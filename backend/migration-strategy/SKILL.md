---
name: migration-strategy
description: Implement safe database migrations with zero-downtime patterns, rollbacks, and strategies for large schema changes without data loss. Use when the user asks about migrations, schema updates, backfills, column/table changes, or tools like Flyway, Liquibase, Alembic, or ActiveRecord.
category: "Backend"
---

# Migration Strategy Skill

## Phase 1 — Discovery

Ask only what context doesn't already reveal:

- **Database engine?** Postgres, MySQL, SQLite, and MSSQL have meaningfully different lock behaviors. What's safe in Postgres can table-lock in MySQL.
- **Table size?** Under 1M rows: standard migration. Over 10M rows: requires a different playbook (see Phase 3).
- **Deployment model?** Blue-green, rolling, or big-bang deploy. Zero-downtime is only achievable with rolling or blue-green.
- **Migration tool in use?** Alembic, Flyway, Rails, Prisma — each has different support for transactional DDL.
- **Can the app tolerate a read-only window?** Affects whether you need full online migration or just careful ordering.

---

## Phase 2 — Strategy Selection

### The core zero-downtime constraint

Two versions of your app run simultaneously during a rolling deploy: the old version and the new version. Your schema must be valid for **both** at the same time. This single rule drives every pattern below.

### What actually causes downtime (non-obvious)

- `ALTER TABLE` on large tables acquires an `ACCESS EXCLUSIVE` lock in Postgres — blocks all reads and writes for the duration.
- `NOT NULL` constraints without a default force a full table rewrite in most engines even if the column already has no nulls.
- Adding a column with a non-null default in Postgres < 11 rewrites the entire table. In Postgres ≥ 11 it's instant — know your version.
- `RENAME COLUMN` is instant on most engines but is an immediate breaking change for any running app instances referencing the old name.
- Foreign key constraints trigger a full table scan to validate existing data unless you add them as `NOT VALID` first.
- `CREATE INDEX` without `CONCURRENTLY` locks the table. Always use `CONCURRENTLY` in production — but note it can't run inside a transaction block.

---

## Phase 3 — Implementation

### Column rename (the most mishandled migration)

Never rename in one step. Use the expand-contract pattern across three deploys:

**Step 1 — Expand (migration only, no app change yet):**
```sql
ALTER TABLE users ADD COLUMN full_name text;
-- Backfill: UPDATE users SET full_name = name WHERE full_name IS NULL;
-- Add write trigger or handle dual-write in app
```

**Step 2 — Migrate app (read from new column, write to both):**
Deploy app code that writes to both `name` and `full_name`. Read from `full_name`. Old instances still write to `name` only — that's fine, the new column is not yet authoritative.

**Step 3 — Contract (after all old instances are gone):**
```sql
ALTER TABLE users DROP COLUMN name;
```

Trap: Teams skip step 2 and deploy the rename and the app change simultaneously. This creates a window where old running instances reference a column that no longer exists.

### Zero-downtime NOT NULL column

```sql
-- Step 1: Add nullable (instant)
ALTER TABLE orders ADD COLUMN shipped_at timestamptz;

-- Step 2: Backfill in batches (see large table section)

-- Step 3: Add check constraint as NOT VALID (skips existing row scan)
ALTER TABLE orders ADD CONSTRAINT orders_shipped_at_not_null
  CHECK (shipped_at IS NOT NULL) NOT VALID;

-- Step 4: Validate in background (takes ShareUpdateExclusiveLock, not AccessExclusive)
ALTER TABLE orders VALIDATE CONSTRAINT orders_shipped_at_not_null;

-- Step 5: Drop check constraint, add real NOT NULL (instant — constraint already proven)
ALTER TABLE orders ALTER COLUMN shipped_at SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT orders_shipped_at_not_null;
```

### Large table migrations (>10M rows)

Never `UPDATE` the whole table in one statement. It creates a massive transaction, holds locks, bloats WAL/undo logs, and can crash replication.

**Batch backfill pattern:**
```sql
-- Run in a loop from application code or a migration script
UPDATE orders
SET status = 'legacy'
WHERE id BETWEEN :batch_start AND :batch_end
  AND status IS NULL;
-- Sleep 10-50ms between batches to let replication catch up
```

