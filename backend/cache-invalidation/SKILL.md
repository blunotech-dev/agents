---
name: cache-invalidation
description: Implement cache invalidation strategies for data consistency. Use when handling stale data, invalidating on updates, designing cache flows, or working with SWR, cache tags, or event-driven invalidation across Redis, CDNs, or client caches.
category: "Backend"
---

# Cache Invalidation

Three patterns, each with different consistency guarantees and complexity. Pick the right one before writing any code.

---

## Phase 1: Discovery

Before recommending a pattern, establish:

**What is being cached?**
- Per-user vs shared data — shared data is harder; one user's write invalidates everyone's cache
- Aggregate/computed data (counts, totals, ranked lists) — these have fan-out invalidation problems
- Relational data — a single entity change may invalidate N cache keys that join against it

**Where is the cache?**
- In-process (Node.js Map, Python dict) — fastest, per-instance, no shared invalidation
- Shared cache (Redis, Memcached) — consistent across instances, but network latency
- HTTP/CDN cache (Cache-Control headers, Cloudflare) — hardest to invalidate, most impact
- Client cache (React Query, SWR, browser cache) — requires client cooperation or full TTL expiry

**What is the write pattern?**
- Who writes? Single service, multiple services, external webhooks?
- How frequent? High-write systems often shouldn't cache at all, or should cache with very short TTL
- Are writes transactional? If yes, invalidation must happen after commit, not before

**What consistency level is acceptable?**
- Strict: cache miss is always correct data (event-driven, synchronous invalidation)
- Eventual: stale data for up to N seconds is OK (TTL-based, SWR)
- Read-your-writes: user sees their own changes immediately, others may see stale (per-user invalidation only)

---

## Phase 2: Pattern Selection

### Pattern A: Event-driven invalidation
**Use when:** writes are infrequent, consistency requirements are strict, or cache fan-out is bounded.

### Pattern B: Tag-based invalidation
**Use when:** one entity change invalidates multiple related cache keys (e.g., updating a product invalidates product page, category page, search results, related product lists).

### Pattern C: Stale-while-revalidate (SWR)
**Use when:** eventual consistency is acceptable, read volume is very high, and the cost of a stale response is low (not financial, not security-sensitive).

These patterns compose. A CDN cache often needs SWR at the edge and event-driven invalidation at the application layer simultaneously.

---

## Phase 3: Implementation

### Pattern A: Event-driven invalidation

**The non-obvious part: invalidate after commit, never before or during.**

```typescript
// WRONG — race condition: cache cleared before transaction commits
async function updateUser(id: string, data: UserUpdate) {
  await cache.del(`user:${id}`)         // cleared too early
  await db.users.update(id, data)       // if this fails, cache was cleared for nothing
}

// WRONG — another race: between delete and re-read, another request repopulates with stale data
async function updateUser(id: string, data: UserUpdate) {
  await db.users.update(id, data)
  await cache.del(`user:${id}`)
  // request races in here, reads old DB replica, repopulates stale cache
}

// CORRECT — invalidate after confirmed write, accept the replication lag window
async function updateUser(id: string, data: UserUpdate) {
  await db.users.update(id, data)
  await cache.del(`user:${id}`)
  // callers should read from primary DB for the next request if read-your-writes is required
}
```

**Fan-out invalidation — the hidden cost:**
A write to one entity may need to invalidate dozens of keys. Enumerate them explicitly:

```typescript
async function invalidateProduct(productId: string, categoryId: string) {
  const keys = [
    `product:${productId}`,
    `category:${categoryId}:products`,
    `category:${categoryId}:count`,
    `search:*`,                          // wildcard — Redis SCAN, not KEYS (KEYS blocks)
    `homepage:featured`,
  ]
  // For wildcard patterns, use SCAN + pipeline delete, never KEYS in production
  await deleteByScan(cache, `search:*`)
  await cache.del(...keys.filter(k => !k.includes('*')))
}
```

**Transactional outbox pattern** — when the cache and DB must stay consistent across service boundaries:
- Write to DB + outbox table in one transaction
- A background worker reads the outbox and fires cache invalidation
- Guarantees at-least-once invalidation even if the app crashes between write and cache.del

### Pattern B: Tag-based invalidation

Tags map a logical group to a set of cache keys. When the group is invalidated, all associated keys are cleared.

**Redis implementation:**

```typescript
// Store key→tags mapping as a Redis Set per tag
async function setWithTags(key: string, value: unknown, tags: string[], ttl: number) {
  const pipeline = cache.pipeline()
  pipeline.set(key, JSON.stringify(value), 'EX', ttl)
  for (const tag of tags) {
    pipeline.sadd(`tag:${tag}`, key)
    pipeline.expire(`tag:${tag}`, ttl * 2)  // tag set outlives the keys it tracks
  }
  await pipeline.exec()
}

async function invalidateByTag(tag: string) {
  const keys = await cache.smembers(`tag:${tag}`)
  if (keys.length === 0) return
  const pipeline = cache.pipeline()
  pipeline.del(...keys)
  pipeline.del(`tag:${tag}`)
  await pipeline.exec()
}
```

