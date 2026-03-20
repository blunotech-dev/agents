---
name: db-transaction-handling
description: Implement correct transaction handling with proper isolation levels, retries, and rollback on failure. Use when dealing with concurrency issues, deadlocks, inconsistent data, or transaction design (e.g., SELECT FOR UPDATE, locking strategies).
category: "Backend"
---

# DB Transaction Handling Skill

## Phase 1 — Discovery

Ask only what context doesn't reveal:

- **Database engine?** Isolation level semantics differ. Postgres uses MVCC and doesn't implement `READ UNCOMMITTED` (treats it as `READ COMMITTED`). MySQL's `REPEATABLE READ` prevents phantom reads via gap locks; Postgres's doesn't (requires `SERIALIZABLE`).
- **What consistency problem are you solving?** Lost update, phantom read, write skew, or double-spend? The isolation level needed depends on which anomaly you're preventing.
- **ORM or raw SQL?** ORMs often silently open transactions per query or wrap saves incorrectly — the fix location differs.
- **Is this a high-contention path?** Optimistic locking is better than pessimistic for low-contention reads. Pessimistic (`SELECT FOR UPDATE`) is better when contention is high and retries are expensive.
- **Any long-running operations inside the transaction?** External HTTP calls, file I/O, or sleep inside a transaction is almost always a bug.

---

## Phase 2 — Isolation Level Selection

### The anomaly → isolation level mapping (the non-obvious part)

Most teams default to `READ COMMITTED` everywhere. It prevents dirty reads but not:

**Lost update:** Two transactions read a value, both compute a new value based on it, both write. One overwrites the other.
```
T1: read balance=100, compute 100-30=70, write 70
T2: read balance=100, compute 100-50=50, write 50  ← T1's update is lost
```
Fix options (in order of preference):
1. Atomic update: `UPDATE accounts SET balance = balance - 30 WHERE id = ?` — no read needed
2. Optimistic locking: read with a `version` column, write with `WHERE version = :read_version`
3. `SELECT FOR UPDATE` (pessimistic lock)
4. `REPEATABLE READ` — prevents lost update in Postgres via write conflict detection

**Write skew:** Two transactions each read an overlapping set, make decisions based on what they read, and write to disjoint rows — neither update conflicts, but the combined result violates an invariant.
```
-- Invariant: at least one doctor must be on call
T1: reads 2 doctors on call, marks doctor A as off
T2: reads 2 doctors on call, marks doctor B as off
-- Result: 0 doctors on call; no conflict detected
```
`REPEATABLE READ` does **not** prevent write skew. Only `SERIALIZABLE` does — or explicit `SELECT FOR UPDATE` on the rows both transactions read.

**Phantom read:** A transaction re-runs a range query and gets different rows because another transaction inserted into that range.
`REPEATABLE READ` prevents phantoms in MySQL (gap locks). In Postgres, `REPEATABLE READ` does not — only `SERIALIZABLE` prevents phantoms in Postgres.

### Isolation level behavior by engine

| Anomaly | PG READ COMMITTED | PG REPEATABLE READ | PG SERIALIZABLE | MySQL RR |
|---|---|---|---|---|
| Dirty read | ✗ | ✗ | ✗ | ✗ |
| Lost update | ✓ possible | ✗ prevented | ✗ prevented | ✗ prevented |
| Phantom read | ✓ possible | ✓ possible | ✗ prevented | ✗ prevented |
| Write skew | ✓ possible | ✓ possible | ✗ prevented | ✓ possible |

### `SERIALIZABLE` is often more practical than teams assume

Teams avoid `SERIALIZABLE` fearing performance cost. The actual cost is retry overhead on serialization failures (`ERROR: could not serialize access due to concurrent update`). For low-contention paths, serialization failures are rare and retry logic is cheap. It's a better tradeoff than implementing complex application-level locking.

---

## Phase 3 — Implementation

### Deadlock prevention

Deadlocks occur when two transactions each hold a lock the other needs. They're prevented by consistent lock ordering — not by shorter transactions alone.

**Consistent ordering rule:**
```python
# Bad: T1 locks account 1 then 2; T2 locks account 2 then 1 → deadlock possible
def transfer(from_id, to_id, amount):
    lock(from_id)
    lock(to_id)  # T2 may hold this

# Good: always lock in ascending ID order
def transfer(from_id, to_id, amount):
    first, second = sorted([from_id, to_id])
    lock(first)
    lock(second)
```

**`SELECT FOR UPDATE SKIP LOCKED` — the job queue pattern:**
Used to claim rows without blocking other workers:
```sql
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```
Without `SKIP LOCKED`, multiple workers queue up waiting for the same locked row. With it, each worker moves to the next available row immediately.

**`SELECT FOR UPDATE OF` — limit which tables are locked in a join:**
```sql
-- Locks only orders row, not joined customer row
SELECT o.* FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.id = ?
FOR UPDATE OF o;
```
Locking more tables than necessary in a join is a common deadlock source.

