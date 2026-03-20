---
name: permission-boundary-test
description: Test that users cannot access others’ resources, covering IDOR, role violations, and ownership checks. Use when validating authorization, access control, or multi-tenant isolation.
category: "Testing"
---

# Permission Boundary Test Skill

## Discovery

Before writing tests, map:

- **Resource inventory** — every entity with an owner (`users`, `documents`, `orders`, `invoices`, etc.)
- **Access patterns** — which HTTP methods and routes touch each resource (GET, PUT, PATCH, DELETE, and non-REST actions like `/share`, `/export`, `/duplicate`)
- **Role model** — flat ownership only, or RBAC with roles (admin, member, viewer)? Org/tenant hierarchy?
- **ID type** — sequential integers are trivially enumerable; UUIDs reduce but don't eliminate IDOR risk
- **Indirect access paths** — can resource B be reached by manipulating resource A? (e.g. a comment endpoint that exposes its parent post's content)

---

## The Two User Pattern

Every permission boundary test needs exactly two authenticated users. Never test with one user and an anonymous request — that tests authentication, not authorization:

```ts
// Setup — two users, each owning their own resource
const [alice, aliceToken] = await createUserWithToken();
const [bob, bobToken] = await createUserWithToken();

const aliceDoc = await api.post('/documents', { title: 'Alice doc' }, aliceToken);
const bobDoc = await api.post('/documents', { title: 'Bob doc' }, bobToken);
```

All cross-ownership assertions use `bobToken` attempting to access `aliceDoc.id` — a real authenticated user, not a missing auth header.

---

## Non-Obvious Patterns

### 1. Test every HTTP method, not just GET

IDOR in read endpoints is obvious. The more dangerous cases are write and delete:

```ts
const methods = [
  { method: 'GET',    path: `/documents/${aliceDoc.id}` },
  { method: 'PUT',    path: `/documents/${aliceDoc.id}`, body: { title: 'hijacked' } },
  { method: 'PATCH',  path: `/documents/${aliceDoc.id}`, body: { title: 'hijacked' } },
  { method: 'DELETE', path: `/documents/${aliceDoc.id}` },
  { method: 'POST',   path: `/documents/${aliceDoc.id}/share` },
  { method: 'POST',   path: `/documents/${aliceDoc.id}/export` },
];

for (const { method, path, body } of methods) {
  const res = await api.request(method, path, body, bobToken);
  expect(res.status).toBe(403); // or 404 — see note below
}
```

Non-REST action endpoints (`/share`, `/publish`, `/transfer`, `/clone`) are the most commonly forgotten.

---

### 2. 403 vs 404 — pick one and enforce it consistently

Returning `404` for unauthorized access is a common security pattern (resource "doesn't exist" to the requester). Returning `403` is explicit. Both are valid, but mixing them is not — it leaks information about resource existence:

```ts
// If the policy is "404 for unauthorized access":
expect(res.status).toBe(404); // consistent — Bob can't tell if Alice's doc exists

// If the policy is "403 for unauthorized access":
expect(res.status).toBe(403); // consistent — Bob knows it exists but can't access it

// BROKEN — leaks existence:
// GET /documents/:id  → 404 (Bob can't tell it exists)
// DELETE /documents/:id → 403 (now Bob knows it exists)
```

Assert the *same* status code across all methods for a given resource. Document the policy choice in the test file.

---

### 3. Test indirect access — IDOR through related resources

A user who can't GET `/invoices/42` directly might be able to reach it via `/payments/99/invoice` if the payment belongs to them but the invoice doesn't:

```ts
// Bob owns payment, but invoice belongs to Alice
const bobPayment = await createPayment(bob, { invoiceId: aliceInvoice.id });

// Direct access — should fail
const direct = await api.get(`/invoices/${aliceInvoice.id}`, bobToken);
expect(direct.status).toBe(403);

// Indirect access through related resource — must also fail
const indirect = await api.get(`/payments/${bobPayment.id}/invoice`, bobToken);
expect(indirect.status).toBe(403); // not 200
```

---

### 4. Test ID manipulation in request bodies, not just URL params

Ownership checks on URL params are common. Ownership checks on body parameters are often missing:

```ts
// Alice's document, Bob's session — ID in body, not URL
const res = await api.post('/exports', {
  documentId: aliceDoc.id,  // Bob referencing Alice's resource in the payload
  format: 'pdf'
}, bobToken);
expect(res.status).toBe(403);

// Also test nested IDs
const res2 = await api.post('/comments', {
  content: 'hijacked comment',
  parentId: aliceDoc.id,  // parent is Alice's resource
}, bobToken);
expect(res2.status).toBe(403);
```

---

### 5. Test mass assignment — can Bob promote himself?

If the API accepts role or ownership fields in update payloads, verify they're ignored or rejected:

```ts
// Bob trying to make himself an admin of Alice's org
const res = await api.patch(`/org-members/${bob.memberId}`, {
  role: 'admin',
  orgId: aliceOrg.id, // Bob referencing Alice's org
}, bobToken);
expect(res.status).toBeOneOf([400, 403]);

// Verify Bob's role was not changed
const member = await db.findMember(bob.memberId);
expect(member.role).toBe('member'); // unchanged
```

---

### 6. Test multi-tenant boundaries — same resource, different org

In multi-tenant apps, the primary isolation boundary is org/tenant, not just user:

```ts
const orgA = await createOrg();
const orgB = await createOrg();
const [userA, tokenA] = await createUserInOrg(orgA);
const [userB, tokenB] = await createUserInOrg(orgB);
const orgAResource = await createResource(orgA);

// OrgB user cannot access OrgA resource — even as admin of OrgB
const res = await api.get(`/resources/${orgAResource.id}`, tokenB);
expect(res.status).toBe(403);

// Verify the response body doesn't leak org metadata
expect(res.body).not.toHaveProperty('orgId');
expect(res.body).not.toHaveProperty('orgName');
```

---

### 7. Test after ownership transfer — stale access

If resources can be transferred or shared, verify the old owner loses access:

```ts
// Alice transfers doc to Bob
await api.post(`/documents/${aliceDoc.id}/transfer`, { newOwnerId: bob.id }, aliceToken);

// Alice can no longer access her old document
const res = await api.get(`/documents/${aliceDoc.id}`, aliceToken);
expect(res.status).toBe(403);

// Bob can now access it
const res2 = await api.get(`/documents/${aliceDoc.id}`, bobToken);
expect(res2.status).toBe(200);
```

---

### 8. Test pagination and list endpoints — no cross-user data leakage

List endpoints commonly filter by the authenticated user, but verify:

```ts
// Alice creates 3 docs; Bob creates 2
await createDocs(aliceToken, 3);
await createDocs(bobToken, 2);

// Bob's list must only contain Bob's docs
const res = await api.get('/documents', bobToken);
expect(res.body.items).toHaveLength(2);
expect(res.body.items.every(doc => doc.ownerId === bob.id)).toBe(true);

// No Alice doc IDs present anywhere in the response
const responseIds = res.body.items.map(d => d.id);
expect(responseIds).not.toContain(aliceDoc.id);
```

Aggregate endpoints (`/stats`, `/search`, `/reports`) are frequently missed here.

---

### 9. Verify the response body, not just the status code

A 403 with the forbidden resource's data in the body is still a leak:

```ts
const res = await api.get(`/documents/${aliceDoc.id}`, bobToken);
expect(res.status).toBe(403);

// Must not contain Alice's data even in the error response
expect(res.body).not.toHaveProperty('content');
expect(res.body).not.toHaveProperty('title');
expect(JSON.stringify(res.body)).not.toContain(aliceDoc.title);
```

---

## Generating Test Coverage Systematically

For codebases without an obvious resource list, extract routes programmatically:

```ts
// Express — extract all registered routes
app._router.stack
  .filter(r => r.route)
  .map(r => ({ path: r.route.path, methods: Object.keys(r.route.methods) }));
```

For each route with a dynamic segment (`:id`, `:resourceId`), generate a cross-user test pair. Routes without dynamic segments typically don't have ownership boundaries — skip them.

---

## Test Structure

```
permission-boundary/
  documents.spec.ts     — one file per resource type
  invoices.spec.ts
  comments.spec.ts
  list-endpoints.spec.ts  — all GET /resource (no ID) endpoints
  indirect-access.spec.ts — relational/nested IDOR cases
```

Shared setup in `fixtures/users.ts` — `createUserWithToken()` factory used by all files. Each test file seeds its own resources; no cross-file state.

---

## What Not to Do

- **Don't test unauthenticated access here** — that's an authentication test; this skill is for authenticated cross-user access only
- **Don't use a single shared test user** — all tests require two distinct users with separately owned resources
- **Don't only test GET** — write, delete, and action endpoints are where IDOR most often ships to production
- **Don't skip body-parameter ID tests** — URL param checks are obvious; body param checks are where authorization is most often missing
- **Don't accept inconsistent status codes** — pick 403 or 404 for unauthorized access and enforce it across all endpoints; mixing leaks resource existence

---

## Output Format

One file per resource type. Each file has a `describe` block per HTTP method. Every `it` is named `'[role/user] cannot [action] [resource] belonging to [other user]'`. After status code assertion, always include a DB or body assertion confirming no data was mutated or leaked.