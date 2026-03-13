# Memoization Tradeoffs

## When Memoization Helps vs Hurts

Memoization is a tradeoff: you pay memory and comparison cost upfront to avoid render cost later. It only pays off when the render cost exceeds the comparison cost.

### Memoization HELPS when:

- The component renders many times with the same props (stable parent, frequent unrelated state changes)
- The component is expensive to render (complex layout, large lists, heavy computation)
- The memoized value is a non-primitive (object, array, function) passed as a prop to a memoized child
- The computation is genuinely expensive (sort + filter of 1000+ items, regex matching, tree traversal)

### Memoization HURTS (or is neutral) when:

- The component almost always renders with different props — the shallow comparison runs every time and never saves a render
- The computation is trivial — `useMemo(() => a + b, [a, b])` costs more than it saves
- The memoized value is a primitive — `useMemo(() => 'hello', [])` is pure overhead
- The deps array is unstable — if deps change every render, memo never skips

### The Comparison Cost

`React.memo` does a shallow equality check on all props. This means:

```tsx
// Cheap comparison (primitives)
<Button label="Click" count={5} disabled={false} />

// More expensive comparison (many props or nested checks)
<ComplexForm 
  config={config}   // object — only checks reference, not deep equality
  validators={validators}
  onSubmit={onSubmit}
  initialValues={initialValues}
  // ... 10 more props
/>
```

For components with many props, the comparison overhead can be non-trivial. Measure before assuming memo helps.

---

## Rules of Thumb for Large Lists

### Virtualization beats memoization for very long lists

If you have 500+ items, `React.memo` on list items helps but virtualization (only rendering visible items) is dramatically more effective.

```bash
npm install @tanstack/react-virtual
# or: npm install react-window
```

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // estimated row height in px
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: rowVirtualizer.getTotalSize() }}>
        {rowVirtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{ transform: `translateY(${virtualItem.start}px)` }}
          >
            <ListItem item={items[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Render count comparison for 1000 items on filter change:**
- No optimization: 1000 renders
- React.memo on item: 0–1000 renders (depends on how many match)
- Virtualization: 15–20 renders (only visible items)
- Both: 0–20 renders (best case)

### Key stability matters more than memo for lists

```tsx
// ❌ Index key — causes remount on every reorder/filter
items.map((item, index) => <Row key={index} item={item} />)

// ✅ Stable ID — React reuses the component instance
items.map(item => <Row key={item.id} item={item} />)
```

Unstable keys cause full unmount/remount, which is far more expensive than a re-render. Fix keys before adding memo.

---

## Benchmark Pattern

When unsure if memoization is helping, measure it:

```tsx
// Temporarily add render counting to confirm the problem
const renders = useRef(0);
if (process.env.NODE_ENV === 'development') {
  renders.current++;
  console.log(`[${componentName}] render #${renders.current}`);
}

// Then add memo, confirm render count drops
// Then remove the logging
```

Or use the Profiler component with `baseDuration` comparison:

```tsx
// baseDuration = estimated render time without any memoization
// actualDuration = actual render time this commit
// If actualDuration << baseDuration, memoization is working
// If actualDuration ≈ baseDuration, memoization isn't helping
```

---

## The Premature Optimization Trap

Kent C. Dodds' rule: **"Memoize when you have a measured performance problem, not before."**

Signs you're over-memoizing:
- Every `useState` callback is wrapped in `useCallback`
- Every object literal is wrapped in `useMemo`
- Components have `React.memo` added "just in case"
- The deps arrays are long and hard to reason about

Signs you're under-memoizing (wait for these before acting):
- DevTools shows the same component rendering 50+ times per second
- A state change in one part of the app causes the entire tree to re-render
- Expensive derived data (sorts, filters, transforms) recomputes on every keystroke
- A list of 100+ items re-renders in full when only one item changes

---

## useMemo vs useCallback — They're the Same Thing

```tsx
// These are equivalent:
const fn = useCallback(() => doSomething(a, b), [a, b]);
const fn = useMemo(() => () => doSomething(a, b), [a, b]);
```

`useCallback(fn, deps)` is just `useMemo(() => fn, deps)` — syntactic sugar for the function case. Use `useCallback` for functions, `useMemo` for values.

---

## Compiler-Based Memoization (React 19+)

React 19 ships with the **React Compiler** (formerly React Forget) which automatically inserts memoization where it's safe to do so. If you're on React 19+:

- The compiler handles most `useMemo` / `useCallback` needs automatically
- You may not need to add manual memoization for new code
- Existing manual memoization is still valid and won't conflict
- Check if your project has `babel-plugin-react-compiler` or `eslint-plugin-react-compiler` configured

```bash
# Check if React Compiler is enabled in your project
cat babel.config.js | grep react-compiler
cat next.config.js | grep reactCompiler
```

If the React Compiler is active, focus performance work on:
1. State architecture (co-location, moving state down)
2. Virtualization for long lists
3. Context splitting (compiler doesn't fix context re-renders)