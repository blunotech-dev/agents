---
name: n-plus-one-fix
description: Detect and fix N+1 query patterns using eager loading, batching, or dataloader approaches. Use when queries run in loops, ORM lazy loading causes repeated DB calls, or the user reports slow endpoints or N+1 issues. Also trigger when the user shares code with loops that call .find(), .load(), .fetch(), or any ORM relation access inside an iteration, or when query logs show the same query repeated with different IDs.
category: "Backend"
---

# N+1 Fix Skill

## Phase 1 — Discovery

Ask only what context doesn't reveal:

- **ORM/framework?** The fix differs significantly: ActiveRecord (`includes`/`preload`/`eager_load`), Django (`select_related`/`prefetch_related`), Prisma (`include`), SQLAlchemy (`joinedload`/`subqueryload`), TypeORM (`relations`), GraphQL (DataLoader).
- **Is this GraphQL?** N+1 in GraphQL is a different problem — eager loading doesn't apply to resolvers. DataLoader is the only correct fix.
- **Can you see query logs?** If not, ask them to enable logging temporarily. Guessing without query counts is unreliable.
- **Read-heavy or write path?** Eager loading on write paths (e.g., validation hooks) can introduce unnecessary queries. Confirm the context is a read endpoint.

---

## Phase 2 — Detection

### What actually signals N+1 (beyond the obvious loop)

**In query logs — the real tell:**
The same query structure repeating with different ID values is definitive. One query to fetch a list, then one per row for a related resource:
```
SELECT * FROM posts LIMIT 20          -- 1 query
SELECT * FROM users WHERE id = 1      -- repeated 20x with different IDs
SELECT * FROM users WHERE id = 2
...
```

**Non-obvious N+1 sources:**

- **Serializers/presenters:** The loop is hidden. A `UserSerializer` that accesses `user.company.name` for each user in a list response — the N+1 is in the serializer, not visible in the controller.
- **Callback chains:** `after_save` or `before_validation` hooks that load associations. Every record save triggers a query, but it looks like a write path issue.
- **Computed properties / `@property` decorators:** A Python model property that accesses a relation — called innocuously in a list comprehension.
- **Conditional association access:** `if user.premium? then user.subscription.expires_at` — only triggers for some rows, making it intermittent and harder to spot in logs.
- **Nested N+1:** Posts → Comments → Authors. Fixing the first level (posts → comments) without fixing the second (comments → authors) still yields N queries.
- **GraphQL N+1:** Every field resolver is called independently per parent object. `post.author` in a resolver runs once per post, regardless of whether the parent query fetched 1 or 1000 posts.

---

## Phase 3 — Fix Patterns

### Choosing the right eager loading method (non-obvious distinctions)

**ActiveRecord — `includes` vs `preload` vs `eager_load`:**
- `preload`: Always issues a second query (`WHERE id IN (...)`). Safe for large associations. Does **not** allow filtering/ordering on the association in the same query.
- `eager_load`: Always uses `LEFT OUTER JOIN`. Required if you need to `WHERE` or `ORDER BY` on the association. Risk: multiplies rows for has_many — can corrupt `limit` behavior.
- `includes`: Chooses between the two automatically. Picks `preload` by default; switches to `eager_load` if you reference the association in a `where`/`order` clause. Trap: this auto-switching is implicit and can silently change query shape when you add a filter later.

Rule: use `preload` explicitly when you don't need to filter on the association; use `eager_load` explicitly when you do. Avoid `includes` in performance-critical paths.

**Django — `select_related` vs `prefetch_related`:**
- `select_related`: SQL JOIN. Only for ForeignKey and OneToOne. Don't use for ManyToMany or reverse ForeignKey — it will generate a cartesian product.
- `prefetch_related`: Separate query with `WHERE id IN (...)`. Works for all relation types including ManyToMany. Use this when the related set could be large.
- Trap: `prefetch_related` is invalidated if you call `.filter()` on the prefetched queryset after the fact. Use `Prefetch()` with a `queryset` argument to filter at prefetch time:
```python
from django.db.models import Prefetch
Post.objects.prefetch_related(
    Prefetch('comments', queryset=Comment.objects.filter(approved=True))
)
```

