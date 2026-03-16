---
name: nested-conditionals
description: Flatten deeply nested conditionals using guard clauses, early returns, or patterns like strategy, without changing behavior. Use when the user asks to simplify conditional logic, reduce nesting/indentation, or clean up “pyramid of doom” style code. Also trigger when pasted code shows 3+ levels of nesting.
category: "Refactoring"

---

# Nested Conditionals

Flatten deeply nested if/else chains into code that reads top-to-bottom, without
changing behavior. The goal: a reader should never have to mentally "stack" more than
one or two conditions to understand what a code path does.

---

## When to Apply This Skill

Trigger on any of:
- 3+ levels of `if`/`else` nesting (the "arrow" or "pyramid of doom" shape)
- `else` blocks that contain most of the meaningful logic (main path buried at the bottom)
- Multiple conditions guarding a single core operation
- Long `if/else if/else if` chains switching on a type, role, or status enum
- Nested ternaries (more than one level deep)

---

## Step 1 — Diagnose the Nesting Type

Before choosing a technique, identify *why* the code is nested:

| Pattern | Symptom | Best technique |
|---|---|---|
| **Validation nesting** | Each level checks a precondition before the "real" work | Early return / guard clauses |
| **Happy path buried** | Core logic at deepest indent; errors handled at outer levels | Invert conditions + early return |
| **Type/role/status switching** | `if type === A … else if type === B …` chains | Strategy pattern or lookup table |
| **Compound conditions** | Multiple `&&`/`||` conditions on a single `if` | Extract named boolean functions |
| **Mixed concerns** | Validation *and* transformation *and* branching all in one block | Combine guard clauses + extraction |

Naming the pattern first prevents applying the wrong fix (e.g., strategy pattern on
simple validation — overkill).

---

## Step 2 — Choose the Right Technique

### Technique A: Guard Clauses (Early Return)

**When to use:** Validations, preconditions, or error cases that should abort early.
Turns inside-out logic into a flat sequence of checks.

**Before:**
```js
function processPayment(user, cart, paymentMethod) {
  if (user) {
    if (cart.items.length > 0) {
      if (paymentMethod.isValid) {
        // 40 lines of actual payment logic
        return result;
      } else {
        throw new Error('Invalid payment method');
      }
    } else {
      throw new Error('Cart is empty');
    }
  } else {
    throw new Error('User not found');
  }
}
```

**After:**
```js
function processPayment(user, cart, paymentMethod) {
  if (!user) throw new Error('User not found');
  if (cart.items.length === 0) throw new Error('Cart is empty');
  if (!paymentMethod.isValid) throw new Error('Invalid payment method');

  // 40 lines of actual payment logic — now readable
  return result;
}
```

**Rules:**
- Guards go at the top, before any main logic.
- Each guard is one line: `if (!condition) return/throw/continue`.
- After all guards, zero nesting — the rest of the function is the happy path.
- Don't combine multiple conditions into one guard unless they share a single error message.

---

### Technique B: Invert the Condition

**When to use:** The `else` branch contains the main logic, making it feel backwards.
Inverting makes the primary path top-level.

**Before:**
```js
if (isAuthenticated) {
  if (hasPermission) {
    // real work — 30 lines deep in nesting
  }
}
```

**After:**
```js
if (!isAuthenticated) return redirectToLogin();
if (!hasPermission) return respondForbidden();

// real work — top-level, no indentation
```

---

### Technique C: Extract Named Boolean Functions

**When to use:** A compound condition is hard to parse at a glance.
Extracting it into a named predicate adds semantic meaning.

**Before:**
```js
if (user.age >= 18 && user.country !== 'US' && !user.isRestricted && account.tier === 'premium') {
  // ...
}
```

**After:**
```js
const isEligibleForInternationalPremium = (user, account) =>
  user.age >= 18 &&
  user.country !== 'US' &&
  !user.isRestricted &&
  account.tier === 'premium';

if (isEligibleForInternationalPremium(user, account)) {
  // ...
}
```

**Rules:**
- Name the predicate from the *business meaning*, not a restatement of the conditions.
- Keep predicates pure (no side effects).
- One predicate per distinct concept — don't bundle unrelated conditions.

---

### Technique D: Strategy Pattern / Lookup Table

**When to use:** Long `if/else if` or `switch` chains dispatching on a type, role,
status, or enum value. Each branch does different work, not just different guards.

**Before:**
```js
function applyDiscount(cart, userType) {
  if (userType === 'student') {
    return cart.total * 0.85;
  } else if (userType === 'employee') {
    return cart.total * 0.70;
  } else if (userType === 'senior') {
    return cart.total * 0.80;
  } else if (userType === 'affiliate') {
    return cart.total * 0.75;
  } else {
    return cart.total;
  }
}
```

