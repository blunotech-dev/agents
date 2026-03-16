---
name: naming-conventions
description: Audit and refactor unclear, abbreviated, or misleading identifiers across a codebase, including variables, functions, parameters, files, and modules. Use this skill when the user asks to rename things, improve naming, audit identifiers, or fix cryptic names like `d`, `tmp`, `handleStuff`, or `doTheThing`. Also trigger for requests like "better names", "clearer code", or when poor naming likely causes readability issues.
category: "Refactoring"
---

# Naming Conventions Refactor Skill

Audit variables, functions, parameters, and files with unclear, abbreviated, or misleading names and replace them with consistent, intent-revealing alternatives.

---

## When to Apply This Skill

Trigger on any of:
- Explicit rename requests: "rename these variables", "better function names", "fix naming"
- Readability complaints where naming is the root cause: "this is confusing", "hard to follow"
- Code review prep: "clean this up before review"
- Onboarding improvements: "new devs can't understand this codebase"
- Audit requests: "audit naming in this file/module/repo"

---

## Step 1: Audit — Identify Naming Problems

Scan the provided code or files and flag names by category:

### Category A: Cryptic Single-Letter / Ultra-Short Names
Names that carry zero semantic content outside a tight loop index.
- `d`, `x`, `n`, `v`, `e`, `r`, `cb`, `fn`, `tmp`, `res`, `req`, `val`, `obj`, `arr`
- **Exception**: `i`, `j`, `k` in simple numeric `for` loops are idiomatic — leave them unless the loop is complex.

### Category B: Vague or Generic Names
Names that describe *form* but not *purpose*.
- `data`, `info`, `stuff`, `thing`, `item`, `element`, `value`, `result`, `output`, `payload`
- Functions: `handleStuff`, `doTheThing`, `process`, `manage`, `update`, `run`, `execute`, `go`

### Category C: Misleading or Lying Names
Names that actively contradict what the code does.
- `isValid` that returns an error message
- `userList` that holds a `Map`
- `fetchData` that also transforms and writes
- `temp` on a variable that lives for the entire module lifecycle

### Category D: Inconsistent Naming Patterns
Mixed conventions within the same codebase or file.
- `getUser` / `fetchUserData` / `loadUser` — three verbs for the same operation
- `userId` / `user_id` / `uid` — three formats for the same concept
- `isLoading` / `loading` / `loadingState` — inconsistent boolean prefix

### Category E: Overly Abbreviated Domain Terms
Jargon compressed past recognition.
- `usr`, `acct`, `cfg`, `auth` (as a boolean), `msg`, `evt`, `ctx` (sometimes fine, sometimes not — context-dependent)
- `calcTax` → `calculateTax`, `mgr` → `manager`

---

## Step 2: Propose Renames

For each flagged name, produce a rename proposal in this format:

```
[CATEGORY] `oldName` → `newName`
Location: <file>:<line> or description of scope
Reason: <one sentence explaining the problem and the intent the new name reveals>
```

### Rename Principles

**Variables — name the concept, not the container**
- Ask: *what does this hold and why does this code care about it?*
- `d` → `invoiceDueDate`, `tmp` → `filteredUserIds`, `res` → `apiResponse`

**Booleans — use positive, present-tense predicates**
- Prefix: `is`, `has`, `can`, `should`, `was`, `needs`
- `flag` → `isEmailVerified`, `check` → `hasAdminRole`, `active` → `isSubscriptionActive`
- Avoid double negatives: `isNotDisabled` → `isEnabled`

**Functions — verb + noun, encode the operation and subject**
- CRUD: `get`, `fetch`, `create`, `update`, `delete`, `remove`
- Transforms: `format`, `parse`, `normalize`, `serialize`, `map`
- Predicates: `is…`, `has…`, `can…`
- `handleStuff(user)` → `updateUserProfile(user)`
- `doTheThing()` → `recalculateInvoiceTotals()`
- `process(order)` → `validateAndSubmitOrder(order)`

