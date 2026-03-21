---
name: deep-linking
description: Implement deep linking so URL state drives UI (filters, modals, tabs, pagination), enabling shareable links and proper browser navigation. Use when syncing UI state to query params, preserving state on refresh, or fixing back/forward behavior.
category: "Fullstack"
---

# Deep Linking

Covers the non-obvious parts of URL-driven state: which history method to use, how to avoid re-render loops, handling defaults without polluting the URL, and keeping multiple independent state slices in sync. Skips basic routing setup — assumes a router exists.

---

## Discovery

Before writing anything, answer:

1. **What state belongs in the URL?** Filters, pagination, selected tab, open modal, sort order — or all of these?
2. **Framework/router**: React Router, Next.js (App Router or Pages), Vue Router, vanilla `history` API?
3. **Shareable vs navigable**: Should back-button step through filter changes, or only page navigations?
4. **Default values**: Should defaults appear in the URL (`?page=1`) or be omitted (cleaner, but requires careful hydration)?
5. **Conflicts**: Does any state also live in a store (Zustand, Redux)? URL must be the source of truth, not both.

---

## Core Patterns

### 1. `push` vs `replace` — The Back Button Decision

**The trap**: using `push` for every state change makes the back button step through every filter tweak the user made — unusable.

| Use `push` (creates history entry) | Use `replace` (overwrites current entry) |
|---|---|
| User explicitly navigates (tab change, pagination) | Filter/sort changes within the same view |
| Modal open that should be back-button-closeable | Autocomplete, debounced search input |
| Distinct "pages" of a wizard | Clearing a filter |

```ts
// React Router v6
import { useSearchParams } from 'react-router-dom';

const [searchParams, setSearchParams] = useSearchParams();

// Filter change — replace, not push
setSearchParams({ ...Object.fromEntries(searchParams), status: 'active' }, { replace: true });

// Tab change — push (back button returns to previous tab)
setSearchParams({ ...Object.fromEntries(searchParams), tab: 'settings' });
```

**Non-obvious**: `setSearchParams` replaces the entire search string by default. Always spread existing params or you'll silently drop other state.

---

### 2. Avoiding the Re-render Loop

**The trap**: reading from URL, updating state, which triggers a URL update, which triggers a read... infinite loop.

```ts
// BAD: dual source of truth causes loop
const [filters, setFilters] = useState({ status: 'active' });

useEffect(() => {
  // This writes back to URL, which triggers this effect again
  setSearchParams({ status: filters.status });
}, [filters]);

// GOOD: URL is the only source of truth; derive state directly
const [searchParams, setSearchParams] = useSearchParams();
const status = searchParams.get('status') ?? 'active'; // default inline

function setStatus(value: string) {
  setSearchParams(prev => {
    const next = new URLSearchParams(prev);
    if (value === 'active') next.delete('status'); // omit defaults
    else next.set('status', value);
    return next;
  }, { replace: true });
}
```

Never maintain a parallel `useState` that mirrors URL params. Read directly from the URL on every render.

---

### 3. Handling Defaults Without Polluting the URL

Two options — pick one and be consistent:

**Option A: Omit defaults (cleaner URLs)**
```ts
const DEFAULT_PAGE = 1;
const DEFAULT_SORT = 'createdAt';

const page = Number(searchParams.get('page') ?? DEFAULT_PAGE);
const sort = searchParams.get('sort') ?? DEFAULT_SORT;

// On change, delete param if it matches the default
function setPage(n: number) {
  setSearchParams(prev => {
    const next = new URLSearchParams(prev);
    if (n === DEFAULT_PAGE) next.delete('page');
    else next.set('page', String(n));
    return next;
  }, { replace: true });
}
```

**Option B: Always include defaults (predictable, easier to parse)**
```ts
// Initialize URL with defaults on mount if params are missing
useEffect(() => {
  const next = new URLSearchParams(searchParams);
  let changed = false;
  if (!next.has('page')) { next.set('page', '1'); changed = true; }
  if (!next.has('sort')) { next.set('sort', 'createdAt'); changed = true; }
  if (changed) setSearchParams(next, { replace: true });
}, []); // intentionally empty — run once on mount
```

**Non-obvious**: Option B's `useEffect` with empty deps will run on every mount including back-navigation, which is usually fine but can cause a flash if the URL briefly lacks the params.

---

### 4. Modals in the URL

Modals that open via URL let users share a link to a specific record, and back-button closes the modal naturally.

```ts
// URL: /users?modal=edit&id=123

function UserList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const modalType = searchParams.get('modal');  // 'edit' | 'delete' | null
  const modalId   = searchParams.get('id');

  function openModal(type: string, id: string) {
    // push — back button should close the modal
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.set('modal', type);
      next.set('id', id);
      return next;
    });
  }

  function closeModal() {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.delete('modal');
      next.delete('id');
      return next;
    }, { replace: true }); // replace — closing shouldn't create a history entry
  }

  return (
    <>
      <UserTable onEdit={(id) => openModal('edit', id)} />
      {modalType === 'edit' && modalId && (
        <EditUserModal userId={modalId} onClose={closeModal} />
      )}
    </>
  );
}
```