**Non-obvious failure mode:** tag sets can contain dead keys (TTL expired on the value but the tag set still references it). This is harmless for correctness but the set grows unbounded. Two mitigations:
1. Set tag set TTL longer than value TTL (shown above) — tag set eventually expires
2. Periodically prune: after `invalidateByTag`, the set is deleted anyway

**Next.js / Vercel edge cache tags:**
```typescript
// Tag cache entries at fetch time
const res = await fetch('/api/products', {
  next: { tags: ['products', `category:${categoryId}`] }
})

// Invalidate from server action or route handler
import { revalidateTag } from 'next/cache'
revalidateTag('products')  // purges all entries tagged 'products' from the edge
```

**Cloudflare Cache-Tag header:**
```
Cache-Tag: product-123,category-45,homepage-featured
```
Invalidate via API: `POST /zones/{zone}/purge_cache` with `{ "tags": ["product-123"] }`
Note: Cache-Tag is a Cloudflare Enterprise feature. On lower plans, use `purge_cache` by URL instead.

### Pattern C: Stale-while-revalidate

SWR serves stale data immediately and triggers a background refresh. The key insight is that the TTL is split into two windows: **fresh** (serve without revalidation) and **stale** (serve stale, revalidate in background).

**HTTP Cache-Control:**
```
Cache-Control: max-age=60, stale-while-revalidate=300
```
Means: fresh for 60s, serve stale for up to 5min while revalidating. After 6min total, the cache must wait for a fresh response.

**Application-level SWR (Redis):**
```typescript
async function getWithSWR<T>(
  key: string,
  fetcher: () => Promise<T>,
  freshTTL: number,     // serve without revalidation
  staleTTL: number      // serve stale + revalidate in background
): Promise<T> {
  const raw = await cache.get(key)
  
  if (raw) {
    const { value, cachedAt } = JSON.parse(raw)
    const age = Date.now() - cachedAt
    
    if (age < freshTTL * 1000) {
      return value                            // fresh, return immediately
    }
    if (age < staleTTL * 1000) {
      // stale but within window — return stale, revalidate in background
      setImmediate(async () => {
        const fresh = await fetcher()
        await cache.set(key, JSON.stringify({ value: fresh, cachedAt: Date.now() }), 'EX', staleTTL)
      })
      return value
    }
  }
  
  // cache miss or beyond stale window — fetch synchronously
  const fresh = await fetcher()
  await cache.set(key, JSON.stringify({ value: fresh, cachedAt: Date.now() }), 'EX', staleTTL)
  return fresh
}
```

**Where SWR breaks down:**
- Data with security implications (permissions, pricing, inventory) — a user might act on stale data
- High-write entities — the stale window means mutations aren't reflected quickly; the cache is providing no useful hit rate
- Aggregates that must be accurate — counts, totals, balances

---

## Phase 4: Non-obvious failure modes to explicitly address

**Cache stampede (thundering herd):** When a popular cache key expires, N concurrent requests all miss and hit the DB simultaneously. Fix: probabilistic early expiration or a per-key lock.

```typescript
// Lock-based stampede prevention
async function getWithLock<T>(key: string, fetcher: () => Promise<T>, ttl: number): Promise<T> {
  const cached = await cache.get(key)
  if (cached) return JSON.parse(cached)
  
  const lockKey = `lock:${key}`
  const acquired = await cache.set(lockKey, '1', 'EX', 5, 'NX')  // NX = only if not exists
  
  if (!acquired) {
    // another request is recomputing — short poll or return a default
    await sleep(50)
    return getWithLock(key, fetcher, ttl)   // retry
  }
  
  const value = await fetcher()
  await cache.set(key, JSON.stringify(value), 'EX', ttl)
  await cache.del(lockKey)
  return value
}
```

**Negative caching:** If a DB lookup returns null, cache the null explicitly with a short TTL. Otherwise, every miss hammers the DB. `cache.set(key, 'NULL', 'EX', 30)` and check for the sentinel on read.

**Replica lag invalidation hole:** If you invalidate and then immediately read from a DB read replica, you may re-populate the cache with stale data. After a write, route the cache-repopulation read to the primary, or skip immediate repopulation and let the next request populate from the replica after lag clears.

**In-process cache in multi-instance deployments:** Local caches (Node.js Map) are invisible to other instances. A write on instance A invalidates A's cache; instances B and C serve stale until their TTL expires. Either: use a shared cache for anything that must be consistent across instances, or accept the TTL window and document it.