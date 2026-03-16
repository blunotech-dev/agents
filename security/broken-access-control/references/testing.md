# Testing Access Control

## Unit Tests — Policy Functions

Test authorization logic in isolation. Fast, no HTTP layer needed.

```ts
describe('invoicePolicy.read', () => {
  const owner   = { id: '1', role: 'user',      org_id: 'org-a' };
  const other   = { id: '2', role: 'user',      org_id: 'org-a' };
  const admin   = { id: '3', role: 'admin',     org_id: 'org-b' };
  const invoice = { id: 'inv-1', owner_id: '1', org_id: 'org-a', status: 'open' };

  it('allows owner',           () => expect(invoicePolicy.read(owner, invoice)).toBe(true));
  it('denies other user',      () => expect(invoicePolicy.read(other, invoice)).toBe(false));
  it('allows admin cross-org', () => expect(invoicePolicy.read(admin, invoice)).toBe(true));
});
```

---

## Integration Tests — IDOR Matrix

For every resource endpoint, test the full ownership matrix with real HTTP calls.

```ts
describe('GET /invoices/:id', () => {
  let ownerToken: string, otherToken: string, adminToken: string;
  let invoiceId: string;

  beforeAll(async () => {
    ownerToken = await loginAs('owner@test.com');
    otherToken = await loginAs('other@test.com');
    adminToken = await loginAs('admin@test.com');
    invoiceId  = await createInvoice(ownerToken);
  });

  it('200 for owner',       () => request(app).get(`/invoices/${invoiceId}`).set('Authorization', `Bearer ${ownerToken}`).expect(200));
  it('404 for other user',  () => request(app).get(`/invoices/${invoiceId}`).set('Authorization', `Bearer ${otherToken}`).expect(404));
  it('200 for admin',       () => request(app).get(`/invoices/${invoiceId}`).set('Authorization', `Bearer ${adminToken}`).expect(200));
  it('401 for unauthenticated', () => request(app).get(`/invoices/${invoiceId}`).expect(401));
});
```

**Test matrix to cover for every resource:**
- Owner → 200
- Other authenticated user (same role) → 404
- Admin → 200 (or 404 if admin shouldn't have access either)
- Unauthenticated → 401

---

## Manual / Exploratory Testing

### IDOR via ID manipulation
1. Log in as User A. Create or find a resource. Note its ID.
2. Log in as User B (different account, same role).
3. Directly request `GET /resource/<User A's ID>`.
4. Expect 404 (or 403). If you get 200 — IDOR.

### Horizontal privilege escalation
1. Log in as User A. Note your `user_id`.
2. Send `PUT /users/<User B's ID>` with User A's token.
3. Expect 404. If 200 — IDOR on write path.

### Vertical privilege escalation
1. Log in as a regular user.
2. Call an admin endpoint: `GET /admin/users`, `DELETE /admin/users/:id`.
3. Expect 403. If 200 — missing role check.

### Role via request body
1. Log in as a regular user.
2. Send `PUT /users/me` with `{ "role": "admin" }`.
3. Fetch your profile. If role changed — mass assignment vulnerability.

### HTTP method bypass
1. Identify a resource endpoint (e.g., `GET /reports/:id` with auth check).
2. Try `POST /reports/:id`, `HEAD /reports/:id`, `OPTIONS /reports/:id` without auth.
3. Any non-401 response on a gated path is worth investigating.

---

## Automated Scanning (Supplementary)

These tools help but don't replace manual testing — they miss business-logic IDOR:

- **OWASP ZAP** — active scan for missing auth headers
- **Burp Suite** — replay requests with different session tokens; compare responses
- **`nuclei`** — template-based; has auth-bypass templates

For CI: write the integration test matrix above and run it on every PR. Automated scanners won't catch "User A accessing User B's invoice" — only tests with two real accounts will.