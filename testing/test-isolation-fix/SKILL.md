---
name: test-isolation-fix
description: Fix test isolation by removing shared state, global side effects, and order dependencies. Use when tests are flaky, fail in CI, or pass individually but fail together.
category: "Testing"
---

# Test Isolation Fix Skill

## Discovery

Before touching anything, run the tests in these modes to diagnose the isolation class:

```bash
# 1. Reverse order — catches forward-only order dependencies
npx jest --testSequencer ./reverse-sequencer.js

# 2. Random order (jest-random-sequencer or --randomize in Vitest)
npx vitest --sequence.shuffle

# 3. Single test in isolation
npx jest --testPathPattern="failing.test.ts" --runInBand

# 4. Full suite with --runInBand (single thread) vs default (parallel workers)
npx jest --runInBand
```

A test that fails only in parallel → shared module-level state or file system collision.
A test that fails only in reverse → writes state that a later test consumes.
A test that fails in isolation → broken setup/teardown, not an isolation issue.

---

## Isolation Failure Taxonomy

### Class 1: Shared module-level state

The most common and hardest to spot — a variable declared outside `describe`/`it` that gets mutated:

```ts
// BROKEN — all tests share and mutate this reference
const db = new Database();
let user: User;

test('creates user', async () => {
  user = await db.create({ name: 'Alice' }); // mutates outer scope
});

test('updates user', async () => {
  await db.update(user.id, { name: 'Bob' }); // depends on previous test
});
```

```ts
// FIXED — each test owns its own setup
test('creates user', async () => {
  const db = createTestDb();
  const user = await db.create({ name: 'Alice' });
  expect(user.name).toBe('Alice');
});

test('updates user', async () => {
  const db = createTestDb();
  const user = await db.create({ name: 'Alice' }); // seeds its own prerequisite
  await db.update(user.id, { name: 'Bob' });
  expect((await db.findById(user.id)).name).toBe('Bob');
});
```

---

### Class 2: `beforeEach` that shares mutable objects by reference

`beforeEach` re-runs the assignment but if the value is an object, tests share the same reference:

```ts
// BROKEN — all tests mutate the same object
let config: Config;
beforeEach(() => {
  config = defaultConfig; // reference, not copy
});

test('overrides timeout', () => {
  config.timeout = 5000; // mutates defaultConfig for all subsequent tests
});
```

```ts
// FIXED — deep copy or factory function
beforeEach(() => {
  config = { ...defaultConfig };       // shallow copy (fine if no nested objects)
  config = structuredClone(defaultConfig); // deep copy for nested structures
  config = buildConfig();              // factory — best for complex objects
});
```

---

### Class 3: Singleton / module cache not reset between tests

Jest caches module imports. A singleton initialized in one test persists into the next:

```ts
// BROKEN — AppState is a module-level singleton
import { AppState } from '../state';

test('sets flag', () => {
  AppState.featureEnabled = true;
});

test('flag is off by default', () => {
  expect(AppState.featureEnabled).toBe(false); // fails — previous test mutated it
});
```

```ts
// FIXED — reset in afterEach, or re-import with jest.resetModules
afterEach(() => {
  AppState.reset(); // if the module exposes a reset method
});

// OR — for deeper isolation, re-require the module fresh
beforeEach(() => {
  jest.resetModules();
  const { AppState } = require('../state'); // fresh instance
});
```

---

### Class 4: Timer and date leakage

Fake timers installed in one test bleed into the next if not torn down:

```ts
// BROKEN
test('debounce fires after 300ms', () => {
  jest.useFakeTimers();
  // ... forgot jest.useRealTimers() in teardown
});

test('fetches data on mount', async () => {
  // real setTimeout never fires — fake timers still active
  await waitFor(() => expect(fetchMock).toHaveBeenCalled()); // hangs
});
```

```ts
// FIXED — always pair useFakeTimers with teardown
beforeEach(() => jest.useFakeTimers());
afterEach(() => jest.useRealTimers());

// OR at describe level if only some tests need fake timers
describe('debounce behavior', () => {
  beforeEach(() => jest.useFakeTimers());
  afterEach(() => jest.useRealTimers());
});
```

---

### Class 5: Mock state accumulating across tests

`jest.spyOn` and `jest.fn()` accumulate call history. `clearAllMocks` vs `resetAllMocks` vs `restoreAllMocks` are not interchangeable:

