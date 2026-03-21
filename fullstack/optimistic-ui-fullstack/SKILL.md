---
name: optimistic-ui-fullstack
description: Implement optimistic UI updates with instant state changes, rollback on failure, and user feedback. Use when making mutations feel immediate while handling errors, race conditions, and data consistency.
category: "Fullstack"
---

# Optimistic UI — Fullstack

## Discovery

Infer from context before asking:

1. **Data fetching library?** — React Query, SWR, RTK Query, tRPC, or raw fetch with local state?
2. **What's being mutated?** — list item (add/delete/reorder), field update, toggle, or form submit?
3. **Is ordering or ID generation involved?** — optimistic items need a temporary ID strategy
4. **What should happen on failure?** — silent rollback, toast, inline error, or retry prompt?

---

## The Three Failure Modes to Design For Upfront

Most implementations only handle the happy path. Design all three before writing code:

| Failure mode | Example | Wrong default | Correct behavior |
|---|---|---|---|
| **Network error** | No connection | Item disappears on rollback | Show error + restore item with retry |
| **Server rejection** | Validation fails (409, 422) | Silent rollback | Show server error message inline |
| **Race condition** | Two mutations in flight | Later response overwrites newer state | Cancel or queue; never merge stale response |

---

## Core Pattern (React Query — most common)

The non-obvious parts are in the ordering of `onMutate`, `onError`, and `onSettled`.

```ts
const queryClient = useQueryClient()

const mutation = useMutation({
  mutationFn: (newItem: CreateItemInput) => api.items.create(newItem),

  onMutate: async (newItem) => {
    // 1. Cancel in-flight refetches — they would overwrite the optimistic state
    await queryClient.cancelQueries({ queryKey: ["items"] })

    // 2. Snapshot current state BEFORE mutating (this is the rollback target)
    const previous = queryClient.getQueryData<Item[]>(["items"])

    // 3. Apply optimistic update with a temporary ID
    queryClient.setQueryData<Item[]>(["items"], (old = []) => [
      ...old,
      { ...newItem, id: crypto.randomUUID(), _optimistic: true },
    ])

    // 4. Return snapshot as context — React Query passes it to onError
    return { previous }
  },

  onError: (_err, _newItem, context) => {
    // Rollback to snapshot
    if (context?.previous) {
      queryClient.setQueryData(["items"], context.previous)
    }
  },

  onSettled: () => {
    // Always refetch after success OR failure — replaces optimistic ID with real one
    queryClient.invalidateQueries({ queryKey: ["items"] })
  },
})
```

**Why `cancelQueries` first**: Without it, a background refetch completing after `onMutate` will overwrite the optimistic item with stale server data. This is the most commonly missed step.

**Why `onSettled` always refetches**: The optimistic item has a fake ID. `onSuccess` alone isn't enough — if the mutation succeeds but `onSuccess` throws, the fake ID stays. `onSettled` fires regardless.

---

## Temporary ID Strategy

Never use `Date.now()` or incrementing integers as temporary IDs — they leak into the UI and can collide with real IDs if the backend uses sequential integers.

```ts
// ✅ Prefix-namespaced to make optimistic items identifiable
const tempId = `optimistic_${crypto.randomUUID()}`

// Use the prefix to filter/style pending items:
const isPending = item.id.startsWith("optimistic_")
```

Flag optimistic items explicitly in the type so the UI can render them differently:

```ts
type Item = { id: string; title: string }
type OptimisticItem = Item & { _optimistic: true }
type ListItem = Item | OptimisticItem

// Render pending state:
<li style={{ opacity: isPending ? 0.5 : 1 }}>
  {item.title}
  {isPending && <Spinner />}
</li>
```

---

## Delete: The Rollback UX Problem

Optimistic delete is the hardest case — if rollback restores the item, users often think their delete "worked" then "undid itself", which is confusing.

