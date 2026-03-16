---
name: rbac-implementation
description: Implement or audit role-based access control (RBAC). Use when the user asks about roles, permissions, admin restrictions, route protection, permission checks, or validating existing RBAC for gaps.
category: "Security"
---

# RBAC Implementation

End-to-end role-based access control: schema design, permission matrix, server enforcement, data-layer scoping, and common mistakes.

---

## Core Concepts

```
User → has Role(s) → Role has Permissions → Permission gates Resources + Actions
```

| Term | Example |
|---|---|
| Role | `admin`, `editor`, `viewer` |
| Permission | `invoices:write`, `users:delete` |
| Resource | `invoices`, `users`, `reports` |
| Action | `read`, `write`, `delete`, `publish` |

**Flat roles** (one role per user) — simple, works for most apps.  
**Permission-based** (roles map to permission sets) — flexible, query once, check many gates.  
**Hierarchical roles** — `admin` inherits `editor` inherits `viewer`. Avoid unless necessary; adds complexity.

---

## Schema Design

### Flat roles (simplest)
```sql
ALTER TABLE users ADD COLUMN role TEXT NOT NULL DEFAULT 'viewer'
  CHECK (role IN ('admin', 'editor', 'viewer'));
```

### Permission-based (recommended for non-trivial apps)
```sql
CREATE TABLE roles (
  id   TEXT PRIMARY KEY,  -- 'admin', 'editor', 'viewer'
  name TEXT NOT NULL
);

CREATE TABLE permissions (
  id     TEXT PRIMARY KEY,  -- 'invoices:write'
  label  TEXT NOT NULL
);

CREATE TABLE role_permissions (
  role_id       TEXT REFERENCES roles(id),
  permission_id TEXT REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);

ALTER TABLE users ADD COLUMN role_id TEXT REFERENCES roles(id) DEFAULT 'viewer';
```

### Permission matrix (define before coding)

|  | `invoices:read` | `invoices:write` | `users:read` | `users:manage` | `reports:read` |
|---|---|---|---|---|---|
| **admin**  | ✅ | ✅ | ✅ | ✅ | ✅ |
| **editor** | ✅ | ✅ | ✅ | ❌ | ✅ |
| **viewer** | ✅ | ❌ | ❌ | ❌ | ✅ |

Write this matrix before writing any middleware. It's the source of truth.

---

## Embedding Permissions in the JWT

Fetch permissions at login, encode in the token — avoids a DB hit on every request.

```ts
async function issueToken(user: User): Promise<string> {
  const permissions = await db.permissions
    .findMany({ where: { role_permissions: { some: { role_id: user.role_id } } } })
    .then(p => p.map(p => p.id));

  return jwt.sign(
    { sub: user.id, role: user.role_id, permissions },
    PRIVATE_KEY,
    { algorithm: 'RS256', expiresIn: '15m' }
  );
}
// Token payload: { sub, role: 'editor', permissions: ['invoices:read', 'invoices:write', ...] }
```

**Tradeoff:** permissions baked into short-lived tokens are slightly stale between refreshes (up to 15 min). Acceptable for most apps. For instant revocation needs, check DB on every request instead.

---

## Server-Side Enforcement

### Middleware

```ts
// requireAuth — validates JWT, attaches req.user
export function requireAuth(req, res, next) {
  const token = req.cookies.access_token || req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).send();
  try {
    req.user = jwt.verify(token, PUBLIC_KEY, { algorithms: ['RS256'] });
    next();
  } catch { res.status(401).send(); }
}

// requireRole — checks role string
export function requireRole(...roles: string[]) {
  return (req, res, next) =>
    roles.includes(req.user.role) ? next() : res.status(403).send();
}

// requirePermission — checks permission bit
export function requirePermission(permission: string) {
  return (req, res, next) =>
    req.user.permissions?.includes(permission) ? next() : res.status(403).send();
}
```

### Route-level enforcement

