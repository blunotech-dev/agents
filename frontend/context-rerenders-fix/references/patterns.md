# Advanced Context Patterns

## Combined Fix A+B: Split + Memoize

The most robust pattern: split by concern AND memoize each value.

```tsx
// auth-context.tsx
interface AuthContextValue {
  user: User | null;
  isLoading: boolean;
}
interface AuthActionsValue {
  login: (creds: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthStateContext = createContext<AuthContextValue>(null!);
const AuthActionsContext = createContext<AuthActionsValue>(null!);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  // Actions are stable — only created once
  const actions = useMemo<AuthActionsValue>(() => ({
    login: async (creds) => {
      setIsLoading(true);
      const user = await api.login(creds);
      setUser(user);
      setIsLoading(false);
    },
    logout: () => setUser(null),
  }), []); // ← no deps: these never need to change

  // State changes on login/logout
  const state = useMemo<AuthContextValue>(
    () => ({ user, isLoading }),
    [user, isLoading]
  );

  return (
    <AuthActionsContext.Provider value={actions}>
      <AuthStateContext.Provider value={state}>
        {children}
      </AuthStateContext.Provider>
    </AuthActionsContext.Provider>
  );
}

// Consumers choose what they need
export const useAuthState = () => useContext(AuthStateContext);
export const useAuthActions = () => useContext(AuthActionsContext);

// A logout button only needs actions — never re-renders on user/loading change
const LogoutButton = () => {
  const { logout } = useAuthActions(); // ← stable, never re-renders
  return <button onClick={logout}>Log out</button>;
};
```

---

## Handler Memoization in Context

If you skip `useCallback` on handlers, `useMemo` on the value object won't help.

```tsx
// ❌ useMemo is useless here — addItem is recreated every render
function CartProvider({ children }) {
  const [cart, setCart] = useState([]);

  const addItem = (item) => setCart(prev => [...prev, item]); // new ref each render!

  const value = useMemo(() => ({ cart, addItem }), [cart]); // addItem always "changed"

  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

// ✅ Stable handler with useCallback
function CartProvider({ children }) {
  const [cart, setCart] = useState([]);

  const addItem = useCallback((item: CartItem) => {
    setCart(prev => [...prev, item]);
  }, []); // ← no deps needed: uses functional updater

  const value = useMemo(() => ({ cart, addItem }), [cart, addItem]);

  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

---

## Context Selector Pattern (without external library)

Simulate Zustand-style selectors using a subscription ref pattern. Use when you can't add a dependency but need fine-grained subscriptions.

```tsx
// context-selector.tsx
import { createContext, useContext, useRef, useSyncExternalStore } from 'react';

function createSelectableContext<T>(defaultValue: T) {
  const Context = createContext<{ getSnapshot: () => T; subscribe: (cb: () => void) => () => void }>(null!);

  function Provider({ value, children }: { value: T; children: React.ReactNode }) {
    const storeRef = useRef(value);
    const listenersRef = useRef(new Set<() => void>());

    storeRef.current = value;

    const store = useRef({
      getSnapshot: () => storeRef.current,
      subscribe: (cb: () => void) => {
        listenersRef.current.add(cb);
        return () => listenersRef.current.delete(cb);
      },
    }).current;

    // Notify listeners when value changes
    useRef(() => {
      listenersRef.current.forEach(cb => cb());
    });

    return <Context.Provider value={store}>{children}</Context.Provider>;
  }

  function useSelector<S>(selector: (state: T) => S): S {
    const store = useContext(Context);
    return useSyncExternalStore(store.subscribe, () => selector(store.getSnapshot()));
  }

  return { Provider, useSelector };
}

// Usage
const { Provider: AppProvider, useSelector: useAppSelector } = createSelectableContext(initialState);

// Only re-renders when user.name changes — ignores cart/theme
const UserName = () => {
  const name = useAppSelector(state => state.user.name);
  return <span>{name}</span>;
};
```

> **Note:** This is a last resort. If you need selectors, use Zustand — it does this out of the box.

---

## Provider Composition Helper

Avoid deep nesting when splitting context into many providers:

```tsx
// providers.tsx
type Provider = ({ children }: { children: React.ReactNode }) => JSX.Element;

function composeProviders(...providers: Provider[]) {
  return function ComposedProviders({ children }: { children: React.ReactNode }) {
    return providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      children as JSX.Element
    );
  };
}

export const AppProviders = composeProviders(
  AuthProvider,
  ThemeProvider,
  CartProvider,
  NotificationProvider,
);

// In main.tsx
root.render(
  <AppProviders>
    <App />
  </AppProviders>
);
```