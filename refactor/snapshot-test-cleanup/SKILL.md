---
name: snapshot-test-cleanup
description: Replace brittle snapshot tests with focused, meaningful assertions that verify behavior. Use when improving test reliability, fixing flaky tests, or migrating away from `toMatchSnapshot` / `toMatchInlineSnapshot`.
category: "Refactoring"
---

# snapshot-test-cleanup

A structured refactoring skill for replacing snapshot tests with targeted,
meaningful assertions — so tests catch real regressions instead of
rubber-stamping every render.

---

## Why Snapshots Become Brittle

Snapshot tests fail for two reasons: **real bugs** and **irrelevant changes**.
The problem is they can't tell which is which. A snapshot fails when:

- A classname was renamed
- A dependency bumped its version and changed whitespace
- A dev ran `--updateSnapshot` to silence CI without reading the diff
- An actual regression occurred

When a test suite cries wolf on every PR, reviewers stop reading failures.
That's when snapshots become **worse than no tests** — they create false
confidence while missing real bugs.

---

## Phase 1 — Triage the Snapshot Suite

### 1.1 Classify existing snapshots

Before rewriting anything, read each snapshot and label it:

| Class | Signal | Action |
|---|---|---|
| **Structure snapshot** | Full component tree, 50+ lines | Replace with targeted assertions |
| **Content snapshot** | Checks rendered text or labels | Replace with `getByText` / `toHaveTextContent` |
| **Style snapshot** | Checks classNames or inline styles | Replace with `toHaveClass` or delete if cosmetic |
| **Behaviour snapshot** | Snapshot taken after user interaction | Replace with interaction + state assertion |
| **Serialised data snapshot** | JSON or API response shape | Keep if schema is stable; otherwise use property assertions |

### 1.2 Identify snapshot update patterns

Check git history: if a snapshot file has been updated more than ~3 times in the
last 6 months with commit messages like "update snapshots" or "fix snapshot" —
it is not catching bugs. Those are the highest-priority targets.

```bash
git log --oneline -- **/__snapshots__/*.snap | grep -i "update\|snapshot\|fix"
```

---

## Phase 2 — Replacement Strategies

### 2.1 Full component tree snapshots → targeted structural assertions

Snapshots of entire component trees are the most brittle. Replace with assertions
on the specific structure that actually matters.

```ts
// Before
it('renders the user card', () => {
  const { container } = render(<UserCard user={mockUser} />)
  expect(container).toMatchSnapshot()
})

// After
it('renders the user card', () => {
  render(<UserCard user={mockUser} />)
  expect(screen.getByRole('heading', { name: mockUser.name })).toBeInTheDocument()
  expect(screen.getByText(mockUser.email)).toBeInTheDocument()
  expect(screen.getByRole('img', { name: /avatar/i })).toHaveAttribute('src', mockUser.avatarUrl)
})
```

**Rule**: assert on meaning, not markup. Ask "what would break if this component
stopped working?" — those are the assertions to write.

---

### 2.2 Text content snapshots → explicit text matchers

```ts
// Before
expect(screen.getByTestId('welcome-message')).toMatchSnapshot()

// After
expect(screen.getByTestId('welcome-message')).toHaveTextContent(
  `Welcome back, ${user.name}`
)
```

For dynamic content, assert on the parts that are semantically meaningful:

```ts
// Before — breaks if punctuation or whitespace changes
expect(container.textContent).toMatchSnapshot()

// After — checks what the user actually reads
expect(screen.getByRole('status')).toHaveTextContent(/3 items selected/i)
```

---

### 2.3 Conditional rendering snapshots → presence/absence assertions

```ts
// Before
it('hides the delete button for non-admins', () => {
  render(<ActionBar user={guestUser} />)
  expect(screen.getByTestId('action-bar')).toMatchSnapshot()
})

// After
it('hides the delete button for non-admins', () => {
  render(<ActionBar user={guestUser} />)
  expect(screen.queryByRole('button', { name: /delete/i })).not.toBeInTheDocument()
})

it('shows the delete button for admins', () => {
  render(<ActionBar user={adminUser} />)
  expect(screen.getByRole('button', { name: /delete/i })).toBeInTheDocument()
})
```

Note: one snapshot that encoded both conditions is now two focused tests — each
with a clear name and a single assertion.

---

### 2.4 Interaction snapshots → fire event + assert outcome