| Method | Clears call history | Resets implementation | Restores original |
|---|---|---|---|
| `clearAllMocks` | ✓ | ✗ | ✗ |
| `resetAllMocks` | ✓ | ✓ | ✗ |
| `restoreAllMocks` | ✓ | ✓ | ✓ |

```ts
// In jest.config — set globally to avoid forgetting per-test
clearMocks: true,      // clears call counts between tests
resetMocks: true,      // also resets mock implementations
restoreMocks: true,    // also restores spyOn originals — use if mixing spies and mocks
```

Using `restoreMocks: true` globally is the safest default for a suite with spies — but requires re-applying spy implementations in each test.

---

### Class 6: File system and environment variable pollution

```ts
// BROKEN — writes to a shared temp file path, collides in parallel
test('exports CSV', async () => {
  await exportToFile('/tmp/test-output.csv');
  const content = fs.readFileSync('/tmp/test-output.csv', 'utf8');
});
```

```ts
// FIXED — unique path per test run
import { randomUUID } from 'crypto';

test('exports CSV', async () => {
  const path = `/tmp/test-output-${randomUUID()}.csv`;
  await exportToFile(path);
  const content = fs.readFileSync(path, 'utf8');
  fs.unlinkSync(path); // cleanup
});

// OR use a temp directory library that auto-cleans
import { mkdtempSync } from 'fs';
const tmpDir = mkdtempSync('/tmp/test-');
afterAll(() => fs.rmSync(tmpDir, { recursive: true }));
```

For environment variables:

```ts
const originalEnv = process.env;
beforeEach(() => { process.env = { ...originalEnv }; });
afterEach(() => { process.env = originalEnv; });
```

---

### Class 7: Database row ID / sequence collisions

Auto-increment IDs from one test run pollute expectations in another if the DB isn't reset:

```ts
// BROKEN — assumes ID is always 1
test('creates first user', async () => {
  const user = await repo.create({ name: 'Alice' });
  expect(user.id).toBe(1); // fails after any previous test creates a user
});
```

```ts
// FIXED — assert shape, not specific IDs
expect(user.id).toEqual(expect.any(Number));
expect(user.id).toBeGreaterThan(0);

// OR reset sequences between tests (Postgres)
await db.query("SELECT setval('users_id_seq', 1, false)");
```

---

## Refactor Checklist

When fixing a contaminated suite, work through in this order:

1. **Move all `let` declarations inside the test or `beforeEach`** — no mutable `let` in `describe` outer scope
2. **Replace reference assignments with factory calls** — `buildUser()` not `user = defaultUser`
3. **Add `afterEach` teardown for every resource opened in `beforeEach`** — timers, DB connections, file handles, server listeners
4. **Set `clearMocks`/`resetMocks`/`restoreMocks` in config** — don't rely on per-test cleanup
5. **Audit `process.env` mutations** — wrap in save/restore around the test
6. **Check for singleton imports** — any module with top-level mutable state needs reset or re-require
7. **Verify parallel safety** — replace shared file paths with unique paths, shared ports with random ports

---

## Detecting Hidden Order Dependencies

If the suite has many tests and the order dependency isn't obvious:

```bash
# Bisect: run first half, then second half, find which group the failing test needs
npx jest --testPathPattern="group-a"
npx jest --testPathPattern="group-b"

# Log test execution order to find the "poisoning" test
# In jest setup file:
beforeEach(() => console.log(`▶ ${expect.getState().currentTestName}`));
```

The poisoning test is the one that runs before the failure and mutates shared state. It often looks completely fine in isolation.

---

## What Not to Do

- **Don't fix order dependency by enforcing order** — `--runInBand` with a fixed sequence treats the symptom, not the cause; parallel execution will still fail
- **Don't use `afterAll` for cleanup that should be `afterEach`** — if a test fails mid-suite, `afterAll` never runs and state leaks into remaining tests
- **Don't share a single DB connection across all tests** — use a connection per test or a transaction-scoped connection (see `db-integration-test` skill)
- **Don't assume `beforeEach` re-creates objects** — re-assignment of a `let` variable to an object reference doesn't copy it; use a factory

---

## Output Format

For each isolation failure found: show the broken pattern, name the class of failure, then show the fixed version. Group fixes by class. After refactoring, include the diagnosis commands the user can run to verify the suite is now order-independent.