# Storage Tradeoffs: httpOnly Cookie vs localStorage

## Threat Model

JWT storage decisions are fundamentally about which attacks you're optimizing against:

| Attack | Cookie (httpOnly) | localStorage |
|---|---|---|
| **XSS** | ✅ JS cannot read httpOnly cookies | ❌ `document.cookie` / `localStorage` fully accessible to injected scripts |
| **CSRF** | ⚠️ Browser auto-sends cookies on requests | ✅ Token must be explicitly added to requests |
| **Network interception** | ✅ `Secure` flag enforces HTTPS | ✅ App-level HTTPS handles this |
| **Physical device access** | ⚠️ Persists until `Max-Age`/`Expires` | ⚠️ Persists until cleared |

XSS is far more common than CSRF in modern apps. **httpOnly cookies win on the more prevalent threat.**

---

## httpOnly Cookie Setup

### Minimum secure configuration

```js
// Express
res.cookie('access_token', jwt, {
  httpOnly: true,       // JS cannot read
  secure: true,         // HTTPS only (set false in dev if needed)
  sameSite: 'strict',   // No cross-site sending
  maxAge: 15 * 60 * 1000,
  path: '/'
});
```

### SameSite values

| Value | Behavior | Use when |
|---|---|---|
| `Strict` | Cookie never sent cross-site | Auth cookies on same-origin apps |
| `Lax` | Sent on top-level navigations (GET) | Default for most apps |
| `None` | Always sent cross-site | Requires `Secure`; for embedded iframes/third-party |

For auth cookies, prefer `Strict`. Fall back to `Lax` only if you have legitimate cross-site GET navigations that need auth.

### CSRF mitigation with cookies

`SameSite=Strict` eliminates CSRF for same-site apps. For cross-origin:

**Double Submit Cookie Pattern:**
```
1. Server sets a non-httpOnly csrf_token cookie
2. Client reads it and sends as X-CSRF-Token header
3. Server compares header value to cookie value
4. Attacker cannot read the cookie value cross-origin, so cannot forge the header
```

```js
// Set csrf token (readable by JS)
res.cookie('csrf_token', crypto.randomUUID(), {
  httpOnly: false,  // Must be readable by JS
  secure: true,
  sameSite: 'strict'
});

// Client sends: headers['X-CSRF-Token'] = getCookieValue('csrf_token')

// Server validates:
if (req.headers['x-csrf-token'] !== req.cookies['csrf_token']) {
  return res.status(403).send('CSRF check failed');
}
```

---

## localStorage Setup

If you must use localStorage (native/mobile, cross-origin constraints):

### Attach token to every request

```js
// Axios interceptor
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### Mitigate XSS with CSP

Add a Content Security Policy to limit what injected scripts can do:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  connect-src 'self' https://api.yourdomain.com;
  object-src 'none';
  base-uri 'self';
```

A tight CSP doesn't eliminate XSS but significantly reduces the attacker's ability to exfiltrate tokens or make authenticated requests.

### Token lifecycle in localStorage

```js
// Store after login
localStorage.setItem('access_token', accessToken);
localStorage.setItem('refresh_token', refreshToken);
localStorage.setItem('token_expiry', Date.now() + 15 * 60 * 1000);

// On each request, check expiry
const expiry = parseInt(localStorage.getItem('token_expiry'));
if (Date.now() > expiry - 60_000) { // refresh 1 min before expiry
  await refreshAccessToken();
}

// On logout — always clear both
localStorage.removeItem('access_token');
localStorage.removeItem('refresh_token');
localStorage.removeItem('token_expiry');
```

---

## Hybrid Pattern (Memory + Cookie)

Used by high-security SPAs:

- **Access token**: Stored in JS memory only (module-level variable). Lost on page refresh.
- **Refresh token**: httpOnly cookie.
- On every page load, silently call `/auth/refresh` to get a new access token.

```js
// auth.js module
let _accessToken = null;

export const getAccessToken = () => _accessToken;
export const setAccessToken = (t) => { _accessToken = t; };

// On app init
async function silentRefresh() {
  const res = await fetch('/auth/refresh', { method: 'POST', credentials: 'include' });
  if (res.ok) {
    const { access_token } = await res.json();
    setAccessToken(access_token);
  }
}
```

**Tradeoff:** Extra network call on page load; token lost on refresh (mitigated by silent refresh). Best XSS/CSRF protection available in a browser context.

---

## Decision Guide

```
Are you building a native mobile app?
  → localStorage (or secure keychain for mobile)

Is your app fully same-origin (frontend + API same domain)?
  → httpOnly cookie, SameSite=Strict

Is your app cross-origin (SPA on cdn.app.com → api.app.com)?
  → httpOnly cookie with SameSite=None + Secure + CSRF token
  → OR localStorage with strict CSP (if cross-origin cookie setup is too complex)

Do you need maximum security (financial, healthcare)?
  → Memory-only access token + httpOnly refresh token cookie
```