---
name: role-enforcement-fullstack
description: Implement end-to-end RBAC across frontend and backend using a shared permission model, avoiding duplicated logic. Use when adding route guards, UI gating, backend enforcement, or syncing role/permission checks. Trigger on RBAC, role-based access, admin-only routes, permission checks, or duplication between frontend and backend.
category: "Fullstack"
---

# Role Enforcement Fullstack

Covers the non-obvious parts of RBAC: keeping frontend and backend in sync without copy-pasting permission logic, and avoiding the traps that create false security or broken UX. Skips basic auth setup — assumes roles exist on the user object.

---

## Discovery

Before writing anything, answer:

1. **Permission model**: Flat roles (`admin`, `editor`) or hierarchical (`org:admin`, `project:viewer`)?
2. **Role source**: JWT claims, database lookup per request, or a session object?
3. **Shared code**: Monorepo (can share a permissions module) or separate repos (must duplicate or use a package)?
4. **Frontend framework**: React, Vue, Next.js? (affects where guards live — middleware file vs component wrapper)
5. **Backend**: Express, Fastify, Next.js API routes, tRPC? (affects middleware shape)
6. **Granularity needed**: Route-level only, or field-level (hide specific data fields by role)?

---

## Core Patterns

### 1. The Single Source of Truth Problem

**The trap**: defining roles and permissions separately on frontend and backend leads to drift. A backend says `editor` can delete; the frontend still shows the delete button for `viewer` because someone forgot to update.

**Fix: a shared permissions map**, ideally in a package both sides import.

```ts
// packages/permissions/index.ts  (imported by both client and server)
export const PERMISSIONS = {
  'post:read':   ['viewer', 'editor', 'admin'],
  'post:write':  ['editor', 'admin'],
  'post:delete': ['admin'],
  'user:manage': ['admin'],
} satisfies Record<string, string[]>;

export type Permission = keyof typeof PERMISSIONS;
export type Role = 'viewer' | 'editor' | 'admin';

export function can(role: Role, permission: Permission): boolean {
  return PERMISSIONS[permission].includes(role);
}
```

If you can't share code, generate the frontend map from the backend at build time (e.g., export as JSON from a `/api/permissions` endpoint that runs at build). Never hand-maintain two copies.

---

### 2. Backend Route Enforcement

**Non-obvious**: middleware runs on every request, but the role often needs context the middleware doesn't have (which resource? whose?). Separate route-level from resource-level checks.

#### Route-level middleware (coarse)

```ts
// Express/Fastify — runs before handler, rejects by role alone
function requireRole(...roles: Role[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role as Role;
    if (!userRole || !roles.includes(userRole)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// Usage
router.delete('/posts/:id', requireRole('admin', 'editor'), deletePost);
router.get('/admin/users',  requireRole('admin'), listUsers);
```

#### Resource-level checks (fine-grained, inside the handler)

```ts
// Non-obvious: route middleware can't know "is this the user's own post?"
// That check must happen inside the handler after fetching the resource.
async function deletePost(req: Request, res: Response) {
  const post = await Post.findById(req.params.id);
  if (!post) return res.status(404).json({ error: 'Not found' });

  const role = req.user.role as Role;
  const isOwner = post.authorId === req.user.id;

  // Admins can delete anything; editors can delete their own only
  if (!can(role, 'post:delete') && !(role === 'editor' && isOwner)) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  await post.delete();
  res.status(204).send();
}
```

**Never** skip the backend check because the frontend already hides the button. Frontend gates are UX; backend gates are security.

---

### 3. Frontend Route Guards

#### React Router v6

```tsx
// ProtectedRoute.tsx
import { can, type Permission, type Role } from '@your-org/permissions';
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from './useAuth';

interface Props {
  permission: Permission;
  redirectTo?: string;
}

export function ProtectedRoute({ permission, redirectTo = '/403' }: Props) {
  const { user } = useAuth();

  if (!user) return <Navigate to="/login" replace />;
  if (!can(user.role as Role, permission)) return <Navigate to={redirectTo} replace />;

  return <Outlet />;
}

// Router setup
<Route element={<ProtectedRoute permission="user:manage" />}>
  <Route path="/admin/users" element={<UserAdmin />} />
</Route>
```

#### Next.js App Router (middleware.ts)

```ts
// middleware.ts — runs on the edge before the page renders
import { NextResponse } from 'next/server';
import { can } from '@your-org/permissions';
import { getSessionFromCookie } from './lib/session';

const ROUTE_PERMISSIONS: Record<string, Permission> = {
  '/admin':        'user:manage',
  '/posts/new':    'post:write',
};

export async function middleware(req: NextRequest) {
  const session = await getSessionFromCookie(req);
  const matchedPermission = Object.entries(ROUTE_PERMISSIONS)
    .find(([path]) => req.nextUrl.pathname.startsWith(path))?.[1];

  if (matchedPermission && !can(session?.role, matchedPermission)) {
    return NextResponse.redirect(new URL('/403', req.url));
  }
}

export const config = { matcher: ['/admin/:path*', '/posts/new'] };
```

