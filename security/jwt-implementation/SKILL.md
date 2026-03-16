---
name: jwt-implementation
description: Audit or implement JWT authentication flows. Use when the user asks about JWTs, tokens, refresh tokens, storage, signing algorithms, or revocation, even if only one part of the auth flow is mentioned.
category: "security"
---

# JWT Implementation

A security skill for auditing existing JWT auth setups and implementing new ones correctly. Covers the full surface area: algorithm selection, token lifecycle, storage, and revocation.

## Scope

Use this skill for:
- Implementing JWT auth from scratch (backend + frontend)
- Auditing an existing JWT setup for security issues
- Answering targeted questions about any one JWT concern (storage, rotation, revocation, etc.)
- Recommending patterns for specific stacks (Node/Express, Python/FastAPI, etc.)

---

## 1. Signing Algorithm: RS256 vs HS256

| | HS256 | RS256 |
|---|---|---|
| Type | Symmetric (shared secret) | Asymmetric (private/public key pair) |
| Verification | Any party with the secret can verify **and** sign | Public key verifies; only private key signs |
| Key distribution | Secret must be shared with every verifying service | Public key can be freely distributed (JWKS endpoint) |
| Best for | Single-service apps, internal services | Multi-service / microservice architectures |
| Key rotation | Must rotate secret across all services simultaneously | Rotate private key; publish new public key via JWKS |

**Recommendation rules:**
- Use **RS256** when more than one service needs to verify tokens (e.g., a separate API gateway, a resource server).
- Use **HS256** for simple single-service setups where the secret never leaves one process.
- Never use `alg: none`. Reject tokens with unexpected algorithms server-side.

**Server-side enforcement (Node example):**
```js
jwt.verify(token, publicKey, { algorithms: ['RS256'] }); // whitelist explicitly
```

---

## 2. Token Expiry Design

### Access Token
- Keep short: **5–15 minutes** is the standard. The shorter the window, the smaller the blast radius if a token is leaked.
- Encode only what's needed in the payload (user ID, roles, tenant). Avoid PII.
- Do **not** use access tokens for anything other than resource access.

### Refresh Token
- Long-lived: **7–30 days** depending on UX requirements.
- Must be opaque (random string, not a JWT) — store server-side so it can be revoked.
- Store only a hashed reference in the DB (bcrypt or SHA-256), not the raw value.

### `exp` and `iat` claims
```json
{
  "sub": "user_123",
  "iat": 1700000000,
  "exp": 1700000900,
  "jti": "uuid-v4-here"
}
```
- Always include `jti` (JWT ID) — needed for revocation and replay protection.
- Validate `iat` to reject tokens issued in the future (clock skew tolerance: ≤30s).

---

## 3. Refresh Token Rotation

**Pattern: Rotating refresh tokens (recommended)**

Each time a refresh token is used, issue a new refresh token and invalidate the old one.

```
Client → POST /auth/refresh { refresh_token: "abc" }
Server:
  1. Look up "abc" in DB (hashed match)
  2. Verify it's not expired or revoked
  3. Invalidate "abc"
  4. Issue new access_token + refresh_token "xyz"
  5. Return both to client
Client stores "xyz", discards "abc"
```

**Replay / theft detection:**
If a refresh token is used after it's already been rotated (i.e., an attacker and a legitimate client both try to use the same token), **revoke the entire token family** — all refresh tokens issued in this login session. Force re-authentication.

**Token families:**
- Assign a `family_id` when a user first logs in.
- Every rotation stays in the same family.
- On replay detection, revoke by `family_id`.

