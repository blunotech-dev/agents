---
name: query-optimization
description: Optimize slow SQL queries using plan analysis, indexing, join tuning, subquery/CTE simplification, and pagination improvements. Use when the user shares slow queries, EXPLAIN output, or reports high execution time, sequential scans, or inefficient joins.
category: "Backend"
---

# Query Optimization Skill

## Phase 1 — Discovery

Ask only what context doesn't reveal:

- **Database engine and version?** Planner behavior, available hints, and index types differ significantly. Postgres 14+ has improved parallel query and CTE materialization defaults vs. older versions.
- **Do you have `EXPLAIN ANALYZE` output?** Without it, optimization is guesswork. Ask them to run it and share — not just `EXPLAIN` (estimates only), but `EXPLAIN (ANALYZE, BUFFERS)` which shows actual row counts and I/O.
- **Table row counts for involved tables?** The planner's choices make sense or don't in context of data volume.
- **Is this a one-time slow query or a recurring endpoint?** One-time queries tolerate different tradeoffs (parallel workers, temp indexes) vs. a hot path that runs thousands of times per minute.
- **Are statistics up to date?** Ask if `ANALYZE` has been run recently on large tables. Stale statistics cause the planner to make systematically wrong choices.

---

## Phase 2 — Plan Analysis

### Reading `EXPLAIN ANALYZE` — what to look for first

**Rows estimate vs. actual rows — the most important signal:**
A large discrepancy between `rows=X` (estimate) and `actual rows=Y` means the planner is working with bad statistics. The rest of the plan is built on a wrong assumption.

```
Seq Scan on orders (cost=0.00..4821.00 rows=5 width=32)
                   (actual time=0.041..89.3 rows=48291 loops=1)
```
Planner expected 5 rows, got 48,291. Every join and operation downstream was sized wrong. Fix: `ANALYZE orders;` then re-run.

**Where to look for the bottleneck — start at the innermost node:**
Cost accumulates upward. The highest-cost leaf node is usually the problem. Don't optimize the outer join if the inner sequential scan is doing 90% of the work.

**`loops=N` multiplies everything:**
A node with `actual time=5ms loops=1000` costs 5 seconds total. Nested loop joins on large tables create this silently. The per-loop time looks fine; the total doesn't appear until you multiply.

**`Buffers: shared hit=X read=Y`:**
`read` = disk I/O. High `read` count means data isn't cached — either the working set is too large for `shared_buffers` or the index isn't being used and is doing full table reads.

### Sequential scan: when it's correct vs. wrong

A sequential scan is correct when:
- Returning >5-10% of the table's rows (index scan has higher overhead at that selectivity)
- The table fits in memory and is small
- Statistics show low cardinality on the filter column

A sequential scan is wrong when:
- The filter is highly selective (returns <1% of rows) and no index covers it
- The table is large and only a few rows match

Trap: adding an index on a low-cardinality column (e.g., `status` with values `active`/`inactive` on a 50/50 split) and wondering why the planner ignores it — it shouldn't use it. Partial indexes fix this:
```sql
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'pending';
-- Planner uses this when filtering on status = 'pending' specifically
```

---

## Phase 3 — Fix Patterns

### Join optimization

**Join order matters; the planner gets it wrong on complex queries:**
Postgres's planner considers all join orderings up to `join_collapse_limit` (default 8) tables. Beyond that it uses a greedy heuristic and can pick suboptimal order. For queries joining 10+ tables, explicitly reordering joins or using CTEs to materialize intermediate results can help.

**Nested loop vs. hash join vs. merge join:**
- Nested loop: good for small outer set + indexed inner. Bad when outer set is large.
- Hash join: good for large unsorted sets. Requires memory — if `work_mem` is too low, spills to disk (shows `Batches: N` in plan where N > 1).
- Merge join: requires both sides sorted. Good for sorted indexed columns. Rare in practice.

If you see a hash join with `Batches: 4`, increase `work_mem` for that session before optimizing indexes:
```sql
SET work_mem = '64MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

**The implicit cross join trap:**
A missing join condition between two tables in a multi-table query produces a cartesian product. It shows up as an astronomically high row estimate in the plan. Look for `Nested Loop` with `rows=` in the millions when your tables are thousands of rows each.

### Subquery elimination

**Correlated subqueries — the hidden N+1 in SQL:**
```sql
-- Bad: executes subquery once per outer row
SELECT p.*, (SELECT COUNT(*) FROM comments c WHERE c.post_id = p.id) AS comment_count
FROM posts p;

