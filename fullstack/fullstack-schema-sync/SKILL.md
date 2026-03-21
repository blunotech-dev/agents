---
name: fullstack-schema-sync
description: Keep database schema, backend types, and frontend types in sync via codegen, migration hooks, and CI validation to prevent drift. Use when generating types from schemas, automating migrations, or enforcing schema/type consistency in CI.
category: "Fullstack"
---

# Fullstack Schema Sync

Covers the non-obvious parts of keeping schema changes propagated end-to-end: what triggers generation, what breaks silently when you skip it, and how to catch drift in CI before it ships. Skips ORM/schema basics — assumes a schema exists and types need to flow from it.

---

## Discovery

Before writing anything, answer:

1. **Stack**: Prisma + REST? Prisma + tRPC? Drizzle? Hasura? GraphQL with a schema file?
2. **API contract format**: OpenAPI spec, GraphQL SDL, tRPC router, or implicit (types shared directly from backend)?
3. **Frontend type consumption**: Generated client, manual fetch with cast types, or a shared types package?
4. **Monorepo or separate repos?** Determines whether types can be imported directly or must be published/fetched.
5. **Migration workflow**: Are migrations auto-generated (Prisma), hand-written (SQL), or both?

---

## Core Patterns

### 1. The Generation Chain — Know What Produces What

Every stack has a directed chain. Document it explicitly or someone will break it by running steps out of order.

```
Prisma stack:
  schema.prisma → [prisma generate] → Prisma Client (backend types)
                → [prisma-zod-generator] → Zod schemas
                                         → [z.infer<>] → Frontend-safe types

OpenAPI stack:
  DB schema → [backend decorators/manual spec] → openapi.json
           → [openapi-typescript] → api.d.ts (frontend)

GraphQL stack:
  schema.graphql → [graphql-codegen] → types.generated.ts (shared)
                 → resolvers type (backend)
                 → query result types (frontend)

tRPC stack:
  Zod input schemas + return types → AppRouter type
                                   → [inferred on frontend via RouterOutputs] → no generation needed
```

**Non-obvious**: tRPC is the only stack where the "generation" step is zero-cost — the frontend infers types from the router at compile time. All other stacks require an explicit generation step that can go stale.

---

### 2. Prisma — What `prisma generate` Does and Doesn't Do

```bash
# What you must run after any schema.prisma change:
npx prisma generate          # regenerates Prisma Client
npx prisma migrate dev       # creates + applies migration, then runs generate
npx prisma migrate deploy    # applies pending migrations in prod (does NOT run generate)
```

**Non-obvious**: `prisma migrate deploy` (used in production CI) does not run `prisma generate`. Your production container must run `generate` at build time, or the deployed Prisma Client won't match the applied schema.

```dockerfile
# Correct Dockerfile pattern
RUN npx prisma generate          # bake generated client into image
RUN npx prisma migrate deploy    # apply pending migrations at startup (or in entrypoint)
```

**Generating Zod schemas from Prisma** (removes hand-maintained validation duplication):

```ts
// schema.prisma
generator zod {
  provider = "zod-prisma-types"
  output   = "./generated/zod"
}

// After generate: use directly in API handlers and share with frontend
import { UserCreateInputSchema } from './generated/zod';

// Backend: validate request body
const body = UserCreateInputSchema.parse(req.body);

// Frontend: import the same schema for form validation — no duplication
import { UserCreateInputSchema } from '@your-org/schemas';
```

---

### 3. OpenAPI → Frontend Types Pipeline

```bash
# One-time setup
npm install -D openapi-typescript

# Generate types from a live server or local spec file
npx openapi-typescript http://localhost:3000/api-spec.json -o src/types/api.d.ts
npx openapi-typescript ./openapi.yaml -o src/types/api.d.ts
```

```ts
// Consuming generated types — non-obvious: use `paths` not `components` for request/response shapes
import type { paths } from './types/api.d.ts';

type GetUserResponse = paths['/users/{id}']['get']['responses']['200']['content']['application/json'];
type CreateUserBody  = paths['/users']['post']['requestBody']['content']['application/json'];

// Type-safe fetch wrapper
async function getUser(id: string): Promise<GetUserResponse> {
  const res = await fetch(`/api/users/${id}`);
  return res.json(); // cast is safe because types came from the spec
}
```

**Non-obvious**: if the spec is generated from backend decorators (NestJS `@ApiProperty`, FastAPI automatic spec), the spec file itself is a build artifact. Frontend generation must run after the backend builds — enforce this order in CI.

---

### 4. GraphQL Codegen

