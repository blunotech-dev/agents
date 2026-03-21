---
name: useeffect-dependencies
description: Fix incorrect or missing useEffect dependency arrays, handling exhaustive-deps issues, stale closures, and intentional omissions. Use when effects re-run incorrectly, read stale values, or trigger eslint warnings across useEffect, useCallback, or useMemo.
category: "React"
---

# useEffect Dependency Fixer

## Phase 1 — Diagnose Before Touching Anything

Identify the *actual* bug category before suggesting any fix. Most mistakes come from misdiagnosing the category.

| Symptom | Likely Category |
|---|---|
| ESLint error, effect works fine | Missing dep that happens to be stable |
| Infinite re-render loop | Object/function dep recreated on every render |
| Stale value read inside effect | Closure captures old binding |
| Effect never re-runs when it should | Dep omitted or effect has wrong placement |
| `useCallback` chain grew just to satisfy deps | Stabilization needed, not suppression |

---

## Phase 2 — The Non-Obvious Rules

### 2.1 Referential Stability Is the Real Problem

`exhaustive-deps` errors on objects/functions almost never mean "add the dep." They mean the dep is unstable. Adding it causes an infinite loop; omitting it causes a stale closure. The fix is stabilization upstream.

**Decision tree:**
- Function defined in component body? → `useCallback` it, or move it outside the component if it doesn't use component state.
- Object defined in component body? → `useMemo` it, or destructure to primitives.
- Value from a library hook that returns a new reference each render? → Check the library's docs — most stable values (e.g., `dispatch` from `useReducer`, `set*` from `useState`) are guaranteed stable and can be **safely omitted** from the array (though including them is also fine).

### 2.2 The Legitimate `eslint-disable` Cases

These are the **only** situations where suppressing exhaustive-deps is defensible:

1. **On-mount-only intent** — intentionally run once. Use `// eslint-disable-next-line react-hooks/exhaustive-deps` with a comment explaining intent. Do NOT use an empty `[]` silently.
2. **Polling interval** — the interval callback needs access to latest state but you don't want to reset the interval on every state change. Use a ref to hold the latest value (see §2.4).
3. **Third-party DOM lib** — effect initializes a lib (e.g., chart, map) once; re-running it destroys and recreates the instance. Document this explicitly.
4. **`previousValue` pattern** — you're intentionally reading a stale value to compare against current.

For every other case: fix the dep array, don't suppress.

### 2.3 Stale Closure Fixes

**Pattern: state or prop read inside a callback passed to a timer/subscription**

```ts
// ❌ stale closure — count captured at setup time
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // always uses initial count
  }, 1000);
  return () => clearInterval(id);
}, []);

// ✅ functional updater — no closure needed
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);
  }, 1000);
  return () => clearInterval(id);
}, []);
```

**Pattern: stale callback in event listener or subscription**

```ts
// ✅ ref pattern — always reads latest value without re-registering
const onMessageRef = useRef(onMessage);
useLayoutEffect(() => { onMessageRef.current = onMessage; });

useEffect(() => {
  const unsub = socket.on('msg', (e) => onMessageRef.current(e));
  return unsub;
}, [socket]); // socket is the only real dep
```

Use `useLayoutEffect` (not `useEffect`) for the ref sync to prevent a window where the ref is stale between render and paint.

### 2.4 The "Latest Ref" Pattern (Canonical)

When you need latest-value access without re-triggering effects:

```ts
function useLatest<T>(value: T) {
  const ref = useRef(value);
  useLayoutEffect(() => { ref.current = value; });
  return ref;
}
```

Use this instead of inline ref sync whenever the pattern appears more than once.

### 2.5 Object/Array Deps — Stabilize, Don't Stringify

**Wrong approach:** `JSON.stringify(obj)` as a dep. Breaks on non-serializable values, hides the real issue, causes bugs with key ordering.

**Right approach:**
- Destructure to primitives: `const { id, type } = config;` then dep on `id, type`
- If the whole object must be stable, `useMemo` it at the call site
- If it comes from props, the parent is responsible for memoizing

### 2.6 When `useCallback` Chains Are the Problem

If fixing a `useEffect` dep causes you to `useCallback` a function, and that function calls another function that also needs `useCallback`, stop. You're papering over a design issue.

**Refactor instead:**
1. Move the function chain outside the component if it doesn't use component state/props
2. Or pass only primitive args to the effect and reconstruct the function inside the effect body (effect-local functions don't need to be in deps)
3. Or use `useReducer` — dispatch is stable, so complex logic moves into the reducer

```ts
// ❌ useCallback chain
const getPayload = useCallback(() => ({ userId, token }), [userId, token]);
const fetchUser = useCallback(() => fetch(getPayload()), [getPayload]);
useEffect(() => { fetchUser(); }, [fetchUser]);

// ✅ primitives into effect, function is effect-local
useEffect(() => {
  const fetchUser = () => fetch({ userId, token });
  fetchUser();
}, [userId, token]);
```

---

## Phase 3 — Execution Pattern

1. Read the full effect + surrounding component scope. Don't fix in isolation.
2. Identify every variable the effect closes over.
3. Classify each as: primitive stable / primitive unstable / object/function stable / object/function unstable.
4. For unstable references: stabilize upstream before touching the dep array.
5. Write the fix. If suppression is unavoidable, add an inline comment explaining why.
6. Check for cleanup — if the dep array changes, the cleanup/re-setup cycle must be correct.

---

## Phase 4 — Output

Produce:
- Fixed code with inline comments on non-obvious decisions
- One-line explanation per changed dep: what it was, what it is, why
- Flag if the root cause is a parent component not memoizing a prop (actionable for the caller)
- If a `useCallback` chain was the cause, show the collapsed version

Do not produce: generic eslint-disable wrappers, JSON.stringify hacks, or dep arrays that will cause infinite loops.