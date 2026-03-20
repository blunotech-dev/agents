---
name: error-handling-test
description: Write tests for error paths, including exceptions, rejected promises, fallback behavior, and invalid inputs. Use when the user wants to cover failure modes, edge cases, or “unhappy paths.”
category: "Testing"
---

# Error Handling Test Skill

## Discovery

Before writing any tests, scan the target code for:

- **Explicit throws** — `throw new Error(...)`, custom error classes, re-throws
- **Implicit throws** — array/object destructuring on null, property access on undefined, calling non-functions
- **Async failure surfaces** — `Promise.reject`, `async` functions that `throw`, unhandled branches in `.then()` chains
- **Fallbacks** — default values, catch blocks that return something, optional chaining with fallback (`?? default`), try/catch that swallows errors silently
- **Error propagation** — does the function rethrow, wrap, or consume the error? Each behavior needs a different assertion

## Non-Obvious Patterns to Cover

### 1. Assert the error *type and message*, not just that something throws

```ts
// WEAK — only verifies something threw
expect(() => fn()).toThrow();

// STRONG — verifies what threw and why
expect(() => fn(null)).toThrow(ValidationError);
expect(() => fn(null)).toThrow('userId is required');
```

For custom error classes, assert the `.code` or `.statusCode` property too if it exists.

---

### 2. Async: use `rejects` not try/catch

```ts
// WRONG — silent pass if promise resolves instead of rejects
try {
  await riskyFn();
  fail('should have thrown');
} catch (e) { ... }

// CORRECT
await expect(riskyFn()).rejects.toThrow(NetworkError);
await expect(riskyFn()).rejects.toMatchObject({ code: 'ECONNREFUSED' });
```

---

### 3. Verify fallback *values*, not just absence of errors

When a function catches internally and returns a default, test the fallback explicitly:

```ts
// Don't just test that it doesn't throw
const result = await fetchWithFallback('/bad-url');
expect(result).toEqual(DEFAULT_CONFIG); // assert the fallback was used
```

---

### 4. Spy on error consumers — catch blocks that log or emit

If error handling calls `logger.error`, `Sentry.captureException`, or emits an event, assert those side effects:

```ts
const spy = jest.spyOn(logger, 'error');
await riskyOperation();
expect(spy).toHaveBeenCalledWith(
  expect.stringContaining('fetch failed'),
  expect.any(Error)
);
```

---

### 5. Error boundary / retry exhaustion

For retry logic, don't just mock a single failure — mock *n* consecutive failures to verify exhaustion behavior:

```ts
fetchMock.mockRejectedValue(new Error('timeout')).mockRejectedValueOnce(...);
// or use mockRejectedValue for all calls then assert maxRetries hit
expect(fetchMock).toHaveBeenCalledTimes(3); // assert retry count
await expect(result).rejects.toThrow('Max retries exceeded');
```

---

### 6. Partial failure in parallel operations

`Promise.allSettled` vs `Promise.all` behave differently — test accordingly:

```ts
// for Promise.all — one rejection should reject the whole call
mockFn.mockResolvedValueOnce('ok').mockRejectedValueOnce(new Error('fail'));
await expect(Promise.all([mockFn(), mockFn()])).rejects.toThrow('fail');

// for Promise.allSettled — partial failure should not throw
const results = await Promise.allSettled([mockFn(), mockFn()]);
expect(results[1].status).toBe('rejected');
```

---

### 7. Error swallowing — the hidden failure mode

If a catch block does nothing (or only logs), write a test that confirms the *caller* receives the expected neutral result — not that an error was thrown:

```ts
// The function eats the error; caller gets undefined/null/empty
const result = await silentFail();
expect(result).toBeNull(); // or undefined, [], {} — whatever the contract says
```

---

### 8. State integrity after failure

For stateful modules, assert that state is unchanged (or rolled back) after a failed operation:

```ts
const before = store.getSnapshot();
await expect(store.update(invalidPayload)).rejects.toThrow();
expect(store.getSnapshot()).toEqual(before); // no partial mutation
```

---

## Strategy by Error Surface

| Surface | Key assertion | Mock strategy |
|---|---|---|
| Sync throw | `toThrow(ErrorClass)` + message | Pass invalid args directly |
| Async reject | `rejects.toThrow()` | `mockRejectedValue` |
| Fetch / HTTP | Status code + error shape | `msw` handler or `fetch` mock returning `{ ok: false, status: 500 }` |
| DB / IO | Connection error, query error | Mock at adapter layer, not at `fetch` |
| Silent catch | Fallback value assertion | Let real error occur, assert return value |
| Event-based errors | Event listener spy | `emitter.emit('error', ...)` or trigger condition |

---

## What Not to Do

- **Don't `console.log` in tests to "verify" an error happened** — use spies
- **Don't use `try/catch` in async tests** — use `rejects`
- **Don't test the mock** — if you mock `throw` and then assert `throw`, you've tested nothing; ensure the production code path is exercised
- **Don't group all error cases in one test** — one assertion per test makes failures legible

---

## Output Format

Produce test blocks grouped by the *type* of error surface (thrown errors, rejected promises, fallbacks, side effects), with one `it`/`test` per distinct failure condition. Include a brief comment on each test block explaining *what contract is being verified*, not just what the code does.