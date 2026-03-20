---
name: unit-test-generator
description: Generate unit tests covering happy paths, edge cases, boundaries, and error conditions with meaningful assertions. Use when the user asks to write tests, improve coverage, or verify behavior of a function or module.
category: "Testing"
---

# Unit Test Generator

## Discovery

Before writing a single test, extract the full contract of the target code:

- **Inputs** — types, optionality, valid ranges, format constraints
- **Outputs** — return type, shape, nullability; does it mutate args or external state?
- **Dependencies** — what does it call that needs mocking? (DB, fetch, logger, Date, Math.random)
- **Implicit assumptions** — things the function *expects* to be true but doesn't validate (e.g. "caller always passes a non-empty array")
- **Branching logic** — every `if`, ternary, `||`, `&&`, and `??` is a branch that needs at least one test each side

## Test Case Taxonomy

Map the function's contract to these four layers before writing:

| Layer | What to test | Common miss |
|---|---|---|
| Happy path | Canonical input → expected output | Only testing one example when the function has meaningful variation |
| Boundary values | Min/max of numeric ranges, empty vs single-item collections, zero | Off-by-one: test `n-1`, `n`, `n+1` around any limit |
| Edge cases | `null`, `undefined`, `""`, `0`, `NaN`, `[]`, `{}`, negative numbers | Assuming non-null is safe because TypeScript says so |
| Error conditions | Invalid input, dependency failure, constraint violations | See `error-handling-test` skill for full async/sync error patterns |

---

## Non-Obvious Patterns

### 1. Test behavior, not implementation

```ts
// WEAK — tests that a method was called (brittle, couples to internals)
expect(formatDate).toHaveBeenCalled();

// STRONG — tests the observable output
expect(result.label).toBe('Jan 5, 2024');
```

Reserve `toHaveBeenCalledWith` for side effects with no observable return value (logging, analytics, event emission).

---

### 2. Boundary values need three points, not one

For any numeric limit or collection size, test the value below, at, and above the boundary:

```ts
// limit is maxItems = 5
expect(fn([...4items])).not.toThrow();   // n-1: allowed
expect(fn([...5items])).not.toThrow();   // n: allowed (boundary)
expect(fn([...6items])).toThrow();       // n+1: rejected
```

---

### 3. Separate *what* from *how many*

Two common mistakes in one test: verifying the return value AND its count. Split them — count bugs and shape bugs fail independently:

```ts
// Split these
expect(result).toHaveLength(3);
expect(result[0]).toMatchObject({ id: '1', active: true });
```

---

### 4. Floating point: never use `toBe`

```ts
// WRONG — fails due to IEEE 754 precision
expect(0.1 + 0.2).toBe(0.3);

// CORRECT
expect(0.1 + 0.2).toBeCloseTo(0.3);
```

---

### 5. Date/time and randomness — always mock at the source

```ts
// Mock Date.now, not the function that calls it
jest.spyOn(Date, 'now').mockReturnValue(1700000000000);

// Mock Math.random for deterministic shuffle/sampling tests
jest.spyOn(Math, 'random').mockReturnValue(0.5);
```

Never rely on `new Date()` in tests — they will drift and fail in CI at midnight or timezone boundaries.

---

### 6. Async: assert resolution *shape*, not just that it resolved

```ts
// WEAK — only proves it didn't reject
await expect(fetchUser('123')).resolves.toBeDefined();

// STRONG — proves the resolved value is correct
await expect(fetchUser('123')).resolves.toMatchObject({
  id: '123',
  email: expect.stringContaining('@'),
});
```

---

### 7. Mutation tests — assert before AND after

When a function mutates state or its arguments, snapshot before calling:

```ts
const original = { count: 0 };
addItem(original, 'x');
expect(original.count).toBe(1); // mutation happened

// or assert immutability
const input = [1, 2, 3];
const result = sortedCopy(input);
expect(input).toEqual([1, 2, 3]); // input unchanged
expect(result).toEqual([1, 2, 3]); // sorted (same here, but proves no mutation)
```

---

### 8. Mock at the right layer

| Dependency type | Mock at |
|---|---|
| `fetch` / HTTP | `msw` request handler or `jest.mock('node-fetch')` |
| DB client | Repository/adapter interface, not the ORM internals |
| `fs` / file I/O | `jest.mock('fs')` or `memfs` |
| Internal module | `jest.mock('../path/to/module')` with named export control |
| Time | `jest.useFakeTimers()` + `jest.setSystemTime()` |

Never mock the function under test itself, and never mock at a deeper layer than needed — mocking `pg` internals when you should mock your own `db.query` wrapper creates fragile tests.

---

### 9. `toMatchObject` vs `toEqual` — use the right one

```ts
// toEqual — asserts exact shape, fails if response has extra fields
expect(result).toEqual({ id: '1', name: 'Alice' });

// toMatchObject — asserts subset, ignores extra fields
expect(result).toMatchObject({ id: '1' }); // passes even if result has more fields
```

Use `toEqual` when the full shape is the contract. Use `toMatchObject` when testing a subset and extra fields are acceptable.

---

### 10. Parameterize repeated structure, not logic

When testing the same behavior across multiple inputs, use `test.each` to keep tests DRY without hiding what's being tested:

```ts
test.each([
  ['empty string', '', false],
  ['whitespace only', '   ', false],
  ['valid email', 'a@b.com', true],
])('isValidEmail(%s) → %s', (_, input, expected) => {
  expect(isValidEmail(input)).toBe(expected);
});
```

Don't use `test.each` for cases that need meaningfully different assertions — write those as separate tests.

---

## Describe Block Structure

Group tests to reflect the function's contract, not its internal implementation:

```ts
describe('calculateDiscount', () => {
  describe('happy path', () => { ... });
  describe('boundary values', () => { ... });
  describe('invalid input', () => { ... }); // or defer to error-handling-test skill
});
```

Name each `it` so the failure message is a readable sentence:
- ✓ `it('returns 0 when cart is empty')`
- ✗ `it('test empty cart')`

---

## What Not to Do

- **Don't write tests that can't fail** — `expect(true).toBe(true)` or asserting a mock you just set up
- **Don't snapshot everything** — snapshot tests for pure logic are a maintenance trap; snapshot only when output shape is genuinely complex and stable
- **Don't `beforeEach` shared mutable state** — reset mocks with `jest.clearAllMocks()` and scope setup to the smallest `describe` block that needs it
- **Don't import the whole module to test one function** — import only what you're testing; side effects in module scope will bite you

---

## Output Format

Group tests in `describe` blocks by behavioral category (not file structure). Each `it` block tests exactly one behavior. Include a one-line comment above non-obvious test cases explaining *why* that case exists — not what the code does.

For error conditions, apply patterns from the `error-handling-test` skill rather than duplicating that logic here.