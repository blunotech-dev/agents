---
name: crud-api-generator
description: Generates complete, production-ready CRUD (Create, Read, List, Update, Delete) API endpoints for any entity or resource. Use this skill whenever the user asks to scaffold API routes, generate REST endpoints, build a backend for a resource, or says things like "generate CRUD for X", "make API endpoints for X", "build routes for my X model", "scaffold a REST API", or "I need endpoints for X". Trigger even for partial requests like "add a create and delete endpoint" or "I need a list and get endpoint". Always use this skill when any combination of create/read/update/delete operations are needed for a data entity, even if the user doesn't say "CRUD" explicitly.
---

# CRUD API Generator

Generates complete, production-ready CRUD endpoints for any entity with proper HTTP status codes, input validation, error handling, and consistent response shapes.

## Workflow

### 1. Gather Entity Info

Before generating, identify:
- **Entity name** (e.g. `User`, `Product`, `Order`)
- **Fields** (names, types, required vs optional)
- **Framework/language** (Express, FastAPI, Django, Next.js API routes, Hono, etc.)
- **Database/ORM** (Prisma, Mongoose, SQLAlchemy, raw SQL, etc.)
- **Auth requirements** (public, JWT-protected, role-based)

If the user hasn't specified framework or DB, ask once — or make a reasonable default (Express + Prisma is a safe assumption for Node.js, FastAPI + SQLAlchemy for Python).

---

### 2. Generate the Five Endpoints

Always generate all five operations unless the user explicitly says to skip one:

| Operation | Method   | Path          | Success Code |
|-----------|----------|---------------|--------------|
| Create    | POST     | /entities     | 201          |
| Read      | GET      | /entities/:id | 200          |
| List      | GET      | /entities     | 200          |
| Update    | PUT/PATCH| /entities/:id | 200          |
| Delete    | DELETE   | /entities/:id | 204          |

---

### 3. Code Standards

**Validation**
- Validate required fields on Create and Update
- Return `400 Bad Request` with a descriptive message for missing/invalid input
- Use the framework's native validator if available (e.g. Zod, Pydantic, class-validator), otherwise inline checks

**Error Handling**
- `400` — validation error (missing/invalid fields)
- `404` — entity not found
- `409` — conflict (e.g. duplicate unique field)
- `500` — unexpected server error (always catch and return generic message, log internally)

**Response Shape**
Use consistent JSON envelopes:
```json
// Success
{ "data": { ...entity } }

// List
{ "data": [...entities], "total": 42 }

// Error
{ "error": "Human-readable message" }
```

**Naming**
- Route params: `:id` (or `{id}` for FastAPI/Django)
- Variables: camelCase for JS/TS, snake_case for Python
- File structure: suggest splitting into `router`, `controller`, and `service` layers for larger apps; inline for small examples

---

### 4. Framework-Specific Patterns

Read the relevant reference file before generating for these frameworks:

- **Express (JS/TS)**: See `references/express.md`
- **FastAPI (Python)**: See `references/fastapi.md`
- **Next.js App Router**: See `references/nextjs.md`
- **Django REST Framework**: See `references/django.md`
- **Hono**: Follow Express patterns with Hono's `c.json()` and `c.req.json()` syntax

For unlisted frameworks, apply the general standards above and match the framework's idiomatic style.

---

### 5. Output Structure

Always output in this order:

1. **Schema / Model definition** (Prisma schema, Pydantic model, Mongoose schema, etc.)
2. **Router / URL config**
3. **Controller / Handler functions** (all five)
4. **Validation logic** (inline or separate validator file)
5. **Usage examples** (curl or fetch snippets for each endpoint)

If the codebase is small, combine into one file with clear section comments. For larger apps, split and show the file path for each block.

---

### 6. Optional Extras (offer after main output)

- Pagination on the List endpoint (`?page=1&limit=20`)
- Soft delete (set `deletedAt` instead of hard delete)
- Search/filter params on List
- Swagger/OpenAPI annotation stubs
- Unit test stubs for each endpoint