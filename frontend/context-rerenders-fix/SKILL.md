---
name: context-rerenders-fix
description: Identify and fix unnecessary React re-renders caused by Context by splitting providers or memoizing values. Use when users report slow React performance, excessive re-renders, or Context-related lag, or ask how to optimize useContext or large Context providers.
category: "Frontend"
---

# Context Re-renders Fix

Diagnose and fix React Context re-renders through context splitting, memoization, or store extraction.

---

## Why Context Causes Re-renders

Every component that calls `useContext(MyContext)` **re-renders whenever the context value changes** — even if the component only uses one field from a large object. This is the most common Context performance pitfall.

```tsx
// ❌ Any change to user, theme, OR cart re-renders ALL consumers
const AppContext = createContext({ user, theme, cart, notifications });
```

---

## Step 1: Diagnose

Before fixing, identify the actual problem. Ask the user or infer from their code:

### Checklist — confirm these before proceeding:

- [ ] Which Context(s) are involved?
- [ ] How often does the context value change? (On every keystroke? Every route change? Only on login?)
- [ ] How many components consume this context?
- [ ] Are consumers close together (same subtree) or spread across the app?
- [ ] Is the value object recreated on every parent render?

### Red flags to look for in code:

| Pattern | Problem |
|---|---|
| `value={{ user, theme, cart }}` inline in JSX | New object reference every render |
| Single context with 5+ unrelated fields | All consumers re-render on any field change |
| `useContext` in a component that only uses 1 of 10 fields | Unnecessary subscription |
| Context provider inside a frequently-updating parent | Value changes propagate constantly |
| No `useMemo` wrapping the context value | Object identity changes on every render |

---

## Step 2: Choose the Right Fix

Use this decision tree:

```
Does the context value change frequently (e.g., on every keystroke or scroll)?
  └─ YES → Is it avoidable? Can state be moved out of context?
       └─ YES → Move high-frequency state to local state or URL state (not context)
       └─ NO  → Use useMemo + consider splitting

Does the context hold multiple unrelated concerns (user + theme + cart)?
  └─ YES → SPLIT THE CONTEXT (Fix A)

Does the context hold one cohesive object that still causes re-renders?
  └─ YES → MEMOIZE THE VALUE (Fix B)

Is the context very widely used across many subtrees with different needs?
  └─ YES → EXTRACT TO EXTERNAL STORE (Fix C) — Zustand / Jotai

Are consumers doing expensive work on each render?
  └─ YES → MEMOIZE CONSUMERS with React.memo / useMemo (Fix D)
```

---

## Fix A: Split the Context

**When:** Single context mixes unrelated concerns. Splitting means components only subscribe to what they need.

```tsx
// ❌ Before — one fat context
const AppContext = createContext<{
  user: User;
  theme: Theme;
  cart: CartItem[];
  setTheme: (t: Theme) => void;
  addToCart: (item: CartItem) => void;
}>(null!);

// ✅ After — three focused contexts
const UserContext = createContext<User>(null!);
const ThemeContext = createContext<{ theme: Theme; setTheme: (t: Theme) => void }>(null!);
const CartContext = createContext<{ cart: CartItem[]; addToCart: (item: CartItem) => void }>(null!);

// Compose providers cleanly
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <UserProvider>
      <ThemeProvider>
        <CartProvider>{children}</CartProvider>
      </ThemeProvider>
    </UserProvider>
  );
}
```

**Result:** A `CartItem` update only re-renders `CartContext` consumers — `UserContext` and `ThemeContext` consumers are unaffected.

---

## Fix B: Memoize the Context Value

**When:** Context holds a cohesive object but the provider's parent re-renders frequently, creating a new object reference each time.