Key parameters:
- Batch size: 1,000–10,000 rows. Larger isn't faster — it just increases lock contention.
- Add `WHERE migrated_at IS NULL` or a similar sentinel rather than scanning by primary key range when rows are non-sequential.
- Track progress in a separate `migration_progress` table or a Redis key. Never rely on "it'll finish eventually."
- Run at off-peak hours and throttle based on replication lag, not time of day alone.

**Online index creation:**
```sql
-- Must be outside a transaction block
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```
If `CONCURRENTLY` fails midway, it leaves an invalid index. Check with:
```sql
SELECT indexname FROM pg_indexes WHERE indexname = 'idx_orders_user_id';
SELECT * FROM pg_stat_user_indexes WHERE indexrelname = 'orders' AND idx_scan = 0;
-- Drop the invalid index and retry
DROP INDEX CONCURRENTLY idx_orders_user_id;
```

### Rollback scripts

Most teams write rollback as an afterthought. Non-obvious rules:

- **Rollback is not always the reverse of forward.** If your forward migration backfilled 2M rows, your rollback script can't un-backfill without knowing the prior state. Either: (a) write rollback as a no-op that accepts the new column exists, or (b) store a pre-migration snapshot of affected row IDs.
- **Data-destructive steps have no safe rollback.** Dropping a column, dropping a table, or deleting rows — once deployed and confirmed, rollback means data loss. Treat these as one-way doors and require a separate forward migration to undo.
- **Test rollback in CI, not just forward migration.** Add a `migrate down` step to your pipeline. It almost never runs in production, but when it does, it has to work.

Rollback template structure:
```sql
-- rollback_20240315_add_shipped_at.sql
-- Safe to run multiple times (idempotent)
ALTER TABLE orders DROP COLUMN IF EXISTS shipped_at;
DROP INDEX CONCURRENTLY IF EXISTS idx_orders_shipped_at;
```

### Foreign key pattern (zero-downtime)

```sql
-- Step 1: Add FK as NOT VALID — no table scan, instant
ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_order_id
  FOREIGN KEY (order_id) REFERENCES orders(id)
  NOT VALID;

-- Step 2: Validate separately — takes ShareUpdateExclusiveLock (non-blocking)
ALTER TABLE order_items
  VALIDATE CONSTRAINT fk_order_items_order_id;
```

### Transactional DDL — know your database

| Engine | DDL in transaction? | Notes |
|---|---|---|
| Postgres | Yes | Most DDL is transactional; `CREATE INDEX CONCURRENTLY` is not |
| MySQL/MariaDB | No | DDL causes implicit commit; rollback won't undo schema changes |
| SQLite | Yes | Full DDL transactional support |
| MSSQL | Yes | Most DDL transactional; some operations are not |

For MySQL: your migration tool's "rollback" button does not undo a completed `ALTER TABLE`. You must write a compensating forward migration.

### Schema change ordering for rolling deploys

Always deploy in this order — never combine schema + app in one deploy:

1. **Deploy migration (additive only):** new columns nullable, new tables, new indexes
2. **Deploy app:** reads/writes new schema; still handles old schema being absent gracefully
3. **Deploy cleanup migration:** drop old columns, add NOT NULL constraints, remove legacy paths

Trap: Deploying a migration and the app simultaneously in a rolling deploy means some instances run new app code against old schema mid-deploy.

---

## Phase 4 — Output

Produce whichever the user needs:

- **Migration file** — forward + rollback, idempotent, with batch size comments
- **Expand-contract plan** — three-step rename/drop sequence with deploy boundaries marked
- **Backfill script** — batched update loop with progress tracking and sleep intervals
- **Rollback risk assessment** — flag which steps in a migration are one-way doors
- **CI migration test snippet** — runs `migrate up` + `migrate down` in pipeline

Always include:
- Postgres version caveat if using `ADD COLUMN WITH DEFAULT` (behavior changed in v11)
- `CONCURRENTLY` reminder on any index operation
- Note if any step cannot run inside a transaction (affects Flyway/Alembic autocommit settings)