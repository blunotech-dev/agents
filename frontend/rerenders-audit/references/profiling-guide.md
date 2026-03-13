# Profiling Guide

## React DevTools Profiler — Step by Step

### Installation
- Chrome/Firefox: Install "React Developer Tools" from the browser extension store
- Only works in development mode (`NODE_ENV=development`)

### Recording a Profile

1. Open DevTools → **React** tab → **Profiler** subtab
2. Click the ⏺ **Record** button
3. Perform the interaction that feels slow (click a button, type in an input, open a dropdown)
4. Click ⏹ **Stop**

### Reading the Flame Graph

- Each bar = one component render
- **Width** = time spent rendering (wider = slower)
- **Color** = relative render time (yellow/orange = slow, blue/teal = fast, gray = did not render)
- **Nested bars** = child components rendered within the parent

**What to look for:**
- Wide bars in unexpected places (components that shouldn't be doing work)
- Components appearing in multiple commits when they should only render once
- Entire subtrees lighting up from a single state change

### "Why did this render?" Panel

Click any bar in the flame graph → right panel shows:
- **Props changed**: which props had a new value
- **State changed**: which state variable triggered the render  
- **Hooks changed**: which hook (by index) caused the render
- **Parent rendered**: parent re-rendered and this component isn't memoized

This is the most important tool — it tells you exactly what to fix.

### Commit Timeline

The bar chart above the flame graph shows each "commit" (React batch of updates).
- Tall bars = expensive commits
- Click each bar to see what rendered in that commit
- Compare adjacent commits — if the same component appears in every commit, it's re-rendering too often

### Profiler Settings (enable these)

React DevTools → ⚙️ Settings → Profiler:
- ✅ **Record why each component rendered** — required for "Why did this render?" panel
- ✅ **Hide commits below X ms** — filter out trivial renders to find real problems

---

## Highlight Updates (Quick Visual Profiling)

React DevTools → ⚙️ Settings → General:
- ✅ **Highlight updates when components render**

Interact with the page:
- **Blue outline** = component re-rendered
- **Green/yellow** = rendered multiple times in quick succession
- **Red** = rendering very frequently (likely a problem)

Use this to quickly spot components re-rendering outside the area you interacted with.

---

## React Scan Setup

React Scan draws red boxes around re-rendering components in real time — faster feedback than DevTools for finding hot spots.

```bash
npm install react-scan
# or: npx react-scan@latest <your-dev-url>
```

```tsx
// src/main.tsx or src/index.tsx — development only
import { scan } from 'react-scan';

if (process.env.NODE_ENV === 'development') {
  scan({
    enabled: true,
    log: true,          // console.log each re-render
    showToolbar: true,  // floating toolbar in the UI
  });
}
```

The toolbar shows:
- Total renders
- Components rendering right now (highlighted in red)
- Render count per component

```tsx
// You can also wrap specific trees
import { withScan } from 'react-scan';
export default withScan(MySlowComponent, { log: true });
```

---

## Reading a Profiler Trace — Example

```
Commit 3 (18ms)
├── App [2ms] — parent rendered
│   ├── Header [0.1ms] — parent rendered ← unnecessary
│   ├── Sidebar [0.1ms] — parent rendered ← unnecessary  
│   └── Dashboard [16ms] — state changed (filters)
│       ├── FilterBar [0.5ms] — props changed (filters) ← necessary
│       └── ProductGrid [15ms] — parent rendered ← INVESTIGATE
│           ├── ProductCard × 200 [14ms total] — parent rendered ← unnecessary
```

**Analysis:**
- `Dashboard` re-rendered because `filters` state changed — correct
- `FilterBar` re-rendered because `filters` prop changed — correct
- `ProductGrid` re-rendered because parent did, but products didn't change → wrap in `React.memo`
- 200x `ProductCard` re-rendered because `ProductGrid` did → `React.memo` on `ProductGrid` fixes all of these

**One fix (memo on ProductGrid) eliminates 201 unnecessary renders per filter change.**

---

## Programmatic Profiling

Use the `Profiler` component to log render times in code:

```tsx
import { Profiler } from 'react';

function onRenderCallback(
  id: string,           // component name passed to Profiler
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,   // time to render this commit
  baseDuration: number,     // estimated time without memoization
  startTime: number,
  commitTime: number
) {
  if (actualDuration > 16) { // flag anything over one frame (60fps)
    console.warn(`[Profiler] ${id} took ${actualDuration.toFixed(2)}ms (${phase})`);
  }
}

// Wrap the subtree you want to measure
<Profiler id="ProductGrid" onRender={onRenderCallback}>
  <ProductGrid products={products} />
</Profiler>
```

**Useful ratios:**
- `baseDuration / actualDuration` ≈ memoization effectiveness (higher = more benefit from memo)
- If `actualDuration` ≈ `baseDuration`, memoization isn't helping

---

## Performance Budget (rule of thumb)

| Scenario | Target render time |
|---|---|
| 60fps interactions | < 16ms per commit |
| 30fps acceptable | < 33ms per commit |
| Initial page load | < 100ms first render |
| List items (per item) | < 1ms |

If a single component exceeds these budgets, it's worth investigating regardless of re-render frequency.