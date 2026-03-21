---
name: route-based-permissions
description: Implement route-level permission enforcement using guard composition and declarative route manifests, supporting nested and wildcard routes. Use when centralizing permissions, chaining guards, or enforcing access across dynamic or hierarchical routes.
category: "Fullstack"
---

# Route-Based Permissions

Covers structural patterns for route permission enforcement: centralized route manifests, guard composition, and nested route inheritance. Assumes basic role/permission primitives exist (`can()`, `requireRole`) — focuses on how to wire them across a route tree without scattering checks everywhere.

---

## Discovery

Before writing anything, answer:

1. **How many guards per route?** Single role check, or multiple independent conditions (auth + role + feature flag + subscription tier)?
2. **Route declaration style**: File-based (Next.js App Router, Remix) or config-based (React Router `createBrowserRouter`, Express router)?
3. **Nested inheritance**: Should child routes inherit parent permissions, or must each route declare its own?
4. **Dynamic segments**: Do permissions depend on the resource at `/:id` (ownership checks), or only on role?
5. **Backend**: Express/Fastify router, or Next.js API/middleware?

---

## Core Patterns

### 1. Route Manifest — Declare Permissions Centrally

**The trap**: scattering `requireRole('admin')` inline across 40 route definitions. When permissions change, you hunt through every file.

**Fix**: a single manifest maps routes to their permission requirements. Guards read from it.

```ts
// routeManifest.ts — single source of route permission declarations
import type { Permission } from '@your-org/permissions';

interface RouteRule {
  permission?: Permission;
  requireAuth?: boolean;
  redirectTo?: string;
}

export const ROUTE_MANIFEST: Record<string, RouteRule> = {
  '/admin':              { permission: 'user:manage', redirectTo: '/403' },
  '/admin/billing':      { permission: 'billing:manage', redirectTo: '/403' },
  '/posts/new':          { permission: 'post:write', redirectTo: '/403' },
  '/posts/:id/edit':     { permission: 'post:write', redirectTo: '/403' },
  '/dashboard':          { requireAuth: true },
};

// Matcher: find the most specific rule for a given path
export function getRuleForPath(pathname: string): RouteRule | null {
  // Exact match first
  if (ROUTE_MANIFEST[pathname]) return ROUTE_MANIFEST[pathname];

  // Parameterized match — replace segments with :param pattern
  const normalized = pathname.replace(/\/[a-f0-9-]{8,}|\d+/g, '/:id');
  return ROUTE_MANIFEST[normalized] ?? null;
}
```

**Non-obvious**: the manifest only works for static and parameterized routes. Resource-level checks (does this user own `/posts/123`?) still belong in the handler — the manifest can't express ownership.

---

### 2. Guard Composition — Chaining Independent Checks

When a route needs multiple independent conditions, don't nest them — compose them.

```ts
// Backend: compose guards as an array, run left-to-right
type Guard = (req: Request, res: Response, next: NextFunction) => void;

function composeGuards(...guards: Guard[]): Guard {
  return (req, res, next) => {
    const run = (index: number) => {
      if (index >= guards.length) return next(); // all passed
      guards[index](req, res, () => run(index + 1));
    };
    run(0);
  };
}

// Guards stay single-responsibility
const requireAuth      = guardFromRule({ requireAuth: true });
const requireEditor    = requireRole('editor', 'admin');
const requireFeatureX  = (req, res, next) =>
  req.user.features?.includes('feature_x') ? next() : res.status(403).json({ error: 'Feature not enabled' });

// Route declaration stays clean
router.post('/posts',
  composeGuards(requireAuth, requireEditor, requireFeatureX),
  createPost
);
```

```tsx
// Frontend: compose route guards as wrappers
function composeGuards(...Guards: React.ComponentType<{ children: React.ReactNode }>[]) {
  return function ComposedGuard({ children }: { children: React.ReactNode }) {
    return Guards.reduceRight(
      (acc, Guard) => <Guard>{acc}</Guard>,
      children as React.ReactElement
    );
  };
}

const AdminEditorGuard = composeGuards(AuthGuard, RoleGuard('editor'), FeatureGuard('feature_x'));

<Route element={<AdminEditorGuard><Outlet /></AdminEditorGuard>}>
  <Route path="/posts/new" element={<NewPost />} />
</Route>
```