```tsx
// ❌ Before — new object on every render of AuthProvider's parent
function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

// ✅ After — stable object reference, only changes when user changes
function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const value = useMemo(() => ({ user, setUser }), [user]);
  // Note: setUser is stable (from useState), so [user] is the only real dep

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Tip:** Separate stable setters from changing values — see Fix A+B combined pattern in `references/patterns.md`.

---

## Fix C: Extract to an External Store

**When:** Context is used across many unrelated subtrees, changes frequently, or you need fine-grained subscriptions.

Zustand and Jotai both use selector-based subscriptions — components only re-render when the slice they select changes.

```tsx
// ❌ Before — Context causes all consumers to re-render on any field change
const useAppContext = () => useContext(AppContext);
const UserName = () => {
  const { user } = useAppContext(); // re-renders when cart changes!
  return <span>{user.name}</span>;
};

// ✅ After — Zustand with selector
import { create } from 'zustand';

interface AppStore {
  user: User;
  cart: CartItem[];
  theme: Theme;
  setTheme: (t: Theme) => void;
}

const useAppStore = create<AppStore>((set) => ({
  user: initialUser,
  cart: [],
  theme: 'light',
  setTheme: (theme) => set({ theme }),
}));

// Only re-renders when user changes — cart/theme changes are ignored
const UserName = () => {
  const userName = useAppStore((state) => state.user.name);
  return <span>{userName}</span>;
};
```

**When to prefer Jotai:** When state is naturally atomic and composed bottom-up (individual atoms vs a single store object).

See `references/zustand-patterns.md` for common Zustand migration patterns.

---

## Fix D: Memoize Expensive Consumers

**When:** The context value must change frequently and splitting/memoizing the provider isn't enough — the consumer itself does expensive work.

```tsx
// ✅ Wrap the consumer component in React.memo
// Re-renders only when its own props change, not when unrelated context changes
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  // ...expensive render
});

// ✅ Memoize derived values inside a consumer
function ProductGrid() {
  const { products, filters } = useProductContext();

  // Only recomputes when products or filters change
  const filtered = useMemo(
    () => products.filter((p) => matchesFilters(p, filters)),
    [products, filters]
  );

  return <ExpensiveList items={filtered} />;
}
```

**Caution:** `React.memo` checks **prop** changes, not context changes. If the component calls `useContext` directly, it will still re-render on context change. Memoization helps most for children passed via props.

---

## Step 3: Output Format

Always produce:

### 3a. Diagnosis Summary
- Which contexts are causing unnecessary re-renders
- Root cause (inline value object / fat context / high-frequency updates / no memoization)
- Estimated blast radius (how many components are affected)

### 3b. Fix Plan
- Which fix(es) to apply (A, B, C, D) and why
- Order of application (start with highest-impact)

### 3c. Before/After Code
- Show the problematic provider and at least one affected consumer
- Show the fixed version with comments explaining what changed and why

### 3d. Verification Steps
Tell the user how to confirm the fix worked:
```
1. Install React DevTools
2. Enable "Highlight updates when components render" in the Profiler
3. Trigger the state change that was causing the problem
4. Verify only the expected components highlight — not the whole tree
```
Or with the Profiler API:
```tsx
<Profiler id="Cart" onRender={(id, phase, duration) => console.log(id, phase, duration)}>
  <CartSection />
</Profiler>
```

---

## Common Mistakes to Warn About

- **Don't memoize everything preemptively.** `useMemo` has overhead — only apply it where re-renders are actually measured as problematic.
- **`useCallback` for handlers in context value.** If the context value includes functions, wrap them in `useCallback` too — otherwise `useMemo` on the object won't help since the function reference changes.
- **Context is not a replacement for a cache.** If the data comes from an API, use React Query/SWR — don't fight re-renders caused by a pattern that shouldn't exist in the first place.
- **Splitting context too aggressively** creates provider hell. Aim for cohesion — split by domain concern, not by individual field.

---

## Reference Files

- `references/patterns.md` — Combined Fix A+B patterns, handler memoization, context selector pattern
- `references/zustand-patterns.md` — Zustand migration recipes and selector patterns

Read these when the user's situation requires more detailed patterns than the examples above.