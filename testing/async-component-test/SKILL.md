---
name: async-component-test
description: Test async UI components (e.g., React) covering loading, success, error, and retry states using proper async queries. Use when testing data-fetching components or avoiding act-related issues.
category: "Testing"
---

# Async Component Test

Generates tests for components with async data lifecycle — covering loading, success, error, and retry states without timing hacks or act() warnings.

---

## Phase 1: Discovery

Extract before writing:

1. **Framework** — React? Vue? (this skill defaults to React + Testing Library)
2. **Data fetching mechanism** — `useEffect` + `fetch`, React Query, SWR, Apollo, or server component?
3. **Test runner** — Jest + jsdom, or Vitest + jsdom/happy-dom?
4. **How fetches are intercepted** — `jest.mock` on a service module, `msw` (Mock Service Worker), or `jest.spyOn(global, 'fetch')`?
5. **Does the component have a retry interaction** — a button, a link, or auto-retry?
6. **Suspense boundaries** — is the component wrapped in `<Suspense>`? (changes how loading state is tested)

Default to `msw` if no interceptor preference is stated — it's the least brittle option.

---

## Phase 2: Architecture Decisions

### Interceptor strategy tradeoffs

| Approach | Non-obvious issue |
|---|---|
| `jest.mock` on service module | Breaks if the component imports fetch directly; mock must be reset per-test or tests bleed |
| `jest.spyOn(global, 'fetch')` | Works everywhere but response shape must be manually constructed; `json()` must be a function, not a value |
| `msw` (recommended) | Intercepts at the network level; works regardless of how the component fetches; requires `server.resetHandlers()` in `afterEach` |

### The `act()` warning problem

`act()` warnings mean the component updated state after the test ended. Root causes:

- Using `getBy` (synchronous) before the async operation resolves — use `findBy` instead
- `waitFor` resolving before all state updates flush — wrap the full assertion, not just the element query
- Unmounted component still has a pending promise — cancel effects in cleanup or use `msw` which handles this gracefully

**Non-obvious:** `findByText` is `waitFor(() => getByText(...))` — it already polls. Don't nest it inside another `waitFor`. That's a common pattern that creates double-polling and masks timing issues.

```ts
// Wrong — double polling
await waitFor(() => expect(await screen.findByText('Loaded')).toBeInTheDocument())

// Right — pick one
await screen.findByText('Loaded')
// or
await waitFor(() => expect(screen.getByText('Loaded')).toBeInTheDocument())
```

---

## Phase 3: Test Case Coverage

### Loading state

Test that the loading indicator appears *before* the fetch resolves, not after. This requires controlling when the mock resolves:

```ts
it('shows a loading spinner while fetching', async () => {
  // Delay the response so we can assert the loading state
  server.use(
    http.get('/api/users', async () => {
      await delay('infinite') // msw helper — never resolves until test ends
      return HttpResponse.json([])
    })
  )

  render(<UserList />)
  expect(screen.getByRole('status')).toBeInTheDocument() // spinner
})
```

**Non-obvious:** if you don't delay the response, the fetch resolves before the first render assertion runs and you never actually see the loading state. Tests that skip this pass vacuously.

For `jest.mock` approach, use a never-resolving promise:

```ts
jest.spyOn(userService, 'getUsers').mockReturnValue(new Promise(() => {})) // never resolves
```

Restore with `jest.restoreAllMocks()` in `afterEach` — a never-resolving mock left in place will cause subsequent tests to hang.

### Success state

```ts
it('renders users after fetch resolves', async () => {
  server.use(
    http.get('/api/users', () =>
      HttpResponse.json([{ id: '1', name: 'Alice' }, { id: '2', name: 'Bob' }])
    )
  )

  render(<UserList />)

  expect(await screen.findByText('Alice')).toBeInTheDocument()
  expect(screen.getByText('Bob')).toBeInTheDocument() // synchronous after findBy resolves
  expect(screen.queryByRole('status')).not.toBeInTheDocument() // spinner gone
})
```

**Non-obvious:** after the first `await findBy` resolves, the component is done rendering. Subsequent `getBy` calls in the same test are synchronous and don't need `await`. Over-awaiting is a red flag in test output — it means the author didn't understand the query semantics.

