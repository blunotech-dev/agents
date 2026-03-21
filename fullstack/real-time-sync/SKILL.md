---
name: real-time-sync
description: Add real-time data sync between backend and frontend using WebSockets, SSE, or polling fallback, including reconnection handling and state reconciliation. Use when implementing live updates, push-based data flows, or syncing client state with server changes. Trigger on real-time features, WebSocket/SSE setup, live feeds, reconnect logic, or keeping UI in sync without manual polling.
category: "Fullstack"
---

# Real-Time Sync

## Discovery

Infer from context, then confirm:

1. **Direction?** — Server → client only (feed, notifications, live count), or bidirectional (chat, collaborative editing, presence)?
2. **Transport constraint?** — Serverless/edge (no persistent connections), self-hosted (full control), or behind a load balancer?
3. **Existing infrastructure?** — Redis available? Is the backend stateless across multiple instances?
4. **Failure tolerance?** — Can the user miss events during a disconnect, or must all events be delivered?

---

## Transport Decision

Pick once. Switching later is expensive.

| Situation | Use |
|---|---|
| Bidirectional (chat, collab, presence) | **WebSocket** |
| Server → client only, simple | **SSE** |
| Serverless / edge (Vercel, Cloudflare) | **SSE** (WebSockets require special runtime support) |
| Behind HTTP/1.1 load balancer with no sticky sessions | **SSE** (WebSockets need sticky or a broker) |
| Multiple backend instances, any transport | **Redis Pub/Sub as the broker** |
| Polling acceptable, infrastructure constrained | **Long polling** (not short polling — see below) |

**SSE is underused.** For most notification/feed/live-count use cases it's simpler than WebSockets, works over standard HTTP, survives proxies, and auto-reconnects natively. Default to SSE unless you need client→server messages.

---

## Pattern A: SSE (Server → Client)

### Backend (Express)

```ts
app.get("/events", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream")
  res.setHeader("Cache-Control", "no-cache")
  res.setHeader("Connection", "keep-alive")
  res.flushHeaders() // sends headers immediately — critical, or client waits

  const send = (event: string, data: unknown) => {
    res.write(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`)
  }

  // Register this connection with a pub/sub mechanism
  const unsubscribe = eventBus.subscribe((event) => send(event.type, event.payload))

  // Clean up on disconnect — without this, dead connections accumulate
  req.on("close", () => {
    unsubscribe()
    res.end()
  })
})
```

**`res.flushHeaders()` is non-optional.** Without it, buffering middleware (compression, some proxies) holds the response until it closes, which defeats SSE entirely.

### Frontend

```ts
function useSSE<T>(url: string, eventType: string, onMessage: (data: T) => void) {
  useEffect(() => {
    const es = new EventSource(url, { withCredentials: true })

    es.addEventListener(eventType, (e) => {
      onMessage(JSON.parse(e.data) as T)
    })

    es.addEventListener("error", () => {
      // EventSource reconnects automatically — this fires during the gap
      // Don't close here unless you want to stop reconnecting
    })

    return () => es.close()
  }, [url, eventType])
}
```

**Native SSE auto-reconnects with exponential backoff.** Don't implement your own reconnection — the browser handles it. The `error` event fires during the reconnect gap, not as a terminal failure.

**SSE auth gotcha**: `EventSource` doesn't support custom headers. Pass auth via cookie (preferred) or a short-lived token in the query string — never a long-lived token in the URL (it logs).

---

## Pattern B: WebSocket (Bidirectional)

### Backend (ws library — framework-agnostic)

```ts
import { WebSocketServer } from "ws"

const wss = new WebSocketServer({ noServer: true })

// Attach to existing HTTP server (avoids port conflict)
server.on("upgrade", (req, socket, head) => {
  // Authenticate before upgrading — after upgrade it's too late to reject cleanly
  const token = new URL(req.url!, "http://x").searchParams.get("token")
  if (!isValidToken(token)) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n")
    socket.destroy()
    return
  }

  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit("connection", ws, req)
  })
})