```yaml
# codegen.ts / codegen.yml
schema: "http://localhost:4000/graphql"   # or ./schema.graphql for offline
documents: "src/**/*.graphql"             # frontend query files
generates:
  src/generated/graphql.ts:
    plugins:
      - typescript
      - typescript-operations           # types for each query/mutation
      - typescript-react-query          # optional: typed hooks
    config:
      strictScalars: true               # fail on unmapped custom scalars
      scalars:
        DateTime: string                # map custom scalars explicitly
```

```bash
npx graphql-codegen --config codegen.ts
```

**Non-obvious**: codegen against a live schema URL requires the server to be running. In CI, generate against the SDL file instead, and add a step that validates the SDL file matches the running server:

```bash
# CI: dump current schema, diff against committed SDL
npx graphql-inspector introspect http://localhost:4000/graphql --write schema.graphql
git diff --exit-code schema.graphql   # fails if schema drifted without updating the file
```

---

### 5. Migration Triggers — What Must Run When

The most common source of drift is running the wrong subset of steps after a schema change.

```json
// package.json scripts — make the full chain a single command
{
  "scripts": {
    "db:change": "prisma migrate dev && prisma generate && npm run codegen",
    "db:deploy": "prisma migrate deploy",
    "codegen": "openapi-typescript ./openapi.yaml -o src/types/api.d.ts",
    "postinstall": "prisma generate"   // regenerate after npm install in any environment
  }
}
```

**`postinstall` for `prisma generate`**: when a teammate pulls changes and runs `npm install`, the Prisma Client regenerates automatically. Without this, the most common bug is "works on my machine" — the developer who made the schema change has the right client; everyone else doesn't.

**Git hooks — catch missing generation before commit:**

```bash
# .husky/pre-commit
#!/bin/sh
# Regenerate and fail if anything changed (means generate wasn't run)
npx prisma generate
git diff --exit-code src/generated/   # or wherever generated files live
```

**Non-obvious**: commit generated files to the repo. Treating them as gitignored means every CI run must regenerate from scratch and you can't diff what changed.

---

### 6. CI Validation — Catching Drift Before Production

Three checks that catch different failure modes:

```yaml
# .github/workflows/schema-sync.yml
jobs:
  schema-sync:
    steps:
      - uses: actions/checkout@v4
      - run: npm ci

      # Check 1: Prisma schema in sync with migrations
      # Fails if schema.prisma has changes not reflected in a migration file
      - run: npx prisma migrate diff
              --from-migrations ./prisma/migrations
              --to-schema-datamodel ./prisma/schema.prisma
              --exit-code

      # Check 2: Generated types are committed and up to date
      - run: npx prisma generate
      - run: git diff --exit-code src/generated/
             # Fails if generated files changed — developer forgot to run generate

      # Check 3: OpenAPI/GraphQL spec matches the implementation
      # (Start the server, dump the spec, compare to committed file)
      - run: npm run build && npm run start:ci &
      - run: sleep 5 && npx openapi-typescript http://localhost:3000/api-spec.json -o /tmp/api-check.d.ts
      - run: diff src/types/api.d.ts /tmp/api-check.d.ts
```

**Non-obvious**: Check 1 (`migrate diff`) catches the case where someone edited `schema.prisma` directly without running `migrate dev` — a common shortcut that creates schema drift without a migration file.

---

### 7. Monorepo vs Separate Repos

**Monorepo** — types flow via workspace imports, no publishing:

```ts
// packages/db/src/index.ts — exports generated types
export type { User, Post } from './generated/prisma';
export { UserCreateInputSchema } from './generated/zod';

// apps/frontend/src/api.ts
import type { User } from '@your-org/db';   // direct, always in sync
```

**Separate repos** — types must be published or fetched:

```bash
# Option A: publish generated types as an npm package on schema change
# Requires: version bump + publish in the backend CI on schema changes

# Option B: generate types from the live API spec at frontend build time
# In frontend CI:
npx openapi-typescript $API_URL/openapi.json -o src/types/api.d.ts
```

**Non-obvious for separate repos**: Option B (generate at frontend build time) means the frontend always has types that match the currently deployed backend — but a breaking backend change will break the frontend build, which is actually the desired behavior. It surfaces the contract break immediately rather than letting it ship silently.

---

## Output

Produce:
- `codegen` scripts wired into `package.json` with a single `db:change` command covering the full chain
- CI workflow file with all three drift-detection checks
- Comment in each generated file header: `// AUTO-GENERATED — do not edit. Run 'npm run db:change' to regenerate.`

Flag clearly in comments:
- Which files are safe to edit vs generated (and will be overwritten)
- The required execution order for the generation chain
- What each CI check catches and what it does not catch