---
name: state-colocation
description: Move React state to the appropriate level, lifting it for shared use, pushing it down when local, and avoiding unnecessary global state. Use when dealing with prop drilling, misplaced state, or deciding between local state, context, or global stores.
category: "React"
---

# State Colocation

## Phase 1 — Diagnose the Colocation Problem

Three distinct problems, three different fixes. Misidentifying them causes the wrong refactor.

| Symptom | Problem | Fix |
|---|---|---|
| Parent holds state, only one child reads/sets it | Over-lifted state | Push down |
| Two siblings need the same value | State too low | Lift up to nearest common ancestor |
| Prop passes through 2+ components that don't use it | Drill or wrong level | Push down OR context — see §2.3 |
| Context/store slice used by exactly one component | Over-globalized | Push down to local state |
| Re-renders cascade to unrelated subtrees on every state change | State too high | Push down or split context |

---

## Phase 2 — Non-Obvious Rules

### 2.1 "Nearest Common Ancestor" Is Not Always the Right Target for Lifting

The reflex is to lift to the nearest common ancestor. But if that ancestor is a layout component (e.g., `<Page>`, `<Shell>`) that renders many unrelated subtrees, lifting there causes the entire page to re-render on every keystroke. The real question is: *what is the nearest ancestor that is cheap to re-render?*

Decision: if the nearest common ancestor renders more than the two components that need the shared state, consider:
1. Composing the two components under a new lightweight wrapper that owns the state
2. Using a ref + callback pattern so the parent never re-renders (valid for "notify but don't display" cases)

### 2.2 Pushing State Down Is the Most Underused Refactor

State that started in a parent "for future flexibility" and never moved is the dominant cause of unnecessary re-renders. The signal is: **if you delete the prop from the parent's JSX and nothing breaks except a type error, the state belongs lower**.

Checklist before lifting state: verify the parent actually renders the value, not just passes it. If the parent only passes it, it shouldn't own it.

```tsx
// ❌ parent owns state it never renders
function Parent() {
  const [query, setQuery] = useState('');
  return <SearchBar query={query} onChange={setQuery} />;
  // Parent does nothing with query — SearchBar should own it
}

// ✅ state colocated with its only consumer
function SearchBar() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

### 2.3 Prop Drilling vs. Wrong Level — Different Fixes

Prop drilling (passing through intermediaries) has two root causes with different solutions:

**Cause A: State is at the right level but the tree is deep.**
Fix: context, or component composition (slot pattern). Don't lift or push — just stop threading through intermediaries.

**Cause B: State is at the wrong level.**
Fix: move the state, don't add context. Adding context to fix a wrong-level problem permanently couples unrelated components to that state.

Tell them apart: if the intermediary components would be *surprised* to know they're passing this prop (it has nothing to do with their job), that's Cause A. If the intermediary *is logically responsible* for the data flow, it's Cause B.

### 2.4 Context Is Not Global State — It's Dependency Injection

Context doesn't solve "where does the state live." State still lives somewhere; context just removes the threading. The state owner is still a component. Colocating context providers matters as much as colocating state:

```tsx
// ❌ theme context at app root when only the sidebar uses it
<AppRoot>
  <ThemeContext.Provider value={theme}> {/* re-renders everything */}
    ...
  </ThemeContext.Provider>
</AppRoot>

// ✅ provider scoped to the subtree that needs it
<Sidebar>
  <ThemeContext.Provider value={theme}> {/* only sidebar re-renders */}
    <SidebarNav />
    <SidebarFooter />
  </ThemeContext.Provider>
</Sidebar>
```

Place the provider at the lowest ancestor that fully contains all consumers — same colocation rule as state.

### 2.5 When Global State Is Actually Warranted

Global state (Zustand, Redux, Jotai, etc.) is warranted when:
1. **Persistence boundary** — state must survive the owning component unmounting (e.g., cached server responses, draft forms)
2. **Cross-cutting identity** — auth session, current user, feature flags; genuinely used by unrelated subtrees
3. **High-frequency writes with selective reads** — many components write, few and specific ones read; a store with selectors prevents re-render storms that context cannot

Global state is NOT warranted for: UI state that resets on navigation (open/closed modals, tab selection, hover state), state used by a single route's subtree, or state lifted there "to avoid prop drilling" without checking §2.3 first.

### 2.6 Derived State Is Not State

Before moving state, check if the consuming component is re-deriving the same value the parent already computed. If a child receives `items` and internally stores `filteredItems` in state, that's derived state masquerading as owned state. It should be computed inline or memoized, not lifted.

```tsx
// ❌ derived state — double source of truth, always stale on first render
const [filteredItems, setFilteredItems] = useState(items.filter(active));
useEffect(() => setFilteredItems(items.filter(active)), [items, active]);

// ✅ computed value — no state needed
const filteredItems = useMemo(() => items.filter(active), [items, active]);
```

Lifting or pushing derived state makes the architecture worse. Eliminate it first.

---

## Phase 3 — Execution

1. Map every piece of state: where it lives, what reads it, what mutates it.
2. Flag any state where the owner never renders the value directly.
3. Flag any prop that passes through a component without being used there.
4. For each flagged item: classify as push-down, lift-up, composition, or context — using §2.3 to distinguish drilling cause.
5. Check for derived state before recommending any move.
6. If lifting: identify the re-render cost of the new owner. If it's a layout component, propose a wrapper instead.

---

## Phase 4 — Output

Produce:
- Refactored component(s) with state at the correct level
- One-line rationale per state moved: what triggered the move, what problem it solves
- If context was added: confirm the provider is also colocated, not at app root
- If global state was recommended: name which of the three warranted cases applies

Do not produce: context added to fix a wrong-level problem, state lifted to a layout component that causes whole-page re-renders, or global state for UI-only concerns.