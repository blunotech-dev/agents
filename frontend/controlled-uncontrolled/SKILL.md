---
name: controlled-uncontrolled
description: Audit React form components for mixed controlled and uncontrolled inputs and normalize them to a consistent pattern with clear reasoning. Use when users mention controlled vs uncontrolled issues, input warnings, value vs defaultValue conflicts, or ask to review or refactor form code.
category: "Frontend"
---

# Controlled vs. Uncontrolled Input Audit Skill

This skill guides a thorough audit of React form/input components for mixed controlled and
uncontrolled usage, and normalizes them to one clear pattern.

---

## Background: The Core Distinction

| Pattern      | How it works                                              | React manages state? |
|--------------|-----------------------------------------------------------|----------------------|
| Controlled   | `value` prop bound to state; `onChange` updates state     | Yes                  |
| Uncontrolled | `defaultValue` or no value prop; DOM holds the value      | No (ref or none)     |

**The cardinal rule:** A component must be **one or the other, consistently**. Mixing them
(e.g., initializing with `defaultValue` then later binding `value`, or omitting `onChange`
when `value` is set) causes React warnings and unpredictable behavior.

---

## Step 1 — Collect All Input Components

Scan the codebase (or provided code) for every component that renders an HTML form element
or a UI library input:

- Native: `<input>`, `<textarea>`, `<select>`
- Common libraries: `<TextField>`, `<Input>`, `<Select>`, `<Checkbox>`, `<Switch>`, `<RadioGroup>`

For **each** input found, record:

```
Component name / location
Props passed:         value? defaultValue? checked? defaultChecked? onChange?
State connection:     useState / useReducer / form library / none
Ref usage:            useRef / createRef / none
```

---

## Step 2 — Classify Each Input

Apply these rules to classify:

### Controlled
- Has `value` (or `checked`) AND `onChange` handler
- State is managed externally (parent state, form library state)

### Uncontrolled
- Has `defaultValue` / `defaultChecked` OR no value prop
- No `onChange` (or onChange used only for side effects, not to update `value`)
- May use `ref` for reads

### Mixed / Broken — Flag these immediately:
| Symptom | Issue |
|---|---|
| `value` set but no `onChange` | Read-only locked input; React warning |
| `value={undefined}` initially, then a real value | Switches from uncontrolled → controlled mid-lifecycle |
| Both `value` and `defaultValue` on same element | `defaultValue` silently ignored; confusing |
| `onChange` updates state but `value` not bound | State changes but UI doesn't reflect it |
| `value` derived from prop with no fallback (`value={prop}` where prop can be `undefined`) | Intermittent switch between controlled/uncontrolled |

---

## Step 3 — Choose a Normalization Pattern

Before fixing, determine the right pattern for the **whole form or component**. Use these
heuristics:

