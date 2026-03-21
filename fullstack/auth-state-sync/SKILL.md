---
name: auth-state-sync
description: Keeps auth state in sync across browser tabs, page refreshes, and token expiry. Use when implementing cross-tab logout, silent token refresh, storage-event-driven auth, or any pattern where auth state must stay consistent without page reload. Triggers on - sync auth across tabs, logout all tabs, silent refresh, token expiry handling, auth state out of sync, refresh token rotation
category: "Fullstack"
---

# Auth State Sync

Covers the non-obvious parts of keeping auth state consistent across tabs, refreshes, and expiry. Skips CRUD auth setup — assumes tokens exist, storage is chosen.

---

## Discovery

Before writing anything, answer:

1. **Token storage**: `localStorage` (cross-tab visible) or `httpOnly cookie` (invisible to JS)?
2. **Refresh strategy**: Silent (iframe/background fetch) or redirect-based?
3. **Multi-tab logout**: Must all tabs log out on one tab's logout?
4. **Token rotation**: Does the server issue a new refresh token on each use?
5. **Framework**: React, Vue, vanilla? (affects broadcast/listener placement)

---

## Core Patterns

### 1. Cross-Tab Sync via `storage` event

`localStorage` changes in one tab fire `storage` events in **other** tabs only — not the current one. Use this for logout and token propagation.

```ts
// Non-obvious: fires in OTHER tabs, not the one that set the value
window.addEventListener('storage', (e) => {
  if (e.key === 'auth:logout' && e.newValue === 'true') {
    clearLocalAuthState();
    redirectToLogin();
  }

  if (e.key === 'auth:token' && e.newValue) {
    // Another tab refreshed the token — update in-memory state
    setInMemoryToken(e.newValue);
  }
});

// Triggering logout across tabs:
localStorage.setItem('auth:logout', 'true');
localStorage.removeItem('auth:logout'); // Remove immediately so future logouts re-trigger
```

**Why remove immediately**: if you leave `auth:logout=true`, subsequent logouts from a freshly loaded tab won't fire the event (value unchanged).

---

### 2. BroadcastChannel (preferred over storage events)

More explicit, no storage side effects, works even when localStorage is disabled.

```ts
const authChannel = new BroadcastChannel('auth');

// Sender
authChannel.postMessage({ type: 'LOGOUT' });
authChannel.postMessage({ type: 'TOKEN_REFRESHED', token: newAccessToken });

// Receiver (all other tabs on same origin)
authChannel.onmessage = ({ data }) => {
  if (data.type === 'LOGOUT') handleForcedLogout();
  if (data.type === 'TOKEN_REFRESHED') setInMemoryToken(data.token);
};
```

**Non-obvious**: BroadcastChannel does NOT reach the sending tab. Close the channel when the component/service unmounts to avoid ghost listeners.

---

### 3. Silent Token Refresh

#### The race condition problem (critical)

Multiple tabs expiring at the same time will all attempt a refresh simultaneously, causing token rotation conflicts.

**Fix: distributed mutex via localStorage**

```ts
const REFRESH_LOCK_KEY = 'auth:refresh_lock';
const LOCK_TTL_MS = 10_000;

async function acquireRefreshLock(): Promise<boolean> {
  const existing = localStorage.getItem(REFRESH_LOCK_KEY);
  if (existing) {
    const { expiresAt } = JSON.parse(existing);
    if (Date.now() < expiresAt) return false; // Another tab holds lock
  }
  localStorage.setItem(REFRESH_LOCK_KEY, JSON.stringify({
    expiresAt: Date.now() + LOCK_TTL_MS,
    tabId: crypto.randomUUID(),
  }));
  // Re-read to detect write collision (non-atomic, but good enough)
  await new Promise(r => setTimeout(r, 50));
  const after = JSON.parse(localStorage.getItem(REFRESH_LOCK_KEY)!);
  return after.tabId === /* our tabId */ after.tabId; // adjust to store tabId in closure
}

async function releaseRefreshLock() {
  localStorage.removeItem(REFRESH_LOCK_KEY);
}
```