**Functions — encode side effects**
- Pure transforms: `formatCurrency(amount)` — no verb prefix needed
- Side-effectful: `saveUserToDatabase(user)`, `sendWelcomeEmail(user)` — verb must communicate the effect

**Parameters — be specific, not generic**
- `(data, cb)` → `(userRecord, onSuccess)`
- `(x, y)` in business logic → `(sourceAccountId, targetAccountId)`
- `(opts)` → `(requestOptions)` or destructure to named params

**Files and Modules — match the primary export**
- `utils.js` → `dateFormatters.js` or `currencyHelpers.js`
- `helpers.ts` → `authTokenHelpers.ts`
- `index.js` re-exports are fine to stay as `index`

**Consistent Verb Vocabulary (pick one per codebase)**

| Operation | Preferred | Avoid mixing with |
|-----------|-----------|-------------------|
| Read from API | `fetch` | `get`, `load`, `retrieve` |
| Read from store/DB | `get` | `fetch`, `read` |
| Write | `save` | `persist`, `store`, `write` |
| Remove | `delete` | `remove`, `destroy`, `clear` |
| Transform | `format` | `transform`, `convert`, `parse` (use `parse` only for string→type) |

> If the codebase already has an established verb pattern, surface it and match it — don't impose a new one.

---

## Step 3: Group and Prioritize Output

Present proposals in priority order:

1. **Misleading names** (Category C) — fix these first, they cause bugs
2. **Cryptic names in non-trivial scope** (Category A, B) — high readability cost
3. **Inconsistent patterns** (Category D) — fix file-by-file or module-by-module
4. **Abbreviations** (Category E) — lowest urgency unless in public API

---

## Step 4: Apply Renames (if asked)

When the user confirms proposals and asks to apply:

1. **Check for all usages** — rename every reference: declaration, call sites, imports, JSDoc/comments, test files.
2. **Preserve exports and public API** — if a name is exported/public, note the breaking change and ask before renaming.
3. **Update comments** — stale comments that reference the old name should be updated.
4. **One rename per commit message suggestion** — if the user wants commit granularity, group by file or by concept.

Output the refactored code in full (or as diffs if the file is large).

---

## Step 5: Summary Report

After auditing or applying, produce a brief summary:

```
## Naming Audit Summary

Files reviewed: X
Names flagged: Y
  - Cryptic/short: N
  - Vague/generic: N
  - Misleading: N
  - Inconsistent: N
  - Abbreviated: N

Renames applied: Z
Deferred (breaking/exported): W — listed below with notes
```

---

## Language-Specific Notes

**JavaScript / TypeScript**
- `camelCase` for variables and functions; `PascalCase` for classes/components/types
- Boolean props in React: `isOpen`, `hasError`, not `open`, `error` (ambiguous with event handlers)
- Event handlers: `onSubmit`, `handleSubmit` — pick one convention per codebase

**Python**
- `snake_case` for variables and functions; `UPPER_SNAKE` for module-level constants
- Avoid shadowing builtins: `id`, `type`, `list`, `dict`, `input`

**General**
- Never use a name that's also a reserved word or a common library export name in that language/ecosystem
- Acronyms: treat as words in camelCase — `userId` not `userID`, `htmlParser` not `HTMLParser` (unless your codebase already uses the latter consistently)

---

## Common Anti-Patterns to Call Out Explicitly

| Anti-pattern | Example | Fix |
|---|---|---|
| Type in name | `userArray`, `nameString` | `users`, `name` |
| Numbered siblings | `data1`, `data2` | `primaryMetrics`, `secondaryMetrics` |
| Negated booleans | `notLoggedIn`, `isNotValid` | `isLoggedOut`, `isInvalid` |
| Redundant context | `user.userName` | `user.name` |
| Too long | `theCurrentlyAuthenticatedUserObjectFromSession` | `currentUser` |
| Too short in complex scope | `e` (an error object in a 50-line catch block) | `networkError` |