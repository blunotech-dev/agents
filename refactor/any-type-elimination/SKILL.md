---
name: any-type-elimination
description: Find and replace `any` types in TypeScript with precise types, generics, or `unknown` plus narrowing. Use when improving type safety, fixing implicit `any`, removing `any`, or migrating code to strict TypeScript.
category: "Refactoring"
---

# any-type-elimination

A structured refactoring skill for finding and replacing `any` types in TypeScript
with accurate, maintainable alternatives — without breaking the build.

---

## Taxonomy of `any` Occurrences

Not all `any` usages are equal. Identify the kind before choosing a replacement strategy.

| Kind | Example | Signal |
|---|---|---|
| **Lazy annotation** | `const user: any = getUser()` | Return type is known, just not written |
| **Untyped third-party data** | `const res: any = await fetch(...)` | Shape is known at runtime, not typed |
| **Weak generic** | `function wrap(val: any): any` | Should be a type parameter |
| **Type escape hatch** | `(obj as any).secret` | Working around a missing/wrong type |
| **Implicit `any`** | `function f(x) { ... }` | Missing parameter annotation |
| **`any[]` collections** | `const items: any[] = []` | Element type is knowable |
| **Callback parameters** | `.map((item: any) => ...)` | Inferred from array type, annotation is redundant |
| **Error catch bindings** | `catch (e: any)` | Should be `unknown` with narrowing |

---

## Phase 1 — Discovery

### 1.1 Find all explicit `any` usages

```bash
# Count and locate every any annotation
grep -rn ": any" src/ --include="*.ts" --include="*.tsx"
grep -rn "as any" src/ --include="*.ts" --include="*.tsx"
grep -rn "<any>" src/ --include="*.ts" --include="*.tsx"

# Implicit any: compile with strict to surface them
npx tsc --noEmit --strict 2>&1 | grep "implicitly has an 'any' type"
```

### 1.2 Audit `tsconfig.json`

Check which flags are currently off — these control what TypeScript will accept:

```json
{
  "strict": true,           // umbrella flag
  "noImplicitAny": true,    // catches untyped parameters
  "strictNullChecks": true  // required for unknown narrowing to be meaningful
}
```

If `strict` is off, enabling it will surface additional hidden `any`s. Plan for that before committing.

### 1.3 Prioritise

Fix in this order:
1. **Public API boundaries** — exported functions, module entry points
2. **Data access layer** — API responses, DB query results
3. **Shared utilities** — functions called across the codebase
4. **Internal implementation details** — local variables, callbacks

---

## Phase 2 — Replacement Strategies

### 2.1 Lazy annotations → use the actual type

The return type already exists; write it down.

```ts
// Before
const user: any = getUser()

// After
const user: User = getUser()
```

If the return type of `getUser` is `any`, fix the function signature first — downstream annotations follow automatically.

---

### 2.2 Untyped third-party / runtime data → model the shape

Define an interface that matches the expected structure, then validate at the boundary.

```ts
// Before
const res: any = await fetch('/api/user').then(r => r.json())

// After
interface UserResponse {
  id: string
  name: string
  email: string
}

const res: UserResponse = await fetch('/api/user').then(r => r.json())
```

For truly unknown shapes (plugin systems, external configs), use `unknown` and narrow explicitly — see §2.5.

---

### 2.3 Weak generics → introduce a type parameter

```ts
// Before
function identity(val: any): any { return val }

// After
function identity<T>(val: T): T { return val }
```

More constrained generics with `extends`:

```ts
// Before
function getProperty(obj: any, key: string): any {
  return obj[key]
}

// After
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
```

---

### 2.4 Type escape hatches (`as any`) → fix the upstream type

`as any` is usually a symptom of a missing overload, wrong return type, or an incomplete third-party `@types` package.

```ts
// Before — working around a missing method on the type
(window as any).__analytics.track(event)

// After option A — declaration merging
declare global {
  interface Window {
    __analytics: AnalyticsClient
  }
}
window.__analytics.track(event)

// After option B — explicit cast through unknown when A isn't feasible
(window as unknown as WindowWithAnalytics).__analytics.track(event)
```

