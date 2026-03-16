# RBAC Patterns

## Multi-Role Users

When a user can hold more than one role simultaneously (e.g., `editor` in Org A, `viewer` in Org B):

```sql
CREATE TABLE user_roles (
  user_id TEXT REFERENCES users(id),
  role_id TEXT REFERENCES roles(id),
  org_id  TEXT REFERENCES orgs(id),   -- scope role to an org (optional)
  PRIMARY KEY (user_id, role_id, org_id)
);
```

```ts
// At login, collect all permissions across all roles
const rows = await db.userRoles.findMany({
  where: { user_id: user.id, org_id: currentOrgId },
  include: { role: { include: { permissions: true } } }
});
const permissions = [...new Set(rows.flatMap(r => r.role.permissions.map(p => p.id)))];
```

Use org-scoped roles when the same user belongs to multiple orgs with different access levels (SaaS, workspace-based products).

---

## Permission Inheritance (Role Hierarchy)

Only add if your domain genuinely has a clear hierarchy. Implement with a `parent_role` column:

```sql
ALTER TABLE roles ADD COLUMN parent_id TEXT REFERENCES roles(id);
-- admin → editor → viewer
```

```ts
// Resolve effective permissions recursively (or flatten on write)
async function resolvePermissions(roleId: string): Promise<string[]> {
  const role = await db.roles.findUnique({
    where: { id: roleId },
    include: { permissions: true, parent: { include: { permissions: true } } }
  });
  const own = role.permissions.map(p => p.id);
  const inherited = role.parent ? await resolvePermissions(role.parent.id) : [];
  return [...new Set([...own, ...inherited])];
}
```

**Simpler alternative:** denormalize — when assigning a role, explicitly copy parent permissions into the child. Slower to update but trivially fast to query.

---

## Dynamic / Contextual Permissions

When access depends on resource state, not just role — use a policy function alongside RBAC:

```ts
// RBAC gate: does the user have the permission at all?
// Policy gate: does the user have permission on THIS resource?
export const invoicePolicy = {
  update(user: User, invoice: Invoice): boolean {
    if (!user.permissions.includes('invoices:write')) return false;
    if (invoice.status === 'locked') return false;     // state check
    if (user.role !== 'admin' && invoice.owner_id !== user.id) return false; // ownership
    return true;
  }
};

// Route handler
const invoice = await db.invoices.findById(req.params.id);
if (!invoice || !invoicePolicy.update(req.user, invoice)) return res.status(404).send();
```

This is lightweight ABAC layered on top of RBAC — use it when a pure role check isn't enough.

---

## Caching Permissions

For high-traffic apps that can't embed permissions in JWTs (e.g., instant revocation required):

```ts
const CACHE_TTL = 60; // seconds

async function getUserPermissions(userId: string): Promise<string[]> {
  const cacheKey = `perms:${userId}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const permissions = await fetchFromDB(userId);
  await redis.set(cacheKey, JSON.stringify(permissions), 'EX', CACHE_TTL);
  return permissions;
}

// On role change: bust the cache
async function assignRole(userId: string, roleId: string) {
  await db.users.update({ where: { id: userId }, data: { role_id: roleId } });
  await redis.del(`perms:${userId}`);
}
```

---

## When RBAC Isn't Enough → ReBAC

If your access model requires "User A can access Document X because they're a member of Team Y which has edit access to Folder Z" — you need relationship-based access control (ReBAC).

Options:
- **OpenFGA** — open-source Google Zanzibar implementation, self-hosted
- **Permify** — managed ReBAC service with an authorization API
- **SpiceDB** — Zanzibar-inspired, open-source

ReBAC is complex — only adopt it when RBAC + ownership genuinely can't express your model.