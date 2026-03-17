---
name: hypothesize-before-fix
description: State the likely root cause before writing code. Use when debugging bugs, errors, or failing tests to enforce hypothesis-driven fixes.
category: "Debugging"
---

# Hypothesize Before Fix

## Instructions

Never write fix code before explicitly stating and validating a root cause hypothesis. This prevents wasted effort on incorrect assumptions and encourages deeper understanding.

### Step 1: Gather Symptoms

Before forming any hypothesis, collect all available evidence:

- **Error messages**: Exact text, error codes, stack traces
- **Observed behavior**: What actually happens
- **Expected behavior**: What should happen
- **Reproduction steps**: How to trigger the bug
- **Context**: When it started, what changed, who is affected

Document symptoms clearly:

```
SYMPTOM REPORT
━━━━━━━━━━━━━
Error: "TypeError: Cannot read property 'map' of undefined"
Location: Dashboard.jsx:47
Observed: Dashboard crashes on load for new users
Expected: Dashboard renders an empty state
First seen: After PR #342 merged
Affects: Only users with no projects
```

### Step 2: Form a Hypothesis

Based on the symptoms, state a clear, falsifiable hypothesis:

**Format:**
> **Hypothesis**: [The bug occurs because] `[specific cause]` [which leads to] `[specific effect]` [when] `[specific condition]`.

**Good hypothesis examples:**
- "The crash occurs because `user.projects` is `undefined` (not an empty array) for new users, and the component calls `.map()` on it without a null check."
- "The timeout happens because the database query joins 3 tables without an index on `orders.user_id`, causing a full table scan when the orders table exceeds 10k rows."

**Bad hypothesis examples:**
- "Something is wrong with the database." *(Too vague)*
- "The code is broken." *(Not falsifiable)*
- "It might be a race condition or maybe a null pointer or perhaps a config issue." *(Multiple unranked guesses)*

### Step 3: Validate the Hypothesis

Before writing any fix, check the hypothesis against evidence:

1. **Predicts the symptoms?** Does the hypothesis explain all observed symptoms?
   - If not, revise or form a new hypothesis.
2. **Predicts what should NOT break?** Does the hypothesis explain why other parts work correctly?
3. **Is testable?** Can you verify the hypothesis with a quick check?
   - Read the relevant code to confirm the suspected logic flaw
   - Check variable values at the suspected failure point
   - Look at logs or database state for confirmation

**Validation checklist:**
- [ ] Hypothesis explains the error message
- [ ] Hypothesis explains why it affects specific users/cases
- [ ] Hypothesis explains the timing (why now, not before)
- [ ] Quick code inspection supports the hypothesis

### Step 4: Declare Confidence and Proceed

Rate your confidence and act accordingly:

| Confidence | Action |
|-----------|--------|
| **High (>80%)** | State the hypothesis, then write the fix |
| **Medium (50-80%)** | State the hypothesis, add a diagnostic step to confirm, then fix |
| **Low (<50%)** | State the hypothesis, recommend investigation steps before any code changes |

### Step 5: Write the Fix (Only Now)

Only after the hypothesis is stated and validated:

1. Write the minimal fix that addresses the root cause
2. Explain how the fix addresses the hypothesis
3. Suggest a test that would catch this bug in the future

## Examples

### Example 1: React Component Crash

User says: "Dashboard crashes for new users."

**Symptoms**: `TypeError: Cannot read property 'map' of undefined` at `Dashboard.jsx:47`

> **Hypothesis**: `user.projects` is `undefined` for users with no projects. The API returns `{user: {name: "..."}}` without a `projects` key instead of `{user: {name: "...", projects: []}}`. The component calls `user.projects.map()` without guarding against `undefined`.

**Validation**: ✅ API response for new user confirms no `projects` key. ✅ Line 47 calls `.map()` directly.

**Fix**: Add default: `const projects = user.projects || []`

### Example 2: Slow API Endpoint

User says: "The /orders endpoint takes 30 seconds."

> **Hypothesis**: The query `SELECT * FROM orders JOIN users ON ...` performs a full table scan because `orders.user_id` lacks an index. This only became noticeable after the orders table grew past 100k rows last week.

**Validation**: ✅ `EXPLAIN` query confirms sequential scan. ✅ Table grew from 50k to 120k rows in the past week.

**Fix**: `CREATE INDEX idx_orders_user_id ON orders(user_id)`

## Troubleshooting

### Error: Multiple plausible hypotheses
**Cause**: Insufficient evidence to distinguish between candidates.
**Solution**: Rank hypotheses by likelihood. Test the most likely one first. If it doesn't hold, move to the next. Do not attempt to fix multiple hypotheses simultaneously.

### Error: Hypothesis does not explain all symptoms
**Cause**: The root cause may be compound (multiple bugs) or the hypothesis is incomplete.
**Solution**: Check if there are multiple independent bugs. Expand the hypothesis to account for all symptoms, or split into separate investigations.

### Error: Cannot validate without running the code
**Cause**: The hypothesis requires runtime verification that isn't possible in the current context.
**Solution**: State the hypothesis with a confidence level, recommend specific logging or debugging steps the user should take, and propose the fix conditionally: "If my hypothesis is correct, the fix is..."
