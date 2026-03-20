---
name: api-integration-test
description: Generate integration tests for REST API endpoints covering auth, validation, success and error responses, and DB side effects. Use when testing endpoints end-to-end or verifying route behavior and data persistence.
category: "Testing"
---

# API Integration Test

Generates integration tests for a REST endpoint that run against a real (or in-memory) server and database, covering the full request lifecycle.

---

## Phase 1: Discovery

Extract before writing anything:

1. **Framework** — Express, Fastify, NestJS, Hapi? (affects how the server is spun up in tests)
2. **Test runner** — Jest or Vitest?
3. **HTTP client** — `supertest`, `undici`, native `fetch`? Default to `supertest` if Express/Fastify.
4. **Auth mechanism** — JWT, session cookie, API key, none?
5. **Database** — real DB with test instance, in-memory (SQLite), or mocked at the service layer?
6. **What the endpoint does** — method, path, request body shape, expected response shape
7. **Existing test setup** — is there already a test app factory, seed util, or DB teardown helper?

If the user pastes the route handler, extract the above directly. If not, ask.

---

## Phase 2: Test Architecture Decisions

### Where to draw the integration boundary

| Approach | When to use | Non-obvious tradeoff |
|---|---|---|
| Real DB + real server | Full confidence | Slow; requires teardown discipline or tests bleed into each other |
| In-memory DB (SQLite, pg-mem) | Fast, isolated | Schema drift if migrations aren't applied to the in-memory instance |
| Real server + mocked service layer | Catches routing/middleware bugs | Misses DB constraint violations and query bugs |

Ask the user which approach they're using. Default to real DB with a test schema if they have migrations — the others require caveats in the output.

### Test isolation: the non-obvious problems

**Row bleed:** tests that insert rows without cleanup cause later tests to see unexpected data. Do not rely on `afterEach` DELETE alone — wrap each test in a transaction and roll it back instead:

```ts
beforeEach(async () => { await db.query('BEGIN') })
afterEach(async () => { await db.query('ROLLBACK') })
```

This works for read/write tests. It does NOT work if the endpoint itself commits a nested transaction or uses a separate connection pool — flag this if the user's ORM (Prisma, TypeORM) manages its own connections.

**Port conflicts:** do not hardcode ports. Let the OS assign one:

```ts
const server = app.listen(0) // 0 = OS-assigned
const { port } = server.address() as AddressInfo
```

With `supertest`, this is handled automatically — `request(app)` never binds a port. Flag if the user is using `fetch` with a hardcoded URL.

**Schema state:** if using real DB, run migrations (not seed scripts) in `beforeAll`. Never run migrations per-test — too slow. Teardown by dropping the test schema, not individual tables.

---

## Phase 3: Test Case Coverage

### Auth tests (non-obvious cases only)

Beyond "no token = 401", generate:
- **Expired token** — JWT with `exp` in the past; requires either a test helper that signs with past exp or a library like `jsonwebtoken` with explicit `expiresIn: -1`
- **Wrong audience/issuer** — if the endpoint validates `aud`/`iss` claims, test that a valid token from a different service is rejected
- **Insufficient scope/role** — authenticated but wrong permission level; should be 403, not 401. Many codebases conflate these

```ts
it('rejects a valid token with insufficient role', async () => {
  const token = signToken({ sub: 'user-1', role: 'viewer' }) // not 'admin'
  const res = await request(app)
    .delete('/api/posts/1')
    .set('Authorization', `Bearer ${token}`)
  expect(res.status).toBe(403) // not 401
})
```

### Validation tests

Generate one test per distinct validation rule, not one test per field. If a field has both `required` and `maxLength`, test them separately — they often hit different code paths.

**Non-obvious:** test the boundary, not just the interior. If `maxLength` is 100, test with 100 chars (should pass) and 101 chars (should fail). Most bugs live at the boundary.

For validation error responses, also assert the **shape** of the error body, not just the status:

```ts
expect(res.body).toMatchObject({
  errors: expect.arrayContaining([
    expect.objectContaining({ field: 'email', message: expect.any(String) })
  ])
})
```

### Success response tests

Assert:
- Status code
- Response body shape (use `toMatchObject`, not `toEqual` — lets the response add fields without breaking the test)
- `Content-Type` header (commonly forgotten, causes client-side parse failures)

Do NOT assert exact timestamps or generated IDs — use `expect.any(String)` / `expect.any(Number)`.

### DB side-effect tests

This is the most commonly skipped coverage. After a mutating request succeeds, query the DB directly and assert the persisted state:

```ts
it('persists the created post', async () => {
  const res = await request(app)
    .post('/api/posts')
    .send({ title: 'Hello', body: 'World' })
    .set('Authorization', `Bearer ${adminToken}`)

  expect(res.status).toBe(201)

  const row = await db.query('SELECT * FROM posts WHERE id = $1', [res.body.id])
  expect(row.rows[0]).toMatchObject({ title: 'Hello', body: 'World' })
})
```

**Non-obvious:** also test that a failed request does NOT persist. If the endpoint has multi-step writes, simulate a failure mid-way (via a mock on the second write) and assert the first write was rolled back.

### Error response tests

Cover:
- **404** — resource not found (use a valid-format but nonexistent ID, not a malformed one)
- **409** — conflict (duplicate unique field); requires seeding the conflicting row first
- **500** — for endpoints with known external dependencies, stub the dependency to throw and assert the endpoint returns 500 with a non-leaking error body (no stack traces, no DB errors)

```ts
it('does not leak internal errors in the response body', async () => {
  jest.spyOn(db, 'query').mockRejectedValueOnce(new Error('connection refused'))
  const res = await request(app).get('/api/users/1').set('Authorization', `Bearer ${token}`)
  expect(res.status).toBe(500)
  expect(res.body).not.toHaveProperty('stack')
  expect(res.body).not.toHaveProperty('query')
})
```

---

## Phase 4: Test File Structure

```ts
// tests/api/posts.test.ts

describe('POST /api/posts', () => {
  describe('auth', () => { /* 401, 403 cases */ })
  describe('validation', () => { /* 400 cases, one per rule */ })
  describe('success', () => { /* 201, response shape, DB state */ })
  describe('errors', () => { /* 404, 409, 500 */ })
})
```

**Why nested describes:** lets you run just `auth` or just `validation` with `--testNamePattern`. Flat test files with long names do not compose.

### Shared setup placement

- `beforeAll` — start server, run migrations, create seeded base rows needed by all tests
- `beforeEach` — begin transaction (if using rollback strategy), generate fresh auth tokens
- `afterEach` — rollback transaction
- `afterAll` — close DB connection, close server

Close the server in `afterAll`. Supertest keeps the process alive if the server is not explicitly closed, causing Jest to hang with `--detectOpenHandles`.

---

## Phase 5: Output

Produce:

1. **Test file** — fully runnable, with all describe blocks, shared setup, and one concrete test per case
2. **Token helper** (if JWT auth) — a `signToken(claims)` util that signs with the test secret
3. **DB seed helper** (if needed) — a minimal `seedPost(overrides?)` factory that inserts a row and returns its ID

### Output notes

- Use real assertions, not `expect(res.status).toBeTruthy()` — that passes on 404
- If the user is on Vitest: `vi.spyOn` instead of `jest.spyOn`; everything else is the same
- If the endpoint is idempotent (GET, PUT with upsert), note which tests can run in any order
- Do not generate a test for every possible invalid input combination — generate the minimal set that covers each distinct code path