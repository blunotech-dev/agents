---
name: long-function-split
description: Refactor long or multi-purpose functions into smaller, single-responsibility functions with clear names. Use when a function is large, complex, hard to read/test, or when the user asks to split, clean up, or reduce function complexity. Also trigger if pasted code contains an obviously large function.
category: "Refactoring"

---

# Long Function Split

Refactor functions that do too many things into a cluster of smaller, well-named,
single-purpose functions. The goal is code that reads like a table of contents —
each function name tells you *what* happens; the body tells you *how*.

---

## When to Apply This Skill

Trigger on any of:
- Function body ≥ 100 lines
- Multiple levels of abstraction in one body (e.g., validation + DB query + formatting + email send)
- Deep nesting (3+ levels of `if`/`for`/`try`)
- Comments like `// Step 1:`, `// Part 2:` acting as section dividers — these are extraction hints
- Multiple `return` paths that each do substantial work
- Hard-to-unit-test function (too many concerns to mock cleanly)

---

## Step 1 — Read and Annotate the Function

Before touching code, fully read the function and mentally annotate it:

1. **Identify abstraction levels.** Mark each logical chunk: is it *orchestration* (deciding what to call) or *implementation* (doing the actual work)?
2. **Find natural seams.** Look for blank lines, comments, variable reassignments, or shifts in subject matter — these are extraction boundaries.
3. **Name the chunks informally.** Before writing code, write out what each extracted function will be called and what it returns. This forces clarity.

**Abstraction level heuristic:**

| What it does | Level |
|---|---|
| Orchestrates a workflow — calls other functions in sequence | High (stays in orchestrator) |
| Validates/sanitizes input | Low (extract to `validateX`) |
| Transforms or maps data | Low (extract to `transformX` / `formatX`) |
| Handles a specific error or edge case | Low (extract to `handleX`) |
| Performs I/O (DB, network, file) | Low (extract to `fetchX` / `saveX`) |

A well-refactored function should contain **only one abstraction level** in its body.

---

## Step 2 — Plan the Decomposition

Write out the decomposition plan *before* generating code. Show the user:

```
Original: processOrder(order) — 140 lines
↓
orchestrator:   processOrder(order)            — calls the 5 below
extracted:      validateOrderInput(order)      — returns validated order or throws
extracted:      calculateOrderTotals(order)    — returns { subtotal, tax, total }
extracted:      checkInventoryAvailability(items) — returns unavailable items[]
extracted:      persistOrderToDatabase(order)  — returns saved order with ID
extracted:      sendOrderConfirmationEmail(order, customer) — returns void
```

Ask the user to confirm or adjust this plan before writing any code. A 30-second check
here prevents a full rewrite later.

---

## Step 3 — Extraction Rules

Follow these rules when writing the extracted functions:

### Naming
- Use **verb + noun** format: `calculateTax`, `validateEmail`, `formatUserProfile`
- Name from the *caller's perspective*, not the implementation: `getEligibleUsers` not `filterByStatusAndTierAndDate`
- Avoid vague names: no `process`, `handle`, `doStuff`, `helper`
- Boolean-returning functions: `is`/`has`/`can` prefix — `isEligible`, `hasPermission`

### Signatures
- Keep parameter count ≤ 3. If you need more, pass an object and destructure.
- Don't pass the entire parent scope as a "context blob" unless truly necessary.
- Prefer returning values over mutating arguments.

### Size targets
- Orchestrator: 10–25 lines (mostly function calls + light glue logic)
- Leaf functions: 5–30 lines
- If an extracted function is still >40 lines, recurse — apply this skill again.

### Side effects
- Isolate side-effectful operations (I/O, mutations, logging) into their own functions.
- Pure transformation functions should have zero side effects.
- Name side-effectful functions with verbs that signal mutation: `save`, `send`, `update`, `delete`.

### Error handling
- Don't duplicate try/catch in every extracted function.
- Prefer letting errors bubble up to a single handler in the orchestrator, unless a specific function needs to catch-and-transform.

---

## Step 4 — Write the Refactored Code

Output the refactored code with:

1. **The orchestrator function first** — reader sees the high-level flow immediately.
2. **Extracted functions below**, in call order.
3. **No behavior changes.** The refactor must be semantically equivalent. Flag any cases where equivalence is unclear.
4. **Preserve all existing comments** that explain *why* (not just *what*). Discard comments that now just restate the function name.

### Output format

```
// ─── Orchestrator ───────────────────────────────────────────
function processOrder(order) {
  ...
}

// ─── Helpers ────────────────────────────────────────────────
function validateOrderInput(order) {
  ...
}

function calculateOrderTotals(order) {
  ...
}
// ... etc
```

---

## Step 5 — Verify and Annotate the Output

After writing the code, always include a brief summary table:

| Function | Lines | Purpose | Pure? |
|---|---|---|---|
| `processOrder` | 18 | Orchestrates full order flow | No (I/O) |
| `validateOrderInput` | 12 | Throws on invalid fields | Yes |
| `calculateOrderTotals` | 9 | Computes subtotal/tax/total | Yes |
| `checkInventoryAvailability` | 14 | Returns out-of-stock items | No (DB read) |
| `persistOrderToDatabase` | 11 | Saves order, returns ID | No (DB write) |
| `sendOrderConfirmationEmail` | 8 | Sends transactional email | No (network) |

Then call out:
- **Testability win**: which functions are now trivially unit-testable (pure functions)
- **Any naming decisions** that weren't obvious — explain your reasoning
- **Any behavior questions**: cases where the original code was ambiguous and you made a call

---

## Common Patterns by Language

### JavaScript / TypeScript
- Prefer `const extractedFn = (args) => { ... }` for pure helpers defined in module scope.
- Use destructuring in params: `function formatAddress({ street, city, zip })` not 6 positional args.
- Arrow functions for short transforms; named `function` declarations for anything with logic.

### Python
- Module-level helper functions for reusable logic; inner functions only for closures that genuinely need to close over local state.
- Type hints on all extracted functions.
- `_private_helper` naming convention for internal-only helpers.

### Java / C#
- Extract to private methods in the same class first; only move to a new class if the function represents a distinct concern (e.g., a full validation layer).
- Watch for over-extraction into separate classes — prefer flat private methods over deep inheritance chains.

### Go
- Package-level unexported functions (`func validateOrder`) for helpers.
- Return `(result, error)` from each extracted function rather than panicking.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Extracting 2-line functions that read worse than inline | Pointless overhead | Only extract when it has a meaningful name and reusable purpose |
| Passing a giant context struct to every helper | Hides coupling | Narrow the interface — pass only what each function needs |
| Renaming + reformatting without changing structure | Cosmetic, not structural | Ensure each extracted fn has a single clear responsibility |
| Extracting purely to hit a line-count target | Mechanical, not thoughtful | Let abstraction boundaries drive extraction, not metrics |
| Creating a `utils.js` dumping ground | Just moves the mess | Name helpers after their domain: `orderUtils`, `addressFormat` |

---

## Quick Checklist Before Delivering

- [ ] Orchestrator reads like a high-level summary of the operation
- [ ] Every extracted function has a name that describes *what* it does, not *how*
- [ ] No extracted function does more than one thing
- [ ] Pure functions are free of side effects
- [ ] Parameter count ≤ 3 per function (or justified)
- [ ] No behavior change from original
- [ ] Summary table included
- [ ] Ambiguous decisions called out