---
name: memo-strategy
description: Decide when to use React.memo, useMemo, and useCallback based on cost vs benefit, referential equality, and re-render behavior. Use when optimizing React performance, fixing unnecessary re-renders, or evaluating memoization patterns.
category: "React"
---

# memo-strategy

Guides precise, justified memoization decisions in React — covering when NOT to memoize as much as when to.

---

## Phase 1 — Discover

Before recommending anything, establish:

- Is the re-render actually measured as slow? (profiler, not intuition)
- What triggers the re-render? (parent re-render, context change, state change)
- Are the props stable across renders? (objects/arrays/functions created inline = never stable)
- Is the expensive computation inside render, or just a prop lookup?

> Skip this phase only if the user is asking a conceptual question, not debugging a specific component.

---

## Phase 2 — Identify the Real Problem

Most "memoization problems" are actually one of these:

| Symptom | Likely cause | Memo helps? |
|---|---|---|
| Child re-renders despite same props | Inline object/array/function prop | Only after fixing the prop |
| Memo'd component still re-renders | Context value is new object each render | No — fix the context |
| `useMemo` result changes every render | Dependency is itself unstable | No — fix the dependency |
| `useCallback` causes downstream re-renders | The callback depends on state that changes frequently | Sometimes — consider restructuring |
| Component is slow on first render | Computation cost is real | `useMemo` may help |

**The referential equality trap**: `React.memo` does a shallow prop comparison. If any prop is an object, array, or function created during render, memo is a no-op — the component will always re-render. Fix the prop first.

---

## Phase 3 — Apply Decision Rules

### `React.memo`

Apply when ALL of these are true:
1. The component re-renders due to parent renders (not its own state).
2. The re-renders are measured as expensive (heavy DOM, large lists, complex calculations).
3. All props are either primitives or referentially stable values.

Do NOT apply when:
- The component almost always receives new props anyway.
- Props include inline objects/functions that aren't wrapped in their own memo hooks.
- The component is trivial (few nodes, no heavy work) — wrapping adds overhead with zero gain.
- You're memoizing "just in case."

### `useMemo`

Apply when:
- The computation is provably expensive (sorting large arrays, heavy math, recursive transforms).
- The result is used as a prop to a memo'd child — and must stay referentially stable.
- You need referential stability for an object/array that feeds a `useEffect` dependency.

Do NOT apply when:
- The value is a primitive — no referential stability benefit.
- The computation is trivial (filtering a 10-item array, string concat).
- You're creating a new object just to pass it to a non-memo'd child — useless.
- The dependency array includes values that change on every render — defeats the purpose.

### `useCallback`

Apply when:
- The function is passed as a prop to a `React.memo`'d child.
- The function is a `useEffect` dependency and you want stable identity.

Do NOT apply when:
- The child isn't memo'd — the callback's identity doesn't matter.
- The function is only used inside the same component.
- The function depends on frequently-changing state — the callback changes anyway, memo is pointless.

---

## Phase 4 — Output

Produce one of:

**A) Annotated component** — Add, remove, or modify memo usage with inline comments explaining each decision. Flag any memo that's currently a no-op and why.

**B) Decision summary** — When the user wants an explanation, not code: a concise table or list covering each candidate (component, value, callback), the recommendation, and the one-line justification.

Always include:
- At least one "do NOT memo this" call if you found one — this is often more valuable than adding memo.
- If the fix requires stabilizing a prop before memo will work, say so explicitly before suggesting `React.memo`.

---

## Non-obvious Rules to Enforce

**Memo invalidation cost is real.** Every memo adds a comparison on each render. For components with many props or deeply nested prop values, the comparison cost can exceed the render cost. Flag this when prop count is high.

**Context kills memo.** If a component consumes a context value that's a new object each render (common with `value={{ state, dispatch }}`), `React.memo` provides zero benefit. The fix is splitting context or stabilizing the value with `useMemo` at the provider level.

**`useCallback` without `React.memo` on the child is always useless.** If the user has `useCallback` wrapping a function that's passed to a non-memo'd child, flag it as dead code.

**Derived state vs. `useMemo`.** If the "expensive computation" is just deriving state from other state, and that state doesn't change frequently, regular variable assignment is fine. `useMemo` only helps when the computation is expensive AND the inputs are stable.

**Stale closure risk.** When `useCallback` has a missing or incorrect dependency, the callback silently captures stale values. Always verify the dependency array is complete — don't silently accept what the user wrote.