**After (lookup table):**
```js
const DISCOUNT_RATES = {
  student:   0.85,
  employee:  0.70,
  senior:    0.80,
  affiliate: 0.75,
};

function applyDiscount(cart, userType) {
  const rate = DISCOUNT_RATES[userType] ?? 1.0;
  return cart.total * rate;
}
```

**After (strategy map — for branches with real logic, not just data):**
```js
const shippingStrategies = {
  standard: (order) => calculateStandardShipping(order),
  express:  (order) => calculateExpressShipping(order),
  overnight: (order) => calculateOvernightShipping(order),
};

function calculateShipping(order) {
  const strategy = shippingStrategies[order.shippingType];
  if (!strategy) throw new Error(`Unknown shipping type: ${order.shippingType}`);
  return strategy(order);
}
```

**When NOT to use:** If branches are genuinely different control flows (not just
dispatching the same operation differently), a lookup table obscures intent. Use
judgment — a 3-branch `if/else` is often fine as-is.

---

### Technique E: Combine Techniques

Real code often needs more than one technique. Apply in this order:
1. Guard clauses first (clear the preconditions)
2. Extract compound conditions into named predicates
3. Strategy/lookup for enum-dispatch branches
4. What remains should be flat and readable

---

## Step 3 — Write the Refactored Code

Output the refactored code with:

1. **One technique per transformation** — don't mix guard clauses and strategy pattern
   in a way that obscures which is doing what.
2. **Behavior must be identical.** Flag any case where the original code had an
   implicit `else undefined` or a swallowed error — don't silently fix bugs during
   refactor; call them out.
3. **Preserve intent comments** — keep comments that explain *why* a condition exists,
   not just *what* it checks.
4. **Show before and after** for any non-trivial transformation. Don't just drop the
   refactored version without context.

---

## Step 4 — Verify and Annotate

After writing the code, include:

**Nesting reduction summary:**

| Before | After | Technique used |
|---|---|---|
| Max depth: 4 | Max depth: 1 | Guard clauses |
| 3 compound conditions | 3 named predicates | Extracted booleans |
| 5-branch `if/else if` | Lookup table | Strategy / data table |

**Then call out:**
- **Any implicit behavior** in the original that was preserved (e.g., `undefined` returns, fall-through)
- **Any ambiguities** where the original logic was unclear — state the assumption made
- **Test surface:** which refactored paths are now independently testable

---

## Language-Specific Notes

### JavaScript / TypeScript
- `??` (nullish coalescing) and `?.` (optional chaining) eliminate many defensive nesting patterns.
- Lookup tables as plain objects (`const MAP = { key: value }`) are idiomatic.
- `Array.find` / `Array.some` / `Array.every` often replace nested loops-with-conditionals.
- TypeScript: use discriminated unions + exhaustive `switch` (with `never` check) instead of open-ended `if/else if` chains on string literals.

### Python
- Python has no early `return` style prohibition — use it freely.
- Dictionary dispatch (`handlers = {'a': fn_a, 'b': fn_b}`) is idiomatic for strategy.
- `match/case` (Python 3.10+) for structural pattern matching — prefer over `if/elif` chains on types.
- Avoid deeply nested list comprehensions; extract to named functions.

### Java / C#
- Use early `return` freely in private methods; it's standard practice.
- `Optional`/nullable chains (Java's `.orElseThrow()`, C#'s `?.` and `??`) reduce null-guard nesting.
- Consider `Map<Key, Supplier<Result>>` for strategy dispatch in Java.
- C# `switch` expressions (C# 8+) are cleaner than `if/else if` for enum dispatch.

### Go
- Early returns for errors are idiomatic Go (`if err != nil { return ..., err }`).
- Avoid `else` after `return` — Go linters often flag it.
- Map dispatch (`map[string]func(...)`) for strategy pattern.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Inverting every condition mechanically | Can make happy path confusing if there's no clear "main path" | Only invert when the else-branch is the primary logic |
| Strategy pattern for 2–3 branches | Over-engineering — a simple `if/else` is fine | Use strategy only when branches ≥ 4 or will grow |
| Extracting conditions into functions that are only called once with a trivial name | Adds indirection without meaning | Only extract when the name adds genuine semantic value |
| Flattening without understanding behavior | Risk of changing semantics (especially `else if` vs independent `if`) | Always trace both original and new paths |
| Merging unrelated guards into a single `if` | Hides which guard fired; hard to debug | One guard per condition, one error message per guard |

---

## Quick Checklist Before Delivering

- [ ] Max nesting depth reduced to ≤ 2 (ideally 1 after guards)
- [ ] No `else` block after a `return`/`throw`
- [ ] Each guard is one line with a clear error or return value
- [ ] Compound conditions extracted to named predicates (if they were hard to read)
- [ ] Lookup table or strategy used only where dispatch has ≥ 4 branches
- [ ] Behavior is identical — implicit returns and edge cases preserved
- [ ] Before/after summary table included
- [ ] Ambiguous original logic called out explicitly