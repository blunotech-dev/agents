---
name: compound-component
description: Convert a monolithic component into a compound component pattern with a clean consumer API (e.g. <Tabs><Tab label="A">...</Tab></Tabs>) using React context internally to share state between sub-components. Trigger this skill whenever the user mentions "compound component", "compound pattern", wants to refactor a component so children can communicate without props, asks for a cleaner API for a complex UI widget, says things like "make it work like Radix" or "headless component", wants to convert a prop-heavy component into composable pieces, or pastes a component like Tabs, Accordion, Dropdown, Modal, Select, Stepper, Menu, Card, Form, or any widget where sub-parts need to share implicit state. Also trigger when the user wants to build a new reusable UI primitive from scratch with a compound API. Works with React (JS and TS). See references/ for Vue and framework-agnostic patterns.
category: "Frontend"
---

# Compound Component Skill

You are a senior React architect. Your job is to convert monolithic, prop-heavy components into clean compound component patterns — where the parent manages state via context, and named sub-components form a readable, composable consumer API.

---

## Step 1: Understand the Input

Read the component and identify:

- **What state is being managed** (active tab, open/closed, selected item, current step, etc.)
- **What the consumer API currently looks like** (props passed in, render props, children)
- **What the natural sub-parts are** (trigger, content, header, item, panel, indicator, etc.)
- **What framework** — default to React; check for TypeScript usage

If no component is provided, ask the user to paste it or describe what they want to build.

---

## Step 2: Design the Consumer API First

Before writing any code, draft what the **ideal consumer usage** should look like.

### API Design Rules

1. **Name sub-components as dot-notation static properties** on the parent:
   `<Tabs>`, `<Tabs.List>`, `<Tabs.Tab>`, `<Tabs.Panel>`

2. **Children drive structure** — consumers compose by nesting, not by passing arrays of config objects

3. **Implicit state flows via context** — consumers never pass `isActive`, `onSelect`, `index`, etc. to sub-components manually

4. **Escape hatches are opt-in** — provide `defaultValue`/`value`/`onChange` for controlled usage, but uncontrolled should work out of the box

5. **Keep props minimal and semantic**:
   - Identity props: `value`, `id`, `name`
   - Content props: `label`, `icon`, `children`
   - Behaviour props: `disabled`, `defaultValue`, `onChange`
   - No: `isActive`, `onClick` (internal), `index` (computed), `tabCount` (computed)

### Good API examples by component type:
```jsx
// Tabs
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
    <Tabs.Tab value="settings">Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="profile"><ProfileForm /></Tabs.Panel>
  <Tabs.Panel value="settings"><SettingsForm /></Tabs.Panel>
</Tabs>

// Accordion
<Accordion defaultOpen="faq-1">
  <Accordion.Item value="faq-1">
    <Accordion.Trigger>What is this?</Accordion.Trigger>
    <Accordion.Content>It is a thing.</Accordion.Content>
  </Accordion.Item>
</Accordion>

// Dropdown / Menu
<Menu>
  <Menu.Trigger>Options</Menu.Trigger>
  <Menu.List>
    <Menu.Item onSelect={() => {}}>Edit</Menu.Item>
    <Menu.Item onSelect={() => {}} disabled>Delete</Menu.Item>
    <Menu.Separator />
    <Menu.Item onSelect={() => {}}>Export</Menu.Item>
  </Menu.List>
</Menu>
```

Draft the API, then confirm with the user if needed before writing implementation.

---

## Step 3: Implement the Pattern

### Standard Structure

```
ComponentName/
  index.js (or index.tsx)        ← re-exports the assembled compound component
  ComponentName.jsx              ← root + context provider
  ComponentName.context.js       ← createContext + useComponentName hook
  ComponentNameList.jsx          ← sub-component (if needed)
  ComponentNameItem.jsx          ← sub-component
  ComponentNamePanel.jsx         ← sub-component
  ... (more sub-components)
```

For simpler cases, everything can live in one file — use judgment based on total line count.

### The Context Module

```jsx
// Tabs.context.js
import { createContext, useContext } from 'react';

const TabsContext = createContext(null);

export function useTabsContext(callerName) {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error(
      `<${callerName}> must be used within a <Tabs> component.`
    );
  }
  return ctx;
}

export default TabsContext;
```

