---
name: webhook-system
description: Build a production-grade outbound webhook system with retries, signing, failure tracking, and re-delivery. Use when implementing webhook delivery, event subscriptions, or handling webhook failures and reliability.
category: "Backend"
---

# Webhook System

Five concerns that must each be correct independently: subscription management, delivery, signing, failure tracking, and re-delivery. A mistake in any one breaks the whole contract with consumers.

---

## Phase 1: Discovery

Before writing any code:

**What triggers webhooks?**
- Domain events from DB writes (user.created, payment.succeeded) — or any state change?
- Who emits events: one service or multiple? If multiple, a shared event bus is needed; per-service pub/sub is not enough
- Are events ordered? Consumers often assume ordering per-entity; delivery order is not guaranteed across retries unless explicitly designed for

**What's the delivery guarantee requirement?**
- At-least-once (standard, acceptable duplicates) — use idempotency keys on consumer side
- Exactly-once — not achievable at the transport layer; requires idempotent consumers + deduplication window

**What's the persistence layer?**
- Queue available (BullMQ, SQS, pg-boss)? — use it; don't implement retry logic in application memory
- No queue? — use a `webhook_deliveries` table as the queue (transactional outbox pattern)
- Postgres available? — `pg-boss` or `pg_cron` + delivery table avoids adding a new infrastructure dependency

**Who are the consumers?**
- Internal services only — simpler; no signing enforcement required but do it anyway
- External third-party developers — signing is non-negotiable; payload schema versioning matters

---

## Phase 2: Data model

Design the tables before any application code. The schema drives everything else.

```sql
-- Endpoint registered by a consumer
CREATE TABLE webhook_endpoints (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID NOT NULL,                          -- owner; omit if single-tenant
  url         TEXT NOT NULL,
  secret      TEXT NOT NULL,                          -- store hashed or encrypted; used for signing
  events      TEXT[] NOT NULL,                        -- ['payment.succeeded', 'user.created']
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  -- rate limiting / circuit breaker state
  failure_count     INT NOT NULL DEFAULT 0,
  last_failure_at   TIMESTAMPTZ,
  disabled_at       TIMESTAMPTZ                       -- NULL = active; set when circuit opens
);

-- One row per delivery attempt
CREATE TABLE webhook_deliveries (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id),
  event_type      TEXT NOT NULL,
  payload         JSONB NOT NULL,
  idempotency_key TEXT NOT NULL,                      -- stable per logical event; reused across retries
  attempt         INT NOT NULL DEFAULT 1,
  status          TEXT NOT NULL DEFAULT 'pending',    -- pending | success | failed | dead
  response_status INT,
  response_body   TEXT,
  error           TEXT,
  scheduled_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  delivered_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deliveries_pending ON webhook_deliveries (scheduled_at)
  WHERE status = 'pending';
CREATE INDEX idx_deliveries_endpoint ON webhook_deliveries (endpoint_id, created_at DESC);
```

**Non-obvious schema decisions:**
- `idempotency_key` lives on the delivery, not the event. The same logical event re-queued for retry shares the same key — consumers can deduplicate
- `secret` should be stored encrypted (AES-256) or as a reference to a secrets manager, not plaintext. Signing uses it but it must never appear in logs or API responses
- `disabled_at` on the endpoint enables circuit breaking without deleting rows — the endpoint is suspendable and re-activatable
- Separate `scheduled_at` from `created_at` — retry scheduling sets `scheduled_at` in the future; the worker polls `WHERE scheduled_at <= NOW()`

---

## Phase 3: Signing

Every delivery must be signed. Consumers verify before processing — this is the only protection against spoofed payloads.

**HMAC-SHA256 — the standard:**
```typescript
import { createHmac, timingSafeEqual } from 'crypto'

function signPayload(secret: string, payload: string, timestamp: number): string {
  const data = `${timestamp}.${payload}`
  return createHmac('sha256', secret).update(data, 'utf8').digest('hex')
}

// Headers sent with every delivery
function buildSignatureHeaders(secret: string, body: string): Record<string, string> {
  const timestamp = Math.floor(Date.now() / 1000)
  const signature = signPayload(secret, body, timestamp)
  return {
    'X-Webhook-Timestamp': String(timestamp),
    'X-Webhook-Signature': `v1=${signature}`,
    'X-Webhook-ID': crypto.randomUUID(),      // delivery ID for consumer logging
  }
}
```

**Why timestamp in the signature:** prevents replay attacks. A consumer rejects any delivery where `|now - timestamp| > 300` seconds. Without it, an intercepted valid payload can be re-delivered arbitrarily.

**Consumer-side verification:**
```typescript
function verifySignature(secret: string, body: string, headers: Record<string, string>): boolean {
  const timestamp = parseInt(headers['x-webhook-timestamp'], 10)
  if (Math.abs(Date.now() / 1000 - timestamp) > 300) return false  // replay window

  const expected = signPayload(secret, body, timestamp)
  const received = headers['x-webhook-signature']?.replace('v1=', '') ?? ''

  // timingSafeEqual prevents timing attacks
  const a = Buffer.from(expected, 'hex')
  const b = Buffer.from(received, 'hex')
  if (a.length !== b.length) return false
  return timingSafeEqual(a, b)
}
```

