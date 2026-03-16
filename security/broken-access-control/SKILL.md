---
name: broken-access-control
description: Audit endpoints for missing or bypassable authorization checks. Use when the user asks about access control, route protection, ownership checks, IDOR, privilege escalation, or whether one user can access another user’s data.
category: "Security"
---

# Broken Access Control

Auditing and fixing authorization gaps — ensuring every route checks *who* can access *which specific resource*, not just whether the caller is logged in.

## The Core Distinction

```
Authentication → "Are you logged in?"         (identity)
Authorization  → "Can you access THIS record?" (permission)
```

Most auth middleware only handles the first. The second must be enforced per-endpoint, per-resource.

---

## Vulnerability Patterns

### 1. IDOR (Insecure Direct Object Reference)
User supplies an ID; server fetches it without checking ownership.

```ts
// ❌ Authenticated but not authorized
app.get('/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findById(req.params.id);
  res.json(invoice); // any logged-in user gets any invoice
});

// ✅ Ownership check
app.get('/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await db.invoices.findOne({
    id: req.params.id,
    owner_id: req.user.id  // scope to caller
  });
  if (!invoice) return res.status(404).send(); // 404, not 403 — don't confirm existence
});
```

**Rule:** Never fetch by ID alone. Always include `owner_id = req.user.id` (or equivalent) in the query.

---

### 2. Missing Role Check
Endpoint exists, auth required, but no role/permission gate.

```ts
// ❌ Any authenticated user can delete users
app.delete('/admin/users/:id', requireAuth, deleteUser);

// ✅
app.delete('/admin/users/:id', requireAuth, requireRole('admin'), deleteUser);
```

---

### 3. Privilege Escalation via User-Supplied Role
User sends their own role in the request body and it gets written to the DB.

```ts
// ❌
app.put('/users/:id', requireAuth, async (req, res) => {
  await db.users.update(req.params.id, req.body); // role, plan, credits — all writable
});

// ✅ Explicit field allowlist
app.put('/users/:id', requireAuth, async (req, res) => {
  const { name, email, bio } = req.body; // only safe fields
  await db.users.update(req.params.id, { name, email, bio });
});
```

---

### 4. Bypassing via HTTP Method
Authorization check tied to a specific method; same route accessible via another.

```ts
// ❌ Only GET is gated; POST to same path skips the check
router.get('/reports/:id', requireAuth, getReport);
router.post('/reports/:id', processReport); // forgot auth here
```

Check every verb on a route independently.

---

### 5. Path Traversal / Parameter Manipulation
ID in URL vs. ID in JWT disagree; server trusts the URL.

```ts
// ❌ User can PUT /users/999 even if their token says id=1
app.put('/users/:id', requireAuth, async (req, res) => {
  await db.users.update(req.params.id, req.body);
});

// ✅ Ignore URL param for self-edit; use token
app.put('/users/me', requireAuth, async (req, res) => {
  await db.users.update(req.user.id, sanitize(req.body));
});
```

---

### 6. Nested Resource Without Parent Check
Child resource ownership is checked, but parent isn't.

```ts
// ❌ Checks comment ownership but not post ownership
app.delete('/posts/:postId/comments/:commentId', requireAuth, async (req, res) => {
  const comment = await db.comments.findOne({ id: req.params.commentId, author_id: req.user.id });
  if (!comment) return res.status(404).send();
  await comment.delete();
});

// ✅ Verify the post belongs to the user too (if that's the intent)
const post = await db.posts.findOne({ id: req.params.postId, owner_id: req.user.id });
if (!post) return res.status(404).send();
```

---

### 7. Tenant Isolation (Multi-Tenant Apps)
Query scoped to user but not to tenant — user from Org A can read Org B's data.

```ts
// ❌ Scoped to user but not tenant
const records = await db.records.findAll({ user_id: req.user.id });

// ✅ Always include tenant scope
const records = await db.records.findAll({
  user_id: req.user.id,
  tenant_id: req.user.tenant_id  // from token, never from request body
});
```

---

## Authorization Models

Choose one and apply it consistently. See `references/models.md` for implementation detail.

| Model | Best for | Key concept |
|---|---|---|
| **RBAC** (Role-Based) | Clear user roles (admin, editor, viewer) | Role → permission set |
| **ABAC** (Attribute-Based) | Fine-grained, context-sensitive | Policy rules on resource + user attributes |
| **Ownership** | User-owned resources | `resource.owner_id === req.user.id` |
| **ReBAC** (Relation-Based) | Hierarchical resources, sharing | Graph of relationships (e.g. Google Zanzibar) |

Most apps need **Ownership + RBAC** together: ownership for user data, roles for admin surfaces.

---

## Audit Process

### Step 1 — Map Every Route
List all endpoints with their HTTP verb. Flag any with no `requireAuth` or equivalent middleware.

### Step 2 — Classify Each Route
For each authenticated route, determine what check is needed:

| Route type | Required check |
|---|---|
| User's own resource | `resource.owner_id === req.user.id` |
| Admin operation | `req.user.role === 'admin'` (or permission bit) |
| Shared/org resource | `resource.org_id === req.user.org_id` |
| Public read | No auth needed — confirm intentional |

### Step 3 — Check the Query
For every DB fetch: does the WHERE clause include the ownership/tenant scope, or does it fetch by ID alone then check in application code?

```ts
// Risky — fetch then check (TOCTOU, extra DB round-trip, easy to forget)
const item = await db.findById(id);
if (item.owner !== req.user.id) return res.status(403);

// Better — let the DB enforce it
const item = await db.findOne({ id, owner_id: req.user.id });
if (!item) return res.status(404);
```

### Step 4 — Check Write Paths
Every `POST / PUT / PATCH / DELETE`: are fields allowlisted? Can a user escalate their own role, balance, or plan via this endpoint?

### Step 5 — Check Indirect References
Are there endpoints that accept email, username, or slug instead of ID? Apply the same ownership check — just via a different lookup key.

---

## Response Codes

| Situation | Code | Reason |
|---|---|---|
| Not logged in | 401 | Client should re-authenticate |
| Logged in, wrong permissions | 403 | Client is authenticated but not authorized |
| Resource exists but caller shouldn't know | 404 | Don't leak existence of records |

Use **404** (not 403) when confirming a record's existence would itself be a data leak (e.g., other users' private invoices).

---

## Audit Checklist

- [ ] Every route has `requireAuth` (or is explicitly marked public)
- [ ] Every resource fetch includes ownership/tenant scope in the query — not just auth check
- [ ] Admin/privileged routes have role/permission checks beyond just `requireAuth`
- [ ] Write endpoints allowlist accepted fields — no `req.body` passed directly to ORM
- [ ] No user-supplied `role`, `plan`, `credits`, or similar fields accepted on update
- [ ] Nested resources: parent ownership verified before child access
- [ ] Multi-tenant: `tenant_id` always in queries, sourced from token not request
- [ ] All HTTP verbs on a route are gated, not just GET
- [ ] URL params not trusted for self-referencing operations (use `req.user.id`)
- [ ] 403 vs 404 used correctly — sensitive resource existence not leaked

---

## Reference Files

- `references/models.md` — RBAC, ABAC, ownership middleware patterns with code
- `references/testing.md` — How to test for access control bugs (unit, integration, manual)