**Non-obvious**: Next.js middleware runs on the Edge runtime — it can't use Node.js APIs or hit a database. Role must be in the cookie/token, not fetched live.

---

### 4. UI Gating (Hiding Elements, Not Just Routes)

Hiding a button is UX only. But it still needs to be consistent with the same permission logic.

```tsx
// usePermission.ts
import { can, type Permission } from '@your-org/permissions';
import { useAuth } from './useAuth';

export function usePermission(permission: Permission): boolean {
  const { user } = useAuth();
  if (!user) return false;
  return can(user.role, permission);
}

// Usage in component
function PostActions({ post }: { post: Post }) {
  const canDelete = usePermission('post:delete');
  const canEdit   = usePermission('post:write');

  return (
    <div>
      {canEdit   && <button onClick={...}>Edit</button>}
      {canDelete && <button onClick={...}>Delete</button>}
    </div>
  );
}
```

**Non-obvious**: don't gate on role strings directly in components (`user.role === 'admin'`). This scatters role knowledge everywhere. Always go through `can()` so the permissions map is the only place to update.

---

### 5. JWT Claims vs Database Roles

| Approach | Tradeoff |
|---|---|
| Role in JWT | Fast (no DB hit), but stale until token expires |
| DB lookup per request | Always fresh, but adds latency + DB pressure |
| Hybrid: role in JWT, invalidation via cache | Best of both; invalidate on role change |

**Stale JWT role problem**: if you revoke admin access, the user keeps it until their token expires. Fix:

```ts
// On role change, write a revocation timestamp to cache (Redis/KV)
await cache.set(`role_changed:${userId}`, Date.now(), { ttl: TOKEN_TTL_SECONDS });

// In middleware, check if token was issued before the role change
function requireRole(...roles: Role[]) {
  return async (req, res, next) => {
    const changedAt = await cache.get(`role_changed:${req.user.id}`);
    if (changedAt && req.user.iat * 1000 < Number(changedAt)) {
      return res.status(401).json({ error: 'Token outdated, please re-login' });
    }
    // ... normal role check
  };
}
```

---

### 6. tRPC / GraphQL — Permission at the Resolver Level

REST middleware doesn't translate directly. Each resolver/procedure must check.

```ts
// tRPC
const adminProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (!can(ctx.user.role, 'user:manage')) {
    throw new TRPCError({ code: 'FORBIDDEN' });
  }
  return next({ ctx });
});

export const userRouter = router({
  list:   adminProcedure.query(() => db.user.findMany()),
  delete: adminProcedure.mutation(({ input }) => db.user.delete({ where: { id: input.id } })),
});
```

```ts
// GraphQL — field-level visibility by role
const PostType = new GraphQLObjectType({
  fields: {
    title:      { type: GraphQLString },
    // Non-obvious: resolvers can return null for unauthorized fields
    // instead of throwing — keeps the response shape intact
    internalId: {
      type: GraphQLString,
      resolve: (post, _, ctx) =>
        can(ctx.user?.role, 'post:delete') ? post.internalId : null,
    },
  },
});
```

---

### 7. 403 vs 404 — Which to Return

**Non-obvious**: returning 403 on a resource the user shouldn't know exists leaks information. Return 404 instead.

```ts
async function getPost(req, res) {
  const post = await Post.findById(req.params.id);

  // Don't 403 here — reveals that the post exists
  if (!post || !can(req.user.role, 'post:read')) {
    return res.status(404).json({ error: 'Not found' });
  }
  res.json(post);
}
```

Reserve 403 for routes where the existence of the resource is already known (e.g., the user navigates to `/admin` which is obviously an admin section).

---

## Output

Produce:
- `permissions.ts` — shared `PERMISSIONS` map and `can()` function (framework-agnostic, importable by both client and server)
- `requireRole.ts` — backend middleware using `can()`
- `ProtectedRoute.tsx` / `middleware.ts` — frontend guard using the same `can()`
- `usePermission.ts` — hook for UI gating

Each file gets a comment header noting: what it enforces, what it deliberately does NOT enforce (security boundary vs UX boundary), and where the single source of truth lives.

Flag clearly in comments:
- Any check that is UX-only (not a security boundary)
- Where stale-role risk exists if using JWT claims
- Any Edge runtime constraints (Next.js middleware)