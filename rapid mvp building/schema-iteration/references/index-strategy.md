# Index Strategy Guide

## When to add an index

### Always index
- Every foreign key column (most commonly missed; causes full scans on joins)
- Every column used in a `WHERE` clause on large tables
- Every column used in `ORDER BY` on paginated queries
- Columns with `UNIQUE` constraints (the constraint creates an index automatically)

### Consider indexing
- Columns used in `GROUP BY` on large aggregations
- Columns used in range filters (`BETWEEN`, `>`, `<`) — B-tree indexes help
- Columns used in `JOIN` conditions that aren't FKs

### Don't index
- Columns on small tables (< ~10k rows) — full scan is faster
- Low-cardinality columns like boolean flags or status enums with 2-3 values — index not selective enough to help (partial index may still help, see below)
- Columns that are updated very frequently on write-heavy tables — index maintenance cost may outweigh read benefit
- Columns never used in queries

---

## Composite indexes

A composite index on `(a, b)` helps queries that filter on:
- `a` alone (uses the leftmost prefix)
- `a` and `b` together

It does NOT help queries that filter on `b` alone.

**Rule:** Put the most selective column first, or the equality column before the range column.

```sql
-- Good: filter on status (equality) then sort by created_at (range/order)
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

-- This query benefits:
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20;
```

---

## Partial indexes

Index only a subset of rows. Useful for:
- Filtering a boolean/status column where only one value is queried often

```sql
-- PostgreSQL: index only unprocessed jobs
CREATE INDEX idx_jobs_pending ON jobs (created_at)
WHERE status = 'pending';

-- This is much smaller and faster than a full index on created_at
```

---

## Covering indexes

Include extra columns in the index so the query can be answered from the index alone (no table heap access).

```sql
-- PostgreSQL: INCLUDE clause
CREATE INDEX idx_orders_user_covering ON orders (user_id)
INCLUDE (status, created_at);

-- Query satisfied entirely from index:
SELECT status, created_at FROM orders WHERE user_id = 42;
```

---

## Avoiding table locks when adding indexes

### PostgreSQL
```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
-- Takes longer but doesn't lock writes
-- Cannot run inside a transaction block
```

### MySQL / MariaDB
```sql
ALTER TABLE orders ADD INDEX idx_user_id (user_id), ALGORITHM=INPLACE, LOCK=NONE;
```

### SQLite
SQLite doesn't support concurrent index creation — adding indexes locks the table.
Acceptable for small datasets; for large ones, do it during a maintenance window.

---

## Redundant index detection

An index is redundant if another index covers the same leftmost prefix:

| Existing index       | New index      | Verdict              |
|----------------------|----------------|----------------------|
| `(a, b)`             | `(a)`          | Redundant — drop `(a)` |
| `(a)`                | `(a, b)`       | Keep both if queries use `(a)` alone AND `(a, b)` together |
| `(a)` UNIQUE         | `(a)`          | Redundant — UNIQUE creates an index |

Use `pg_stat_user_indexes` (PostgreSQL) or `sys.dm_db_index_usage_stats` (SQL Server) to find indexes that are never used in production.

---

## Index size estimates (rough)

| Rows     | B-tree index on INT column | B-tree index on VARCHAR(255) |
|----------|---------------------------|------------------------------|
| 100k     | ~3 MB                     | ~10-30 MB                   |
| 1M       | ~30 MB                    | ~100-300 MB                 |
| 10M      | ~300 MB                   | ~1-3 GB                     |

Keep this in mind when adding many indexes to large tables — disk and memory pressure add up.