**Non-obvious**: closing the modal should use `replace`, not `push`. If you `push` on close, the back button re-opens the modal — which is rarely what users expect.

---

### 5. Multiple Independent State Slices

**The trap**: multiple components each managing their own search params slice will clobber each other because `setSearchParams` replaces all params.

```ts
// Reusable hook that safely merges its own slice
function useQueryParam(key: string, defaultValue: string, replace = true) {
  const [searchParams, setSearchParams] = useSearchParams();
  const value = searchParams.get(key) ?? defaultValue;

  const setValue = useCallback((next: string) => {
    setSearchParams(prev => {
      const params = new URLSearchParams(prev); // preserve other params
      if (next === defaultValue) params.delete(key);
      else params.set(key, next);
      return params;
    }, { replace });
  }, [key, defaultValue, replace, setSearchParams]);

  return [value, setValue] as const;
}

// Used independently in two components — no clobbering
const [status, setStatus] = useQueryParam('status', 'all', true);
const [tab, setTab]       = useQueryParam('tab', 'overview', false); // push
```

---

### 6. Next.js App Router Specifics

`useSearchParams` in the App Router is read-only. To write, use `useRouter` + `router.push/replace`.

```ts
'use client';
import { useSearchParams, useRouter, usePathname } from 'next/navigation';
import { useCallback } from 'react';

export function useNextQueryParam(key: string, defaultValue: string) {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const value = searchParams.get(key) ?? defaultValue;

  const setValue = useCallback((next: string) => {
    const params = new URLSearchParams(searchParams.toString());
    if (next === defaultValue) params.delete(key);
    else params.set(key, next);
    // replace to avoid polluting history on filter changes
    router.replace(`${pathname}?${params.toString()}`);
  }, [key, defaultValue, router, pathname, searchParams]);

  return [value, setValue] as const;
}
```

**Non-obvious**: in the App Router, `useSearchParams()` must be wrapped in `<Suspense>` in a Server Component tree or you'll get a build error. Wrap the component using URL params, not the whole page.

```tsx
// page.tsx (Server Component)
import { Suspense } from 'react';
import { FilterBar } from './FilterBar'; // 'use client' inside

export default function Page() {
  return (
    <Suspense fallback={<FilterSkeleton />}>
      <FilterBar />
    </Suspense>
  );
}
```

---

### 7. Serializing Complex State (Arrays, Objects)

Search params are strings. Arrays and nested objects need a consistent serialization strategy.

```ts
// Arrays — use repeated keys (native URLSearchParams support)
// URL: ?tag=react&tag=typescript
const tags = searchParams.getAll('tag'); // ['react', 'typescript']

function setTags(values: string[]) {
  setSearchParams(prev => {
    const next = new URLSearchParams(prev);
    next.delete('tag');
    values.forEach(v => next.append('tag', v));
    return next;
  }, { replace: true });
}

// Objects / ranges — use dot notation keys or JSON (keep it readable)
// URL: ?price.min=10&price.max=100  (preferred: human-readable)
// URL: ?price=%7B%22min%22%3A10%7D  (avoid: opaque when shared)

const priceMin = Number(searchParams.get('price.min') ?? 0);
const priceMax = Number(searchParams.get('price.max') ?? 1000);
```

**Non-obvious**: `JSON.stringify` in query params produces percent-encoded blobs that break readability and sharing. Flatten nested state into dot-notation keys instead.

---

### 8. Validation and Type Safety

Raw URL params are untyped strings from user input — validate before use.

```ts
const VALID_TABS = ['overview', 'settings', 'billing'] as const;
type Tab = typeof VALID_TABS[number];

function parseTab(raw: string | null): Tab {
  if (VALID_TABS.includes(raw as Tab)) return raw as Tab;
  return 'overview'; // fallback to default, don't throw
}

const tab = parseTab(searchParams.get('tab'));

// For numbers, guard against NaN
const page = Math.max(1, Number(searchParams.get('page')) || 1);

// For enums from an API, validate against the known set at runtime
const VALID_STATUSES = new Set(['active', 'inactive', 'pending']);
const status = VALID_STATUSES.has(searchParams.get('status') ?? '')
  ? searchParams.get('status')!
  : 'active';
```

---

## Output

Produce:
- `useQueryParam.ts` — reusable hook that safely merges a single param without clobbering others; accepts `push`/`replace` option
- `useFilters.ts` — domain-specific hook composing `useQueryParam` for a concrete filter set, with typed defaults and validation
- Component integration example showing modal URL pattern and tab URL pattern with correct push/replace choices

Flag clearly in comments:
- Every `push` vs `replace` decision and why
- Where defaults are omitted vs included and the chosen convention
- Any framework-specific constraint (Next.js Suspense requirement, read-only `useSearchParams`)