**`timingSafeEqual` is non-negotiable.** String equality (`===`) short-circuits on first mismatch — an attacker can measure response time to brute-force the signature byte by byte. `timingSafeEqual` always runs in constant time.

**Secret rotation:** support multiple active secrets (a list, not a single value) so consumers can rotate without a delivery gap. Sign with the current secret; verify against all active secrets. Stripe does this — the `v1=` prefix is designed to support multiple comma-separated signatures.

---

## Phase 4: Delivery worker

**Never deliver synchronously in the request handler.** Always enqueue; a worker delivers asynchronously.

```typescript
// Enqueue on event emission — this is the only thing the application does synchronously
async function emitEvent(tenantId: string, eventType: string, payload: object) {
  const endpoints = await db.query<WebhookEndpoint>(
    `SELECT * FROM webhook_endpoints
     WHERE tenant_id = $1
       AND $2 = ANY(events)
       AND is_active = true
       AND disabled_at IS NULL`,
    [tenantId, eventType]
  )

  const idempotencyKey = `${eventType}:${payload.id}:${Date.now()}`
  
  await db.query(
    `INSERT INTO webhook_deliveries (endpoint_id, event_type, payload, idempotency_key, scheduled_at)
     SELECT unnest($1::uuid[]), $2, $3, $4, NOW()`,
    [endpoints.map(e => e.id), eventType, JSON.stringify(payload), idempotencyKey]
  )
}
```

**Delivery worker:**
```typescript
async function deliverWebhook(delivery: WebhookDelivery, endpoint: WebhookEndpoint) {
  const body = JSON.stringify({
    id: delivery.idempotency_key,
    event: delivery.event_type,
    created_at: new Date().toISOString(),
    data: delivery.payload,
  })

  const headers = buildSignatureHeaders(endpoint.secret, body)

  let responseStatus: number | null = null
  let responseBody: string | null = null
  let error: string | null = null

  try {
    const res = await fetch(endpoint.url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...headers },
      body,
      signal: AbortSignal.timeout(30_000),      // 30s hard timeout
    })
    responseStatus = res.status
    responseBody = await res.text().catch(() => null)
    
    const success = res.status >= 200 && res.status < 300
    
    if (success) {
      await markDelivered(delivery.id, responseStatus, responseBody)
      await resetFailureCount(endpoint.id)
      return
    }
    
    // 4xx from consumer — do not retry (their bug, not a transient failure)
    // Exception: 429 Too Many Requests — retry with backoff
    if (res.status >= 400 && res.status < 500 && res.status !== 429) {
      await markFailed(delivery.id, responseStatus, responseBody, 'non-retryable 4xx')
      return
    }
    
    error = `HTTP ${res.status}`
  } catch (err) {
    error = err instanceof Error ? err.message : String(err)
  }

  await scheduleRetry(delivery, endpoint, error, responseStatus, responseBody)
}
```

**4xx vs 5xx retry logic is non-obvious.** A 400/401/403/404 from the consumer means their endpoint rejected the payload — retrying won't help. A 500/502/503 means their server is down — retry is appropriate. 429 is the exception: their server is up but rate-limiting; retry with extended backoff.

**Retry schedule — exponential backoff with jitter:**
```typescript
const RETRY_DELAYS_SECONDS = [60, 300, 1800, 7200, 86400]  // 1m, 5m, 30m, 2h, 24h

async function scheduleRetry(
  delivery: WebhookDelivery,
  endpoint: WebhookEndpoint,
  error: string | null,
  responseStatus: number | null,
  responseBody: string | null
) {
  const nextAttempt = delivery.attempt + 1
  const maxAttempts = RETRY_DELAYS_SECONDS.length + 1

  if (nextAttempt > maxAttempts) {
    await markDead(delivery.id, error, responseStatus, responseBody)
    await incrementEndpointFailures(endpoint.id)
    await maybeDisableEndpoint(endpoint.id)
    return
  }

  const baseDelay = RETRY_DELAYS_SECONDS[nextAttempt - 2]
  const jitter = Math.random() * baseDelay * 0.25   // ±25% jitter
  const delaySeconds = Math.floor(baseDelay + jitter)

  await db.query(
    `INSERT INTO webhook_deliveries
       (endpoint_id, event_type, payload, idempotency_key, attempt, scheduled_at)
     VALUES ($1, $2, $3, $4, $5, NOW() + ($6 || ' seconds')::interval)`,
    [endpoint.id, delivery.event_type, delivery.payload,
     delivery.idempotency_key, nextAttempt, delaySeconds]
  )
  await updateDeliveryStatus(delivery.id, 'failed', error, responseStatus, responseBody)
}
```

