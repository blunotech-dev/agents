---
name: response-shape-mismatch
description: Audit frontend API consumption against actual backend responses to detect mismatches such as assumed-but-missing fields, incorrect types, and unhandled null/undefined values. Use when fixing runtime crashes (e.g., “cannot read property of undefined”), validating response shapes, or hardening data handling. Trigger on signs like API mismatch, undefined fields at runtime, differing backend shapes, excessive optional chaining, or unstable UI due to inconsistent API data.
category: "Fullstack"
---

# Response Shape Mismatch

## Discovery

Infer as much as possible from the codebase before asking. Then confirm:

1. **Where are responses consumed?** — fetch/axios calls, React Query hooks, SWR, RTK Query, or tRPC?
2. **Are response types declared?** — hand-written interfaces, generated from OpenAPI, inferred from tRPC, or `any`/untyped?
3. **Is the backend accessible?** — can we read route handlers, Prisma schema, or serializers directly, or only the frontend?
4. **What's the failure mode?** — known crash, silent wrong data, or proactive audit before it breaks?

---

## Audit Strategy

Run all four checks. Each catches a distinct failure class.

| Check | What it catches |
|---|---|
| **Structural diff** | Fields frontend assumes that backend never sends |
| **Nullability gap** | Fields typed as `T` but backend returns `T \| null` |
| **Optional chaining audit** | `?.` masking real errors vs. `!` causing crashes |
| **Serialization delta** | Shape in DB vs. shape after backend transforms it |

---

## Check 1: Structural Diff (Assumed-but-Missing Fields)

The most common source of silent bugs — the frontend types a field that exists in the DB but the backend never includes in the response.

**How to find it:**

Trace from the fetch call backward to the serializer/controller:

```ts
// Frontend assumes:
type OrderResponse = { id: string; user: { name: string; email: string }; items: Item[] }

// Backend actually sends (Express example):
res.json({ id: order.id, userId: order.userId, items: order.items })
// ↑ "user" is never populated — frontend gets undefined, not an error
```

**Pattern to look for:** Any nested object type on the frontend that maps to a foreign key (`userId`, `authorId`) on the backend model. The backend often returns the ID, not the hydrated relation.

**Fix:**

```ts
// Option A: explicitly select and include the relation in the query
const order = await prisma.order.findUnique({
  where: { id },
  include: { user: { select: { name: true, email: true } } }
})

// Option B: flatten the frontend type to match what's actually sent
type OrderResponse = { id: string; userId: string; items: Item[] }
```

---

## Check 2: Nullability Gap

TypeScript's strictest flaw for API work: a field typed as `string` is trusted at compile time, but the backend may return `null` — and TS never catches it because the cast happened at the fetch boundary.

**Where it hides:**

```ts
// The cast at the fetch boundary silently strips null from the type:
const data = await res.json() as UserResponse
// TS now believes data.bio is string — but Prisma's bio is String? (nullable)
```

**How to find it systematically:**

Compare the frontend interface against the Prisma schema (or DB schema) field by field:

```
Prisma:   bio       String?    → nullable
Frontend: bio       string     → assumed non-null ← GAP
```

If there's no Prisma schema, look at the backend serializer/controller for any field that conditionally exists:

```ts
// Backend
const response = {
  ...user,
  avatar: user.avatarUrl ?? null,  // ← null possible, check frontend
  lastLogin: user.sessions[0]?.createdAt  // ← undefined possible
}
```

**Fix — validate at the boundary, not after:**

```ts
import { z } from "zod"

const UserSchema = z.object({
  id: z.string(),
  bio: z.string().nullable(),   // matches backend reality
  avatar: z.string().url().nullable(),
})

// In the fetch hook:
const data = UserSchema.parse(await res.json())
// Now TypeScript knows bio is string | null — and enforces it everywhere
```

---

## Check 3: Optional Chaining Audit

`?.` is not always safe — it suppresses both "field doesn't exist" (real bug) and "field is intentionally optional" (correct). Audit which is which.

**Dangerous pattern — `?.` hiding a structural mismatch:**

```ts
// If user.address is always present per the API contract,
// this silently renders nothing instead of throwing:
<p>{user.address?.city}</p>
// A backend change that stops sending `address` becomes invisible
```

**Dangerous pattern — non-null assertion on API data:**

```ts
// This crashes if backend ever returns null/undefined:
const name = user.profile!.displayName
```

**How to audit:**

Search for these patterns on any variable that originated from an API response:

- `?.` on fields typed as required — signals an unacknowledged mismatch
- `!` on fields from `json()` casts — guaranteed future crash
- `|| ""` / `?? ""` on fields used in logic (not just display) — often masks wrong type

**Fix — use discriminated unions for partial responses:**

```ts
// Instead of optional chaining on ambiguous shape:
type ApiUser =
  | { status: "complete"; profile: { displayName: string; avatar: string } }
  | { status: "pending"; profile: null }

// Now TS forces you to check status before accessing profile
```

---

## Check 4: Serialization Delta

The DB shape and the API response shape are often different. Middleware, serializers, `toJSON()` overrides, and ORM transforms all mutate data between DB and wire.

**Common deltas to check:**

| Transform | What changes |
|---|---|
| `JSON.stringify` on `Date` | `Date` object → ISO string — frontend types it as `string` but often forgets to parse back |
| Prisma `select` | Only selected fields exist — `include`d relations absent unless explicitly selected |
| `class-transformer` / NestJS `@Exclude()` | Fields present in the class but stripped from response |
| Express middleware (e.g. `camelCase` transform) | `snake_case` DB fields renamed — types must match the transformed name |

**Date pitfall (extremely common):**

```ts
// Backend sends: { "createdAt": "2024-01-15T10:30:00.000Z" }
// Frontend type:   createdAt: Date   ← WRONG, it's a string after JSON parse

// Correct:
type Post = { createdAt: string }
// Parse explicitly where needed:
const date = new Date(post.createdAt)
```

**NestJS `@Exclude()` pitfall:**

```ts
@Entity()
class User {
  @Expose() id: string
  @Expose() email: string
  @Exclude() passwordHash: string  // stripped from response
}
// Frontend must not type passwordHash — it will always be undefined
```

---

## Hardening: Validate at the Fetch Boundary

The only permanent fix. All other checks are audits — this prevents regressions.

**With Zod (framework-agnostic):**

```ts
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`)
  const json = await res.json()
  return UserSchema.parse(json)  // throws with a clear error if shape is wrong
}
```

**With React Query — centralize in `queryFn`:**

```ts
const { data } = useQuery({
  queryKey: ["user", id],
  queryFn: async () => {
    const res = await fetch(`/api/users/${id}`)
    return UserSchema.parse(await res.json())
  }
})
// data is now User (not User | undefined in shape), null handling is explicit
```

**Safe parse for non-throwing validation (dev logging):**

```ts
const result = UserSchema.safeParse(json)
if (!result.success) {
  console.error("API shape mismatch:", result.error.flatten())
  // log to Sentry, show fallback UI, etc.
}
```

---

## Output Checklist

- [ ] All nested object types verified against backend serializer (not just DB schema)
- [ ] Nullable DB fields matched to `T | null` in frontend types
- [ ] `?.` on required fields replaced with explicit null checks or discriminated unions
- [ ] `!` assertions on API data eliminated
- [ ] `Date` fields typed as `string` on the wire, parsed explicitly where needed
- [ ] `@Exclude()` / `select`-omitted fields removed from frontend types
- [ ] Zod (or equivalent) parse added at fetch boundary for at-risk endpoints