**SQLAlchemy — `joinedload` vs `subqueryload` vs `selectinload`:**
- `joinedload`: JOIN. Same cartesian product risk as `eager_load`. Avoid for collections; use for many-to-one only.
- `subqueryload`: Subquery per relationship. Deprecated in SQLAlchemy 1.4+.
- `selectinload`: `SELECT IN` pattern. Correct default for collections in modern SQLAlchemy. Use this unless you have a specific reason for a join.

**Prisma:**
No join vs. subquery distinction — `include` always issues a separate query per relation level. For deeply nested includes, consider raw queries or restructuring the data access.

### DataLoader pattern (GraphQL and beyond)

DataLoader is not GraphQL-specific — it applies anywhere you have a per-item async fetch inside a loop.

Core mechanic: batch + deduplicate. Requests within the same event loop tick are coalesced into one batch fetch.

```typescript
// Wrong: per-resolver fetch
const authorResolver = async (post) => {
  return await db.user.findUnique({ where: { id: post.authorId } }); // N queries
};

// Right: DataLoader
const userLoader = new DataLoader(async (userIds: string[]) => {
  const users = await db.user.findMany({ where: { id: { in: userIds } } });
  // Must return array in same order as userIds — DataLoader contract
  const userMap = new Map(users.map(u => [u.id, u]));
  return userIds.map(id => userMap.get(id) ?? null);
});

const authorResolver = async (post) => {
  return userLoader.load(post.authorId); // batched automatically
};
```

Critical DataLoader trap: the batch function **must** return results in the same order as the input keys, with `null` for misses. Returning unordered results silently maps wrong objects to wrong parents.

DataLoader scope: create one DataLoader instance **per request**, not per application. A shared DataLoader caches across requests and causes data leakage between users.

### Batch query without an ORM

When no ORM is available or the ORM fix is insufficient:

```sql
-- Replace: SELECT * FROM users WHERE id = ?  (called N times)
-- With:
SELECT * FROM users WHERE id = ANY($1::int[])  -- Postgres
SELECT * FROM users WHERE id IN (...)           -- universal
```

Then reassemble in application code:
```typescript
const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [postAuthorIds]);
const userMap = new Map(users.rows.map(u => [u.id, u]));
posts.forEach(post => post.author = userMap.get(post.authorId));
```

Trap: `IN (...)` with >32,000 values hits Postgres parameter limits. Chunk batches at 1,000–5,000 IDs.

### When eager loading is the wrong fix

- **Rarely-accessed associations on large lists:** Eagerly loading `user.permissions` for 10,000 users when only 3% are admin users wastes more than it saves. Use conditional loading or restructure the query to filter first.
- **Polymorphic associations:** Eager loading a polymorphic `belongs_to` (e.g., `commentable` that could be Post, Video, or Article) generates one query per type after deduplication — this is correct behavior, not an N+1, but it looks like one in logs.
- **Already paginated to 1 row:** An N+1 on a detail page (single record) costs exactly 1 extra query. Not worth the complexity of eager loading unless the page is high-traffic.

---

## Phase 4 — Output

Produce whichever the user needs:

- **Fixed code** — replace the looping access with the correct eager load or batch pattern for their ORM
- **Query log analysis** — annotate their log output with which lines are the N+1 pattern and what the fix collapses them to
- **DataLoader scaffold** — per-request loader setup with correct ordering and null-handling
- **Detection comment** — inline code comment explaining why the original code causes N+1, for PR review context
- **Nested N+1 audit** — trace all relation accesses across serializer/presenter/resolver layers, not just the top-level loop

Always include:
- The expected query count before and after the fix
- A note if the fix changes query shape (JOIN vs. subquery) and what that means for result ordering or `limit` behavior
- DataLoader per-request scope warning if the context is GraphQL or async resolvers