### Choose Controlled when:
- Form values must be validated on every keystroke
- Fields depend on each other (e.g., disabling one field based on another's value)
- Values are submitted programmatically or prefilled from an API
- Using a form library like React Hook Form (controlled mode), Formik, or Zod-integrated forms
- The component needs to be reset or cleared programmatically

### Choose Uncontrolled when:
- Simple, one-time form submissions (no real-time validation)
- Performance is critical and the form has many fields (avoids re-renders per keystroke)
- Integrating with non-React code or legacy DOM manipulation
- Using `ref`-based reads on submit (e.g., `inputRef.current.value`)

**Default recommendation: prefer controlled.** It is more predictable, easier to debug, and
better supported by the React ecosystem. Uncontrolled is a deliberate optimization, not a default.

---

## Step 4 — Apply the Normalization

### Normalizing to Controlled

```tsx
// Before: mixed / broken
function BadForm() {
  const [name, setName] = useState('');
  return (
    <input
      defaultValue="John"   // uncontrolled default
      value={name}          // controlled binding — CONFLICT
      onChange={e => setName(e.target.value)}
    />
  );
}

// After: fully controlled
function GoodForm() {
  const [name, setName] = useState('John'); // initial value in state
  return (
    <input
      value={name}
      onChange={e => setName(e.target.value)}
    />
  );
}
```

**Key rules for controlled normalization:**
1. Move all initial values into `useState` initializer — never use `defaultValue`
2. Every `value` prop must have a corresponding `onChange`
3. Guard against `undefined`: use `value={name ?? ''}` not `value={name}`
4. For `<select>`, `value` goes on the `<select>` tag, not `<option>`
5. For checkboxes: use `checked` + `onChange`, not `defaultChecked`

---

### Normalizing to Uncontrolled

```tsx
// Before: mixed
function BadForm() {
  const [email, setEmail] = useState('');
  return (
    <input
      value={email}             // controlled
      onChange={e => setEmail(e.target.value)}
      defaultValue="fallback"   // ignored, noise
    />
  );
}

// After: fully uncontrolled
function GoodForm() {
  const emailRef = useRef<HTMLInputElement>(null);

  const handleSubmit = () => {
    console.log(emailRef.current?.value);
  };

  return (
    <>
      <input ref={emailRef} defaultValue="" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

**Key rules for uncontrolled normalization:**
1. Remove all `value` bindings and state that only existed to hold field value
2. Replace with `defaultValue` for initial values
3. Use `ref` for any reads
4. Do NOT use `onChange` to update state that feeds back into `value`

---

## Step 5 — Handle Special Cases

### Form Libraries (React Hook Form, Formik)
- **React Hook Form (uncontrolled mode):** uses `register()` — do not add `value` props manually
- **React Hook Form (controlled mode):** use `Controller` wrapper with `render` prop
- **Formik:** always controlled — use `field.value` + `field.onChange` from `useField()`
- **Never mix** RHF's `register` with manual `useState` for the same field

### Select / Multi-select
```tsx
// Controlled select
<select value={selectedVal} onChange={e => setSelectedVal(e.target.value)}>
  <option value="a">A</option>
</select>
```

### Checkbox / Radio
```tsx
// Controlled checkbox
<input type="checkbox" checked={isChecked} onChange={e => setIsChecked(e.target.checked)} />

// Uncontrolled checkbox
<input type="checkbox" defaultChecked={false} ref={checkRef} />
```

### Dynamic fields (field arrays)
- Always use controlled pattern for dynamic field arrays
- Each field needs its own state slice or form library registration
- Avoid using array index as key for controlled inputs (use stable IDs)

---

## Step 6 — Verification Checklist

After normalization, verify:

- [ ] No component produces the React warning: *"A component is changing an uncontrolled input to be controlled"*
- [ ] No input has both `value` and `defaultValue`
- [ ] No `value` prop is ever `undefined` (use `?? ''` fallback)
- [ ] Every controlled input has an `onChange` handler
- [ ] Every uncontrolled input that needs reading has a `ref`
- [ ] Form reset works correctly (controlled: reset state; uncontrolled: use `form.reset()` or set `key` to remount)
- [ ] Form library fields are not mixed with manual state bindings

---

## Step 7 — Output Format

When presenting the audit results, structure the output as:

### 1. Audit Summary Table
List every input with its classification (Controlled / Uncontrolled / Mixed).

### 2. Issues Found
For each mixed/broken input: describe the exact problem and why it matters.

### 3. Recommended Pattern
State clearly: "Normalize to **controlled**" or "Normalize to **uncontrolled**" with 1–2 sentence rationale.

### 4. Refactored Code
Provide the corrected code with inline comments on each fix.

### 5. Notes on Edge Cases
Call out any dynamic fields, form libraries, or reset logic that needs special attention.

---

## Common Pitfalls to Highlight

- **`value={someVar}` where `someVar` starts as `undefined`** — This is the #1 cause of the
  controlled/uncontrolled switch warning. Always initialize state to `''`, `false`, or `[]`.
- **Conditional rendering of controlled inputs** — Unmounting a controlled input and remounting
  it does not reset state unless you explicitly reset. Use `key` prop or explicit state reset.
- **Spreading props onto inputs** — `<input {...props} />` can accidentally merge `value` from
  one place and `defaultValue` from another. Audit spreads carefully.
- **Third-party UI components** — Libraries like MUI, Ant Design, Radix, and Headless UI often
  have their own controlled/uncontrolled semantics. Read their docs; don't assume they mirror
  native HTML behavior.