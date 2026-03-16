---
name: cors-config
description: Audit and fix CORS configuration. Use this skill when the user mentions CORS, cross-origin requests, Access-Control headers, wildcard origins, preflight, OPTIONS requests, credentials with CORS, or gets a CORS error — including "how do I fix CORS?" or "is my CORS config safe?".
category: "Security"
---

# CORS Configuration

CORS is a browser enforcement mechanism. It doesn't protect your API from non-browser clients (curl, server-to-server) — it prevents *other websites* from making credentialed requests on behalf of your users.

A misconfigured CORS policy is a security boundary failure, not just a developer inconvenience.

---

## The Critical Mistake: Credentials + Wildcard

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

Browsers reject this combination — but some servers approximate it unsafely:

```ts
// ❌ Reflecting the request Origin without validation
res.setHeader('Access-Control-Allow-Origin', req.headers.origin); // any origin allowed
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

This is equivalent to `*` with credentials — any site can make authenticated requests as your users. Cookies, session tokens, everything.

---

## Correct Pattern: Explicit Allowlist

```ts
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://admin.example.com',
  // 'http://localhost:3000', // dev only — keep out of production
]);

function corsMiddleware(req, res, next) {
  const origin = req.headers.origin;

  if (origin && ALLOWED_ORIGINS.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);  // exact match only
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Vary', 'Origin');  // required — tells caches this varies by origin
  }
  // No header set for disallowed origins — browser blocks the response

  if (req.method === 'OPTIONS') {
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Max-Age', '86400'); // cache preflight 24h
    return res.status(204).send();
  }

  next();
}
```

**`Vary: Origin` is required** when reflecting a dynamic origin. Without it, a CDN or proxy may cache a response with one origin's CORS headers and serve it to a different origin.

---

## Using the `cors` npm Package

```ts
import cors from 'cors';

const corsOptions: cors.CorsOptions = {
  origin(requestOrigin, callback) {
    // requestOrigin is undefined for same-origin and non-browser requests
    if (!requestOrigin || ALLOWED_ORIGINS.has(requestOrigin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
};

app.use(cors(corsOptions));
```

---

## Preflight (OPTIONS) Requests

Browsers send a preflight `OPTIONS` request before any "non-simple" request. A non-simple request is:
- Any method other than GET / POST / HEAD
- Any custom header (including `Authorization`, `Content-Type: application/json`)
- Any `Content-Type` other than form/text

```
Preflight:  OPTIONS /api/invoices
            Origin: https://app.example.com
            Access-Control-Request-Method: POST
            Access-Control-Request-Headers: Content-Type, Authorization

Response:   204 No Content
            Access-Control-Allow-Origin: https://app.example.com
            Access-Control-Allow-Methods: POST
            Access-Control-Allow-Headers: Content-Type, Authorization
            Access-Control-Max-Age: 86400
```

**Preflight must respond before auth middleware runs.** A common mistake: placing CORS middleware after `requireAuth`, causing preflight to return 401.

```ts
// ❌ Preflight hits auth before CORS headers are set
app.use(requireAuth);
app.use(cors(corsOptions));

// ✅ CORS first — always
app.use(cors(corsOptions));
app.use(requireAuth);
```

---

## Per-Route CORS (Public + Credentialed Mixed API)

When some routes are public (CDN-cacheable) and some require credentials:

```ts
const publicCors = cors({ origin: '*', credentials: false });
const privateCors = cors(corsOptions); // allowlist + credentials

app.get('/public/prices',  publicCors,  getPrices);   // open — wildcard fine here
app.get('/account/data',   privateCors, requireAuth, getAccountData);
```

Wildcard is only safe on routes that return no user-specific data and set no cookies.

---

## Non-Obvious Issues

| Issue | Why it matters | Fix |
|---|---|---|
| Reflecting `Origin` without validation | Every origin allowed — same as `*` with credentials | Check against allowlist before reflecting |
| Missing `Vary: Origin` | CDN serves wrong CORS headers to other origins | Always set when reflecting dynamic origin |
| `localhost` in production allowlist | Attacker running local server can make credentialed requests | Strip from production; use env-based config |
| Subdomain wildcard via regex mistake | `req.origin.endsWith('.example.com')` matches `evil.example.com` | Use exact Set lookup or validated regex |
| CORS on preflight only, not actual response | Browser checks both; missing headers on actual response causes failure | Apply same headers to all responses, not just OPTIONS |
| Trusting `Origin` for server-side auth | CORS is browser-only; server-to-server requests can forge `Origin` | Use API keys or mTLS for server auth — not Origin |

### Subdomain allowlist — do it right

```ts
// ❌ Too loose — matches attacker.myexample.com or evilexample.com
if (origin.endsWith('example.com')) { ... }

// ✅ Exact subdomain match
const SUBDOMAIN_RE = /^https:\/\/[a-z0-9-]+\.example\.com$/;
if (SUBDOMAIN_RE.test(origin)) { ... }

// ✅ Or just use an explicit Set — simpler and safer
```

---

## Python / FastAPI

```python
from fastapi.middleware.cors import CORSMiddleware

ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://admin.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,   # never ["*"] with allow_credentials=True
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    max_age=86400,
)
```

---

## Audit Checklist

- [ ] No `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`
- [ ] Origin reflected only after Set/allowlist lookup — never blindly echoed
- [ ] `Vary: Origin` set on all responses that reflect a dynamic origin
- [ ] CORS middleware runs before auth middleware (preflight gets headers, not 401)
- [ ] `localhost` / dev origins not in production allowlist
- [ ] Subdomain matching uses exact Set or anchored regex — not `endsWith`
- [ ] Preflight `OPTIONS` returns 204 with correct `Allow-Headers` covering all custom headers used
- [ ] `Access-Control-Max-Age` set to reduce preflight volume
- [ ] Wildcard only used on genuinely public, non-credentialed routes
- [ ] `Origin` header not used as a server-side auth mechanism