```ts
// Before
it('opens the dropdown on click', async () => {
  render(<Select options={opts} />)
  await userEvent.click(screen.getByRole('button'))
  expect(document.body).toMatchSnapshot()
})

// After
it('opens the dropdown on click', async () => {
  render(<Select options={opts} />)
  await userEvent.click(screen.getByRole('combobox'))
  expect(screen.getByRole('listbox')).toBeVisible()
  expect(screen.getAllByRole('option')).toHaveLength(opts.length)
})
```

---

### 2.5 Error / loading state snapshots → state-specific assertions

```ts
// Before
it('shows loading state', () => {
  render(<DataTable loading />)
  expect(container).toMatchSnapshot()
})

// After
it('shows loading state', () => {
  render(<DataTable loading />)
  expect(screen.getByRole('progressbar')).toBeInTheDocument()
  expect(screen.queryByRole('table')).not.toBeInTheDocument()
})
```

---

### 2.6 Serialised data / API response snapshots — when to keep them

Not all snapshots are bad. A snapshot is appropriate when:

- The output is a **stable serialised format** (a generated SQL query, a config
  file, a CLI output) that should be reviewed when it changes
- The object has **10+ fields** and you want to assert on all of them — a
  snapshot is clearer than 10 separate `expect` lines
- It is a **pure transformation** (no DOM, no side effects) with well-understood
  input/output

When keeping, make the snapshot small and focused:

```ts
// Before — snapshots the entire API response (200+ lines)
expect(response).toMatchSnapshot()

// After — snapshots only the shape that matters for this test
expect(pick(response, ['id', 'status', 'items'])).toMatchSnapshot()
```

---

### 2.7 Inline snapshots → convert or delete

`toMatchInlineSnapshot` is slightly better (the snapshot is visible in the test
file) but still brittle. Apply the same strategies above. Delete the inline
snapshot if the test already has meaningful assertions and the snapshot adds
nothing.

---

## Phase 3 — Test Quality Checklist Per Converted Test

After rewriting each test, verify:

- [ ] The test name describes **behaviour**, not implementation (`'shows error when email is invalid'` not `'renders correctly'`)
- [ ] Assertions use **semantic queries** — `getByRole`, `getByLabelText`, `getByText` — not `getByTestId` or `querySelector`
- [ ] Each `it` block tests **one behaviour** — if it has 10 assertions, consider splitting
- [ ] The test would **fail if the feature broke** (delete the assertion mentally — would a bug slip through?)
- [ ] The test would **pass after a harmless refactor** (rename a CSS class, reorder DOM siblings)

---

## Phase 4 — Output Format

### Per-test diff

Show before/after for each converted test, grouped by file:

```
src/components/UserCard.test.tsx
  'renders the user card'
    removed: toMatchSnapshot()
    added:   getByRole('heading'), toHaveTextContent, toHaveAttribute

  'shows error state'
    removed: toMatchSnapshot()
    added:   getByRole('alert'), toHaveTextContent(/required/i)
```

### Snapshot files to delete

List `.snap` files that are now empty or fully superseded. These should be
deleted from the repo — an empty snapshot file still creates noise.

```bash
# After conversion, find and remove empty snapshot files
find . -name "*.snap" -empty -delete
```

### Tests that were split

Note any single snapshot tests that became multiple focused tests — this
increases the test count intentionally and is a feature, not scope creep.

### Skipped snapshots (with rationale)

If any snapshots were left in place, document why:

```
UserCard.test.tsx — serialised CSV export snapshot kept: stable format,
no DOM involved, reviewed on change.
```

---

## Decision Reference

| What the snapshot was testing | Replace with |
|---|---|
| Component renders without crashing | `render()` with no assertion (smoke test) |
| Correct text is displayed | `toHaveTextContent` / `getByText` |
| Element is present / absent | `toBeInTheDocument` / `not.toBeInTheDocument` |
| Element is visible / hidden | `toBeVisible` / `not.toBeVisible` |
| Correct number of items | `toHaveLength` on `getAllByRole` |
| Correct attribute or prop | `toHaveAttribute` |
| Correct CSS class | `toHaveClass` |
| Input value | `toHaveValue` |
| Button is disabled | `toBeDisabled` |
| Form validation message | `getByRole('alert')` + `toHaveTextContent` |
| State after interaction | `userEvent` + state-specific assertion |
| Serialised stable output | Keep snapshot, but scope it tightly |

---

## Checklist Before Calling It Done

- [ ] No `toMatchSnapshot()` calls remain except intentionally kept ones (documented)
- [ ] All remaining snapshot files either have a written rationale or are deleted
- [ ] Every converted test has a descriptive name that reflects behaviour
- [ ] `--updateSnapshot` was not run — all passing tests pass on first run
- [ ] CI passes without any snapshot update commits