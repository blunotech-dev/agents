---
name: api-documentation-sync
description: Keep API docs in sync with implementation by generating or validating OpenAPI/Swagger specs, detecting undocumented routes, and fixing spec drift. Use when the user asks to generate docs, validate a spec, find undocumented endpoints, or update stale API documentation.
category: "Backend"
---

# API Documentation Sync

Two modes: **Generate** (routes → spec) or **Validate** (spec ↔ routes). Identify which mode applies before proceeding.

---

## Phase 1: Discovery

### Route extraction — non-obvious patterns to catch

Most routes are not just `app.get(...)`. Scan for:

- **Router mounts**: `app.use('/prefix', router)` — the prefix compounds. Reconstruct full paths.
- **Dynamic registrations**: routes registered in loops, from config arrays, or via decorators — trace the source data.
- **Middleware-as-route**: auth middleware on `router.use(...)` that also terminates requests (e.g., returns 401) — these are implicit endpoints.
- **Nested routers**: routers mounted on routers. Build a full prefix chain.
- **Framework-specific**: NestJS `@Controller` + `@Get/@Post` decorators; FastAPI `APIRouter`; Django `urlpatterns` with `include()`; Hono `.route()`.

Collect for each route:
- Full path (with all prefix segments resolved)
- HTTP method(s)
- Handler function name/location
- Any inline validation schema (Zod, Pydantic, Joi, Yup, class-validator)
- Auth guard presence (middleware name, decorator)

### Schema extraction — non-obvious sources

Don't just look at request body. Extract:
- **Path params**: named captures in route strings (`:id`, `{id}`, `<int:pk>`)
- **Query params**: `req.query.*` accesses or framework query decorators
- **Response shapes**: infer from `res.json(...)` calls or explicit return type annotations
- **Validation schemas**: Zod `.parse()`/`.safeParse()`, Pydantic model in function signature, Joi `.validate()` — these are ground truth for request shape
- **Error responses**: catch blocks that `res.status(4xx).json(...)` — document these as response variants

---

## Phase 2: Strategy

### Generate mode

1. Confirm OpenAPI version target (default: 3.1.0 unless codebase shows 3.0.x or Swagger 2.0)
2. Determine output format: YAML (preferred for readability) or JSON
3. Ask if existing partial spec exists to merge into — don't overwrite human-authored descriptions
4. Identify reusable schemas (same shape used across multiple routes → `$ref` to `components/schemas`)

### Validate mode

1. Parse the existing spec — note the `servers[].url` base path, it affects path matching
2. Extract all `paths` entries from spec
3. Cross-reference against discovered routes using normalized path comparison:
   - OpenAPI uses `{param}` notation; Express uses `:param` — normalize before comparing
   - Ignore trailing slashes unless the framework distinguishes them
4. Classify findings into three buckets:
   - **Ghost routes**: in spec, not in code (deleted or renamed)
   - **Undocumented routes**: in code, not in spec
   - **Drift**: route exists in both but method, params, or response schema diverge

---

## Phase 3: Execution

### Generating the spec

Build the spec incrementally:

```yaml
openapi: 3.1.0
info:
  title: <infer from package.json name or ask>
  version: <infer from package.json version or "0.1.0">
paths:
  /resource/{id}:
    get:
      summary: <derive from handler name, e.g. getUserById → "Get user by ID">
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string   # or integer — infer from validation schema or usage
      responses:
        "200":
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        "404":
          description: Not found
```

**Non-obvious generation rules:**
- Handler name → `operationId`: camelCase handler name maps directly. `getUserById` → `operationId: getUserById`
- Auth middleware presence → add `security` field on that operation (use `BearerAuth` as default scheme name)
- If validation schema exists (Zod/Pydantic/Joi), derive the JSON Schema directly from it — don't guess
- For routes with no response type info, use `description: Success` with no schema rather than fabricating a shape
- Group routes under shared tags by their first path segment (`/users/*` → tag: `users`)

### Validating the spec

Output a structured diff report:

```
GHOST ROUTES (in spec, not in code):
  DELETE /api/v1/users/{id}  — handler not found

UNDOCUMENTED ROUTES (in code, not in spec):
  POST /api/v1/auth/refresh
  GET  /api/v1/admin/stats

DRIFT DETECTED:
  GET /api/v1/users/{id}
    spec: response 200 → schema $ref User
    code: handler returns { user, meta } — schema mismatch
  
  POST /api/v1/orders
    spec: body requires { productId, quantity }
    code: Zod schema requires { productId, quantity, shippingAddressId }
```

For each drift item, suggest the fix (update spec or update code) based on which is more likely to be ground truth.

---

## Phase 4: Output

### Generate mode output
- Single `openapi.yaml` (or `.json`) at project root or `docs/` — ask if no convention is visible
- If a partial spec existed, merge: preserve existing `description`, `summary`, `example` fields; update `parameters`, `requestBody`, `responses` from code

### Validate mode output
- Report in the format above, grouped by severity
- Offer to: (a) auto-add undocumented routes to spec as stubs, (b) remove ghost routes, (c) flag drift items for manual review
- Never silently mutate the spec — always confirm before writing

---

## Edge cases to handle

- **Versioned APIs** (`/v1/`, `/v2/`): treat as separate tag groups, not separate specs unless user requests it
- **File upload routes**: `multipart/form-data` content type — set explicitly, not `application/json`
- **Streaming endpoints** (SSE, chunked): note in spec as `text/event-stream`, flag that schema coverage is limited
- **Health/internal routes** (`/health`, `/metrics`, `/__internal`): ask if these should be excluded from public spec
- **Optional auth**: some routes accept both authenticated and unauthenticated requests with different responses — document both response variants