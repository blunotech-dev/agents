---
name: idor-fix
description: Identify and fix Insecure Direct Object Reference (IDOR) vulnerabilities. Use this skill when the user mentions IDOR, sequential IDs, guessable IDs, object reference, users accessing other users' data, or asks "can a user access another user's resource?" or "are my IDs safe to expose?".
category: "Security"
---

# IDOR Fix

IDOR occurs when an ID in a URL/body maps directly to a DB record with no ownership check. An attacker increments `?invoice_id=1042` to `1043` and gets someone else's data.

---

## Two Independent Fixes (Both Required)

```
1. Opaque IDs     → make guessing futile
2. Ownership check → deny access even if ID is known
```

Neither alone is sufficient. Sequential IDs with ownership checks work. UUIDs without ownership checks don't.

---

## Fix 1: Use Non-Guessable IDs

| ID type | Guessable? | Recommendation |
|---|---|---|
| Auto-increment `1, 2, 3` | ✅ trivially | Replace |
| Short hash / base62 | ⚠️ low entropy | Replace |
| UUIDv4 | ❌ 122 bits entropy | Use this |
| ULID / UUIDv7 | ❌ + sortable | Use for time-ordered records |
| CUID2 | ❌ + collision-resistant | Good alternative |

```sql
-- Postgres: use gen_random_uuid() as default
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES users(id),
  ...
);
```

**Keep numeric PKs internally** if needed for joins/indexes — expose only the UUID externally.

---

## Fix 2: Ownership Check in Every Query

Never fetch by ID alone. Always scope to the authenticated caller.

```ts
// ❌ Authenticated but not authorized
const invoice = await db.invoices.findById(req.params.id);

// ✅ Ownership enforced at the DB layer
const invoice = await db.invoices.findOne({
  id: req.params.id,
  owner_id: req.user.id,   // from JWT — never from request body
});
if (!invoice) return res.status(404).send(); // 404, not 403
```

**Return 404, not 403.** Returning 403 confirms the resource exists — itself an information leak.

---

## Non-Obvious Patterns to Check

### Indirect references (not just `/:id`)
IDOR isn't only in URL params — check every place user input selects a record:

```ts
// Email/username lookup — same risk
const user = await db.users.findOne({ email: req.body.email });
// Fix: scope to org or confirm intent is public lookup

// File download by filename
res.sendFile(`/uploads/${req.query.filename}`); // path traversal + IDOR
// Fix: lookup file record by name scoped to owner, serve from that path
```

### Write paths (often missed)
```ts
// ❌ User can update any invoice by ID
await db.invoices.update({ id: req.body.invoice_id }, data);

// ✅
await db.invoices.update(
  { id: req.body.invoice_id, owner_id: req.user.id },
  data
);
// If 0 rows updated → 404
```

### Nested resources — check the parent too
```ts
// ❌ Checks comment owner but not post owner
const comment = await db.comments.findOne({ id: req.params.cid, author_id: req.user.id });

// ✅ Verify the parent belongs to caller (if that's the model)
const post = await db.posts.findOne({ id: req.params.pid, owner_id: req.user.id });
if (!post) return res.status(404).send();
const comment = await db.comments.findOne({ id: req.params.cid, post_id: post.id });
```

### Batch endpoints
```ts
// ❌ Deletes any IDs passed — no ownership check
await db.invoices.deleteMany({ id: { in: req.body.ids } });

// ✅ Scope batch operation to owner
await db.invoices.deleteMany({
  id: { in: req.body.ids },
  owner_id: req.user.id,
});
```

### Multi-tenant: user-scoped isn't enough
```ts
// ❌ Scoped to user but crosses tenant boundary
const records = await db.records.findMany({ owner_id: req.user.id });

// ✅ Always include tenant scope
const records = await db.records.findMany({
  owner_id: req.user.id,
  tenant_id: req.user.tenant_id, // from token, never from request
});
```

---

## Migrating Sequential IDs to UUIDs

```sql
-- 1. Add UUID column
ALTER TABLE invoices ADD COLUMN public_id UUID DEFAULT gen_random_uuid();
UPDATE invoices SET public_id = gen_random_uuid() WHERE public_id IS NULL;
ALTER TABLE invoices ALTER COLUMN public_id SET NOT NULL;
CREATE UNIQUE INDEX ON invoices(public_id);

-- 2. Keep internal id for FK integrity
-- 3. Expose only public_id in API responses
-- 4. Accept only public_id in API inputs — look up internal id server-side
```

```ts
// API layer translates: public_id → internal id
const invoice = await db.invoices.findOne({ public_id: req.params.id, owner_id: req.user.id });
```

---

## Audit Checklist

- [ ] All externally exposed IDs are non-sequential (UUID/ULID/CUID2)
- [ ] Every `findById` / `findOne` / `update` / `delete` includes `owner_id` or `tenant_id` scope
- [ ] Write paths (POST/PUT/PATCH/DELETE) ownership-checked, not just reads
- [ ] Nested resource routes verify parent ownership before child access
- [ ] Batch operations scoped to owner — not just array of IDs
- [ ] File/asset endpoints look up record by name+owner, never serve by raw path
- [ ] 404 returned for unauthorized access (not 403)
- [ ] `owner_id` / `tenant_id` sourced from JWT — never from request body or query param