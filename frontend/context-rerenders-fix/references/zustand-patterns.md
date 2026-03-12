# Zustand Migration Patterns

## Basic Context → Zustand Migration

```tsx
// ❌ Before: Context with re-render issues
interface StoreState {
  user: User | null;
  cart: CartItem[];
  theme: 'light' | 'dark';
  setUser: (u: User) => void;
  addToCart: (item: CartItem) => void;
  setTheme: (t: 'light' | 'dark') => void;
}

const AppContext = createContext<StoreState>(null!);

// ✅ After: Zustand store
import { create } from 'zustand';

const useAppStore = create<StoreState>((set) => ({
  user: null,
  cart: [],
  theme: 'light',
  setUser: (user) => set({ user }),
  addToCart: (item) => set((state) => ({ cart: [...state.cart, item] })),
  setTheme: (theme) => set({ theme }),
}));

// Consumers use selectors — only re-render on selected slice change
const useUser = () => useAppStore((s) => s.user);
const useCart = () => useAppStore((s) => s.cart);
const useTheme = () => useAppStore((s) => s.theme);
```

---

## Slice Pattern (for large stores)

```tsx
// user-slice.ts
interface UserSlice {
  user: User | null;
  setUser: (u: User | null) => void;
}

const createUserSlice = (set: any): UserSlice => ({
  user: null,
  setUser: (user) => set({ user }),
});

// cart-slice.ts
interface CartSlice {
  cart: CartItem[];
  addToCart: (item: CartItem) => void;
  removeFromCart: (id: string) => void;
}

const createCartSlice = (set: any): CartSlice => ({
  cart: [],
  addToCart: (item) => set((s: any) => ({ cart: [...s.cart, item] })),
  removeFromCart: (id) => set((s: any) => ({ cart: s.cart.filter((i: CartItem) => i.id !== id) })),
});

// store.ts
type AppStore = UserSlice & CartSlice;

const useAppStore = create<AppStore>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}));
```

---

## Jotai Alternative (atomic model)

Better than Zustand when state is naturally atomic or derived.

```tsx
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// Atoms
const userAtom = atom<User | null>(null);
const cartAtom = atom<CartItem[]>([]);
const themeAtom = atom<'light' | 'dark'>('light');

// Derived atom (computed, no re-render if inputs unchanged)
const cartCountAtom = atom((get) => get(cartAtom).length);
const cartTotalAtom = atom((get) =>
  get(cartAtom).reduce((sum, item) => sum + item.price * item.quantity, 0)
);

// Components subscribe only to what they read
const CartBadge = () => {
  const count = useAtomValue(cartCountAtom); // only re-renders when count changes
  return <span>{count}</span>;
};

const ThemeToggle = () => {
  const [theme, setTheme] = useAtom(themeAtom);
  return <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>{theme}</button>;
};
```

**Choose Jotai when:**
- State naturally decomposes into independent atoms
- You have many derived/computed values
- You want React Suspense integration for async atoms

**Choose Zustand when:**
- State is a cohesive object with related fields
- You want devtools, persistence middleware, or immer integration
- You're migrating from Redux and want a familiar pattern