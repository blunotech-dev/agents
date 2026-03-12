---
name: rerenders-audit
description: Profile and fix unnecessary React re-renders using memo, useCallback, and useMemo, explaining what changed and why. Use when users report slow React performance, frequent re-renders, laggy UI, or ask how to profile, memoize, or optimize components.
category: "Frontend"
---

# Re-renders Audit

Profile React components, identify unnecessary re-renders, and apply the right fix — with a clear explanation of what changed and why it works.

---

## Mental Model First

Re-renders are not inherently bad. React is designed to re-render. The goal is to eliminate **unnecessary** re-renders — ones where the output would be identical to the previous render.

A component re-renders when:
1. Its own **state** changes
2. Its **parent** re-renders (even if props didn't change)
3. A **context** it consumes changes
4. Its **key** changes

Fixes only help when the re-render is truly unnecessary. Always measure before and after.

---

## Step 1: Profile First

Never guess. Before writing any fix, identify what's actually re-rendering and why.

### Option A: React DevTools Profiler (recommended)
1. Open React DevTools → **Profiler** tab
2. Click **Record**, interact with the slow part of the UI, click **Stop**
3. Inspect the flame graph — look for components that rendered when they shouldn't have
4. Click a component bar → check **"Why did this render?"** panel on the right

### Option B: Highlight Updates (quick visual check)
- React DevTools → ⚙️ Settings → **"Highlight updates when components render"**
- Interact with the UI — blue/green flashes = re-renders
- Look for components outside the interaction area flashing

### Option C: Add render logging (when you can't use DevTools)
```tsx
// Drop this inside any component to count renders
const renderCount = useRef(0);
console.log(`[MyComponent] render #${++renderCount.current}`, { props });
```

### Option D: React Scan (third-party, excellent DX)
```bash
npm install react-scan
```
```tsx
// In your entry point
import { scan } from 'react-scan';
scan({ enabled: true, log: true });
```
Draws red outlines around re-rendering components with render count overlays.

---

## Step 2: Classify the Re-render

Once you've identified the component, classify the cause:

| Cause | Symptom | Fix |
|---|---|---|
| Parent re-renders, child output unchanged | Child re-renders with same props | `React.memo` on child |
| Inline object/array prop | New reference every render | `useMemo` for the value |
| Inline function prop | New reference every render | `useCallback` for the handler |
| Derived value recomputed each render | Expensive calculation runs unnecessarily | `useMemo` for the computation |
| Context re-render | All consumers re-render on any context change | See `context-rerenders-fix` skill |
| Key instability | Component unmounts/remounts on each render | Fix key to be stable identifier |
| State too high | Unrelated state change triggers wide re-render | Move state down (co-locate) |

---

## Step 3: Apply the Right Fix

### Fix 1: React.memo — Stop child re-renders from parent

**When:** A child component re-renders because its parent does, but its props haven't changed.

```tsx
// ❌ Before — re-renders every time Parent re-renders
function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

// ✅ After — skips render if user prop is the same reference
const UserCard = React.memo(function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
});
```

**Why it works:** `React.memo` wraps the component in a shallow equality check on props. If all props are `===` to their previous values, React skips the render entirely and reuses the last output.

**Pitfall:** If a prop is an object or function created inline in the parent, it will always be a new reference — `memo` won't help without also fixing the prop. Pair with `useMemo`/`useCallback`.

**Custom comparator (use sparingly):**
```tsx
const UserCard = React.memo(UserCardFn, (prev, next) => {
  return prev.user.id === next.user.id; // only re-render if ID changes
});
```

---

### Fix 2: useCallback — Stable function references

**When:** A function is defined inside a component and passed as a prop. New function reference on every render breaks `memo` on the child.

```tsx
// ❌ Before — new onDelete reference every render, breaks memo on Row
function Table({ rows }: { rows: Row[] }) {
  const handleDelete = (id: string) => {
    deleteRow(id); // or setState(...)
  };

  return rows.map(row => <Row key={row.id} row={row} onDelete={handleDelete} />);
}

// ✅ After — stable reference, Row's memo check passes
function Table({ rows }: { rows: Row[] }) {
  const handleDelete = useCallback((id: string) => {
    deleteRow(id);
  }, []); // ← deps: add anything from closure that can change

  return rows.map(row => <Row key={row.id} row={row} onDelete={handleDelete} />);
}
```

**Why it works:** `useCallback(fn, deps)` returns the same function reference between renders as long as `deps` haven't changed. Children receiving it via props will see a stable reference and skip re-renders.

**Common deps mistake:**
```tsx
// ❌ Stale closure — items is captured from first render only
const handleSubmit = useCallback(() => {
  processItems(items); // items may be stale!
}, []);

// ✅ Correct deps
const handleSubmit = useCallback(() => {
  processItems(items);
}, [items]);

// ✅ Or use functional updater to avoid the dep entirely
const handleAdd = useCallback((item: Item) => {
  setItems(prev => [...prev, item]); // no `items` dep needed
}, []);
```

---

### Fix 3: useMemo — Stable object/array references and expensive calculations

**Two distinct use cases:**

#### 3a. Stable reference for object/array props

```tsx
// ❌ Before — new config object every render, breaks memo on Chart
function Dashboard({ data }: { data: DataPoint[] }) {
  const chartConfig = { color: 'blue', animate: true }; // new ref each render

  return <Chart data={data} config={chartConfig} />;
}

