---
name: useeffect-cleanup
description: Audit useEffect hooks for proper cleanup, including removing listeners, canceling subscriptions, clearing timers, aborting fetches, and disconnecting observers. Use when debugging memory leaks, duplicate effects, or state updates on unmounted components.
category: "React"
---

# useEffect Cleanup Auditor

## Phase 1 — Identify What Needs Cleanup

Not every effect needs cleanup. Scan for these and flag every one missing a `return`:

| Setup call | Required cleanup |
|---|---|
| `addEventListener` | `removeEventListener` — exact same fn reference |
| `setTimeout` | `clearTimeout` |
| `setInterval` | `clearInterval` |
| `fetch` / `axios` / any HTTP | `AbortController.abort()` |
| `new WebSocket` | `ws.close()` |
| `new IntersectionObserver` | `observer.disconnect()` |
| `new ResizeObserver` | `observer.disconnect()` |
| `new MutationObserver` | `observer.disconnect()` |
| `eventEmitter.on` / `.subscribe` | `.off()` / `.unsubscribe()` |
| `setInterval` inside a recursive `setTimeout` | clear the *current* timeout id, not a stale one |

Effects that only compute, set state, or call stable setters need no cleanup.

---

## Phase 2 — Non-Obvious Failure Modes

### 2.1 `addEventListener` With an Inline Arrow Creates an Unremovable Listener

```ts
// ❌ anonymous arrow — removeEventListener does nothing, new listener added every re-render
useEffect(() => {
  window.addEventListener('resize', () => setWidth(window.innerWidth));
}, []);

// ✅ named reference — same fn identity required for removal
useEffect(() => {
  const handler = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handler);
  return () => window.removeEventListener('resize', handler);
}, []);
```

The listener must be the **same reference** passed to both `add` and `remove`. Even extracting to a `useCallback` outside the effect doesn't help if the callback is recreated between registration and removal — define it inside the effect.

### 2.2 `AbortController` — What Actually Gets Aborted and What Doesn't

`abort()` cancels the network request (stops bandwidth), but the `fetch` promise rejects with a `DOMException`. You must catch it specifically, or you'll log an error on every normal unmount.

```ts
useEffect(() => {
  const controller = new AbortController();

  async function load() {
    try {
      const res = await fetch(url, { signal: controller.signal });
      const data = await res.json();
      setData(data); // safe — only reached if not aborted
    } catch (err) {
      if (err.name === 'AbortError') return; // expected, not a real error
      setError(err);
    }
  }

  load();
  return () => controller.abort();
}, [url]);
```

**What `AbortController` does NOT cover:** axios (use `CancelToken` or pass `signal` with axios ≥0.22), custom polling, WebSocket messages already in-flight. Each needs its own cancellation mechanism.

### 2.3 The Race Condition `AbortController` Doesn't Fix

Aborting the fetch prevents the network round-trip, but if you're setting state conditionally *after* an `await`, a second effect run can start before cleanup fires (React 18 strict mode fires effects twice in dev; concurrent mode can interrupt). Add an `ignored` flag for anything after an await that isn't already guarded by the signal:

```ts
useEffect(() => {
  let ignored = false;
  const controller = new AbortController();

  async function load() {
    try {
      const res = await fetch(url, { signal: controller.signal });
      const data = await res.json();
      if (!ignored) setData(data); // guards against out-of-order responses
    } catch (err) {
      if (err.name !== 'AbortError' && !ignored) setError(err);
    }
  }

  load();
  return () => {
    ignored = true;
    controller.abort();
  };
}, [url]);
```

Use both: `AbortController` for the network, `ignored` flag for any state update after an `await`.

### 2.4 Timers — `clearTimeout` on the Right ID

A recursive `setTimeout` pattern reassigns the id on every tick. You must capture the *current* id in the cleanup closure, not a stale outer variable.

```ts
// ❌ id is reassigned; cleanup clears the first id, not the last
useEffect(() => {
  let id: ReturnType<typeof setTimeout>;
  const poll = () => { id = setTimeout(poll, 1000); };
  poll();
  return () => clearTimeout(id); // only clears whatever id was at cleanup time
}, []);

// ✅ use a ref to always hold the current id
useEffect(() => {
  const idRef = { current: 0 as ReturnType<typeof setTimeout> };
  const poll = () => { idRef.current = setTimeout(poll, 1000); };
  poll();
  return () => clearTimeout(idRef.current);
}, []);
```

### 2.5 Observer Pattern — `disconnect` vs `unobserve`

`IntersectionObserver` and `ResizeObserver` have two methods:
- `observer.unobserve(element)` — stops watching one element but keeps the observer alive
- `observer.disconnect()` — stops everything

In cleanup, always call `disconnect()` unless you're sharing one observer across multiple components (rare; requires a registry pattern). Calling `unobserve` in cleanup is a common partial fix that leaks the observer instance itself.

### 2.6 Cleanup Runs on Every Re-Run, Not Just Unmount

This is the most misunderstood thing about cleanup. If the dep array is `[userId]`, the cleanup runs every time `userId` changes, *before* the next effect setup. A cleanup that calls `ws.close()` will close the socket on every re-render if the socket is unstable. Fix the dep (see `useeffect-dependencies` skill) before assuming the cleanup is the problem.

---

## Phase 3 — Execution

1. Find every `useEffect` in scope.
2. Check each for a `return` statement.
3. For effects without a return: identify what was opened/registered. Apply the matching cleanup from Phase 1.
4. For effects with a return: verify the cleanup actually matches the setup (same fn reference for listeners, correct id for timers, correct signal for fetch).
5. Check if any `async` logic after an `await` sets state without an `ignored` guard.
6. Note any effect where cleanup runs on dep change — confirm that re-setup is intentional and cheap.

---

## Phase 4 — Output

Produce:
- Fixed effect(s) with cleanup added or corrected
- One line per fix: what was missing and what failure mode it caused (memory leak / duplicate listener / stale update / race condition)
- If `AbortError` handling was missing, include the catch guard
- Flag if an `ignored` flag is needed in addition to `AbortController`

Do not produce: cleanup that only calls `unobserve` when `disconnect` is correct, cleanup that references a stale timer id, or `AbortController` without the `AbortError` catch.