Also assert that the loading indicator is **gone** in the success test. Many tests forget this and miss regressions where the spinner never clears.

### Error state

```ts
it('shows an error message when fetch fails', async () => {
  server.use(
    http.get('/api/users', () => HttpResponse.error())
  )

  render(<UserList />)

  expect(await screen.findByRole('alert')).toBeInTheDocument()
  expect(screen.getByText(/failed to load/i)).toBeInTheDocument()
  expect(screen.queryByRole('status')).not.toBeInTheDocument() // spinner cleared
})
```

**Non-obvious:** `HttpResponse.error()` simulates a network failure (no response). To test HTTP error status codes (404, 500), use `HttpResponse.json({ error: '...' }, { status: 500 })` — these are different code paths in the component and need separate tests if the component handles them differently.

### Retry interaction

```ts
it('retries the fetch when the retry button is clicked', async () => {
  // First call fails, second succeeds
  server.use(
    http.get('/api/users', ({ request }) => {
      // msw doesn't have built-in call counting — track externally
    })
  )

  let callCount = 0
  server.use(
    http.get('/api/users', () => {
      callCount++
      if (callCount === 1) return HttpResponse.error()
      return HttpResponse.json([{ id: '1', name: 'Alice' }])
    })
  )

  render(<UserList />)
  await screen.findByRole('alert') // wait for error state

  await userEvent.click(screen.getByRole('button', { name: /retry/i }))

  expect(await screen.findByText('Alice')).toBeInTheDocument()
  expect(screen.queryByRole('alert')).not.toBeInTheDocument()
})
```

**Non-obvious:** use `userEvent.click` not `fireEvent.click` for interactive elements — `userEvent` runs the full event chain (pointerdown, mousedown, click) which is needed if the component uses pointer event handlers. `fireEvent.click` is a shortcut that skips those and can miss bugs.

Also assert that the error state **clears** after successful retry — not just that the data appears.

### Suspense boundary (if applicable)

When the component is wrapped in `<Suspense fallback={<Spinner />}>`:

```ts
it('shows the Suspense fallback while suspended', async () => {
  render(
    <Suspense fallback={<div role="status">Loading...</div>}>
      <UserList />
    </Suspense>
  )

  expect(screen.getByRole('status')).toBeInTheDocument()
  expect(await screen.findByText('Alice')).toBeInTheDocument()
})
```

**Non-obvious:** do NOT wrap `<Suspense>` in your test render if the component already uses it internally — you'll get a double-fallback and the test assertions won't match real behavior. Render the component exactly as it appears in your app tree.

---

## Phase 4: Test Structure

```ts
describe('UserList', () => {
  describe('loading state', () => { /* spinner visible, data absent */ })
  describe('success state', () => { /* data visible, spinner gone */ })
  describe('error state', () => { /* error message visible, spinner gone */ })
  describe('retry', () => { /* error clears, data appears after click */ })
})
```

### msw setup (if used)

```ts
// test-utils/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers' // shared defaults

export const server = setupServer(...handlers)

// In jest setup file (jest.setup.ts):
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

**Non-obvious:** `onUnhandledRequest: 'error'` is the right default — it fails tests loudly when a component makes a request with no handler, instead of silently returning nothing. The silent behavior masks missing test setup.

---

## Phase 5: Output

Produce:

1. **Test file** — all four states covered, correct query types per async context
2. **msw handler setup** (if applicable) — per-test overrides using `server.use()`
3. **Notes on any `act()` warning sources** if detected in the user's existing tests

### Output notes

- `findBy` for the first async assertion, `getBy` for subsequent ones in the same test
- `queryBy` only for asserting absence (`not.toBeInTheDocument()`)
- Never use `waitFor` with `findBy` inside it
- `userEvent` over `fireEvent` for all user interactions
- If Vitest: same API — Testing Library is framework-agnostic; only the test runner setup differs
- For React Query / SWR: wrap renders in their provider; assert using the same `findBy` pattern — the fetching mechanism doesn't change query semantics