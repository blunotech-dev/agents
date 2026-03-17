---
name: validate-inputs-first
description: Validate that function inputs match the expected shape. Use when debugging unexpected behavior, null errors, type issues, or incorrect arguments.
category: "Debugging"
---

# Validate Inputs First

## Instructions

Before investigating internal logic, always verify that the function receives the correct inputs. A large percentage of bugs are caused by unexpected input data—not broken logic.

### Step 1: Catalog Expected Inputs

For the function under investigation, document what each input should be:

```
EXPECTED INPUT CONTRACT
━━━━━━━━━━━━━━━━━━━━━━
Function: createUser(userData, options)

| Parameter  | Type   | Required | Shape / Constraints            |
|------------|--------|----------|--------------------------------|
| userData   | Object | Yes      | { name: string, email: string, |
|            |        |          |   age: number (>= 0) }         |
| options    | Object | No       | { sendEmail: boolean,          |
|            |        |          |   role: "admin"|"user" }       |
```

Determine the expected contract from:
- Type annotations / TypeScript interfaces / JSDoc
- Function documentation or comments
- Validation logic within the function
- How callers invoke the function (look at actual call sites)
- Test cases that exercise the function

### Step 2: Inspect Actual Inputs

Determine what is actually being passed in:

1. **Trace upstream**: Follow the data from its origin to the function call.
2. **Check transformations**: Identify any mapping, filtering, or formatting applied before the call.
3. **Look for mutations**: Has the data been modified by another function between creation and use?

Document actual vs. expected:

```
INPUT COMPARISON
━━━━━━━━━━━━━━━━
Parameter: userData

Expected:                    Actual:
{                            {
  name: "Alice",               name: "Alice",
  email: "a@b.com",           email: null,        ← MISMATCH
  age: 30                     age: "30"           ← WRONG TYPE
}                            }
```

### Step 3: Identify Mismatches

For each input, check:

- [ ] **Presence**: Is the parameter provided at all? (not `undefined`)
- [ ] **Type**: Is it the correct data type? (`string` vs `number` vs `object`)
- [ ] **Shape**: Does the object/array have the expected structure and keys?
- [ ] **Values**: Are the values within valid ranges? (non-empty, positive, valid enum)
- [ ] **Encoding**: Is the data in the expected format? (UTF-8, URL-encoded, base64)
- [ ] **Staleness**: Is the data current, or could it be stale/cached?

**Common mismatch patterns:**

| Pattern | Example | Usual Cause |
|---------|---------|-------------|
| `undefined` parameter | `fn(a, undefined, c)` | Destructuring error, missing prop |
| String instead of number | `age: "30"` | Form input, query params, JSON parsing |
| Null instead of object | `user: null` | Failed API call, missing DB record |
| Array instead of single item | `items: [{...}]` vs `item: {...}` | API returns list, code expects one |
| Extra wrapping | `{data: {data: {...}}}` | Double-wrapped API response |
| Stale data | Correct shape, wrong values | Caching, race condition, stale closure |

### Step 4: Trace the Root Cause Upstream

Once a mismatch is found, trace it backward:

1. **Where does the bad value originate?** (API response, user input, database, computation)
2. **Where does the corruption happen?** (transformation, middleware, serialization)
3. **Why wasn't it caught?** (missing validation, no type checking, no tests)

```
UPSTREAM TRACE
━━━━━━━━━━━━━━
Bad value: userData.email = null

← createUser(userData)           [UserService.js:12]
← handleSubmit(formData)         [SignupForm.jsx:45]
← formData = getFormValues()     [SignupForm.jsx:38]
← email input has name="e-mail"  [SignupForm.jsx:15]  ← ROOT CAUSE

Mismatch: Form field named "e-mail", code reads "email"
```

### Step 5: Recommend Defensive Measures

After identifying the input issue, suggest:

1. **Immediate fix**: Correct the data at its source
2. **Validation guard**: Add input validation at the function boundary
3. **Type safety**: Add TypeScript types, PropTypes, or schema validation (e.g., Zod, Joi)
4. **Fail-fast**: Throw descriptive errors for invalid inputs instead of silently proceeding

```javascript
// Example: Adding input validation
function createUser(userData, options = {}) {
  // Validate inputs first
  if (!userData || typeof userData !== 'object') {
    throw new Error('createUser: userData must be an object');
  }
  if (!userData.name || typeof userData.name !== 'string') {
    throw new Error('createUser: userData.name is required and must be a string');
  }
  if (!userData.email || typeof userData.email !== 'string') {
    throw new Error('createUser: userData.email is required and must be a string');
  }
  // ... proceed with valid inputs
}
```

## Examples

### Example 1: API Response Shape Change

User says: "My user list page crashes after the API update."

**Expected input**: `{ users: [{id, name, email}] }`
**Actual input**: `{ data: { users: [{id, name, email}] } }`

**Root cause**: API v2 wraps response in a `data` envelope. Code reads `response.users` but should read `response.data.users`.

### Example 2: Type Coercion Bug

User says: "The discount calculation is wrong."

**Expected input**: `discount: 0.15` (number)
**Actual input**: `discount: "15"` (string from URL query param)

**Root cause**: `price * discount` computes `100 * "15"` = `1500` instead of `100 * 0.15` = `15`. Query params are always strings.

## Troubleshooting

### Error: Cannot determine expected input shape
**Cause**: No types, no docs, no validation—the function accepts anything.
**Solution**: Examine the function body to see which properties/methods it accesses on each parameter. Build the implied contract from usage. Flag the missing contract as a code quality issue.

### Error: Input looks correct but function still fails
**Cause**: The bug is in internal logic, not inputs.
**Solution**: Confirm inputs are genuinely correct (log them at function entry). If inputs are validated and correct, switch to the `trace-execution` skill to analyze internal logic.

### Error: Input varies across different callers
**Cause**: Multiple callers pass different shapes.
**Solution**: Check each call site. Identify which caller passes the malformed data. Consider normalizing inputs at the function boundary to accept multiple formats gracefully.
