---
name: in-app-notifications-fullstack
description: Implement in-app notifications end-to-end with backend event generation, persistent storage, real-time delivery, and read/unread state. Use when building notification systems, notification UIs (bell/feed), or syncing backend events to users. Trigger on in-app notifications, unread counts, mark as read, notification center, or real-time user alerts.
category: "Fullstack"
---

# In-App Notifications — Fullstack

## Discovery

Infer from context, then confirm:

1. **What triggers notifications?** — User actions (someone liked your post), system events (job finished), or both?
2. **Delivery requirement?** — Real-time (SSE/WebSocket already in use) or near-real-time (polling acceptable)?
3. **Persistence required?** — Should notifications survive page reload / be accessible later, or ephemeral?
4. **Multi-device?** — Does marking read on one device need to sync to others?

---

## Architecture Overview

Four distinct responsibilities — keep them separate:

```
[Backend Event] → [Notification Writer] → [DB] → [Delivery Layer] → [Frontend State]
```

- **Event** — something happens (order placed, comment posted, job failed)
- **Writer** — translates the event into a `Notification` row(s) for the right recipients
- **Delivery** — pushes to connected clients in real-time (SSE) or waits for poll
- **Frontend state** — unread count, notification list, mark-read mutations

The most common mistake: coupling the event directly to delivery (e.g., emitting a WebSocket event without writing to DB). Notifications become ephemeral — a user who was offline misses them forever.

---

## Backend: Data Model

```sql
notifications (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type        TEXT NOT NULL,          -- "comment.created", "order.shipped", etc.
  title       TEXT NOT NULL,
  body        TEXT,
  action_url  TEXT,                   -- where to go on click
  read_at     TIMESTAMPTZ,            -- NULL = unread; timestamp = when read
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
)

CREATE INDEX ON notifications (user_id, created_at DESC);  -- list query
CREATE INDEX ON notifications (user_id) WHERE read_at IS NULL;  -- unread count
```

**`read_at` as a timestamp, not a boolean** — gives "read X minutes ago" for free and makes bulk-mark-read queries cheap (`WHERE read_at IS NULL`). A boolean requires a separate timestamp column anyway if you ever need audit data.

**Partial index on `read_at IS NULL`** — unread count is queried on every page load. The partial index makes it O(unread rows), not O(all notifications).

**`action_url` stored on the notification** — don't reconstruct it on the frontend from `type` + payload. Notification shapes change over time; the URL stored at creation time is always correct.

---

## Backend: Notification Writer

Centralise creation in one place. Don't scatter `prisma.notification.create` calls across route handlers.

```ts
// lib/notifications.ts

type NotificationInput = {
  userId: string
  type: string
  title: string
  body?: string
  actionUrl?: string
}

export async function createNotification(input: NotificationInput) {
  const notification = await prisma.notification.create({ data: input })

  // Deliver to connected client immediately, but don't block on it
  deliverToClient(input.userId, notification).catch(console.error)

  return notification
}
```

**Fire-and-forget delivery** — `deliverToClient` failing (user is offline) must never cause the triggering operation to fail. Catch and log, never await in the critical path.

**Don't call the writer inside a database transaction.** If the transaction rolls back, the notification was already created. Call it after the transaction commits.

### Idempotency (prevents notification spam)

If the triggering event can fire multiple times (webhook retries, at-least-once queues), add an idempotency key:

```ts
await prisma.notification.upsert({
  where: { idempotencyKey: `order.placed:${order.id}` },
  create: { ...notificationData, idempotencyKey: `order.placed:${order.id}` },
  update: {},  // no-op if already exists
})
```

Add `idempotencyKey TEXT UNIQUE` to the schema. One extra column; webhook retries and double-fires never produce duplicate notifications.

---

## Backend: Delivery Layer (SSE)

```ts
// In-memory registry — replace with Redis Pub/Sub for multi-instance deploys
const connections = new Map<string, Set<(n: Notification) => void>>()

export function registerConnection(userId: string, send: (n: Notification) => void) {
  if (!connections.has(userId)) connections.set(userId, new Set())
  connections.get(userId)!.add(send)
  return () => connections.get(userId)?.delete(send)  // cleanup fn
}

export async function deliverToClient(userId: string, notification: Notification) {
  connections.get(userId)?.forEach(send => send(notification))
}

app.get("/api/notifications/stream", requireAuth, async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream")
  res.setHeader("Cache-Control", "no-cache")
  res.setHeader("Connection", "keep-alive")
  res.flushHeaders()  // mandatory — buffering middleware holds response without this

  const send = (n: Notification) =>
    res.write(`event: notification\ndata: ${JSON.stringify(n)}\n\n`)

  const unregister = registerConnection(req.user.id, send)

  // Send unread count immediately on connect — don't make the client do a second request
  const count = await prisma.notification.count({
    where: { userId: req.user.id, readAt: null },
  })
  res.write(`event: unread_count\ndata: ${count}\n\n`)

  req.on("close", () => { unregister(); res.end() })
})
```

**Send `unread_count` on connect** — the badge needs to populate immediately. Piggyback on the SSE connection rather than forcing a separate HTTP call.

---

## Backend: Read/Unread Endpoints

