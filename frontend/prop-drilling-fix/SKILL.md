---
name: prop-drilling-fix
description: Identify prop drilling in React component trees and refactor it using the most appropriate pattern, such as lifted state, React Context, or component composition. Use when users mention passing props through many layers, prop drilling, or ask to simplify or refactor component communication.
category: "Frontend"
---

# Prop Drilling Fix Skill

Diagnose prop drilling chains and refactor them to the cleanest solution — lifted state,
Context, or composition — with clear reasoning for the choice.

---

## What Is Prop Drilling?

Prop drilling occurs when a value is passed through intermediate components that don't use
it themselves — they only exist in the chain to forward the prop to a deeper consumer.

```
App (has `user`)
  └── Layout (receives `user`, doesn't use it)
        └── Sidebar (receives `user`, doesn't use it)
              └── UserAvatar (finally uses `user`) ← actual consumer
```

The smell: `user` appears in Layout's and Sidebar's prop signatures purely as a conduit.

**Why it matters:**
- Intermediate components become unnecessarily coupled to data they don't care about
- Adding a new consumer deep in the tree requires updating every component in the chain
- Refactoring the data shape requires touching all intermediate components
- It makes components harder to reuse in isolation

---

## Step 1 — Detect Drilling Chains

Scan the provided code for these patterns:

### Direct indicators
- A prop appears in a component's function signature but is never read in its JSX/logic —
  it's only spread or passed down: `function Layout({ user, ...rest }) { return <Sidebar user={user} /> }`
- The same prop name threads through 3+ component levels
- A component accepts many props and immediately passes them all to a single child:
  `<Child propA={propA} propB={propB} propC={propC} />`

### Heuristics for finding chains
For each prop, trace its path:
1. Where is it **defined or sourced**? (State, API call, context, parent)
2. Where is it **actually consumed**? (Read, rendered, used in logic)
3. How many components sit **between** source and consumer without using it?

If 2+ components are pure conduits → drilling confirmed.

### Build a chain map
Document each chain before refactoring:

```
Prop: `onThemeChange` (function)
Source: App.tsx (line 12, useState)
Chain: App → DashboardLayout → TopBar → ThemeToggle
Used by: ThemeToggle only
Conduits: DashboardLayout, TopBar (2 levels of drilling)
```

---

## Step 2 — Choose the Right Fix

This is the most important decision. The wrong pattern creates new problems.

### Option A: Component Composition (Render Props / Children)

**Use when:**
- The intermediate component is a layout/wrapper that doesn't need the data
- You can restructure the tree so the consumer is composed in from above
- Avoids adding a Context provider for data that's only needed in one place

**The key insight:** Instead of passing data *through* a component, pass the *already-rendered*
component as `children` or a render prop. The intermediate component never sees the data.

```tsx
// Before: Layout drills `user` it doesn't need
function App() {
  const [user] = useState(currentUser);
  return <Layout user={user} />;
}
function Layout({ user }) {
  return <div className="layout"><Sidebar user={user} /></div>;
}
function Sidebar({ user }) {
  return <nav><UserAvatar user={user} /></nav>;
}

// After: App owns the composition; Layout/Sidebar are pure shells
function App() {
  const [user] = useState(currentUser);
  return (
    <Layout>
      <Sidebar>
        <UserAvatar user={user} />
      </Sidebar>
    </Layout>
  );
}
function Layout({ children }) {
  return <div className="layout">{children}</div>;
}
function Sidebar({ children }) {
  return <nav>{children}</nav>;
}
```

**Best for:** Layout shells, wrappers, page skeletons, slot-based UIs.

---

### Option B: React Context

**Use when:**
- The data is consumed in multiple places across the tree (2+ unrelated consumers)
- Composition would require restructuring too much of the component tree
- The data is truly "ambient" to a subtree: theme, auth user, locale, feature flags
- You want consumers to subscribe independently without parent re-renders propagating manually

```tsx
// Context approach
const UserContext = createContext<User | null>(null);

function App() {
  const [user] = useState(currentUser);
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

// Any descendant can consume directly — no prop threading
function UserAvatar() {
  const user = useContext(UserContext);
  return <img src={user?.avatar} />;
}
```

**Also provide a custom hook** to enforce correct usage and give better errors:
```tsx
export function useUser() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error('useUser must be used within UserContext.Provider');
  return ctx;
}
```

**Best for:** Auth state, theme, locale, user preferences, permissions — global or subtree-wide concerns.

**Avoid Context for:** High-frequency updates (every keystroke), data that's only needed in
one place, server-fetched data (use a data-fetching library instead).