```ts
// Role gate
app.delete('/admin/users/:id',  requireAuth, requireRole('admin'), deleteUser);

// Permission gate (preferred — decoupled from specific role names)
app.post('/invoices',           requireAuth, requirePermission('invoices:write'), createInvoice);
app.get('/invoices',            requireAuth, requirePermission('invoices:read'),  listInvoices);
app.get('/reports',             requireAuth, requirePermission('reports:read'),   getReports);

// Multiple roles
app.get('/dashboard/analytics', requireAuth, requireRole('admin', 'analyst'), getAnalytics);
```

**Prefer permission checks over role checks in route handlers.** Role names change; permissions are stable. Exception: coarse admin-only gates where a role check is clearer.

---

## Data-Layer Enforcement

Route middleware stops unauthorized requests, but data-layer scoping ensures queries never return data the caller shouldn't see — a second line of defense.

```ts
// ❌ Middleware guards the route but DB fetches all rows
app.get('/invoices', requireAuth, requirePermission('invoices:read'), async (req, res) => {
  const invoices = await db.invoices.findMany(); // returns everyone's invoices
});

// ✅ Scope the query to the caller's org/ownership
app.get('/invoices', requireAuth, requirePermission('invoices:read'), async (req, res) => {
  const invoices = await db.invoices.findMany({
    where: {
      org_id: req.user.org_id,
      // admins see all; editors/viewers see only their own
      ...(req.user.role !== 'admin' && { owner_id: req.user.id }),
    }
  });
  res.json(invoices);
});
```

---

## Client-Side Checks: UI Only, Never Security

Client-side permission checks are **UI affordances** — hide buttons, disable routes. They are never a security boundary.

```ts
// React — fine for hiding UI elements
function InvoiceActions({ invoice }) {
  const { permissions } = useAuth();
  return (
    <>
      {permissions.includes('invoices:write') && <EditButton />}
      {permissions.includes('invoices:delete') && <DeleteButton />}
    </>
  );
}
```

The corresponding API endpoints must still have server-side `requirePermission` checks. A user can bypass the React UI trivially with curl.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Auth middleware on router, not individual routes | Explicitly gate each sensitive route |
| Role check but no resource ownership check | Combine with `owner_id` / `org_id` scope (see `broken-access-control`) |
| Permissions checked client-side only | Every permission check must exist server-side |
| Role stored in request body or JWT claims writable by client | Source role from DB or server-signed JWT only |
| Super-admin bypass that skips all checks | Admins should pass the same middleware; just have broader permissions |
| Permission matrix lives only in developer's head | Document in code or DB — see schema above |
| Flat `isAdmin` boolean instead of permission set | Grows unmanageable; use permission strings from the start |

---

## Role Assignment Safety

```ts
// ❌ User can escalate themselves
app.put('/users/me', requireAuth, async (req, res) => {
  await db.users.update(req.user.id, req.body); // role field writable
});

// ✅ Allowlist fields; role changes admin-only
app.put('/users/me', requireAuth, async (req, res) => {
  const { name, email } = req.body;
  await db.users.update(req.user.id, { name, email });
});

app.put('/users/:id/role', requireAuth, requireRole('admin'), async (req, res) => {
  const { role } = req.body;
  if (!VALID_ROLES.includes(role)) return res.status(400).send();
  await db.users.update(req.params.id, { role_id: role });
});
```

---

## Audit Checklist

- [ ] Permission matrix documented before implementation
- [ ] Every non-public route has `requireAuth` + `requireRole`/`requirePermission`
- [ ] Permissions embedded in JWT (or fetched from DB) — not derived from request
- [ ] Data queries scoped to org/owner, not just role-gated at route level
- [ ] Role assignment endpoint is admin-only; user profile update allowlists fields
- [ ] No permission logic lives only in the client
- [ ] Token refresh re-fetches permissions (role changes take effect within one token TTL)
- [ ] `VALID_ROLES` constant used to validate role values before writing to DB

---

## Reference Files

- `references/patterns.md` — Permission inheritance, multi-role users, dynamic permissions, ReBAC pointer
- `references/stack-examples.md` — Full RBAC setup for Node/Express + Prisma and Python/FastAPI