`as unknown as T` is safer than `as any` — it requires you to name the type you expect.

---

### 2.5 Unknown runtime data → `unknown` with narrowing

`unknown` is `any` with discipline: you must check before you use.

```ts
// Before
function parse(input: any) {
  return input.data.items
}

// After
function parse(input: unknown): Item[] {
  if (
    typeof input === 'object' &&
    input !== null &&
    'data' in input &&
    typeof (input as { data: unknown }).data === 'object'
  ) {
    // narrow further...
  }
  throw new Error('Unexpected shape')
}
```

For complex shapes, use a validation library to avoid hand-written narrowing chains:

```ts
// zod
import { z } from 'zod'
const ResponseSchema = z.object({ items: z.array(ItemSchema) })
const parsed = ResponseSchema.parse(input)  // typed as { items: Item[] }
```

---

### 2.6 Implicit `any` parameters → annotate or infer

```ts
// Before — implicit any on x
function double(x) { return x * 2 }

// After
function double(x: number): number { return x * 2 }
```

For callbacks whose type can be inferred from context, remove the annotation entirely rather than adding `any`:

```ts
// Before — redundant any
const lengths = ['a', 'bb', 'ccc'].map((s: any) => s.length)

// After — inferred as string
const lengths = ['a', 'bb', 'ccc'].map(s => s.length)
```

---

### 2.7 `catch (e: any)` → `catch (e: unknown)`

```ts
// Before
try {
  riskyOp()
} catch (e: any) {
  console.error(e.message)
}

// After
try {
  riskyOp()
} catch (e: unknown) {
  const message = e instanceof Error ? e.message : String(e)
  console.error(message)
}
```

---

### 2.8 `any[]` → typed arrays or tuples

```ts
// Before
const results: any[] = []

// After — homogeneous
const results: QueryResult[] = []

// After — heterogeneous known shape
const pair: [string, number] = ['score', 42]
```

---

## Phase 3 — Execution Order

Refactor **outside-in**: types at the boundary (API responses, function signatures) propagate inward automatically, eliminating downstream `any`s without touching them.

1. Fix return types of data-fetching / parsing functions
2. Fix exported function signatures
3. Fix shared utility generics
4. Fix internal call sites (many will self-resolve from step 1–3)
5. Fix remaining catch blocks and local variables
6. Enable `noImplicitAny` (or `strict`) and resolve surfaced errors

Never fix every `any` in one pass. Work file-by-file, running `tsc --noEmit` after each file to catch regressions.

---

## Phase 4 — Output Format

### Per-occurrence diff

Show before/after for each replaced `any`, grouped by file:

```
src/api/user.ts
  line 12: const user: any  →  const user: User
  line 34: (res as any)     →  (res as unknown as RawResponse)

src/utils/transform.ts
  line 7:  function wrap(x: any): any  →  function wrap<T>(x: T): T
```

### Type definitions introduced

List every new interface, type alias, or generic constraint added — these are the real deliverable.

### `tsconfig` changes recommended

If the audit reveals flags that should be enabled, note them explicitly with the expected impact (e.g., "enabling `strictNullChecks` will surface ~12 additional errors in src/legacy/").

---

## Decision Reference

| Situation | Use |
|---|---|
| Shape is fully known | Concrete interface or type alias |
| Shape varies by caller | Generic `<T>` |
| Shape is known, source untrusted | `unknown` + Zod/validation |
| Shape is genuinely unknowable | `unknown` (never `any`) |
| Third-party type is wrong/missing | Declaration merging or `@types` patch |
| Callback param inferred by context | Remove annotation entirely |
| Error catch binding | `unknown` + `instanceof Error` guard |

---

## Checklist Before Calling It Done

- [ ] `grep -r ": any"` returns zero results in scope
- [ ] `grep -r "as any"` returns zero results (or each is documented with a `// eslint-disable` explaining why)
- [ ] `tsc --noEmit --strict` passes
- [ ] No new `@ts-ignore` or `@ts-expect-error` comments introduced as a workaround
- [ ] Every new interface/type is exported if used across files
- [ ] Validation boundaries (API responses, JSON.parse) use `unknown`, not a naked type assertion