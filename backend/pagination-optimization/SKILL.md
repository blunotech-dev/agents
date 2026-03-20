---
name: pagination-optimization
description: Implement efficient pagination using offset, cursor, or keyset strategies. Use when fixing slow or deep pagination, high offsets, infinite scroll, or designing list endpoints.
category: "Backend"
---

# Pagination Optimization

Three pagination strategies with fundamentally different performance characteristics and API contracts. The choice is irreversible once clients depend on it — pick correctly upfront.

---

## Phase 1: Discovery

Establish before recommending anything:

**What does the sort order depend on?**
- Stable unique column (created_at + id, auto-increment id) → keyset is viable
- Arbitrary user-defined sort (price, name, relevance score) → keyset is harder; offset may be necessary with mitigation
- Relevance/ranking from full-text search → offset only; keyset doesn't compose with score-based ranking

**What are the access patterns?**
- Sequential forward traversal only (infinite scroll, feeds) → cursor/keyset
- Random access to arbitrary page numbers ("jump to page 47") → offset required
- Bidirectional navigation (previous page) → keyset requires storing both directions; cursor is forward-only by default
- Does the dataset mutate while the user paginates? (inserts/deletes mid-traversal) → offset causes skips/duplicates; keyset is stable

**What's the dataset size and expected max offset?**
- Under 10k rows with indexed sort column: offset is fine, stop here
- Over 100k rows with deep pagination: offset WILL degrade; quantify the current OFFSET value at the slowest queries
- Unbounded/append-only (event log, activity feed): never use offset

**Who consumes the API?**
- Internal/controlled client: cursor or keyset is fine, migration is easy
- Public API with existing consumers: offset → cursor migration requires versioning; plan for this

---

## Phase 2: The core problem with OFFSET

`SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000`

The DB must generate and discard 10,000 rows before returning 20. Even with an index on `created_at`, rows must be fetched and counted. At page 500 with page size 20, you're paying for 10,000 discarded rows on every request.

Demonstrate the actual query plan to the user if they haven't seen it:
```sql
EXPLAIN ANALYZE SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Look for: rows removed by filter, actual rows scanned vs returned ratio
```

---

## Phase 3: Strategy selection

```
Need random page access?
  └─ Yes → Offset with late-join optimization (see below)
  └─ No → Sort column is unique or made unique?
              └─ Yes → Keyset pagination
              └─ No (relevance score, non-unique column) → Cursor with composite tie-breaking
```

---

## Phase 4: Implementation

### Offset — with late-join optimization

The fix for deep offset: don't fetch full rows during the offset scan. Scan only the index, then join back for full rows.

```sql
-- SLOW: full row fetch × offset count
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;

-- FAST: scan index only, then fetch 20 full rows
SELECT p.*
FROM posts p
JOIN (
  SELECT id FROM posts
  ORDER BY created_at DESC
  LIMIT 20 OFFSET 10000
) ids ON p.id = ids.id
ORDER BY p.created_at DESC;
```

The inner query hits only the index (covering index on `(created_at, id)`). The outer join fetches only 20 full rows. At high offsets this is dramatically faster — but it still scans N index entries. It degrades gracefully, not fully.

**Prisma equivalent** — Prisma has no native late-join; drop to `$queryRaw` for this optimization.

**Count query problem:** `SELECT COUNT(*) FROM posts WHERE ...` is expensive on large tables. Options:
- PostgreSQL: use `reltuples` from `pg_class` for approximate count (±5%, updates via ANALYZE)
- For filtered counts: maintain a counter table updated via trigger, or accept the cost and cache the result with a short TTL
- Present "10,000+ results" instead of exact count if the query is expensive

### Keyset pagination

Keyset encodes position as values, not row numbers. The DB seeks directly to the position using an index — O(log n) regardless of depth.

```sql
-- First page
SELECT id, created_at, title FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page — use last row's values as the cursor
SELECT id, created_at, title FROM posts
WHERE (created_at, id) < ('2024-01-15 10:30:00', 8472)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**The composite key requirement:** sort column alone is rarely unique. Two posts can share a `created_at` timestamp. Always append a unique tiebreaker (usually `id`). The `WHERE` clause must use the same composite tuple comparison.

**Index requirement** — must have a composite index matching the sort:
```sql
CREATE INDEX idx_posts_keyset ON posts (created_at DESC, id DESC);
```
Without this index, keyset is no faster than offset.

**Encoding the cursor** — expose as an opaque token, not raw values. Clients should not construct cursors:
```typescript
function encodeCursor(createdAt: Date, id: number): string {
  return Buffer.from(JSON.stringify({ createdAt: createdAt.toISOString(), id })).toString('base64url')
}