**Advisory locks — for application-level mutual exclusion:**
When you need to coordinate at a level above row locks (e.g., "only one process should run this job"):
```sql
-- Session-level: released on disconnect
SELECT pg_try_advisory_lock(hashtext('job:export:user:123'));

-- Transaction-level: released on commit/rollback
SELECT pg_try_advisory_xact_lock(hashtext('job:export:user:123'));
```
`pg_try_advisory_lock` returns false instead of blocking — always check the return value. Use `pg_try_advisory_xact_lock` in transaction scope; it auto-releases and can't be leaked by a crashed process.

### Retry logic for serialization failures

`SERIALIZABLE` isolation and `SELECT FOR UPDATE` can both raise errors that are safe to retry. These are not bugs — they're the database telling you to re-run the transaction.

Postgres error codes to retry:
- `40001` — serialization failure
- `40P01` — deadlock detected

```python
import psycopg2
import time
import random

RETRYABLE_CODES = {'40001', '40P01'}

def with_retry(fn, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return fn()
        except psycopg2.errors.lookup(RETRYABLE_CODES[0]) as e:
            if attempt == max_attempts - 1:
                raise
            # Exponential backoff with jitter
            sleep = (2 ** attempt) * 0.1 + random.uniform(0, 0.05)
            time.sleep(sleep)
```

Critical: the entire transaction must be re-run from the start, including the initial reads. Retrying only the write half means the decision was made on stale data.

Trap: ORMs that silently retry only the failing statement rather than the full transaction — this produces the lost update the isolation level was meant to prevent.

### Rollback on partial failure

**The implicit rollback trap in some ORMs:**
Some ORMs (ActiveRecord, SQLAlchemy) silently begin a transaction per request but don't roll back on unhandled exceptions in all code paths. Verify your framework's behavior — don't assume.

**Savepoints for nested partial rollback:**
When you need to attempt an operation and roll back only that part on failure without aborting the outer transaction:
```sql
BEGIN;
INSERT INTO orders ...;

SAVEPOINT before_notification;
INSERT INTO notifications ...;  -- might fail; non-critical
ROLLBACK TO SAVEPOINT before_notification;  -- undo only notification insert
RELEASE SAVEPOINT before_notification;

COMMIT;  -- order still committed
```

**What can't be rolled back:**
- Sequences (`nextval`) — always increment even on rollback. Don't use sequence gaps as evidence of missing records.
- External side effects — emails sent, webhooks fired, Stripe charges initiated inside a transaction that later rolls back. Always move external calls outside the transaction or use an outbox pattern.

### Long transactions — the systemic problem

A transaction held open for seconds or minutes blocks:
- Autovacuum from reclaiming dead rows (Postgres)
- Replication slot advancement
- Lock escalation on any row touched

**Symptoms in Postgres:**
```sql
-- Find long-running transactions
SELECT pid, now() - xact_start AS duration, query, state
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY duration DESC;

-- Find locks blocking other queries
SELECT blocked.pid, blocking.pid AS blocking_pid, blocked.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

Common causes:
- HTTP calls inside a transaction (`with db.transaction(): response = requests.post(...)`)
- Holding a transaction open while waiting for user input or a queue message
- ORM lazy-loading related objects inside an open transaction that was opened much earlier

Fix: keep transactions as short as possible. Do all reads and computation first, open the transaction only for the writes, commit immediately.

### Optimistic locking without an ORM

```sql
-- Read
SELECT id, balance, version FROM accounts WHERE id = ?;

-- Write: fails silently if version changed
UPDATE accounts
SET balance = :new_balance, version = version + 1
WHERE id = :id AND version = :read_version;

-- Check rows affected — 0 means conflict
```

If `rowcount == 0`: another transaction updated the row between read and write. Retry from the read. This is safe under `READ COMMITTED` — no elevated isolation level needed.

---

## Phase 4 — Output

Produce whichever the user needs:

- **Transaction wrapper** — correct begin/commit/rollback structure with retry logic for the user's language and ORM
- **Isolation level recommendation** — identify the anomaly they're vulnerable to and the minimum isolation level that prevents it
- **Deadlock fix** — reorder lock acquisition or replace with `SKIP LOCKED` / advisory locks
- **Optimistic locking implementation** — version column approach with conflict detection
- **Long transaction audit** — identify external calls or lazy loads inside open transactions in their code
- **Savepoint pattern** — partial rollback for non-critical nested operations

Always include:
- The specific Postgres error codes to catch for retry logic (`40001`, `40P01`)
- A note if the chosen isolation level behaves differently in MySQL vs. Postgres
- Warning if any operation inside the transaction can't be rolled back (sequences, external calls)
- Confirmation that retry logic re-runs the full transaction, not just the failing statement