# Accessibility Patterns for Compound Components

## Tabs

```jsx
// Tabs.List — container for tab buttons
<div role="tablist" aria-label="Section navigation">
  {children}
</div>

// Tabs.Tab — each tab button
<button
  role="tab"
  id={`tab-${value}`}
  aria-selected={isActive}
  aria-controls={`panel-${value}`}
  tabIndex={isActive ? 0 : -1}
  onClick={() => setActiveTab(value)}
>
  {children}
</button>

// Tabs.Panel — content panel
<div
  role="tabpanel"
  id={`panel-${value}`}
  aria-labelledby={`tab-${value}`}
  tabIndex={0}
  hidden={activeTab !== value}
>
  {children}
</div>
```

**Keyboard nav**: Arrow keys move between tabs. Implement in `Tabs.List` with `onKeyDown`:
```jsx
function handleKeyDown(e) {
  const tabs = [...e.currentTarget.querySelectorAll('[role="tab"]:not([disabled])')];
  const idx = tabs.indexOf(document.activeElement);
  if (e.key === 'ArrowRight') tabs[(idx + 1) % tabs.length]?.focus();
  if (e.key === 'ArrowLeft') tabs[(idx - 1 + tabs.length) % tabs.length]?.focus();
  if (e.key === 'Home') tabs[0]?.focus();
  if (e.key === 'End') tabs[tabs.length - 1]?.focus();
}
```

---

## Accordion

```jsx
// Accordion.Trigger
<button
  aria-expanded={isOpen}
  aria-controls={`accordion-content-${value}`}
  id={`accordion-trigger-${value}`}
  onClick={() => toggle(value)}
>
  {children}
</button>

// Accordion.Content
<div
  id={`accordion-content-${value}`}
  role="region"
  aria-labelledby={`accordion-trigger-${value}`}
  hidden={!isOpen}
>
  {children}
</div>
```

---

## Menu / Dropdown

```jsx
// Menu.Trigger
<button
  aria-haspopup="menu"
  aria-expanded={isOpen}
  aria-controls="menu-list"
  onClick={toggle}
>
  {children}
</button>

// Menu.List
<ul
  id="menu-list"
  role="menu"
  aria-orientation="vertical"
>
  {children}
</ul>

// Menu.Item
<li role="none">
  <button
    role="menuitem"
    tabIndex={-1}
    aria-disabled={disabled}
    onClick={!disabled ? onSelect : undefined}
  >
    {children}
  </button>
</li>

// Menu.Separator
<li role="separator" aria-orientation="horizontal" />
```

**Keyboard nav for Menu**: Arrow keys navigate items, Enter/Space select, Escape closes, Tab closes and moves focus out.

---

## Modal / Dialog

```jsx
// Modal root — rendered in a portal
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  aria-describedby="modal-description"
>
  {children}
</div>

// Modal.Title
<h2 id="modal-title">{children}</h2>

// Modal.Description (optional)
<p id="modal-description">{children}</p>
```

**Focus trap**: On open, focus the first focusable element. On close, return focus to the trigger.
Use a library like `focus-trap-react` or implement manually:
```jsx
useEffect(() => {
  if (!isOpen) return;
  const focusable = dialogRef.current.querySelectorAll(
    'a[href], button:not([disabled]), input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  focusable[0]?.focus();
}, [isOpen]);
```

---

## Select / Combobox

For accessible Select, prefer using native `<select>` when possible. For custom implementations, follow the [ARIA Combobox Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/):

```jsx
// Trigger input
<input
  role="combobox"
  aria-expanded={isOpen}
  aria-haspopup="listbox"
  aria-controls="listbox-id"
  aria-activedescendant={activeOptionId}
  value={inputValue}
  onChange={handleInput}
/>

// Options list
<ul id="listbox-id" role="listbox">
  {options.map(opt => (
    <li
      key={opt.value}
      id={`option-${opt.value}`}
      role="option"
      aria-selected={opt.value === selectedValue}
    >
      {opt.label}
    </li>
  ))}
</ul>
```

---

## Stepper / Wizard

```jsx
// Step list nav
<nav aria-label="Progress">
  <ol role="list">
    {steps.map((step, i) => (
      <li key={step.id} aria-current={i === currentStep ? 'step' : undefined}>
        <span aria-hidden="true">{i + 1}</span>
        {step.label}
        {i < currentStep && <span className="sr-only">(completed)</span>}
      </li>
    ))}
  </ol>
</nav>

// Step panel
<section aria-labelledby={`step-${currentStep}-heading`}>
  <h2 id={`step-${currentStep}-heading`}>{currentStepLabel}</h2>
  {children}
</section>
```

---

## General Principles

- Always use semantic HTML where possible (`<button>`, `<nav>`, `<ul>`, `<li>`) — ARIA supplements, doesn't replace
- Every interactive element needs a focus style — never `outline: none` without an alternative
- Use `sr-only` class (Tailwind) or `visually-hidden` for screen-reader-only text
- Test with keyboard navigation before shipping
- When in doubt, follow [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/)