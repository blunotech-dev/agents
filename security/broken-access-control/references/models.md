# Authorization Models

## Ownership Middleware

The simplest pattern — attach the resource to `req` after verifying ownership, so the route handler never needs to re-check.

```ts
// middleware/ownResource.ts
export function ownResource(
  model: { findOne: Function },
  paramKey = 'id'
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const resource = await model.findOne({
      id: req.params[paramKey],
      owner_id: req.user.id,
    });
    if (!resource) return res.status(404).send();
    req.resource = resource; // attach for handler use
    next();
  };
}

// Usage
app.get('/invoices/:id',   requireAuth, ownResource(Invoice), getInvoice);
app.put('/invoices/:id',   requireAuth, ownResource(Invoice), updateInvoice);
app.delete('/invoices/:id',requireAuth, ownResource(Invoice), deleteInvoice);
```

---

## RBAC (Role-Based Access Control)

### Simple role check middleware

```ts
export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) return res.status(403).send();
    next();
  };
}

// Usage
app.delete('/admin/users/:id', requireAuth, requireRole('admin'), deleteUser);
app.get('/reports',            requireAuth, requireRole('admin', 'analyst'), getReports);
```

### Permission-based (more granular than roles)

```ts
// Store permissions as an array on the user/token
// e.g. req.user.permissions = ['invoices:read', 'invoices:write', 'users:read']

export function requirePermission(permission: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user.permissions?.includes(permission)) return res.status(403).send();
    next();
  };
}

app.post('/invoices', requireAuth, requirePermission('invoices:write'), createInvoice);
```

**Encode permissions in the JWT** so you avoid a DB lookup on every request:
```json
{ "sub": "user_123", "role": "editor", "permissions": ["invoices:read", "invoices:write"] }
```

---

## ABAC (Attribute-Based Access Control)

Policy functions that evaluate user + resource + context. Use when RBAC conditions get complex.

```ts
// policies/invoice.ts
export const invoicePolicy = {
  read(user: User, invoice: Invoice): boolean {
    return invoice.owner_id === user.id
      || user.role === 'admin'
      || (user.role === 'accountant' && invoice.org_id === user.org_id);
  },
  update(user: User, invoice: Invoice): boolean {
    return invoice.owner_id === user.id && invoice.status !== 'locked';
  },
  delete(user: User, invoice: Invoice): boolean {
    return user.role === 'admin';
  },
};

// In route handler
const invoice = await Invoice.findById(req.params.id);
if (!invoice || !invoicePolicy.read(req.user, invoice)) return res.status(404).send();
```

**When to use ABAC over RBAC:** when the decision depends on resource state (e.g., `status !== 'locked'`), time, or multiple attributes beyond just the user's role.

---

## Multi-Tenant Scoping

Add a `tenantScope` middleware that injects `tenant_id` into all queries automatically:

```ts
// Extend Prisma (or your ORM) with tenant context
export function tenantScope(req: Request) {
  return {
    where: { tenant_id: req.user.tenant_id }
  };
}

// Route handler — tenant_id automatically included
const invoices = await db.invoices.findMany({
  ...tenantScope(req),
  where: { owner_id: req.user.id },
});
```

**For ORMs with global middleware** (Prisma extensions, Sequelize scopes):
```ts
// Prisma: row-level tenant isolation via extension
const scopedDb = db.$extends({
  query: {
    $allModels: {
      async findMany({ args, query }) {
        args.where = { ...args.where, tenant_id: currentTenantId() };
        return query(args);
      }
    }
  }
});
```

**Never trust `tenant_id` from the request body.** Always source it from the JWT or session.