wss.on("connection", (ws) => {
  ws.on("message", (raw) => {
    const msg = JSON.parse(raw.toString())
    // handle msg
  })

  // Heartbeat — clients behind NAT/proxies drop idle connections silently
  const ping = setInterval(() => {
    if (ws.readyState === ws.OPEN) ws.ping()
  }, 30_000)

  ws.on("close", () => clearInterval(ping))
})
```

**Authenticate during HTTP upgrade, not after.** Once the WebSocket handshake completes, you can't reject with an HTTP status code — you can only close the socket abruptly.

**Heartbeat is mandatory in production.** NAT gateways and load balancers silently drop idle TCP connections (often at 60–90s). Without a ping/pong, the server holds a socket that's actually dead.

### Frontend

```ts
function useWebSocket(url: string) {
  const ws = useRef<WebSocket | null>(null)
  const reconnectDelay = useRef(1000)

  const connect = useCallback(() => {
    ws.current = new WebSocket(url)

    ws.current.onopen = () => {
      reconnectDelay.current = 1000 // reset backoff on successful connect
    }

    ws.current.onmessage = (e) => {
      const msg = JSON.parse(e.data)
      // dispatch to state
    }

    ws.current.onclose = (e) => {
      if (e.code === 1000) return // intentional close, don't reconnect
      // Exponential backoff with jitter — jitter prevents thundering herd on server restart
      const delay = reconnectDelay.current + Math.random() * 500
      reconnectDelay.current = Math.min(delay * 2, 30_000)
      setTimeout(connect, delay)
    }
  }, [url])

  useEffect(() => {
    connect()
    return () => ws.current?.close(1000) // code 1000 = intentional, stops reconnect loop
  }, [connect])

  return ws
}
```

**Jitter on reconnect backoff** prevents every client reconnecting simultaneously when the server restarts (thundering herd). `Math.random() * 500` added to the delay is enough.

**Close code 1000** signals intentional disconnect. Check it in `onclose` to skip reconnection on component unmount — otherwise your hook reconnects after cleanup.

---

## Multi-Instance Problem (Horizontal Scaling)

WebSockets and SSE connections are sticky to one server process. If you have multiple backend instances, a message published on instance A won't reach clients connected to instance B.

**Fix: Redis Pub/Sub as the broker**

```ts
import { createClient } from "redis"

const pub = createClient()
const sub = createClient()
await pub.connect()
await sub.connect()

// When something changes, publish to Redis (any instance can do this)
await pub.publish("item:updated", JSON.stringify({ id, ...data }))

// Each instance subscribes and forwards to its local connections
await sub.subscribe("item:updated", (message) => {
  const payload = JSON.parse(message)
  // broadcast to all local SSE/WS connections interested in this item
  localConnections.forEach((conn) => conn.send("item:updated", payload))
})
```

**Don't skip this even in early stages if you deploy to platforms that run multiple workers** (Railway, Render, Heroku with multiple dynos). A single Redis instance handles tens of thousands of pub/sub messages per second — it's not a bottleneck until very high scale.

---

## State Reconciliation After Reconnect

The gap between disconnect and reconnect is where state corrupts silently. Events fired during the gap are lost unless you account for them.

**Cursor/sequence approach — the reliable pattern:**

```ts
// Backend: every event has a monotonic sequence number
type SyncEvent = { seq: number; type: string; payload: unknown }

// Client: tracks the last seen sequence
const lastSeq = useRef(0)

// On reconnect, request a replay from the last known position:
es.addEventListener("open", () => {
  fetch(`/api/state?since=${lastSeq.current}`)
    .then(r => r.json())
    .then(({ events }) => {
      events.forEach(applyEvent)        // replay missed events in order
      lastSeq.current = events.at(-1)?.seq ?? lastSeq.current
    })
})

// On each event, update the cursor:
es.addEventListener("message", (e) => {
  const event = JSON.parse(e.data)
  applyEvent(event)
  lastSeq.current = event.seq
})
```

**If sequence tracking is too complex**, the fallback is a full refetch on reconnect:

```ts
es.addEventListener("open", () => {
  queryClient.invalidateQueries({ queryKey: ["items"] })
  // Simple, correct, slightly heavier — fine for most apps
})
```

Full refetch on reconnect is not lazy — it's often the right tradeoff. Sequence replay only pays off when the data volume makes a full refetch expensive.

---

## Polling Fallback (When Nothing Else Works)

Use **long polling**, not short polling. Short polling (`setInterval` + fetch) hammers the server for no benefit.

```ts
async function longPoll(signal: AbortSignal) {
  while (!signal.aborted) {
    try {
      // Server holds the request open until there's data or timeout
      const res = await fetch("/api/poll?timeout=30", { signal })
      if (res.ok) {
        const data = await res.json()
        applyUpdate(data)
      }
    } catch {
      if (signal.aborted) break
      await sleep(2000) // back off on error
    }
    // If server returned immediately (no data), loop continues naturally
  }
}

// In component:
useEffect(() => {
  const controller = new AbortController()
  longPoll(controller.signal)
  return () => controller.abort()
}, [])
```

Backend holds the request open up to 30s, responds when data arrives, closes. Client immediately re-requests. Net effect: near-real-time with one open connection, no extra infrastructure.

---

## Output Checklist

- [ ] Transport chosen based on direction, infrastructure, and deployment constraints
- [ ] SSE: `res.flushHeaders()` called immediately after setting headers
- [ ] SSE: auth via cookie or short-lived query token (not custom headers)
- [ ] SSE: `req.on("close")` cleans up subscription to prevent connection accumulation
- [ ] WebSocket: auth validated during HTTP upgrade, before handshake completes
- [ ] WebSocket: heartbeat ping/pong on 30s interval to keep connection alive through NAT
- [ ] WebSocket: exponential backoff with jitter on reconnect; code 1000 skips reconnect
- [ ] Multi-instance: Redis Pub/Sub broker if more than one backend process
- [ ] Reconnect: either sequence-based replay or full invalidate on `open` event
- [ ] Polling: long polling used (not short polling) with AbortController cleanup