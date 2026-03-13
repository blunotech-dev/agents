---
name: rest-api-doc
description: Generate structured REST API documentation for a set of endpoints. Use when users want API reference docs, endpoint documentation, or descriptions of HTTP routes. Trigger when endpoints, route definitions, controller code, or HTTP methods like GET, POST, PUT, PATCH, or DELETE are provided.
category: "Documentation"
---

# REST API Documentation Generator

Produces clear, consistent, professional API reference documentation. Covers: HTTP method, path, description, authentication, request parameters (path, query, body), response schema, status codes, and usage examples.

---

## Workflow

1. **Gather input** — accept any of: raw endpoint list, code (routes/controllers), curl examples, OpenAPI YAML/JSON, or plain prose description.
2. **Infer missing details** — use naming conventions, HTTP verbs, and context to fill in reasonable defaults. Flag assumptions explicitly.
3. **Choose output format** — default is a clean Markdown file; see Format Options below.
4. **Generate documentation** — follow the Endpoint Block Template for every endpoint.
5. **Save to file** — always write output to `/mnt/user-data/outputs/api-docs.md` (or `.html` / `openapi.yaml` if requested) and present it with `present_files`.

---

## Format Options

| Format | When to use |
|--------|-------------|
| **Markdown** (default) | General purpose, README embeds, GitHub wikis |
| **HTML** | Standalone browsable page; use the HTML template in `references/html-template.md` |
| **OpenAPI 3.0 YAML** | Machine-readable spec; use when user says "OpenAPI", "Swagger", or needs tooling integration |

Ask the user if ambiguous; default to Markdown when not specified.

---

## Endpoint Block Template (Markdown)

Repeat this block for every endpoint. Keep the order and headings consistent.

````markdown
## `METHOD /path/to/endpoint`

**Summary**: One-sentence description of what this endpoint does.

**Auth**: `Bearer token` | `API Key (header: X-Api-Key)` | `None` *(pick one)*

---

### Path Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | string | ✅ | The resource identifier |

*(Omit section if none)*

### Query Parameters

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `page` | integer | ❌ | `1` | Page number for pagination |

*(Omit section if none)*

### Request Body

**Content-Type**: `application/json`

```json
{
  "name": "string",       // required
  "email": "string",      // required — must be valid email
  "role": "string"        // optional — "admin" | "user", default "user"
}
```

*(Omit section for GET/DELETE with no body)*

---

### Responses

| Status | Meaning | Schema |
|--------|---------|--------|
| `200 OK` | Success | See below |
| `400 Bad Request` | Validation error | `{ "error": "string" }` |
| `401 Unauthorized` | Missing/invalid auth | `{ "error": "string" }` |
| `404 Not Found` | Resource not found | `{ "error": "string" }` |
| `500 Internal Server Error` | Server error | `{ "error": "string" }` |

**200 Response Body**:
```json
{
  "id": "abc123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

---

### Example

**Request**:
```http
POST /users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

**Response**:
```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "abc123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```
````

---

## Document Structure (full API reference)

When documenting multiple endpoints, wrap them in this top-level structure:

```markdown
# API Reference

**Base URL**: `https://api.example.com/v1`  
**Auth**: All endpoints require `Authorization: Bearer <token>` unless marked `None`.  
**Rate Limiting**: 1000 req/min per token (include if known).

---

## Table of Contents

- [Endpoint 1](#method-path)
- [Endpoint 2](#method-path)

---

<!-- Endpoint blocks go here -->
```

---

## Inference Rules

When input is incomplete, apply these defaults before flagging gaps:

- **Auth**: If any endpoint in the set uses auth, assume all do unless clearly public.
- **Error responses**: Always include `400`, `401`, `404`, `500` unless method/context rules them out (e.g., no 404 on collection GETs).
- **Pagination**: If a GET returns a list, add `page` + `per_page` query params and a paginated response shape.
- **IDs**: Assume `string` (UUID-compatible) unless codebase signals integer.
- **Timestamps**: Use ISO 8601 (`"2024-01-15T10:30:00Z"`) in all examples.

Always add a **⚠️ Assumed** callout at the bottom of any endpoint block where you had to infer non-obvious details, so the user can verify.

---

## OpenAPI Output

When generating OpenAPI 3.0 YAML, follow the structure in `references/openapi-template.md`. Key rules:
- Use `$ref` components for reused schemas (e.g., `ErrorResponse`, pagination wrappers).
- Include `securitySchemes` at the top level.
- Validate mentally: every `$ref` must resolve, every path parameter must appear in the path string.

Read `references/openapi-template.md` when the user requests OpenAPI output.

---

## Quality Checklist

Before saving the file, verify:

- [ ] Every endpoint has: method, path, summary, auth, at least one response
- [ ] All path parameters documented
- [ ] Request body present for POST/PUT/PATCH
- [ ] At least one full HTTP request+response example per endpoint
- [ ] Status codes are realistic for the method (no 201 on GET, no 200 on DELETE if 204 is standard)
- [ ] Consistent naming: camelCase in JSON, kebab-case in paths
- [ ] Assumed details flagged with ⚠️

---

## Tips for Common Input Types

**User pastes route definitions (Express, FastAPI, Rails, etc.)**  
→ Extract method + path from decorator/router call. Infer request/response shape from handler logic or type hints if present.

**User pastes curl commands**  
→ Parse `-X METHOD`, URL, `-H` headers, `-d` body. Reverse-engineer the response schema from any example output provided.

**User describes endpoints in prose**  
→ Ask for a numbered list of endpoints before proceeding, to avoid ambiguity.

**OpenAPI/Swagger input**  
→ Parse and reformat into the Markdown block template for human-readable output, or pass through with improvements for OpenAPI output.