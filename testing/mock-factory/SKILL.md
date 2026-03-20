---
name: mock-factory
description: Create reusable mock or stub factories that generate configurable, typed fakes for tests. Use when mocks are repetitive, hard to maintain, or need per-test customization.
category: "Testing"
---

# Mock Factory

Generates a typed, reusable factory function for mocking a module, class, or interface — with per-test override support and clean reset semantics.

---

## Phase 1: Discovery

Before writing anything, extract:

1. **Target shape** — is it a TS interface, class, or CommonJS/ESM module?
2. **Test runner** — Jest or Vitest? (API is nearly identical; note differences inline)
3. **Override depth** — do tests need to override individual method return values, or whole sub-objects?
4. **Async surface** — are any methods `async` or returning `Promise`?
5. **Reset strategy** — per-test (`beforeEach`) or per-file (`beforeAll`)?

If the user pastes the interface/class, extract the shape directly. If not, ask for it.

---

## Phase 2: Shape Strategy

### Choosing the factory base

| Source type | Strategy |
|---|---|
| TS `interface` | Build object literal with `jest.fn()` per method; cast as the interface |
| `class` | Do NOT instantiate — build plain object matching the public API; avoids constructor side-effects |
| Module (named exports) | `jest.mock()` the module path, then use `jest.mocked()` to get typed access; factory wraps the setup |

**Non-obvious:** for a class with private fields that affect public behavior, note that the factory produces a structural fake — tests depending on `instanceof` checks will fail. Flag this if detected.

### Handling optional properties

Omit optional fields from the factory default. Only include them if the user's tests need them. Adding everything inflates the factory and creates false assumptions about presence.

### Overloaded methods

Pick the broadest overload signature for the mock type. Use `jest.fn() as jest.MockedFunction<typeof target.method>` to preserve call signature inference.

### Async methods

Default return should be `Promise.resolve(undefined)` or a sensible typed default — not a raw `jest.fn()` with no return. Tests that `await` an unresolved mock hang silently.

```ts
fetchUser: jest.fn().mockResolvedValue(null) as jest.MockedFunction<UserService['fetchUser']>
```

---

## Phase 3: Factory Implementation

### Core pattern

```ts
// factories/userService.factory.ts

import type { UserService } from '../services/UserService'

type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends (...args: any[]) => any
    ? jest.MockedFunction<T[K]>
    : DeepPartial<T[K]>
}

export function createMockUserService(
  overrides: DeepPartial<UserService> = {}
): jest.Mocked<UserService> {
  const defaults: jest.Mocked<UserService> = {
    getUser: jest.fn().mockResolvedValue(null),
    createUser: jest.fn().mockResolvedValue({ id: '1', name: 'Test' }),
    deleteUser: jest.fn().mockResolvedValue(undefined),
  }

  return { ...defaults, ...overrides } as jest.Mocked<UserService>
}
```

**Key decisions embedded here:**
- Shallow merge is intentional for flat service shapes; use deep merge only if the target has nested objects with their own methods
- `jest.Mocked<T>` preserves autocomplete on `.mockReturnValue`, `.mockImplementation`, etc.
- The factory file lives next to the test or in a shared `__mocks__` / `factories/` dir — ask the user which

### Module-level mock pattern (when mocking imports)

```ts
// At top of test file — must be hoisted
jest.mock('../services/UserService')

import { UserService } from '../services/UserService'
const MockUserService = jest.mocked(UserService)

function createMockInstance(overrides = {}) {
  const instance = {
    getUser: jest.fn().mockResolvedValue(null),
    ...overrides,
  }
  MockUserService.mockImplementation(() => instance as unknown as UserService)
  return instance
}
```

**Non-obvious:** `jest.mock()` hoisting means the import above it still works due to Babel/ts-jest transform order. Do not move the `jest.mock()` call.

---

## Phase 4: Reset and Override Patterns

### Reset without reconstruction

Prefer `mockReset()` over rebuilding the factory each test — it clears calls and return values but keeps the reference stable (important if the mock is injected once into a class under test).

```ts
let mockService: jest.Mocked<UserService>

beforeAll(() => {
  mockService = createMockUserService()
  // inject into SUT once
})

beforeEach(() => {
  jest.resetAllMocks()
  // re-apply defaults if needed
  mockService.getUser.mockResolvedValue(null)
})
```

**When to use `mockReset` vs `mockClear` vs `mockRestore`:**
- `mockClear` — clears call history only; return values persist. Use when asserting call counts per-test but sharing behavior.
- `mockReset` — clears calls + return values. Use when each test sets its own behavior.
- `mockRestore` — only relevant for `jest.spyOn`; restores original implementation. Not applicable to factory-built mocks.

### Per-test overrides (non-destructive)

```ts
it('handles not found', async () => {
  mockService.getUser.mockResolvedValueOnce(null) // one-time override
  // rest of test
})
```

`mockResolvedValueOnce` stacks — multiple calls consume overrides in order, then fall back to the default `mockResolvedValue`. Use this instead of replacing the mock entirely.

---

## Phase 5: Output

Produce:

1. **Factory file** — typed, exportable, with sane defaults for all async methods
2. **Usage example** — one test file showing construction, per-test override, and reset
3. **If module mock:** the hoisted `jest.mock()` block + `jest.mocked()` usage

### Output notes

- Co-locate factory with tests unless the user has a shared `factories/` pattern already
- Use `jest.Mocked<T>` not `Partial<T>` — the former gives `.mock.calls` access
- If Vitest: replace `jest.fn()` with `vi.fn()`, `jest.Mocked<T>` with `vi.Mocked<T>` — rest is identical
- Do not generate barrel exports for factories unless the user has >3 factories and asks for it