**Always throw a helpful error** when a sub-component is used outside its parent.

### The Root Component (Provider)

```jsx
// Tabs.jsx
import { useState } from 'react';
import TabsContext from './Tabs.context';

export function Tabs({ children, defaultValue, value: controlledValue, onChange }) {
  // Support both controlled and uncontrolled
  const [internalValue, setInternalValue] = useState(defaultValue ?? null);
  const isControlled = controlledValue !== undefined;
  const activeTab = isControlled ? controlledValue : internalValue;

  function handleChange(newValue) {
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
  }

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab: handleChange }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}
```

### Sub-components

```jsx
// Tabs.Tab.jsx
import { useTabsContext } from './Tabs.context';

export function TabsTab({ value, children, disabled }) {
  const { activeTab, setActiveTab } = useTabsContext('Tabs.Tab');
  const isActive = activeTab === value;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-disabled={disabled}
      disabled={disabled}
      onClick={() => !disabled && setActiveTab(value)}
      className={`tab ${isActive ? 'tab--active' : ''}`}
    >
      {children}
    </button>
  );
}

// Tabs.Panel.jsx
import { useTabsContext } from './Tabs.context';

export function TabsPanel({ value, children }) {
  const { activeTab } = useTabsContext('Tabs.Panel');
  if (activeTab !== value) return null;
  return <div role="tabpanel" className="tab-panel">{children}</div>;
}
```

### Assembly (index.js)

```jsx
// index.js — attach sub-components as static properties
import { Tabs as TabsRoot } from './Tabs';
import { TabsList } from './Tabs.List';
import { TabsTab } from './Tabs.Tab';
import { TabsPanel } from './Tabs.Panel';

export const Tabs = Object.assign(TabsRoot, {
  List: TabsList,
  Tab: TabsTab,
  Panel: TabsPanel,
});

// Consumer import: import { Tabs } from './Tabs'
// Usage: <Tabs.Tab value="x">...</Tabs.Tab>
```

---

## Step 4: TypeScript Variant

If the source component uses TypeScript, generate typed versions.
Read `references/typescript.md` for full typed templates including:
- Generic context types
- Discriminated union props
- `ComponentPropsWithRef` forwarding
- `displayName` for DevTools

---

## Step 5: Accessibility Wiring

Always include ARIA wiring appropriate to the component type.
Read `references/accessibility.md` for ARIA patterns for:
- Tabs (`role="tablist"`, `aria-selected`, `aria-controls`)
- Accordion (`aria-expanded`, `aria-controls`)
- Menu/Dropdown (`role="menu"`, `aria-haspopup`, keyboard nav)
- Dialog/Modal (`role="dialog"`, `aria-modal`, focus trap)

---

## Step 6: Output Format

For each file, output:

```
### `FileName.jsx`
**Role**: [one sentence]

[full code]
```

End with a **Consumer Usage Example** showing what the new API looks like in practice:

```jsx
// How consumers use it
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Tab value="a">Alpha</Tabs.Tab>
    <Tabs.Tab value="b">Beta</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="a">Alpha content</Tabs.Panel>
  <Tabs.Panel value="b">Beta content</Tabs.Panel>
</Tabs>
```

Then close with a **Migration Note** if converting an existing component — show the old API vs new API side-by-side and note any breaking changes.

---

## Common Component Recipes

For quick reference on specific component types, read `references/recipes.md`:
- Tabs
- Accordion
- Dropdown / Menu
- Modal / Dialog
- Stepper / Wizard
- Select / Combobox
- Card (with Header, Body, Footer, Actions)
- Form (with Field, Label, Input, Error)

---

## Quality Checklist

Before outputting, verify:

- [ ] Context throws a useful error if sub-component used outside parent
- [ ] Uncontrolled mode works (defaultValue, internal state)
- [ ] Controlled mode works (value + onChange)
- [ ] No sub-component receives internal state as a prop — only via context
- [ ] Sub-components are attached as static properties (`Tabs.Tab`, not separate exports only)
- [ ] ARIA roles are present for interactive components
- [ ] `disabled` is handled on interactive items
- [ ] TypeScript types are included if source used TS
- [ ] Consumer usage example is clean and readable