```ts
// PATCH /api/notifications/:id/read
app.patch("/api/notifications/:id/read", requireAuth, async (req, res) => {
  await prisma.notification.updateMany({
    where: { id: req.params.id, userId: req.user.id },  // userId prevents IDOR
    data: { readAt: new Date() },
  })
  res.json({ ok: true })
})

// PATCH /api/notifications/read-all
app.patch("/api/notifications/read-all", requireAuth, async (req, res) => {
  await prisma.notification.updateMany({
    where: { userId: req.user.id, readAt: null },
    data: { readAt: new Date() },
  })
  res.json({ ok: true })
})
```

**`updateMany` with `userId` guard** — never `findUnique` then update. Filtering only by `id` is an IDOR — any authenticated user can mark anyone's notification as read. The `userId` in `updateMany` is both the auth check and the query.

---

## Frontend: State Management

Two separate pieces of state — don't conflate them:

- **Unread count** — badge on the bell, updates in real-time via SSE
- **Notification list** — loaded lazily when dropdown opens, paginated

```ts
// useNotifications.ts
export function useNotifications() {
  const queryClient = useQueryClient()
  const [unreadCount, setUnreadCount] = useState(0)

  useEffect(() => {
    const es = new EventSource("/api/notifications/stream", { withCredentials: true })

    es.addEventListener("unread_count", (e) => {
      setUnreadCount(JSON.parse(e.data))
    })

    es.addEventListener("notification", (e) => {
      const notification = JSON.parse(e.data)
      setUnreadCount(c => c + 1)

      // Prepend to cached list — don't invalidate (would cause refetch mid-read)
      queryClient.setQueryData<InfiniteData<NotificationPage>>(
        ["notifications"],
        (old) => old ? prependToFirstPage(old, notification) : old
      )
    })

    return () => es.close()
  }, [queryClient])

  return { unreadCount }
}

function prependToFirstPage(
  data: InfiniteData<NotificationPage>,
  notification: Notification,
): InfiniteData<NotificationPage> {
  const [first, ...rest] = data.pages
  return {
    ...data,
    pages: [{ ...first, items: [notification, ...first.items] }, ...rest],
  }
}
```

**Prepend to cache, don't invalidate** — invalidating on every incoming notification causes a refetch that reorders the list while the user is reading it. Prepend is instant and authoritative since the data came directly from the server.

### Notification List

```ts
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ["notifications"],
  queryFn: ({ pageParam }) =>
    fetch(`/api/notifications${pageParam ? `?cursor=${pageParam}` : ""}`).then(r => r.json()),
  initialPageParam: null,
  getNextPageParam: (last) => last.nextCursor ?? undefined,
  enabled: isDropdownOpen,  // lazy — don't fetch until dropdown opens
  staleTime: 30_000,
})
```

### Mark-Read (optimistic)

```ts
const markRead = useMutation({
  mutationFn: (id: string) =>
    fetch(`/api/notifications/${id}/read`, { method: "PATCH" }),

  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ["notifications"] })
    const previous = queryClient.getQueryData(["notifications"])

    queryClient.setQueryData<InfiniteData<NotificationPage>>(
      ["notifications"],
      (old) => old
        ? updateItemInPages(old, id, n => ({ ...n, readAt: new Date().toISOString() }))
        : old
    )
    setUnreadCount(c => Math.max(0, c - 1))  // badge and list must stay in sync

    return { previous }
  },

  onError: (_err, _id, ctx) => {
    queryClient.setQueryData(["notifications"], ctx?.previous)
    setUnreadCount(c => c + 1)
  },
})
```

**Update badge and list together in `onMutate`** — they're separate state. Missing one produces visible inconsistency (badge says 3, list shows 0 unread).

---

## Notification Bell UI

```tsx
function NotificationBell() {
  const [open, setOpen] = useState(false)
  const { unreadCount } = useNotifications()

  return (
    <div style={{ position: "relative" }}>
      <button
        onClick={() => setOpen(o => !o)}
        aria-label={`Notifications, ${unreadCount} unread`}
      >
        <BellIcon />
        {unreadCount > 0 && (
          <span aria-hidden>{unreadCount > 99 ? "99+" : unreadCount}</span>
        )}
      </button>
      {open && <NotificationPanel onClose={() => setOpen(false)} />}
    </div>
  )
}
```

**Cap badge at 99+** — "1,847" in a badge breaks layouts and leaks internal volume. 99+ is the universal convention.

**`aria-label` on the button** includes the count; `aria-hidden` on the badge prevents double-announcement by screen readers.

---

## Output Checklist

- [ ] Notification written to DB before delivery — offline users receive on next load
- [ ] `deliverToClient` is fire-and-forget; failure never breaks the triggering operation
- [ ] Writer called after transaction commits, not inside it
- [ ] `read_at` is a timestamp, not a boolean
- [ ] Partial index on `read_at IS NULL` for unread count performance
- [ ] `action_url` stored at creation time, not reconstructed on frontend
- [ ] Idempotency key on creation if trigger can fire multiple times
- [ ] SSE sends `unread_count` immediately on connect
- [ ] Mark-read uses `updateMany` with `userId` guard (prevents IDOR)
- [ ] Incoming SSE notifications prepended to cache, not invalidated
- [ ] Notification list query gated on `enabled: isDropdownOpen`
- [ ] Badge and list state updated together in mark-read `onMutate`
- [ ] Badge capped at 99+; `aria-label` on button includes count