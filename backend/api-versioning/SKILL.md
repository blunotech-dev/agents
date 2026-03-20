---
name: api-versioning
description: Implement API versioning with URL or header strategies, deprecation handling, and backward compatibility. Use when the user asks about versioning, managing breaking changes, evolving APIs, supporting multiple versions, or handling v1/v2 migrations.
category: "Backend"
---

# API Versioning Skill

## Phase 1 — Discovery

Ask only what isn't obvious from context:

- **Strategy already chosen?** URL prefix (`/v1/`) vs. header (`Accept: application/vnd.api+json;version=2`) vs. query param — each has non-obvious tradeoffs (see below).
- **Who are the clients?** Internal services (fast migration possible), third-party developers (long deprecation windows), or SDKs you own (can force upgrades). This dictates sunset timelines.
- **Existing traffic?** If v1 is live, ask for any analytics on usage before deprecating. Never deprecate blind.
- **Monolith or microservices?** Microservices complicate versioning — each service may be on a different version cycle.

---

## Phase 2 — Strategy Selection

### Non-obvious tradeoffs

**URL versioning** (`/v1/users`)
- Easiest to route, log, and cache. Default choice for public APIs.
- Trap: teams version the entire API when only one resource changed. Version at the resource level if changes are isolated (`/v1/users`, `/v2/orders` simultaneously is valid).

**Header versioning** (`API-Version: 2` or `Accept` negotiation)
- Keeps URLs clean. Preferred for internal/RPC-style APIs.
- Trap: CDN and proxy caches often ignore custom headers. You must add `Vary: API-Version` or caching will silently serve wrong versions.
- Trap: Harder to test in a browser or with curl without extra flags. Raises support burden for third-party devs.

**Query param versioning** (`?version=2`)
- Only acceptable for read-only, public, non-cacheable endpoints (e.g., a public data API).
- Never use for mutation endpoints — params get dropped in logs, proxies, and form submissions.

---

## Phase 3 — Implementation

### Backward compatibility rules (non-obvious)

These are breaking changes most teams miss:

- **Adding a required field to a request** — breaking. Always optional with a default.
- **Narrowing an existing field's type** — breaking (e.g., `id` was `string | number`, now `string` only).
- **Changing error codes or error shapes** — breaking if clients pattern-match on them.
- **Reordering enum values** — breaking if clients use positional index.
- **Removing a field from a response** — breaking. Deprecate first; return `null` for one full version cycle before removing.
- **Changing pagination shape** (`page`/`limit` → cursor-based) — breaking even if field names stay the same.
- **Tightening validation** — breaking. If you previously accepted `age: -1`, rejecting it now is a breaking change for clients that stored and re-sent that value.

Safe non-breaking changes:
- Adding optional request fields
- Adding new fields to responses (clients should ignore unknown fields — enforce this in SDK design)
- Adding new enum values (warn clients to handle unknown enums gracefully)
- Relaxing validation

### Deprecation headers (exact spec)

Use standard headers so tooling (Postman, API gateways) picks them up automatically:

```http
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

- `Deprecation` header: RFC 8594 standard. Use `true` or an ISO date for when deprecation started.
- `Sunset` header: RFC 8594. Exact datetime in HTTP-date format. Must be honored — if you miss the sunset date, clients lose trust permanently.
- `Link` rel successor: Gives clients a machine-readable migration target. Include on every deprecated response, not just docs.

Add these at the router/middleware level, not per-handler. One missed handler means silent deprecation.

### Version routing patterns

**Gateway-level routing** (preferred):
Route at the API gateway or reverse proxy. Application code never sees the version prefix.

```
/v1/* → service:8001
/v2/* → service:8002  (or same service, different handler namespace)
```

Trap: Don't run two full copies of your service per version unless truly necessary. Instead, use a version flag in the request context and branch at the handler level for changed behavior only.

**Shared vs. forked handlers:**
- If < 20% of endpoints changed between versions: shared codebase with version branching
- If > 50% changed: fork the handler layer, share the service/data layer
- Never fork the database layer — version in the API, not the schema

### Sunset execution checklist

1. Add deprecation headers to all v_old endpoints (at least 3 months before sunset for third-party APIs; 2 weeks for internal).
2. Log every request hitting deprecated endpoints — export to a dashboard. Track unique client IDs/IPs still on old version.
3. Reach out directly to top-10 clients by traffic before sunset.
4. Return `410 Gone` (not `404`) after sunset. Include a `Link` header pointing to the migration guide.
5. Keep the route registered as a `410` tombstone for 6+ months. Removing it entirely causes cryptic errors for clients that cached DNS or hardcoded the path.

### Version negotiation (if using header versioning)

Always define a **default version** behavior — what happens when no version header is sent:

- **Return latest** — Bad. Silently breaks old clients as you release new versions.
- **Return oldest stable** — Safer default. Document it explicitly.
- **Return 400** with a clear error — Best for internal APIs. Forces explicit versioning from day one.

```json
{
  "error": "missing_version_header",
  "message": "Include 'API-Version' header. Current supported versions: [1, 2]. Default is not assumed.",
  "docs": "https://docs.example.com/versioning"
}
```

---

## Phase 4 — Output

Produce whichever of these the user needs:

- **Middleware snippet** — deprecation header injection (Express, FastAPI, or Go chi pattern)
- **Version routing config** — nginx, Kong, or AWS API Gateway route rules
- **Breaking change audit** — diff two OpenAPI specs and flag breaking changes
- **Deprecation notice template** — email/changelog copy for notifying API consumers
- **Migration guide skeleton** — v1 → v2 endpoint mapping table with change descriptions

Always include:
- The exact sunset date in HTTP-date format if generating headers
- A `410` handler stub if the sunset has passed or is imminent
- `Vary` header reminder if header versioning is chosen