function decodeCursor(cursor: string): { createdAt: Date; id: number } {
  const { createdAt, id } = JSON.parse(Buffer.from(cursor, 'base64url').toString())
  return { createdAt: new Date(createdAt), id }
}
```

**Keyset with nullable sort columns** — NULLs break tuple comparison in most DBs. Two approaches:
1. Coalesce to a sentinel: `COALESCE(published_at, '1970-01-01')` — predictable but semantic distortion
2. Split into two queries: one for non-null rows, one for null rows, union with application-level merge
Prefer approach 1 unless NULL semantics matter to users.

**Keyset with non-unique sort (price, name):**
```sql
-- price is non-unique; use (price, id) composite as keyset
WHERE (price, id) > (last_price, last_id)
ORDER BY price ASC, id ASC
```
This works as long as `id` is appended as tiebreaker — the tuple comparison is now globally unique.

**Previous page with keyset:**
Reverse the comparison operator and sort direction, then reverse the result set in application code:
```sql
-- Going backwards from cursor
SELECT * FROM posts
WHERE (created_at, id) > (:cursor_created_at, :cursor_id)  -- note: > not <
ORDER BY created_at ASC, id ASC                             -- note: ASC not DESC
LIMIT 20;
-- Then reverse the returned array
```

### Cursor pagination (for ORMs / non-SQL)

Prisma/TypeORM cursor pagination uses a single unique field as cursor. It's simpler than keyset but less flexible:

```typescript
// Prisma
const posts = await prisma.post.findMany({
  take: 20,
  skip: cursor ? 1 : 0,       // skip the cursor row itself
  cursor: cursor ? { id: cursor } : undefined,
  orderBy: { createdAt: 'desc' },
})
const nextCursor = posts.length === 20 ? posts[posts.length - 1].id : null
```

**The `skip: 1` is non-obvious.** Without it, the cursor row is included in the result. Prisma's cursor is inclusive; you must skip 1 to exclude it.

**MongoDB:**
```typescript
const posts = await Post.find(
  cursor ? { _id: { $lt: new ObjectId(cursor) } } : {}
).sort({ _id: -1 }).limit(20)
// ObjectId encodes timestamp — monotonic _id gives you time-ordered keyset for free
```

---

## Phase 5: API contract

The response envelope matters. Establish this once; clients depend on it:

```typescript
// Preferred — includes both next cursor and hasMore signal
{
  data: T[],
  pagination: {
    nextCursor: string | null,   // null = last page
    hasMore: boolean,
    // For offset only:
    total?: number,              // omit if expensive
    page?: number,
    pageSize?: number,
  }
}
```

**Non-obvious contract rules:**
- Return `nextCursor: null` (not omit the field) on the last page — clients should not need to check for field presence
- `hasMore: false` when `data.length < pageSize` — client can determine this, but explicit is safer
- If `total` is omitted, document why; don't silently remove it and break existing clients
- Cursor tokens must survive URL encoding — use `base64url` not `base64` (no `+`, `/`, `=`)

---

## Phase 6: Failure modes

**Cursor invalidation on delete:** if the cursor row is deleted between pages, keyset skips no rows — it resumes after the deleted position. This is correct behavior for keyset. Document it; it surprises users who expect exact continuity.

**Cursor invalidation on update to sort column:** if `created_at` on a row changes after a cursor is issued, traversal may skip or duplicate that row. Immutable sort columns (created_at, auto-increment id) are safe; mutable sort columns (updated_at, score) are not safe for stable traversal.

**Total count drift with offset:** user loads page 1 (total: 100), a row is deleted, user loads page 2 — they see a duplicate (the row that was at position 21 is now at 20, served again). With keyset this doesn't happen.

**ORM N+1 on cursor join:** some ORMs execute the cursor lookup and the data fetch as separate queries. Verify with query logging; use `include` or `joinedLoad` to collapse into one query.