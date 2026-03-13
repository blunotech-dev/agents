# TypeScript Patterns for Compound Components

## Typed Context

```tsx
// Tabs.context.tsx
import { createContext, useContext } from 'react';

interface TabsContextValue {
  activeTab: string | null;
  setActiveTab: (value: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

export function useTabsContext(callerName: string): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error(`<${callerName}> must be rendered inside a <Tabs> component.`);
  }
  return ctx;
}

export default TabsContext;
```

## Typed Root Component with Controlled/Uncontrolled

```tsx
interface TabsProps {
  children: React.ReactNode;
  /** Uncontrolled: initial active tab */
  defaultValue?: string;
  /** Controlled: current active tab */
  value?: string;
  /** Controlled: change handler */
  onChange?: (value: string) => void;
  className?: string;
}

export function Tabs({ children, defaultValue, value, onChange, className }: TabsProps) {
  const [internalValue, setInternalValue] = useState<string | null>(defaultValue ?? null);
  const isControlled = value !== undefined;
  const activeTab = isControlled ? value : internalValue;

  const setActiveTab = (newValue: string) => {
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
  };

  return (
    <TabsContext.Provider value={{ activeTab: activeTab ?? null, setActiveTab }}>
      <div className={className}>{children}</div>
    </TabsContext.Provider>
  );
}
```

## Typed Sub-components

```tsx
interface TabsTabProps {
  value: string;
  children: React.ReactNode;
  disabled?: boolean;
  className?: string;
}

export function TabsTab({ value, children, disabled, className }: TabsTabProps) {
  const { activeTab, setActiveTab } = useTabsContext('Tabs.Tab');
  const isActive = activeTab === value;
  // ...
}
```

## Generic Context (for reusable patterns)

When building a utility that generates compound components, use a generic context factory:

```tsx
function createContext<T>(componentName: string) {
  const Context = React.createContext<T | null>(null);

  function useCtx(callerName: string): T {
    const ctx = React.useContext(Context);
    if (!ctx) {
      throw new Error(`<${callerName}> must be used within <${componentName}>.`);
    }
    return ctx;
  }

  return [Context, useCtx] as const;
}

// Usage:
const [TabsContext, useTabsContext] = createContext<TabsContextValue>('Tabs');
```

## Assembly with Types

```tsx
// index.tsx
import { Tabs as TabsRoot } from './Tabs';
import { TabsList } from './Tabs.List';
import { TabsTab } from './Tabs.Tab';
import { TabsPanel } from './Tabs.Panel';

export const Tabs = Object.assign(TabsRoot, {
  List: TabsList,
  Tab: TabsTab,
  Panel: TabsPanel,
});

export type { TabsProps } from './Tabs';
export type { TabsTabProps } from './Tabs.Tab';
export type { TabsPanelProps } from './Tabs.Panel';
```

## Forwarding Refs

For components that need ref forwarding (e.g., for focus management):

```tsx
export const TabsTab = React.forwardRef<HTMLButtonElement, TabsTabProps>(
  ({ value, children, disabled }, ref) => {
    const { activeTab, setActiveTab } = useTabsContext('Tabs.Tab');
    return (
      <button ref={ref} onClick={() => setActiveTab(value)} aria-selected={activeTab === value}>
        {children}
      </button>
    );
  }
);
TabsTab.displayName = 'Tabs.Tab';
```

## displayName for DevTools

Always set displayName on sub-components so React DevTools shows meaningful names:

```tsx
TabsRoot.displayName = 'Tabs';
TabsList.displayName = 'Tabs.List';
TabsTab.displayName = 'Tabs.Tab';
TabsPanel.displayName = 'Tabs.Panel';
```