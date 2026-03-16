# Stack Examples

## Node.js + Express + Prisma

### Prisma schema

```prisma
model Role {
  id          String           @id  // 'admin', 'editor', 'viewer'
  permissions RolePermission[]
  users       User[]
}

model Permission {
  id    String           @id  // 'invoices:write'
  roles RolePermission[]
}

model RolePermission {
  role_id       String
  permission_id String
  role          Role       @relation(fields: [role_id], references: [id])
  permission    Permission @relation(fields: [permission_id], references: [id])
  @@id([role_id, permission_id])
}

model User {
  id      String @id @default(cuid())
  email   String @unique
  role_id String @default("viewer")
  role    Role   @relation(fields: [role_id], references: [id])
}
```

### Seed permission matrix

```ts
await prisma.role.createMany({ data: [{ id: 'admin' }, { id: 'editor' }, { id: 'viewer' }] });
await prisma.permission.createMany({
  data: ['invoices:read','invoices:write','users:read','users:manage','reports:read']
    .map(id => ({ id }))
});

const matrix: Record<string, string[]> = {
  admin:  ['invoices:read','invoices:write','users:read','users:manage','reports:read'],
  editor: ['invoices:read','invoices:write','users:read','reports:read'],
  viewer: ['invoices:read','reports:read'],
};

for (const [role_id, perms] of Object.entries(matrix)) {
  await prisma.rolePermission.createMany({
    data: perms.map(permission_id => ({ role_id, permission_id }))
  });
}
```

### Auth middleware stack

```ts
// middleware/auth.ts
import jwt from 'jsonwebtoken';

export async function requireAuth(req, res, next) {
  const token = req.cookies.access_token;
  if (!token) return res.status(401).send();
  try {
    req.user = jwt.verify(token, process.env.JWT_PUBLIC_KEY!, { algorithms: ['RS256'] });
    next();
  } catch { res.status(401).send(); }
}

export const requireRole = (...roles: string[]) => (req, res, next) =>
  roles.includes(req.user.role) ? next() : res.status(403).send();

export const requirePermission = (perm: string) => (req, res, next) =>
  req.user.permissions?.includes(perm) ? next() : res.status(403).send();
```

### Token issuance with permissions

```ts
export async function issueAccessToken(userId: string): Promise<string> {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { role: { include: { permissions: true } } }
  });
  return jwt.sign(
    {
      sub: user.id,
      role: user.role_id,
      permissions: user.role.permissions.map(p => p.id),
    },
    process.env.JWT_PRIVATE_KEY!,
    { algorithm: 'RS256', expiresIn: '15m' }
  );
}
```

### Routes

```ts
const VALID_ROLES = ['admin', 'editor', 'viewer'] as const;

app.get('/invoices',     requireAuth, requirePermission('invoices:read'),   listInvoices);
app.post('/invoices',    requireAuth, requirePermission('invoices:write'),  createInvoice);
app.get('/users',        requireAuth, requirePermission('users:read'),      listUsers);
app.delete('/users/:id', requireAuth, requireRole('admin'),                 deleteUser);

app.put('/users/:id/role', requireAuth, requireRole('admin'), async (req, res) => {
  const { role } = req.body;
  if (!VALID_ROLES.includes(role)) return res.status(400).send('Invalid role');
  await prisma.user.update({ where: { id: req.params.id }, data: { role_id: role } });
  res.json({ ok: true });
});
```

---

## Python + FastAPI

```python
from fastapi import Depends, HTTPException, status
from jose import jwt, JWTError
from functools import wraps

def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    try:
        return jwt.decode(token, PUBLIC_KEY, algorithms=["RS256"])
    except JWTError:
        raise HTTPException(status_code=401)

def require_permission(permission: str):
    def dependency(user: dict = Depends(get_current_user)):
        if permission not in user.get("permissions", []):
            raise HTTPException(status_code=403)
        return user
    return dependency

def require_role(*roles: str):
    def dependency(user: dict = Depends(get_current_user)):
        if user.get("role") not in roles:
            raise HTTPException(status_code=403)
        return user
    return dependency

# Routes
@router.get("/invoices")
async def list_invoices(user=Depends(require_permission("invoices:read"))):
    return await db.invoices.find_many(where={"org_id": user["org_id"]})

@router.post("/invoices")
async def create_invoice(user=Depends(require_permission("invoices:write")), body=Body(...)):
    ...

@router.delete("/users/{user_id}")
async def delete_user(user_id: str, user=Depends(require_role("admin"))):
    ...

# Token issuance
async def issue_token(user_id: str) -> str:
    user = await db.users.find_unique(where={"id": user_id}, include={"role": {"include": {"permissions": True}}})
    permissions = [p.id for p in user.role.permissions]
    payload = {"sub": user.id, "role": user.role_id, "permissions": permissions}
    return jwt.encode(payload, PRIVATE_KEY, algorithm="RS256")
```