Tabs that fail to acquire the lock wait and then re-read the token from storage once the winning tab broadcasts it.

---

#### Proactive vs reactive refresh

| Approach | When to use |
|---|---|
| Timer-based (proactive) | You control token TTL, predictable expiry |
| 401-interceptor (reactive) | Third-party tokens, unknown expiry |
| Both (recommended) | Proactive with 401 fallback |

```ts
// Proactive: schedule refresh before expiry
function scheduleRefresh(expiresAt: number) {
  const refreshAt = expiresAt - 60_000; // 60s before expiry
  const delay = refreshAt - Date.now();
  if (delay <= 0) return doRefresh(); // Already close to expiry
  setTimeout(doRefresh, delay);
}

// Reactive: intercept 401, queue pending requests
let refreshPromise: Promise<string> | null = null;

async function withAuthRetry(request: () => Promise<Response>) {
  const res = await request();
  if (res.status !== 401) return res;

  // Deduplicate: if refresh already in-flight, await it
  if (!refreshPromise) {
    refreshPromise = doRefresh().finally(() => { refreshPromise = null; });
  }
  await refreshPromise;
  return request(); // Retry once
}
```

---

### 4. httpOnly Cookie Tokens (no JS access)

When access tokens are in httpOnly cookies, you can't read them in JS. Sync strategy changes:

- **Auth state source of truth**: a lightweight `isAuthenticated` flag in localStorage or a short-lived session cookie readable by JS
- **Token expiry detection**: rely entirely on 401 responses — can't decode JWT expiry client-side
- **Cross-tab sync**: use BroadcastChannel for logout; storage events for flag changes

```ts
// Set by your own server after login (not httpOnly, just a flag)
document.cookie = 'auth_hint=1; SameSite=Strict; Secure; Max-Age=3600';

// JS checks this cookie, not the actual token
function isLikelyAuthenticated(): boolean {
  return document.cookie.includes('auth_hint=1');
}
```

---

### 5. Page Refresh / Hydration Sync

**Problem**: In-memory auth state is lost on refresh. Re-reading from storage on mount can cause a flash of unauthenticated UI.

```ts
// Non-obvious: validate token signature/expiry locally before trusting storage
import { jwtDecode } from 'jwt-decode';

function getValidTokenFromStorage(): string | null {
  const token = localStorage.getItem('auth:token');
  if (!token) return null;
  try {
    const { exp } = jwtDecode<{ exp: number }>(token);
    if (exp * 1000 < Date.now()) return null; // Expired, don't hydrate with it
    return token;
  } catch {
    return null;
  }
}

// Initialize auth state synchronously before first render
const initialToken = getValidTokenFromStorage();
```

**Prevent flash**: wrap root render in a check, or use a boolean `authReady` flag that gates rendering — set it only after storage is read.

---

### 6. Logout Propagation Checklist

Not obvious which things need to be cleared and where:

```ts
function fullLogout() {
  // In-memory
  inMemoryToken = null;
  clearRefreshTimer();

  // Storage
  localStorage.removeItem('auth:token');
  localStorage.removeItem('auth:refresh_lock');
  sessionStorage.clear(); // Any session-scoped auth state

  // Broadcast to other tabs
  authChannel.postMessage({ type: 'LOGOUT' });
  // Also cover storage-event-based listeners
  localStorage.setItem('auth:logout', 'true');
  localStorage.removeItem('auth:logout');

  // httpOnly cookie (requires server round-trip)
  await fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });

  // Clear any in-flight requests (cancel pending fetches)
  abortController.abort();

  redirect('/login');
}
```

---

## Output

Produce:
- `authSync.ts` — BroadcastChannel setup, storage listeners, refresh lock
- `useAuth.ts` (or framework equivalent) — hook/store that consumes sync events
- `tokenRefresh.ts` — refresh logic with lock, proactive timer, 401 interceptor

Each file gets a comment header explaining which problem it solves and why the approach is non-obvious.

Flag clearly in comments:
- Where race conditions can occur
- Which events fire in current vs other tabs
- Any browser support caveats (BroadcastChannel: all modern browsers, no IE)