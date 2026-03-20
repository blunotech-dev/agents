---
name: jwt-backend
description: Implement JWT auth on the backend, including token generation, validation, refresh rotation, revocation, and secure storage. Use when building token-based auth, securing endpoints, or handling expiry and access/refresh flows.
category: "Backend"
---

# JWT Backend Skill

## Phase 1 — Discovery

Ask only what context doesn't reveal:

- **Single service or multiple services?** HS256 (shared secret) works for single service. RS256/ES256 (asymmetric) is required when multiple services need to verify tokens independently — each service gets the public key only.
- **What's the token consumer?** Browser (cookie storage viable), mobile (secure storage), or service-to-service (in-memory)? Storage recommendation differs significantly.
- **Is logout/revocation required?** JWTs are stateless — if you need revocation, you need a denylist. If you don't want that complexity, you're accepting that tokens are valid until expiry.
- **Framework/language?** Middleware integration pattern differs across Express, FastAPI, Rails, Go chi, etc.
- **Session duration expectations?** Short-lived access tokens (5–15 min) + long-lived refresh tokens is the correct architecture. If they want a single long-lived token, that's a design smell to address.

---

## Phase 2 — Architecture Decisions

### Algorithm selection (the part teams get wrong first)

**Never use `alg: none`** — some libraries accepted unsigned tokens when `none` was passed. Always explicitly whitelist the expected algorithm in validation:
```python
jwt.decode(token, secret, algorithms=["HS256"])  # explicit allowlist, never omit
```

**HS256 vs RS256 vs ES256:**
- HS256: single shared secret signs and verifies. If the secret leaks from any service, all tokens are forgeable. Use only when one service both issues and verifies.
- RS256: private key signs, public key verifies. Services only need the public key to verify — no secret exposure risk. Standard for any multi-service setup.
- ES256: same asymmetric model as RS256 but smaller tokens and faster verification. Preferred over RS256 for new systems.

**Key size matters for HS256:** The secret must be at least as long as the hash output — 256 bits (32 bytes) minimum for HS256. `secret123` is not a valid JWT secret. Generate with `openssl rand -base64 32`.

### Access + refresh token architecture

**Access token:** short-lived (5–15 minutes), stateless, validated without database lookup. Contains only what's needed for authorization decisions — user ID, roles, tenant ID. Nothing PII beyond what's required.

**Refresh token:** long-lived (7–30 days), opaque (not a JWT), stored server-side with a reference to the user. Used only to issue new access tokens. Must be rotated on every use.

Trap: making the refresh token a JWT. A JWT refresh token can't be invalidated without a denylist — you've gained nothing over a longer-lived access token.

---

## Phase 3 — Implementation

### Token generation — what to include and what not to

**Required claims:**
```json
{
  "sub": "user_123",        // subject — stable user identifier
  "iat": 1710000000,        // issued at — for age checks
  "exp": 1710000900,        // expiry — always set; never omit
  "jti": "uuid-v4-here"     // JWT ID — required if you run a denylist
}
```

**What not to put in the payload:**
- Passwords, PII beyond user ID, full permission sets, anything you'd regret leaking
- Mutable data that might change before expiry (email, username) — it'll be stale during the token lifetime
- Roles that could be elevated — a token claiming `admin: true` issued today is still valid in 14 minutes even if the role was revoked

**The `nbf` claim trap:** `nbf` (not before) is rarely needed and creates clock skew issues in distributed systems. Omit unless you have a specific use case.

### Validation middleware — the full check sequence

Validating signature alone is insufficient. A correct middleware validates in this order:

1. **Token present and well-formed** — three base64url segments separated by dots
2. **Algorithm matches allowlist** — reject if `alg` header doesn't match expected
3. **Signature valid** — cryptographic verification
4. **`exp` not passed** — reject expired tokens (with a small clock skew tolerance: ≤30 seconds)
5. **`nbf` not in the future** — if claim is present
6. **`iss` matches expected issuer** — prevents tokens from one service being used at another
7. **`aud` matches this service** — prevents tokens intended for service A being replayed at service B
8. **`jti` not in denylist** — only if revocation is implemented

```typescript
// Express middleware skeleton
const validateToken = async (req, res, next) => {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) return res.status(401).json({ error: 'missing_token' });

  const token = header.slice(7);
  try {
    const payload = jwt.verify(token, publicKey, {
      algorithms: ['RS256'],          // explicit allowlist
      issuer: 'https://auth.example.com',
      audience: 'https://api.example.com',
    });
    req.user = payload;
    next();
  } catch (err) {
    // Distinguish error types for logging — don't expose details to client
    if (err.name === 'TokenExpiredError') return res.status(401).json({ error: 'token_expired' });
    return res.status(401).json({ error: 'invalid_token' });
  }
};
```