---

### Option C: Lifted State

**Use when:**
- Two sibling components need to share state, but neither is an ancestor of the other
- The current drilling is happening because state is placed *too low* in the tree
- The fix is simply moving `useState` to the nearest common ancestor

```tsx
// Before: selectedId lives in FilterPanel, but ResultsList also needs it
function Page() {
  return (
    <>
      <FilterPanel />   {/* owns selectedId */}
      <ResultsList />   {/* needs selectedId — can't reach it! */}
    </>
  );
}

// After: lift selectedId to Page (nearest common ancestor)
function Page() {
  const [selectedId, setSelectedId] = useState(null);
  return (
    <>
      <FilterPanel selectedId={selectedId} onSelect={setSelectedId} />
      <ResultsList selectedId={selectedId} />
    </>
  );
}
```

**Best for:** Coordinating two or three siblings. Not a fix for deep drilling — if the lifted
state immediately starts drilling through 3+ levels, reach for Context instead.

---

## Step 3 — Decision Matrix

Use this to pick quickly:

| Situation | Best Fix |
|---|---|
| Layout/wrapper components that don't use the data | **Composition** |
| Same data needed in 2+ distant components | **Context** |
| Siblings need to share state | **Lifted State** |
| Only 1 consumer but 2–3 conduit levels | **Composition** first, Context if composition is awkward |
| Frequently-updating state (e.g. input values) | **Lifted State** (avoid Context — causes excess re-renders) |
| Auth, theme, locale, permissions | **Context** always |
| Using a state manager (Redux, Zustand, Jotai) | **Store slice** — same principle as Context |

---

## Step 4 — Perform the Refactor

### Composition refactor checklist
- [ ] Move JSX composition up to the component that owns the data
- [ ] Replace drilled props with `children` or named slot props (`header`, `footer`, etc.)
- [ ] Remove the now-unused prop from each intermediate component's signature
- [ ] Verify the intermediate components have no implicit dependency on the removed prop

### Context refactor checklist
- [ ] Create a dedicated context file: `<FeatureName>Context.tsx`
- [ ] Define the context shape with TypeScript interface
- [ ] Create a `Provider` component that owns the state
- [ ] Export a `useFeatureName()` custom hook
- [ ] Wrap the relevant subtree (not necessarily the whole app) with the Provider
- [ ] Replace all prop reads in consumers with `useFeatureName()`
- [ ] Remove the now-unused prop from all intermediate component signatures
- [ ] Confirm the Provider is placed at the *lowest* ancestor that covers all consumers

### Lifted state checklist
- [ ] Find the nearest common ancestor of all consumers
- [ ] Move `useState` (and any derived logic) to that ancestor
- [ ] Pass state and setter down *only one level* if possible (or use Context if it goes deeper)
- [ ] Remove state from the child that previously owned it

---

## Step 5 — Output Format

Structure refactor output as:

### 1. Drilling Chains Found
A chain map for each identified drilling instance (prop name, source, chain, consumers, conduit count).

### 2. Recommended Fix Per Chain
For each chain: chosen pattern + 1–2 sentence rationale explaining *why* this pattern
fits better than the alternatives.

### 3. Refactored Code
Full corrected code with:
- Before/after for each affected component
- Inline comments marking what changed and why
- New files (e.g., context files) shown in full

### 4. What Was Removed
Explicitly list every prop that was removed from intermediate component signatures —
this is the proof the drilling is fixed, not just papered over.

### 5. Caveats / Follow-up
Note any performance considerations (Context re-renders), components that may need
`React.memo`, or places where a state management library would be a better long-term fit.

---

## Common Mistakes to Avoid

**Overusing Context.** Context is not a replacement for all prop passing. If a prop only
travels one level (parent → direct child), that's normal and fine — don't add a Context
for it.

**Putting everything in one giant Context.** Split Contexts by concern. A `UserContext` and
a `ThemeContext` are better than one `AppContext` with everything — consumers only re-render
when *their* context value changes.

**Forgetting memoization.** When a Context value is an object or function created inline in
the Provider, every render recreates it and all consumers re-render. Use `useMemo` and
`useCallback`:
```tsx
const value = useMemo(() => ({ user, updateUser }), [user, updateUser]);
<UserContext.Provider value={value}>
```

**Confusing "drilling" with "passing."** Passing a prop from a parent to its *direct child*
that actually uses it is not drilling. Only flag chains where intermediate components are
pure conduits.

**Using Context for high-frequency state.** Form field values, hover state, scroll position —
these change constantly. Context re-renders every subscriber on each change. Keep these local
or use a ref.