**Jitter is required** on retry delays. Without it, if 1000 endpoints all fail simultaneously (e.g., your own outage), they all retry at exactly t+60s — a thundering herd that repeats at every retry interval.

---

## Phase 5: Circuit breaker

An endpoint that's been failing for 24h should stop receiving deliveries — it wastes queue capacity and creates noise.

```typescript
const FAILURE_THRESHOLD = 10        // failures before circuit opens
const CIRCUIT_RESET_HOURS = 24      // how long until auto-retry

async function maybeDisableEndpoint(endpointId: string) {
  await db.query(
    `UPDATE webhook_endpoints
     SET disabled_at = NOW()
     WHERE id = $1 AND failure_count >= $2 AND disabled_at IS NULL`,
    [endpointId, FAILURE_THRESHOLD]
  )
  // Notify the endpoint owner (email, dashboard alert) that their endpoint is suspended
  await notifyEndpointDisabled(endpointId)
}

async function resetFailureCount(endpointId: string) {
  await db.query(
    `UPDATE webhook_endpoints
     SET failure_count = 0, last_failure_at = NULL, disabled_at = NULL
     WHERE id = $1`,
    [endpointId]
  )
}
```

**Always notify on circuit open.** A suspended endpoint is silent — the consumer sees no deliveries and may not notice unless you tell them. Send an email or surface a dashboard alert with a link to re-enable and trigger re-delivery.

---

## Phase 6: Re-delivery API

Consumers need to recover missed events. Provide two re-delivery mechanisms:

```typescript
// Re-deliver a single failed delivery by ID
router.post('/webhooks/deliveries/:id/redeliver', async (req, res) => {
  const delivery = await getDelivery(req.params.id, req.tenantId)
  if (!delivery) return res.status(404).json({ error: 'not found' })
  if (!['failed', 'dead'].includes(delivery.status)) {
    return res.status(409).json({ error: 'only failed or dead deliveries can be redelivered' })
  }

  // Insert a new delivery row — same idempotency_key, attempt resets to 1
  const newDelivery = await db.query(
    `INSERT INTO webhook_deliveries
       (endpoint_id, event_type, payload, idempotency_key, scheduled_at)
     VALUES ($1, $2, $3, $4, NOW())
     RETURNING id`,
    [delivery.endpoint_id, delivery.event_type, delivery.payload, delivery.idempotency_key]
  )
  res.json({ delivery_id: newDelivery.id })
})

// Bulk re-delivery: re-queue all dead deliveries in a time window
router.post('/webhooks/endpoints/:id/redeliver', async (req, res) => {
  const { since, until, event_type } = req.body  // ISO timestamps
  
  // Re-insert dead deliveries as new pending rows — don't mutate originals
  const result = await db.query(
    `INSERT INTO webhook_deliveries (endpoint_id, event_type, payload, idempotency_key, scheduled_at)
     SELECT endpoint_id, event_type, payload, idempotency_key, NOW()
     FROM webhook_deliveries
     WHERE endpoint_id = $1
       AND status = 'dead'
       AND created_at BETWEEN $2 AND $3
       AND ($4::text IS NULL OR event_type = $4)
     RETURNING id`,
    [req.params.id, since, until, event_type ?? null]
  )
  res.json({ queued: result.rowCount })
})
```

**Re-delivery inserts new rows; it never mutates the original delivery record.** The original is an immutable audit log. The new delivery shares the `idempotency_key` so consumers can deduplicate if they received the event via another path.

---

## Phase 7: Non-obvious failure modes

**DNS resolution at delivery time:** consumer endpoints change IPs. Resolve DNS fresh per delivery; don't cache the resolved address from subscription registration. Also: validate that the URL doesn't resolve to a private IP range (SSRF protection — reject 10.x, 172.16.x, 192.168.x, 127.x).

**Payload schema versioning:** include a `version` field in the envelope (`"version": "2024-01-01"`). When you change the payload shape, bump the version. Consumers can opt into versions at subscription time. Without this, any payload change is a breaking change for all consumers simultaneously.

**Request body streaming vs buffering:** `fetch` and most HTTP clients buffer the response body. A consumer that streams a slow response (e.g., returns chunked 200 gradually) will hold the worker's connection open. The 30s `AbortSignal.timeout` prevents this from blocking indefinitely — but log it when it fires; it usually indicates the consumer is confused about the protocol.

**Log scrubbing:** delivery logs contain full request/response bodies. These may include PII or secrets. Either: scrub before persisting `response_body`, or apply a TTL to delivery rows (e.g., delete after 30 days) and document the retention policy to consumers who expect to query historical logs.

**Worker concurrency vs endpoint rate limits:** if a single endpoint has 500 queued deliveries, a naive worker will fire all 500 concurrently. Add per-endpoint concurrency limits (1-5 concurrent deliveries per endpoint) to avoid overwhelming slow consumers. This is especially important during re-delivery.