---
name: data-transformation-layer
description: Implement a data transformation layer that maps DB models → API DTOs and API responses → UI models without leaking internal fields. Use when separating DB schema from API contracts, building mappers/serializers, or shaping frontend-specific data. Trigger on DTOs, response shaping, presenter patterns, or preventing internal field exposure.
category: "Fullstack"
---

# Data Transformation Layer

Covers the non-obvious parts of transformation: where mappers live, what they must never do, how to handle nested relations and optional fields safely, and how to keep frontend UI models separate from raw API shapes without over-engineering. Skips ORM setup — assumes data is fetched, focus is on shaping it.

---

## Discovery

Before writing anything, answer:

1. **How many layers are needed?** DB → API only (backend-rendered), or DB → API → UI model (fullstack with a frontend)?
2. **Sensitive fields**: Which DB fields must never reach the client (`password`, `internalScore`, `stripeCustomerId`)?
3. **Derived fields**: Does the UI need computed values not in the DB (e.g., `displayName`, `isExpired`, `formattedDate`)?
4. **Nested relations**: Does the API response include related objects (author, tags, permissions) that also need transforming?
5. **Nullability**: Does the DB schema allow nulls that the API or UI must normalize to defaults or omit?

---

## Core Patterns

### 1. The Layer Boundary — What Each Layer Owns

Define this once, enforce it everywhere:

```
DB Model       — raw Prisma/Drizzle output, internal fields included
     ↓ [toApiResponse()]
API DTO        — public contract, no internal fields, stable shape for consumers
     ↓ [toUIModel()]
UI Model       — frontend-specific, derived fields, display-ready values
```

**Non-obvious**: the UI model layer is often skipped, with components consuming raw API responses directly. This is fine until the API changes a field name and 15 components break, or until you need `displayName` computed in three different places.

**The rule**: nothing above a layer should import types from below it.

```ts
// WRONG — frontend importing a DB type
import type { User } from '@prisma/client';           // leaks DB shape to UI

// RIGHT — each layer owns its own type
import type { UserResponse } from '@your-org/api';    // API DTO only
import type { UserUIModel } from './models/user';     // UI model only
```

---

### 2. DB → API Mapper (Backend)

Explicit allowlist, not blocklist. Never use `{ ...dbRecord }` and then delete fields — you will forget one.

```ts
// types/api.ts — the public contract
export interface UserResponse {
  id:        string;
  email:     string;
  name:      string;
  role:      'admin' | 'editor' | 'viewer';
  createdAt: string;  // ISO string, not Date — JSON-safe
}

// mappers/user.ts
import type { User } from '@prisma/client';
import type { UserResponse } from '../types/api';

export function toUserResponse(user: User): UserResponse {
  return {
    id:        user.id,
    email:     user.email,
    name:      user.name ?? 'Unknown',      // normalize nulls at the boundary
    role:      user.role,
    createdAt: user.createdAt.toISOString(), // convert types at the boundary
  };
  // password, stripeCustomerId, internalScore — omitted by not including them
}

export function toUserResponseList(users: User[]): UserResponse[] {
  return users.map(toUserResponse);
}
```

**Non-obvious**: `Date` objects serialize to strings in JSON responses anyway — but the type is still `Date` unless you convert it. If TypeScript thinks `createdAt: Date` and the actual payload sends a string, consumers will get runtime surprises. Convert all non-JSON-primitive types at the mapper boundary.

---

### 3. Handling Nested Relations

Prisma includes relations as optional fields (`user.posts` is `Post[] | undefined` depending on the query). Mappers must handle both cases explicitly.

```ts
// DB type with optional relation
type UserWithPosts = User & { posts?: Post[] };

interface UserDetailResponse extends UserResponse {
  posts: PostSummaryResponse[];
}

// Non-obvious: check for undefined, don't assume the relation was included
export function toUserDetailResponse(user: UserWithPosts): UserDetailResponse {
  return {
    ...toUserResponse(user),
    posts: user.posts?.map(toPostSummaryResponse) ?? [],
  };
}

// Separate summary shape for nested context — not the same as the top-level PostResponse
interface PostSummaryResponse {
  id:    string;
  title: string;
  // No body, no author — avoids circular nesting and keeps payload small
}

export function toPostSummaryResponse(post: Post): PostSummaryResponse {
  return { id: post.id, title: post.title };
}
```

**Non-obvious**: nested objects often need a trimmed shape (summary vs detail) to avoid circular references and payload bloat. Define separate mapper functions for each context — `toPostResponse` (top-level) vs `toPostSummaryResponse` (when nested inside a user).

---

### 4. API → UI Model (Frontend)

The UI model adds derived, display-ready fields computed from the API response. Keep this transformation in one place, not scattered across components.