---

### 3. Nested Route Permission Inheritance

**The trap**: each child route re-declares the same parent permission. Easy to miss one.

```tsx
// React Router v6 — parent guard wraps all children via Outlet
// Children get the permission "for free" without declaring it
<Route element={<ProtectedRoute permission="user:manage" />}>
  <Route path="/admin"              element={<AdminDashboard />} />
  <Route path="/admin/users"        element={<UserList />} />
  <Route path="/admin/users/:id"    element={<UserDetail />} />
  {/* All three inherit the parent permission check */}
</Route>

// Stricter child — add a second guard for a subset of routes
<Route element={<ProtectedRoute permission="user:manage" />}>
  <Route path="/admin/users"        element={<UserList />} />
  <Route element={<ProtectedRoute permission="billing:manage" />}>
    <Route path="/admin/billing"    element={<Billing />} />
  </Route>
</Route>
```

```ts
// Express — apply parent guard to the router, not individual routes
const adminRouter = express.Router();
adminRouter.use(requireRole('admin')); // applies to all routes below

adminRouter.get('/users',       listUsers);
adminRouter.get('/users/:id',   getUser);
adminRouter.delete('/users/:id', deleteUser);

// Stricter sub-router
const billingRouter = express.Router();
billingRouter.use(requireRole('admin'), requireFeatureBilling);
billingRouter.get('/', getBilling);

adminRouter.use('/billing', billingRouter);
app.use('/admin', adminRouter);
```

**Non-obvious**: in Express, `router.use()` order matters. A guard added after a route definition won't protect that route.

---

### 4. Next.js Middleware — Wildcard and Prefix Matching

Next.js middleware runs on the Edge and must match routes via config `matcher`. Most teams under-specify this.

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import { getRuleForPath } from './routeManifest';
import { can } from '@your-org/permissions';
import { getSessionFromCookie } from './lib/session';

export async function middleware(req: NextRequest) {
  const rule = getRuleForPath(req.nextUrl.pathname);
  if (!rule) return NextResponse.next(); // no rule = public route

  const session = await getSessionFromCookie(req);

  if (rule.requireAuth && !session) {
    return NextResponse.redirect(new URL(`/login?next=${req.nextUrl.pathname}`, req.url));
  }

  if (rule.permission && !can(session?.role, rule.permission)) {
    return NextResponse.redirect(new URL(rule.redirectTo ?? '/403', req.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    // Non-obvious: exclude static assets and API routes that handle their own auth
    '/((?!_next/static|_next/image|favicon.ico|api/).*)',
  ],
};
```

**Non-obvious**: the `matcher` negative lookahead `(?!api/)` means middleware skips `/api/*` entirely. API routes must enforce their own permissions — middleware won't cover them.

---

### 5. Dynamic Segments — What the Manifest Can't Express

Route manifests handle role checks. Ownership checks (`/posts/:id` — does this user own post 123?) cannot be expressed in a manifest and must stay in the handler.

```ts
// Pattern: split enforcement explicitly in the handler
async function editPost(req: Request, res: Response) {
  // Layer 1: role check (could be middleware, but shown inline for clarity)
  if (!can(req.user.role, 'post:write')) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  // Layer 2: ownership check — requires the resource, can't be in middleware
  const post = await Post.findById(req.params.id);
  if (!post) return res.status(404).json({ error: 'Not found' });

  const isOwner = post.authorId === req.user.id;
  const isAdmin = req.user.role === 'admin';

  if (!isOwner && !isAdmin) {
    // Return 404, not 403 — don't confirm the resource exists to unauthorized users
    return res.status(404).json({ error: 'Not found' });
  }

  // Proceed
}
```

Keep the manifest for what it's good at (role/auth gates on known paths) and don't try to stretch it to express ownership or resource-level conditions.

---

## Output

Produce:
- `routeManifest.ts` — centralized route-to-permission map with path matcher
- `composeGuards.ts` — backend and frontend composition utilities
- Integration snippets showing nested route inheritance (React Router + Express router)

Flag clearly in comments:
- What the manifest can and cannot express (role vs ownership)
- Guard execution order in composition
- Next.js `matcher` exclusions and what they leave unprotected