---
name: type-sharing
description: Set up shared TypeScript types between frontend and backend to prevent type drift. Use this skill when the user wants to share types across a fullstack TypeScript project, set up a shared types package in a monorepo, generate types from an API schema or database, or eliminate runtime type mismatches between client and server. Trigger on phrases like "shared types", "type drift", "frontend and backend out of sync", "generate types from schema", "zod + trpc", "prisma types", or any request to create a shared types layer in a fullstack app.
category: "Fullstack"
---

# Type Sharing

## Discovery

Before generating anything, ask (or infer from context):

1. **Architecture**: Monorepo (Turborepo/Nx/pnpm workspaces) or separate repos?
2. **Schema source of truth**: Database schema (Prisma), API spec (OpenAPI/GraphQL), or hand-written TypeScript?
3. **Stack**: What's the backend (Express, tRPC, Fastify, NestJS)? What's the frontend (Next.js, React, Vite)?
4. **Existing drift problem?** — Are types already duplicated, or is this greenfield?

---

## Strategy: Pick One Pattern

Choose based on answers above. Don't mix patterns.

| Situation | Pattern |
|---|---|
| Monorepo, schema-first (Prisma/OpenAPI) | **Generated types package** |
| Monorepo, hand-written types | **Shared `@repo/types` package** |
| Separate repos | **Published npm package + version pinning** |
| tRPC already in use | **Router inference — no separate package needed** |

---

## Non-Obvious Implementation Notes

### Pattern A: Shared `@repo/types` Package (Monorepo)

**The non-obvious parts:**

```
packages/types/
├── package.json        ← name: "@repo/types"
├── tsconfig.json       ← must extend root tsconfig
└── src/
    └── index.ts
```

`packages/types/package.json` — the critical field people miss:
```json
{
  "name": "@repo/types",
  "exports": {
    ".": {
      "types": "./src/index.ts",   // ← point to source, not dist
      "default": "./src/index.ts"  // ← avoids a build step entirely
    }
  }
}
```

Root `tsconfig.json` needs path aliases or the package won't resolve:
```json
{
  "compilerOptions": {
    "paths": {
      "@repo/types": ["./packages/types/src/index.ts"]
    }
  }
}
```

**Why source-first matters**: Skipping a build step eliminates the most common drift source — forgetting to rebuild the types package before running the app. Both consumer apps see live TypeScript source.

---

### Pattern B: Prisma → Generated Types (Schema-First)

Prisma exports its types but they're tied to `@prisma/client`. Don't re-export them directly into a shared package — the frontend will try to import server-only code.

**Correct approach — extract only what's safe:**

```ts
// packages/types/src/index.ts
import type { User, Post } from "@prisma/client"

// Re-export only plain data shapes (no Prisma methods, no PrismaClient)
export type { User, Post }

// For frontend-safe subsets, use Pick/Omit explicitly:
export type PublicUser = Pick<User, "id" | "name" | "email">
```

Add a postgenerate hook so types stay in sync automatically:
```json
// package.json (root)
{
  "scripts": {
    "db:generate": "prisma generate && tsc -p packages/types/tsconfig.json --noEmit"
  }
}
```

---

### Pattern C: OpenAPI → TypeScript (API-First)

Use `openapi-typescript` (not `swagger-typescript-api` — the output is harder to use):

```bash
npx openapi-typescript ./openapi.yaml -o packages/types/src/api.ts
```

The generated output uses a `paths` namespace — people usually want helper types:

```ts
// packages/types/src/helpers.ts
import type { paths } from "./api"

// Extract request/response types without manually indexing paths every time
export type ApiResponse<P extends keyof paths, M extends keyof paths[P]> =
  paths[P][M] extends { responses: { 200: { content: { "application/json": infer R } } } }
    ? R
    : never

// Usage: ApiResponse<"/users/{id}", "get">
```

---

### Pattern D: tRPC (Inference — No Package Needed)

If you're already on tRPC, a shared types package is redundant. The router IS the type contract.

```ts
// server: export the router type only
export type AppRouter = typeof appRouter

// client: import and use
import type { AppRouter } from "@repo/server"
const trpc = createTRPCProxyClient<AppRouter>(...)
```

**Non-obvious**: Export only `typeof appRouter`, never the router itself — keeping backend deps off the frontend.

---

### Pattern E: Zod as the Single Source of Truth

Define schemas once, derive types everywhere:

```ts
// packages/types/src/user.ts
import { z } from "zod"

export const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
})

export type User = z.infer<typeof UserSchema>

// Backend: use schema for validation
// Frontend: use schema for form validation + type inference
// Both: same source, zero drift
```

---

## Preventing Type Drift

### CI Check (most teams skip this, don't)

Add a typecheck step that spans all packages:

```json
// package.json (root)
{
  "scripts": {
    "typecheck": "tsc --build --force"
  }
}
```

Requires each package to have a proper `tsconfig.json` with `composite: true` and `references` set up.

### The `satisfies` Operator (TS 4.9+)

Catches shape mismatches that `as` masks:

```ts
// backend
const handler: RequestHandler = (req, res) => {
  const body = req.body satisfies CreateUserInput  // errors if shape changed
}
```

### Shared Enum Pitfall

Don't use TypeScript `enum` in shared packages — it compiles to an IIFE and adds runtime overhead on the frontend. Use `const` objects + type union instead:

```ts
// ❌ shared enum
export enum Role { Admin = "admin", User = "user" }

// ✅ shared const + union
export const Role = { Admin: "admin", User: "user" } as const
export type Role = typeof Role[keyof typeof Role]
```

---

## Output Checklist

- [ ] Single source of truth identified (Prisma / OpenAPI / Zod / hand-written)
- [ ] Types package or inference pattern set up
- [ ] No server-only imports leaking into the types package
- [ ] `typecheck` script runs across all packages in CI
- [ ] Shared enums use `const` + union, not TypeScript `enum`
- [ ] If schema-generated: postgenerate hook wired up