```ts
// models/user.ts — frontend only
import type { UserResponse } from '@your-org/api';

export interface UserUIModel {
  id:          string;
  email:       string;
  displayName: string;          // derived: name or email fallback
  role:        string;
  roleLabel:   string;          // derived: 'Administrator' from 'admin'
  createdAt:   Date;            // re-parsed to Date for local formatting
  isNew:       boolean;         // derived: createdAt within last 7 days
  initials:    string;          // derived: for avatar component
}

const ROLE_LABELS: Record<UserResponse['role'], string> = {
  admin:  'Administrator',
  editor: 'Editor',
  viewer: 'Viewer',
};

export function toUserUIModel(user: UserResponse): UserUIModel {
  const createdAt = new Date(user.createdAt);
  const name = user.name || user.email;

  return {
    id:          user.id,
    email:       user.email,
    displayName: name,
    role:        user.role,
    roleLabel:   ROLE_LABELS[user.role],
    createdAt,
    isNew:       Date.now() - createdAt.getTime() < 7 * 24 * 60 * 60 * 1000,
    initials:    name.split(' ').map(p => p[0]).join('').toUpperCase().slice(0, 2),
  };
}
```

**Non-obvious**: re-parse `createdAt` back to a `Date` on the frontend. The API sends a string (correct for JSON), but the UI model should hold a `Date` so components can use `Intl.DateTimeFormat` or date-fns without each one parsing strings.

---

### 5. Where Mappers Live and When They Run

**Backend**: call mappers in the controller/handler, not in the service or repository. The service returns DB types; the handler is responsible for shaping the response.

```ts
// WRONG — service knows about API shape
async function getUserService(id: string): Promise<UserResponse> {
  const user = await db.user.findUniqueOrThrow({ where: { id } });
  return toUserResponse(user); // service shouldn't know about response shape
}

// RIGHT — service returns DB type, handler maps
async function getUserService(id: string): Promise<User> {
  return db.user.findUniqueOrThrow({ where: { id } });
}

async function getUserHandler(req: Request, res: Response) {
  const user = await getUserService(req.params.id);
  res.json(toUserResponse(user)); // mapping happens at the edge
}
```

**Frontend**: call `toUIModel` at the data-fetching boundary (query hook, store action), not inside components.

```ts
// useUser.ts
import { useQuery } from '@tanstack/react-query';
import { toUserUIModel } from './models/user';

export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: async () => {
      const res = await fetch(`/api/users/${id}`);
      const data: UserResponse = await res.json();
      return toUserUIModel(data); // transform once, components get UI model
    },
  });
}

// Component — never sees UserResponse, only UserUIModel
function UserCard() {
  const { data: user } = useUser(id);
  return <Avatar initials={user.initials} label={user.roleLabel} />;
}
```

---

### 6. Partial Updates and PATCH Responses

**The trap**: a PATCH endpoint returns the full updated object — mapped through the same `toUserResponse`. Don't write a separate mapper for each endpoint.

```ts
// One mapper handles full and partial — return type stays consistent
router.patch('/users/:id', async (req, res) => {
  const updated = await db.user.update({
    where: { id: req.params.id },
    data: req.body,
  });
  res.json(toUserResponse(updated)); // same mapper, always returns full shape
});
```

**Non-obvious**: if you return partial shapes from PATCH (`{ name: 'New Name' }` only), the frontend cache gets a partial object that's missing fields — causing null reference errors or stale data. Always return the full mapped shape, even from updates.

---

### 7. Type-Safe Mapper Testing

Mappers are pure functions — test them without a running server or DB.

```ts
// mappers/user.test.ts
import { toUserResponse } from './user';

const dbUser = {
  id: 'u1',
  email: 'a@b.com',
  name: null,                        // test null normalization
  role: 'admin' as const,
  password: 'hashed',                // must not appear in output
  stripeCustomerId: 'cus_123',       // must not appear in output
  createdAt: new Date('2024-01-01'),
  updatedAt: new Date('2024-01-01'),
};

it('maps db user to response shape', () => {
  const result = toUserResponse(dbUser);

  expect(result.name).toBe('Unknown');           // null normalized
  expect(result.createdAt).toBe('2024-01-01T00:00:00.000Z');  // Date → ISO string
  expect(result).not.toHaveProperty('password'); // internal field excluded
  expect(result).not.toHaveProperty('stripeCustomerId');
});
```

**Non-obvious**: `expect(result).not.toHaveProperty('password')` is the critical assertion — it fails if you accidentally spread the whole DB record. Most mapper tests skip this, which means a future refactor can silently expose internal fields.

---

## Output

Produce:
- `mappers/[entity].ts` — DB → API mapper with explicit allowlist return, null normalization, and type conversion
- `models/[entity].ts` (frontend) — API → UI model with derived fields, re-parsed dates, display values
- `hooks/use[Entity].ts` — query hook that calls `toUIModel` at the fetch boundary so components never see raw API types
- `mappers/[entity].test.ts` — mapper unit tests including the `not.toHaveProperty` assertion for every sensitive field

Flag clearly in comments:
- Which fields are omitted and why (security vs noise)
- Which fields are derived and from what source
- Any null normalization decisions and what the fallback is