// ✅ After — same reference until deps change
function Dashboard({ data }: { data: DataPoint[] }) {
  const chartConfig = useMemo(() => ({ color: 'blue', animate: true }), []);

  return <Chart data={data} config={chartConfig} />;
}
```

#### 3b. Expensive computation

```tsx
// ❌ Before — full sort + filter runs on every render
function ProductList({ products, query, sortBy }: Props) {
  const results = products
    .filter(p => p.name.toLowerCase().includes(query.toLowerCase()))
    .sort((a, b) => a[sortBy] > b[sortBy] ? 1 : -1);

  return results.map(p => <ProductCard key={p.id} product={p} />);
}

// ✅ After — only recomputes when products, query, or sortBy change
function ProductList({ products, query, sortBy }: Props) {
  const results = useMemo(() => 
    products
      .filter(p => p.name.toLowerCase().includes(query.toLowerCase()))
      .sort((a, b) => a[sortBy] > b[sortBy] ? 1 : -1),
    [products, query, sortBy]
  );

  return results.map(p => <ProductCard key={p.id} product={p} />);
}
```

**Why it works:** `useMemo(fn, deps)` caches the return value and only recomputes when deps change. For objects/arrays, this preserves reference identity. For calculations, it avoids redundant work.

**When NOT to use useMemo:**
- The computation is trivial (adding two numbers, string concat)
- The value is already a primitive (string, number, boolean)
- The deps change as often as the component renders anyway

---

### Fix 4: Move State Down (Co-location)

**When:** State lives higher than it needs to, causing broad re-renders when only a small subtree cares.

```tsx
// ❌ Before — isOpen in parent causes entire parent + siblings to re-render
function Page() {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <ExpensiveSection /> {/* re-renders on every toggle! */}
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} />
      <button onClick={() => setIsOpen(true)}>Open</button>
    </>
  );
}

// ✅ After — modal state lives inside its own component
function Page() {
  return (
    <>
      <ExpensiveSection /> {/* never re-renders from modal state */}
      <ModalWithTrigger />
    </>
  );
}

function ModalWithTrigger() {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open</button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} />
    </>
  );
}
```

**Why it works:** State changes only re-render the component that owns the state and its descendants. By moving state down, the blast radius shrinks.

---

### Fix 5: Lift JSX / Children as Props

**When:** A parent re-renders frequently, but some children don't depend on the changing state. Pass them as `children` or props — they won't re-render.

```tsx
// ❌ Before — SlowTree re-renders every time count changes
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <SlowTree /> {/* re-renders on every count change */}
    </div>
  );
}

// ✅ After — SlowTree is passed as children, doesn't re-render
function CounterWrapper({ children }: { children: React.ReactNode }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children} {/* SlowTree's JSX is created by the parent, not here */}
    </div>
  );
}

function App() {
  return (
    <CounterWrapper>
      <SlowTree />
    </CounterWrapper>
  );
}
```

**Why it works:** `children` is a prop. When `CounterWrapper` re-renders, React sees the same `children` reference (created in `App`, which didn't re-render) and bails out of re-rendering `SlowTree`.

---

## Step 4: Output Format

Always produce:

### 4a. Re-render Audit Table

```
| Component     | Re-render Cause                        | Necessary? | Fix Applied        |
|---------------|----------------------------------------|------------|--------------------|
| UserCard      | Parent (Dashboard) re-renders          | No         | React.memo         |
| Table         | onDelete new ref each render           | No         | useCallback        |
| ProductList   | Full filter+sort on every render       | No         | useMemo            |
| ModalTrigger  | isOpen state lifted too high           | No         | Move state down    |
| CountDisplay  | count state changed                    | Yes        | None needed        |
```

### 4b. Root Cause Explanation
For each unnecessary re-render, one sentence explaining why it happens — no jargon assumed.

### 4c. Before/After Code Snippets
Focused diffs for each fix. Include a comment on the fix line explaining the mechanism.

### 4d. Why Each Fix Works
After each snippet, a 2–3 sentence plain-English explanation:
- What the fix does mechanically
- Why the previous code was triggering a render
- Any caveats or follow-up to watch for

### 4e. Verification
How to confirm the fix worked — DevTools steps or a render-count log pattern.

---

## Decision Guide: Which Tool to Reach For

```
Is the component re-rendering because its parent does, with unchanged props?
  └─ Wrap component in React.memo

Is a prop that's an object or array created inline in the parent?
  └─ useMemo for that value in the parent

Is a prop that's a function created inline in the parent?
  └─ useCallback for that function in the parent

Is a derived value expensive to compute and computed every render?
  └─ useMemo for that computation

Is state higher than it needs to be?
  └─ Move state down to the component that owns it

Does a frequently-updating parent contain stable subtrees?
  └─ Pass stable subtrees as children/props

Is a Context causing broad re-renders?
  └─ See context-rerenders-fix skill
```

---

## Anti-patterns to Flag

- **Memoizing everything** — adds overhead, makes code harder to read, and often doesn't help if deps change as often as the render
- **Missing deps in useCallback/useMemo** — leads to stale closures and subtle bugs; always include all referenced variables
- **React.memo on components with frequently-changing props** — the shallow comparison itself costs time with no benefit
- **Object/array in useCallback deps** — `[{ id }]` will never be equal; extract the primitive: `[id]`
- **Skipping the profiler** — fixing the wrong thing is worse than fixing nothing; always measure first

---

## Reference Files

- `references/profiling-guide.md` — Detailed React DevTools Profiler walkthrough, React Scan setup, reading flame graphs
- `references/memoization-tradeoffs.md` — When memoization helps vs hurts, benchmark patterns, rules of thumb for large lists