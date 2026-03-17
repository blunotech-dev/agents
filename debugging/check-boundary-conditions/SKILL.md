---
name: check-boundary-conditions
description: Evaluate how logic handles edge cases and extreme inputs. Use when checking robustness, debugging failures, or testing boundaries like empty input, zero values, or limits
category: "Debugging"
---

# Check Boundary Conditions

## Instructions

Systematically evaluate how code behaves at every boundary and extreme. Bugs cluster at edges—check them all before declaring code correct.

### Step 1: Identify the Input Domain

For each parameter or input to the code under review, determine:

- **Data type**: number, string, array, object, boolean, date, etc.
- **Valid range**: What values are acceptable?
- **Source**: User input, API response, database query, config file?

Document the input domain:

```
FUNCTION: processOrder(items, discount, userId)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
| Parameter | Type   | Valid Range         | Source     |
|-----------|--------|---------------------|------------|
| items     | Array  | 1+ order items      | User cart  |
| discount  | Number | 0.0 to 1.0 (0-100%) | Promo code |
| userId    | String | Non-empty UUID      | Auth token |
```

### Step 2: Enumerate Boundary Values

For each input, systematically list boundary conditions:

**Numeric inputs:**
- Zero (`0`)
- Negative values (`-1`, `-0.001`)
- Maximum value (`Number.MAX_SAFE_INTEGER`, `INT_MAX`)
- Minimum value (`Number.MIN_SAFE_INTEGER`, `INT_MIN`)
- Floating point precision (`0.1 + 0.2`)
- `NaN`, `Infinity`, `-Infinity`
- Just above/below threshold values

**String inputs:**
- Empty string (`""`)
- Single character (`"a"`)
- Very long string (10k+ characters)
- Unicode / special characters (`"café"`, `"🎉"`, `"<script>"`)
- Whitespace-only (`"   "`)
- `null` / `undefined`

**Array/Collection inputs:**
- Empty collection (`[]`)
- Single element (`[item]`)
- Very large collection (100k+ elements)
- Collection with duplicates
- Collection with `null` elements
- Nested collections

**Object inputs:**
- Missing required properties
- Extra unexpected properties
- `null` / `undefined`
- Circular references

**Date/Time inputs:**
- Epoch (`1970-01-01`)
- Leap year dates (`2024-02-29`)
- Timezone boundaries (DST transitions)
- Far future / far past dates
- Invalid date strings

### Step 3: Evaluate Each Boundary

For each boundary value, answer:

1. **Does the code handle it?** Is there an explicit check, default, or guard?
2. **What happens if it's not handled?** Crash, wrong result, silent corruption?
3. **Should it be handled?** Is this a realistic input or purely theoretical?

Use this evaluation format:

```
BOUNDARY EVALUATION
━━━━━━━━━━━━━━━━━━
Input: items = []
Handled: ❌ No guard for empty array
Result: Returns NaN (sum of empty array / 0)
Risk: HIGH - users can submit empty carts
Fix: Add early return with validation error

Input: discount = 1.5
Handled: ❌ No upper bound check
Result: Negative total (150% discount)
Risk: HIGH - malicious promo codes
Fix: Clamp discount to [0.0, 1.0]

Input: userId = null
Handled: ✅ Auth middleware rejects before this function
Risk: LOW - cannot reach this code path
Fix: None needed (defense-in-depth optional)
```

### Step 4: Assess Combinations

Some bugs only appear when multiple inputs are at boundaries simultaneously:

- Empty array AND zero discount
- Maximum array size AND maximum numeric values (overflow)
- Null userId AND empty items (which error triggers first?)

Prioritize combinations that are:
- Likely in production
- Security-sensitive
- Data-corruption risks

### Step 5: Report Findings

Present a prioritized summary:

1. **Critical** (must fix): Boundaries that cause crashes, data loss, or security issues
2. **Warning** (should fix): Boundaries that produce incorrect results
3. **Info** (nice to fix): Boundaries that degrade user experience

## Examples

### Example 1: Division Function

```python
def calculate_average(numbers):
    return sum(numbers) / len(numbers)
```

**Boundary evaluation:**
| Input | Result | Risk |
|-------|--------|------|
| `[]` | `ZeroDivisionError` | 🔴 Critical |
| `[0]` | `0.0` (correct) | ✅ OK |
| `[1e308, 1e308]` | `inf` (overflow) | 🟡 Warning |
| `[None, 1, 2]` | `TypeError` | 🔴 Critical |
| `None` | `TypeError` | 🔴 Critical |

### Example 2: String Truncation

```javascript
function truncate(str, maxLen) {
    return str.substring(0, maxLen) + '...';
}
```

**Boundary evaluation:**
| Input | Result | Risk |
|-------|--------|------|
| `str=""`, `maxLen=5` | `"..."` (misleading) | 🟡 Warning |
| `str="Hi"`, `maxLen=100` | `"Hi..."` (adds dots to short string) | 🟡 Warning |
| `str="Hi"`, `maxLen=0` | `"..."` (truncates everything) | 🟡 Warning |
| `str=null` | `TypeError` | 🔴 Critical |
| `maxLen=-1` | `""` + `"..."` = `"..."` | 🟡 Warning |

## Troubleshooting

### Error: Too many boundary combinations
**Cause**: Multiple inputs create combinatorial explosion.
**Solution**: Use pairwise testing—cover all pairs of boundary values rather than all combinations. Focus on high-risk combinations first (security, data integrity).

### Error: Cannot determine valid range
**Cause**: No documentation, types, or validation for the input.
**Solution**: Check upstream callers to infer the input domain. Look at existing tests for implied valid ranges. Flag the missing validation as a finding itself.

### Error: Boundary behavior depends on external state
**Cause**: Function behavior changes based on database state, time, or config.
**Solution**: Document the external dependency. Test boundaries with various external states. Recommend making the dependency explicit (inject it as a parameter).