### Refresh token rotation

Rotation means: each refresh token use issues a new refresh token and invalidates the old one. This limits the window of a stolen refresh token.

**Rotation with theft detection:**
Store refresh tokens hashed in the database with a `family` identifier. If a refresh token that's already been rotated is presented (i.e., the old one after a new one was issued), invalidate the entire family — a token has been stolen and replayed.

```sql
CREATE TABLE refresh_tokens (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   uuid NOT NULL,           -- all rotations of one lineage
  token_hash  text NOT NULL UNIQUE,    -- store hash, never plaintext
  user_id     uuid NOT NULL,
  expires_at  timestamptz NOT NULL,
  revoked_at  timestamptz,
  created_at  timestamptz DEFAULT now()
);
```

Rotation flow:
1. Client presents refresh token
2. Look up by hash — if not found or `revoked_at` set: deny, invalidate entire family (theft detected)
3. If valid: mark current token as revoked, issue new access token + new refresh token in the same family
4. Return new pair atomically — if the response fails after the DB write, the client is locked out. Handle with a short grace window or idempotency key.

### Revocation

**The fundamental tradeoff:** JWTs are designed to be stateless. Revocation reintroduces state. Choose one:

**Option A — Accept non-revocability:** Use very short access token expiry (5 min). On logout, delete the refresh token server-side. The access token remains valid until expiry — this window is acceptable for most apps.

**Option B — Denylist (Redis):** On logout or forced invalidation, add the `jti` to a Redis set with TTL matching the token's remaining lifetime. Middleware checks denylist on every request.

```python
def revoke_token(jti: str, expires_at: datetime):
    ttl = int((expires_at - datetime.utcnow()).total_seconds())
    if ttl > 0:
        redis.setex(f"revoked:{jti}", ttl, "1")

def is_revoked(jti: str) -> bool:
    return redis.exists(f"revoked:{jti}") == 1
```

Trap: storing revoked JTIs in Postgres and querying on every request. This turns your stateless auth into a slow synchronous DB call per request. Redis (or Memcached) only for the denylist.

**Option C — Short-lived tokens + token version on user:** Add a `token_version` integer to the user record. Embed it in the JWT. On logout or force-revoke, increment `token_version`. Middleware rejects tokens where `version` < current. Requires one DB read per request — only acceptable if you already have a user cache.

### Secure storage recommendations

**Browser:**
- `HttpOnly` + `Secure` + `SameSite=Strict` cookies for both access and refresh tokens — not `localStorage`. `localStorage` is readable by any JavaScript on the page; a single XSS vulnerability exposes every token.
- BFF (Backend for Frontend) pattern: the browser never sees the JWT. The BFF holds tokens server-side and issues session cookies to the browser.
- If you must use `Authorization: Bearer` from JS, store in memory only (a closure or React state) — lost on page refresh, but XSS-safe.

**Mobile:**
- iOS: Keychain
- Android: EncryptedSharedPreferences or Android Keystore
- Never `AsyncStorage` or unencrypted shared preferences — accessible on rooted devices

**Service-to-service:**
- Environment variable or secrets manager (Vault, AWS Secrets Manager) — never hardcoded, never in source control
- In-memory only at runtime; re-fetch from secrets manager on rotation

### Clock skew and distributed validation

All services validating tokens must have synchronized clocks (NTP). A 5-minute clock drift can make valid tokens appear expired or not-yet-valid. Allow a small tolerance (`leeway`) in validation — 30 seconds is sufficient, 5 minutes defeats the purpose of short expiry.

Trap: deploying a token validation service in a container without NTP sync. Container clocks can drift from the host. Validate with `timedatectl` or `date` in CI.

---

## Phase 4 — Output

Produce whichever the user needs:

- **Token generation function** — with correct claims, algorithm, and key loading
- **Validation middleware** — full claim check sequence for their framework
- **Refresh token rotation implementation** — DB schema + rotation logic + theft detection
- **Revocation setup** — Redis denylist with TTL-based cleanup
- **Storage recommendation** — specific guidance for their client type (browser/mobile/service)
- **Algorithm migration plan** — HS256 → RS256 for existing systems, with key rotation steps

Always include:
- Explicit algorithm allowlist in any validation code — never omit
- `iss` and `aud` claim validation — not just signature
- A note if they're using a single long-lived token (design smell, push toward access+refresh split)
- Warning if refresh token is a JWT (can't be truly revoked)
- Redis (not Postgres) for any denylist implementation