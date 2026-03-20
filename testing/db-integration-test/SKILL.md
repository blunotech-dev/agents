---
name: db-integration-test
description: Write tests against a real test database to verify CRUD operations, queries, constraints, transactions, and migrations. Use when validating DB logic, ORM/repository layers, or data integrity without mocks.
category: "Testing"
---

# DB Integration Test Skill

## Discovery

Before writing tests, extract:

- **DB engine** — Postgres, MySQL, SQLite, MongoDB, etc. (affects transaction syntax, constraint behavior, JSON support)
- **Access layer** — raw SQL, query builder (Knex, Drizzle), ORM (Prisma, TypeORM, Sequelize)
- **Schema** — tables, columns, constraints (NOT NULL, UNIQUE, FK, CHECK), indexes
- **Test DB strategy** — is there an existing test DB, Docker Compose setup, or in-process option (SQLite, `pg-mem`)?
- **Seeding approach** — fixtures, factories, or raw inserts? Already established or needs creating?

---

## Test DB Setup — Non-Obvious Decisions

### 1. Isolate tests with transactions, not table truncation

Transaction rollback is faster and cleaner than truncating between tests:

```ts
beforeEach(async () => {
  await db.query('BEGIN');
});

afterEach(async () => {
  await db.query('ROLLBACK');
});
```

Truncation is only necessary when the code under test explicitly commits (e.g. testing `COMMIT` behavior itself, or cross-connection visibility).

---

### 2. Never share connection state between tests

Each test should get a fresh connection or a connection scoped to a transaction. Pooled connections that carry session state (temp tables, `SET` variables, advisory locks) will cause flaky cross-test contamination.

```ts
// Correct — scoped connection per test
const client = await pool.connect();
await client.query('BEGIN');
// ... test
await client.query('ROLLBACK');
client.release();
```

---

### 3. Schema setup: migrations over raw DDL

Run the real migration files in test setup, not hand-rolled `CREATE TABLE`. This ensures tests break when migrations change — which is the point:

```ts
beforeAll(async () => {
  await runMigrations(testDb); // same migration runner used in production
});
```

---

## What to Test

### CRUD — assert the round-trip, not just absence of error

```ts
// WEAK
await repo.create({ name: 'Alice' });
// no assertion

// STRONG — assert the persisted state, not the in-memory return value
await repo.create({ name: 'Alice' });
const row = await db.query('SELECT * FROM users WHERE name = $1', ['Alice']);
expect(row.rows[0]).toMatchObject({ name: 'Alice', active: true });
```

Always read back from the DB directly for write tests — ORM return values can diverge from actual persisted state.

---

### Constraints — trigger them deliberately

Test each constraint type with a case that violates it:

```ts
// UNIQUE constraint
await repo.create({ email: 'a@b.com' });
await expect(repo.create({ email: 'a@b.com' })).rejects.toThrow(/unique/i);

// NOT NULL
await expect(repo.create({ email: null })).rejects.toThrow(/null/i);

// FK constraint
await expect(
  repo.createPost({ userId: 99999 }) // non-existent FK
).rejects.toThrow(/foreign key/i);

// CHECK constraint
await expect(
  repo.create({ age: -1 }) // violates CHECK age > 0
).rejects.toThrow(/check/i);
```

Match on error message pattern, not exact string — DB engines phrase these differently across versions.

---

### Transactions — test the rollback, not just the commit

```ts
// Verify atomicity: partial failure rolls back all changes
await expect(async () => {
  await db.transaction(async (trx) => {
    await trx.insert(users).values({ id: 1, name: 'Alice' });
    await trx.insert(users).values({ id: 1, name: 'Bob' }); // duplicate PK — throws
  });
}).rejects.toThrow();

// Assert Alice was NOT persisted (rollback succeeded)
const result = await db.select().from(users).where(eq(users.id, 1));
expect(result).toHaveLength(0);
```

This is the only test that proves your transaction boundary is real. Without it, you're trusting the ORM.

---

### Query correctness — test data shape and filtering, not just row count

```ts
// WEAK — only verifies something came back
const results = await repo.findActive();
expect(results.length).toBeGreaterThan(0);

// STRONG — seed known data, assert exact filtering behavior
await seedUsers([
  { name: 'Alice', active: true },
  { name: 'Bob', active: false },
]);
const results = await repo.findActive();
expect(results).toHaveLength(1);
expect(results[0].name).toBe('Alice');
```

Always seed the exact data you need — never rely on pre-existing DB state.

---

### Soft deletes — assert the record is hidden, not gone

```ts
await repo.delete(userId);

// The record must be invisible to normal queries
const visible = await repo.findById(userId);
expect(visible).toBeNull();

// But still present in the DB with deleted_at set
const raw = await db.query(
  'SELECT deleted_at FROM users WHERE id = $1', [userId]
);
expect(raw.rows[0].deleted_at).not.toBeNull();
```

---

### Pagination and ordering — test the edges

```ts
// Seed 5 records, request page 2 of size 2
await seedUsers(5);
const page2 = await repo.list({ page: 2, size: 2 });
expect(page2).toHaveLength(2);

// Last page — partial results
const page3 = await repo.list({ page: 3, size: 2 });
expect(page3).toHaveLength(1);

// Empty page beyond range
const page4 = await repo.list({ page: 4, size: 2 });
expect(page4).toHaveLength(0);

// Ordering is stable — same seed, same order on repeated calls
const a = await repo.list({ sort: 'created_at' });
const b = await repo.list({ sort: 'created_at' });
expect(a.map(r => r.id)).toEqual(b.map(r => r.id));
```

---

### Concurrent writes — test optimistic locking or last-write-wins

If the schema uses a `version` or `updated_at` for optimistic locking:

```ts
const record = await repo.findById(id);

// Simulate concurrent update elsewhere
await db.query('UPDATE items SET version = version + 1 WHERE id = $1', [id]);

// This write should fail — stale version
await expect(
  repo.update({ ...record, name: 'new' })
).rejects.toThrow(/conflict|stale|version/i);
```

---

## Seeding Patterns

### Minimal seed — only insert what the test needs

```ts
// Bad — seeding 20 users when only 1 is needed makes assertions fragile
await seedFixture('users-large.json');

// Good — seed the exact shape you assert against
const user = await factory.create('user', { email: 'test@example.com', active: true });
```

### Factory pattern over static fixtures

Static fixture files drift from schema changes silently. A factory function that calls `repo.create(defaults)` with overrides will break loudly when the schema changes:

```ts
const createUser = (overrides = {}) =>
  db.insert(users).values({ name: 'Default', active: true, ...overrides }).returning();
```

---

## What Not to Do

- **Don't mock the DB in integration tests** — that's a unit test; the entire point here is hitting real SQL
- **Don't assert ORM return values as proof of persistence** — read back from the DB directly for write assertions
- **Don't use `TRUNCATE` between tests if transaction rollback works** — truncation resets sequences and is 10-100x slower
- **Don't share seed data across unrelated tests** — each test owns its data; shared state causes ordering dependencies
- **Don't catch DB errors in test setup** — let setup failures surface loudly; silent setup failures produce false passes

---

## Output Format

Group tests by operation type: setup/teardown at the top, then CRUD, constraints, transactions, and query correctness. Each `it` asserts one DB contract. Include a comment on non-obvious cases explaining the invariant being verified, not the mechanics.