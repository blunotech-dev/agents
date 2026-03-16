---
name: oauth-flow
description: Implement or audit OAuth 2.0 authorization code flow. Use this skill for OAuth, PKCE, authorization codes, token exchange, scopes, redirect URIs, state parameter, social login, or "login with Google/GitHub/etc." — including narrow questions like "do I need PKCE?" or "what scopes should I request?".
category: "Security"
---

# OAuth 2.0 Authorization Code Flow

## Flow at a Glance

```
1. Client → Auth Server: GET /authorize?response_type=code&client_id=...&state=...&code_challenge=...
2. Auth Server → Client: redirect to /callback?code=...&state=...
3. Client → Auth Server: POST /token { code, code_verifier, redirect_uri }
4. Auth Server → Client: { access_token, refresh_token, expires_in }
5. Client → Resource Server: GET /api with Authorization: Bearer <access_token>
```

**Rule:** Always `response_type=code`. Never `response_type=token` (implicit flow — deprecated).

---

## PKCE

Required for all public clients (SPAs, mobile). Recommended for server-side too.

```ts
// 1. Generate before redirect
const verifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = base64url(await crypto.subtle.digest('SHA-256', encode(verifier)));
sessionStorage.setItem('pkce_verifier', verifier);

// 2. Authorization request
params.set('code_challenge', challenge);
params.set('code_challenge_method', 'S256'); // never 'plain'

// 3. Token exchange
body.set('code_verifier', sessionStorage.getItem('pkce_verifier')!);
sessionStorage.removeItem('pkce_verifier');

function base64url(buf: ArrayBuffer | Uint8Array) {
  return btoa(String.fromCharCode(...new Uint8Array(buf as ArrayBuffer)))
    .replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
}
```

---

## State Parameter

Prevents CSRF. Must be cryptographically random, stored, and validated **before** token exchange.

```ts
// Before redirect
const state = base64url(crypto.getRandomValues(new Uint8Array(16)));
sessionStorage.setItem('oauth_state', state);

// On callback — validate first, then use code
const returned = params.get('state');
const saved = sessionStorage.getItem('oauth_state');
sessionStorage.removeItem('oauth_state');
if (!returned || returned !== saved) throw new Error('State mismatch');
```

**Tip:** Embed `returnTo` path inside state — `btoa(JSON.stringify({ nonce, returnTo }))` — gives CSRF protection plus post-login redirect in one field.

---

## Authorization Request

```ts
const url = new URL('https://provider.com/authorize');
url.searchParams.set('response_type', 'code');
url.searchParams.set('client_id', CLIENT_ID);
url.searchParams.set('redirect_uri', REDIRECT_URI); // exact match, HTTPS
url.searchParams.set('scope', 'openid profile email');
url.searchParams.set('state', state);
url.searchParams.set('code_challenge', challenge);
url.searchParams.set('code_challenge_method', 'S256');
```

---

## Callback Handling

```ts
const params = new URLSearchParams(location.search);

// 1. Error first
if (params.get('error')) throw new Error(params.get('error_description')!);

// 2. State before code
validateState(params.get('state'));

// 3. Exchange
const code = params.get('code')!;
const tokens = await exchangeCode(code);
```

---

## Token Exchange

**Public client (SPA):**
```ts
const res = await fetch('https://provider.com/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    code, redirect_uri: REDIRECT_URI,
    client_id: CLIENT_ID,
    code_verifier: verifier,
  }),
});
```

**Confidential client (server-side):** swap `code_verifier` for `client_secret` (or use both).

---

## Scopes

- Request minimum necessary; add more incrementally when features need them
- Validate granted scopes — server may return fewer than requested
- Include `offline_access` (or provider equivalent) to get a refresh token
- Include `openid` for OIDC `id_token`

---

## Common Mistakes

| Mistake | Why it's wrong | Fix |
|---|---|---|
| `response_type=token` (implicit) | Token in URL → leaks into history/logs | Use `response_type=code` |
| No state / static state | CSRF → attacker links victim session to their account | Random state per request |
| No PKCE | Code interception → attacker exchanges code | PKCE with S256 |
| Token in redirect URL | Leaks in browser history, server logs, Referer | Tokens in body/headers only |
| Skip id_token validation | Can't verify issuer, audience, expiry | Validate sig + iss + aud + exp |
| Open post-login redirect | Phishing via `?next=https://evil.com` | Allowlist or same-origin check |
| `client_secret` in frontend | Extractable from bundle | Public clients use PKCE only |
| localStorage for tokens | XSS exfiltrates tokens | Memory (access) + httpOnly cookie (refresh) |
| `code_challenge_method=plain` | No security — challenge == verifier | Always S256 |

---

## id_token Validation (OIDC)

Always validate when using OpenID Connect:
- Signature against provider's JWKS endpoint
- `iss` = expected issuer
- `aud` = your `client_id`
- `exp` > now (allow ≤30s clock skew)
- `nonce` matches what you sent

Use a library (`jose`, `openid-client`, `python-jose`). Never roll your own.

---

## Flow Selection

| Client type | Flow |
|---|---|
| SPA / mobile | Authorization code + PKCE (no secret) |
| Server-side web app | Authorization code + client secret (+ PKCE) |
| Machine-to-machine | Client credentials (no user) |
| Input-constrained device | Device authorization flow |

---

## Audit Checklist

- [ ] `response_type=code`, not `token`
- [ ] PKCE with S256 on all client types
- [ ] State: random per request, validated before code use
- [ ] `redirect_uri`: exact string, HTTPS, registered with provider
- [ ] Callback errors handled (`?error=`) 
- [ ] Tokens never in URLs
- [ ] id_token validated (if OIDC)
- [ ] Post-login redirect: allowlist or same-origin only
- [ ] `client_secret` server-side only, not in frontend bundles
- [ ] Minimum scopes; granted scopes validated
- [ ] Access token in memory; refresh token in httpOnly cookie

---

## Reference Files

Load on demand:
- `references/stack-examples.md` — Node/Express, Python/FastAPI, SPA full implementations
- `references/provider-quirks.md` — Google, GitHub, Microsoft, Auth0 provider-specific notes