```ts
onMutate: async (id) => {
  await queryClient.cancelQueries({ queryKey: ["items"] })
  const previous = queryClient.getQueryData<Item[]>(["items"])

  // Mark as deleting instead of removing — gives rollback a better UX hook
  queryClient.setQueryData<Item[]>(["items"], (old = []) =>
    old.map(item => item.id === id ? { ...item, _deleting: true } : item)
  )

  return { previous }
},

onError: (_err, _id, context) => {
  queryClient.setQueryData(["items"], context?.previous)
  toast.error("Couldn't delete — item restored")  // explain the restoration
},
```

Style `_deleting` items with strikethrough + opacity so restoration doesn't look like a ghost appearing.

---

## Race Condition: Mutation Queue

When multiple mutations can fire on the same resource (e.g. user rapidly toggling a checkbox), naive optimistic updates will corrupt state.

**The problem:**

```
User clicks toggle → optimistic: true  → POST /items/1 (req A)
User clicks again  → optimistic: false → POST /items/1 (req B)
req A resolves → onSettled refetches → gets "false" (correct)
req B resolves → onSettled refetches AGAIN → gets "false" (still correct, but flickered)
```

**Worse case — req B resolves before req A:**

```
Server processes B (false) first, then A (true) → final state is "true" (wrong)
```

**Fix — debounce the mutation, not the UI update:**

```ts
// Optimistic state updates instantly (good UX)
// But the actual API call is debounced so only the final value is sent
const debouncedMutate = useMemo(
  () => debounce((value: boolean) => mutation.mutate(value), 500),
  []
)

const handleToggle = (value: boolean) => {
  setOptimisticValue(value)   // instant local state
  debouncedMutate(value)      // debounced server call
}
```

For cases where debounce isn't appropriate (sequential operations like reordering), use mutation queueing via `useMutation`'s `onMutate` to cancel the previous in-flight request before issuing the next.

---

## Error Feedback: Inline vs. Toast

The choice depends on whether the user can act on the error in place:

| Scenario | Pattern |
|---|---|
| Form field rejected (422 validation) | Inline error on the field — user can fix and retry |
| Network failure | Toast + retry button — no field to annotate |
| Conflict (409) | Toast with explanation — "Edited by someone else, reload to see changes" |
| Server 500 | Toast — user can't fix it |

Wire the server error message through, not a generic fallback:

```ts
onError: (err, _vars, context) => {
  queryClient.setQueryData(["items"], context?.previous)

  // Prefer the server's message over a generic one
  const message = err instanceof ApiError
    ? err.message                     // "Title must be under 100 characters"
    : "Something went wrong. Try again."

  toast.error(message)
}
```

**`ApiError` class pattern** — wrap fetch so API errors carry the server body:

```ts
class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
  }
}

async function apiFetch(url: string, init?: RequestInit) {
  const res = await fetch(url, init)
  if (!res.ok) {
    const body = await res.json().catch(() => ({ message: "Request failed" }))
    throw new ApiError(res.status, body.message ?? "Request failed")
  }
  return res.json()
}
```

---

## tRPC Variant

tRPC doesn't expose React Query's `onMutate` context directly — use `useUtils()` for cache access:

```ts
const utils = trpc.useUtils()

const mutation = trpc.items.create.useMutation({
  onMutate: async (newItem) => {
    await utils.items.list.cancel()
    const previous = utils.items.list.getData()

    utils.items.list.setData(undefined, (old = []) => [
      ...old,
      { ...newItem, id: `optimistic_${crypto.randomUUID()}`, _optimistic: true },
    ])

    return { previous }
  },
  onError: (_err, _vars, context) => {
    if (context?.previous) utils.items.list.setData(undefined, context.previous)
  },
  onSettled: () => utils.items.list.invalidate(),
})
```

---

## Output Checklist

- [ ] `cancelQueries` called before every optimistic `setQueryData`
- [ ] Snapshot stored in `onMutate` and returned as context
- [ ] `onSettled` invalidates regardless of success/failure (not just `onSuccess`)
- [ ] Temporary IDs are prefixed, not `Date.now()` or sequential integers
- [ ] Optimistic items visually flagged (opacity, spinner, strikethrough for deletes)
- [ ] Rollback paired with user-visible explanation (not silent)
- [ ] Server error message surfaced, not replaced with generic string
- [ ] Race condition addressed: debounce or queue for rapidly-fired mutations