-- Good: single aggregation join
SELECT p.*, COALESCE(c.comment_count, 0) AS comment_count
FROM posts p
LEFT JOIN (
    SELECT post_id, COUNT(*) AS comment_count
    FROM comments GROUP BY post_id
) c ON c.post_id = p.id;
```

**`EXISTS` vs `IN` vs `JOIN` for filtering:**
- `IN (subquery)` materializes the full subquery result — problematic if large.
- `EXISTS` short-circuits on first match — correct for "does at least one exist" checks.
- `JOIN` + `DISTINCT` or `GROUP BY` is often faster than either for large sets with indexes.
- For `NOT IN`: a single NULL in the subquery result makes `NOT IN` return no rows. Always use `NOT EXISTS` instead.

**CTEs: materialization trap (Postgres < 12):**
Before Postgres 12, CTEs were always materialized (optimization fence). The planner couldn't push predicates inside them. A CTE that looked like a clean refactor was actually forcing a full intermediate result. In Postgres 12+, CTEs are inlined by default unless they're recursive or contain side effects. If you're on < 12, rewrite performance-critical CTEs as subqueries.

In Postgres 12+, you can force materialization when you want it (to avoid re-execution):
```sql
WITH expensive AS MATERIALIZED (
    SELECT ... FROM large_table WHERE ...
)
SELECT * FROM expensive JOIN other ON ...;
```

### Pagination optimization

**`OFFSET` pagination degrades linearly — the fundamental problem:**
`OFFSET 10000 LIMIT 20` scans and discards 10,000 rows every time. Page 500 is 500x slower than page 1. This is not fixable with indexes.

**Keyset (cursor) pagination — the correct replacement:**
```sql
-- Instead of: SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 400
-- Use: pass the last seen value as a cursor
SELECT * FROM posts
WHERE created_at < :last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;
```
Requirements: sort column must be indexed, and must be unique or combined with a tiebreaker (usually `id`):
```sql
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Limitation: keyset pagination doesn't support jumping to arbitrary pages. Only valid for "load more" / infinite scroll patterns. If the product requires page numbers, `OFFSET` is unavoidable — mitigate with a covering index and a hard cap on max offset.

**COUNT(*) for total pages is expensive — avoid or cache it:**
`SELECT COUNT(*) FROM posts WHERE ...` with the same filter as your paginated query often does a full index scan. Options:
- Return an estimated count from `pg_stat_user_tables` for rough totals
- Cache the count and invalidate on write
- Return `has_next_page` boolean instead of total count (requires fetching `LIMIT + 1` rows)

### Index patterns for common slow query shapes

**Covering index — eliminates table heap fetch:**
If a query selects only a few columns and filters on others, a covering index that includes all referenced columns lets Postgres answer entirely from the index without touching the table:
```sql
-- Query: SELECT id, status FROM orders WHERE user_id = ? AND created_at > ?
CREATE INDEX idx_orders_user_covering ON orders(user_id, created_at) INCLUDE (status, id);
-- INCLUDE columns are stored but not part of the sort key
```

**Index on expression — when the filter transforms the column:**
```sql
-- This won't use an index on email:
WHERE LOWER(email) = 'foo@example.com'

-- Fix:
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```
Any function applied to a column in a `WHERE` clause — `DATE()`, `EXTRACT()`, `::text` cast — disables index usage unless an expression index matches exactly.

**Why an index is ignored even when it exists:**
- Filter is not selective enough (planner prefers seq scan)
- Cast mismatch: `WHERE id = '123'` on an integer column causes implicit cast, disabling the index
- `LIKE '%suffix'` — leading wildcard disables B-tree index; use `pg_trgm` + GIN index for arbitrary substring search
- Index has not been analyzed recently; planner thinks it's larger or smaller than it is
- `OR` conditions: `WHERE a = 1 OR b = 2` won't use indexes on `a` and `b` separately; use `UNION ALL` or a multi-column index

---

## Phase 4 — Output

Produce whichever the user needs:

- **Plan annotation** — walk through their `EXPLAIN ANALYZE` output and flag the bottleneck node, the rows estimate mismatch, and the fix
- **Rewritten query** — subquery → join conversion, correlated subquery elimination, or CTE inlining
- **Index recommendation** — specific `CREATE INDEX` statement with column order rationale and whether `INCLUDE` or a partial index applies
- **Pagination rewrite** — keyset cursor implementation replacing `OFFSET`, including tiebreaker handling
- **Session-level diagnostic commands** — the exact `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` invocation to run, plus `ANALYZE tablename` if statistics look stale

Always include:
- Expected plan change after the fix (seq scan → index scan, nested loop → hash join)
- A note if the fix requires `work_mem` tuning rather than an index change
- Postgres version caveat if CTE materialization behavior is relevant
- Warning if `NOT IN` is used with a nullable subquery column