**DB schema (minimal):**
```sql
CREATE TABLE refresh_tokens (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID NOT NULL,
  token_hash  TEXT NOT NULL,        -- SHA-256 of raw token
  user_id     UUID NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked     BOOLEAN DEFAULT FALSE,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. Storage: httpOnly Cookie vs localStorage

See `references/storage-tradeoffs.md` for a detailed breakdown. Summary:

| | httpOnly Cookie | localStorage |
|---|---|---|
| XSS exposure | ✅ Not accessible via JS | ❌ Fully exposed to XSS |
| CSRF exposure | ⚠️ Requires CSRF mitigation | ✅ Not sent automatically |
| Works cross-origin | ⚠️ Needs `SameSite` + CORS config | ✅ Easier |
| Mobile/native clients | ❌ Cookie jars vary | ✅ Simple |

**Default recommendation: httpOnly cookies with `SameSite=Strict` or `SameSite=Lax`.**

Cookie config (Node/Express):
```js
res.cookie('access_token', token, {
  httpOnly: true,
  secure: true,           // HTTPS only
  sameSite: 'strict',
  maxAge: 15 * 60 * 1000 // 15 min
});
```

If using cookies, **always** add CSRF protection:
- `SameSite=Strict` handles most cases for same-origin flows.
- For cross-origin, use the **Double Submit Cookie** or **Synchronizer Token** pattern.

**When localStorage is acceptable:**
- Native mobile apps (no browser cookie store)
- SPAs calling third-party APIs cross-origin, where cookies are impractical
- Always pair with a strict CSP to reduce XSS blast radius

---

## 5. Revocation Strategy

JWTs are stateless — once issued, they're valid until `exp`. Revocation requires server-side state.

### Option A: Short expiry + blocklist (recommended default)
- Keep access tokens short (≤15 min). Most revocation needs are covered by natural expiry.
- Maintain a blocklist of `jti` values for tokens that must be immediately invalidated (logout, password change, account compromise).
- Blocklist only needs to live until the token's `exp`. Use Redis with TTL:

```js
// On logout / forced revocation:
await redis.set(`blocklist:${jti}`, '1', 'EX', tokenTTLSeconds);

// On every request:
const blocked = await redis.get(`blocklist:${jti}`);
if (blocked) return res.status(401).send('Token revoked');
```

### Option B: Version counter (no Redis required)
- Store a `token_version` per user in DB.
- Embed version in JWT payload.
- On verify, reject if JWT version < DB version.
- Increment `token_version` to invalidate all existing tokens for that user.

```sql
ALTER TABLE users ADD COLUMN token_version INT DEFAULT 0;
```

```js
// JWT payload
{ sub: "user_123", ver: 5, exp: ... }

// On verify
const user = await db.users.findById(payload.sub);
if (payload.ver < user.token_version) throw new Error('Token invalidated');
```

**Tradeoffs:**
- Blocklist: fine-grained (per token), requires Redis
- Version counter: coarse-grained (all tokens for user), DB-only, simpler

### Option C: Reference tokens
Issue opaque tokens; validate against DB on every request. Full revocation control but eliminates statelessness benefit. Only use when real-time revocation is non-negotiable (financial, healthcare).

---

## Audit Checklist

When reviewing an existing JWT implementation, check:

- [ ] Algorithm explicitly whitelisted server-side (no `alg: none`)
- [ ] Access token expiry ≤ 15 minutes
- [ ] Refresh tokens stored as hash, not plaintext
- [ ] Refresh token rotation enabled
- [ ] Replay detection with family-level revocation
- [ ] `jti` included in all tokens
- [ ] Storage: httpOnly cookie preferred; if localStorage, CSP in place
- [ ] CSRF protection if using cookies
- [ ] Revocation path exists for logout + account compromise
- [ ] `exp` and `iat` both validated on verify
- [ ] Secrets / private keys not hardcoded; loaded from env/secrets manager
- [ ] HTTPS enforced in production

---

## Output Format

When producing recommendations or an audit report, use Markdown. Structure output as:

1. **Summary** — what's in place, what's missing
2. **Issues Found** (if auditing) — severity: Critical / High / Medium / Low
3. **Recommendations** — concrete, implementable steps
4. **Code Snippets** — for any non-trivial implementation detail

For implementation tasks, produce working code in the user's stack. If stack is unknown, default to Node.js + Express with TypeScript.

---

## Reference Files

- `references/storage-tradeoffs.md` — Deep dive on cookie vs localStorage with CSRF patterns
- `references/stack-examples.md` — Stack-specific implementation snippets (Node, Python, Go)