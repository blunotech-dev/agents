---
name: react-query-mutations
description: Implement TanStack Query mutations with proper optimistic updates, including cancel → snapshot → update → rollback sequencing, cache invalidation, and loading/error handling. Use when working with useMutation, fixing rollback issues, or is invalidating too early/late.
category: "React"
---

# react-query-mutations

Implements TanStack Query mutations correctly — focused on the optimistic update lifecycle, which is where almost all real mistakes happen.

---

## Phase 1 — Discover

Before writing any mutation:

- Does this mutation need **optimistic updates**? (Instant UI feedback before API confirms.) If no, the implementation is trivial — skip to Phase 3B.
- What **query key** holds the data being mutated? (Needed for cancel, snapshot, and invalidation.)
- Is the cache entry a **list** (add/remove item) or a **single record** (update in place)? The optimistic update logic differs.
- Should the UI reflect **only the mutation's loading state**, or also **per-item loading states**? (The latter requires storing mutation variables in local state alongside the cache update.)

---

## Phase 2 — The Optimistic Update Lifecycle (and where it breaks)

The four callbacks run in this order: `onMutate` → (API call) → `onSuccess` OR `onError` → `onSettled`.

### What people get wrong

**Forgetting to cancel outgoing queries before snapshot**
If a background refetch is in-flight when `onMutate` fires, the refetch can land *after* your optimistic update and overwrite it. The cancel must happen before the snapshot, not after.

```ts
// ❌ Race condition — refetch can still land and overwrite optimistic state
onMutate: async (newItem) => {
  const previous = queryClient.getQueryData(key)  // snapshot before cancel
  await queryClient.cancelQueries({ queryKey: key })
  // ...
}

// ✓ Cancel first, then snapshot
onMutate: async (newItem) => {
  await queryClient.cancelQueries({ queryKey: key })
  const previous = queryClient.getQueryData(key)
  // ...
}
```

**Returning the snapshot incorrectly**
`onMutate` must return the snapshot as `{ previousData }` (or any object). TanStack Query passes this return value as `context` to `onError` and `onSettled`. If you forget the return, rollback is impossible without a closure hack.

**Rolling back in `onError` but not cleaning up in `onSettled`**
`onSettled` fires regardless of success or failure. Invalidation belongs in `onSettled`, not `onSuccess` — otherwise a failed mutation leaves the cache stale indefinitely.

```ts
// ❌ Stale cache on failure
onSuccess: () => queryClient.invalidateQueries({ queryKey: key }),
onError: (err, vars, context) => queryClient.setQueryData(key, context.previousData),

// ✓ Always invalidate, roll back only on error
onError: (err, vars, context) => queryClient.setQueryData(key, context.previousData),
onSettled: () => queryClient.invalidateQueries({ queryKey: key }),
```

**`onSettled` invalidation refetching the old data**
When `onSettled` invalidates immediately after `onError` rolls back, the invalidation triggers a refetch which returns the correct server state and replaces the rollback. This is actually correct behavior — don't suppress it.

---

## Phase 3 — Implementation Patterns

### A. Optimistic update — list (add item)

```ts
const queryKey = ['todos']

const addTodo = useMutation({
  mutationFn: (newTodo: NewTodo) => api.post('/todos', newTodo),

  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey })
    const previousTodos = queryClient.getQueryData<Todo[]>(queryKey)

    queryClient.setQueryData<Todo[]>(queryKey, (old = []) => [
      ...old,
      { ...newTodo, id: crypto.randomUUID(), optimistic: true },
    ])

    return { previousTodos }
  },

  onError: (_err, _vars, context) => {
    queryClient.setQueryData(queryKey, context?.previousTodos)
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey })
  },
})
```

**Why `crypto.randomUUID()` for the temp ID**: The optimistic item needs a stable key for React reconciliation. Don't use `Date.now()` — it's not unique enough under fast mutations. The real ID arrives via `onSettled` refetch.

### B. Optimistic update — update in place

```ts
const updateTodo = useMutation({
  mutationFn: ({ id, ...patch }: UpdateTodo) => api.patch(`/todos/${id}`, patch),

  onMutate: async ({ id, ...patch }) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })
    const previousTodos = queryClient.getQueryData<Todo[]>(['todos'])

    queryClient.setQueryData<Todo[]>(['todos'], (old = []) =>
      old.map(todo => todo.id === id ? { ...todo, ...patch } : todo)
    )

    return { previousTodos }
  },

  onError: (_err, _vars, context) => {
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

### C. No optimistic update (simple invalidation)

```ts
const deleteTodo = useMutation({
  mutationFn: (id: string) => api.delete(`/todos/${id}`),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['todos'] }),
})
```

Use `onSuccess` here (not `onSettled`) — you only want to refetch if the delete succeeded.

### D. Non-list mutation (single resource)

When mutating a single cached record:

```ts
const queryKey = ['todo', id]

onMutate: async (patch) => {
  await queryClient.cancelQueries({ queryKey })
  const previous = queryClient.getQueryData<Todo>(queryKey)
  queryClient.setQueryData<Todo>(queryKey, old => ({ ...old!, ...patch }))
  return { previous }
},
```

---

## Phase 4 — Loading and Error States

### Basic wiring

```ts
const { mutate, isPending, isError, error, reset } = addTodo

// isPending is true from mutate() call until onSettled completes
// isError persists until reset() or the next mutate() call
```

### Per-item loading state (lists)

`isPending` is global to the mutation instance — it doesn't tell you *which* item is loading. For per-row spinners:

```ts
// Track in the mutation itself via variables
const { mutate, isPending, variables } = updateTodo

// In the list item:
const isUpdating = isPending && variables?.id === todo.id
```

`variables` holds the arguments from the most recent `mutate()` call. This is available synchronously during the pending state — no local state needed.

### Error display and retry

```ts
// After error, mutation stays in error state. Call reset() to clear.
<button onClick={() => mutate(payload)}>
  {isPending ? 'Saving...' : isError ? 'Retry' : 'Save'}
</button>
{isError && <p>{error.message}</p>}
```

---

## Phase 5 — Output

Produce a complete mutation hook, either inline or extracted. Always include:

1. **All four callbacks** if optimistic — even if `onSuccess` is absent, comment why.
2. **Typed `context`** in `onError` — `context?.previousData` without a type is a common source of silent bugs.
3. **The `variables` pattern** for per-item loading state if the component renders a list.
4. Call out explicitly if `onSettled` vs `onSuccess` for invalidation and why.

---

## Non-obvious Rules to Enforce

**`cancelQueries` is async — always await it**
It returns a promise that resolves when all matching queries have been cancelled. Skipping the await means the cancel may not complete before your snapshot, and the race condition from Phase 2 applies.

**`setQueryData` updater function vs direct value**
Always use the updater function form `(old) => newValue` when the update depends on current cache state. Passing a value directly can cause stale closure bugs in concurrent renders.

**Multiple query keys for the same data**
If the same resource is cached under multiple keys (e.g., `['todos']` list AND `['todo', id]` individual), you must snapshot and rollback both. Miss one and the invalidation may restore stale data to the other.

**`invalidateQueries` after `setQueryData` in `onSettled`**
The invalidation marks the query stale and triggers a background refetch — it does not immediately clear the cache. The UI won't flash blank; it will show the optimistic state until the refetch resolves. This is intentional.

**Mutation deduplication doesn't exist**
Unlike queries, mutations don't deduplicate. Calling `mutate()` three times fires three API requests. If the user's UI allows rapid re-triggering, add a `disabled={isPending}` guard on the trigger element.