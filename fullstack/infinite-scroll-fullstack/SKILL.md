---
name: infinite-scroll-fullstack
description: Implement infinite scroll end-to-end with cursor-based pagination, IntersectionObserver, loading states, and scroll position preservation. Use when building feeds, “load more” lists, or replacing offset pagination with cursor-based loading.
category: "Fullstack"
---

# Infinite Scroll — Fullstack

## Discovery

Infer from context, then confirm:

1. **Data fetching library?** — React Query (`useInfiniteQuery`), SWR, RTK Query, or raw fetch?
2. **Current pagination type?** — Offset (`page=2&limit=20`) or already cursor-based?
3. **Navigation pattern?** — SPA (no page reload) or MPA/SSR (scroll position lost on back-nav)?
4. **Data mutates while user scrolls?** — Feed where new items are inserted at the top (cursor matters more), or stable dataset?

---

## Cursor vs. Offset: Pick Cursor

Offset pagination is tempting because it's simple, but breaks under any of these conditions:

| Problem | Offset | Cursor |
|---|---|---|
| Row inserted while user scrolls | Skips or duplicates item at page boundary | Unaffected |
| Large offset on deep pages | `OFFSET 10000` does a full table scan | Constant time via indexed column |
| "Load more" UX | Works but degrades | Designed for it |

Use offset only if the dataset is static and small. Otherwise cursor.

---

## Backend: Cursor Pagination API

### Cursor Design (non-obvious)

The cursor encodes *where you are*, not *what page you're on*. Encode it as a base64 opaque string — callers should never parse or construct cursors manually.

```ts
// Encode: combine sort key + id to handle ties (same createdAt)
function encodeCursor(row: { id: string; createdAt: Date }): string {
  return Buffer.from(JSON.stringify({
    id: row.id,
    createdAt: row.createdAt.toISOString(),
  })).toString("base64url")
}

function decodeCursor(cursor: string): { id: string; createdAt: string } {
  return JSON.parse(Buffer.from(cursor, "base64url").toString())
}
```

**Why include `id` in the cursor alongside `createdAt`?** If two rows share the exact same `createdAt`, pure timestamp cursors produce nondeterministic page boundaries. The `id` tie-breaks deterministically.

### Prisma Query

```ts
async function getItems(cursor?: string, limit = 20) {
  const decoded = cursor ? decodeCursor(cursor) : null

  const items = await prisma.item.findMany({
    take: limit + 1,  // fetch one extra to determine if there's a next page
    ...(decoded && {
      cursor: { id: decoded.id },
      skip: 1,         // skip the cursor row itself
    }),
    orderBy: [
      { createdAt: "desc" },
      { id: "desc" },  // tie-break — must match cursor fields
    ],
  })

  const hasNextPage = items.length > limit
  const page = hasNextPage ? items.slice(0, limit) : items

  return {
    items: page,
    nextCursor: hasNextPage ? encodeCursor(page[page.length - 1]) : null,
  }
}
```

**`take: limit + 1` pattern** — fetching one extra row is cheaper than a separate `COUNT` query. If you get `limit + 1` rows back, there's a next page; slice off the extra before returning.

**Response shape:**
```ts
type PaginatedResponse<T> = {
  items: T[]
  nextCursor: string | null  // null = end of list
}
```

Never return `hasNextPage: boolean` separately — `nextCursor === null` already encodes this. Fewer fields, no state sync needed.

---

## Frontend: `useInfiniteQuery`

```ts
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
  status,
} = useInfiniteQuery({
  queryKey: ["items"],
  queryFn: ({ pageParam }) =>
    fetch(`/api/items${pageParam ? `?cursor=${pageParam}` : ""}`).then(r => r.json()),

  initialPageParam: null,
  getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  // undefined = no next page (React Query convention); null would refetch with null cursor
})

// Flatten pages into a single array — the only correct pattern
const items = data?.pages.flatMap(p => p.items) ?? []
```

**`null` vs `undefined` in `getNextPageParam`**: React Query treats `undefined` as "no next page." Return `null` by mistake and it passes `null` as `pageParam` to the next fetch — your backend gets `cursor=null` as a string. Always return `undefined` (or nothing) to signal end of list.

---

## IntersectionObserver Sentinel

Place an invisible sentinel element at the bottom of the list. When it enters the viewport, fetch the next page.

```tsx
function useInfiniteScrollTrigger(
  fetchNextPage: () => void,
  hasNextPage: boolean,
  isFetchingNextPage: boolean,
) {
  const sentinelRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const el = sentinelRef.current
    if (!el) return

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage()
        }
      },
      { rootMargin: "200px" }  // trigger 200px before sentinel is visible
    )

    observer.observe(el)
    return () => observer.disconnect()
  }, [fetchNextPage, hasNextPage, isFetchingNextPage])

  return sentinelRef
}
```

**`rootMargin: "200px"`** — triggers the fetch before the user hits the bottom, so the next page is ready before they need it. Without this, users see a loading spinner on every page boundary.

**Re-create the observer when deps change**, not just on mount. If `fetchNextPage` reference changes (it does when `queryKey` changes), a stale closure will call the wrong function.

### Usage

```tsx
function ItemList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, status } =
    useInfiniteQuery(/* ... */)

  const items = data?.pages.flatMap(p => p.items) ?? []
  const sentinelRef = useInfiniteScrollTrigger(fetchNextPage, hasNextPage, isFetchingNextPage)

  if (status === "pending") return <SkeletonList />
  if (status === "error") return <ErrorState />

  return (
    <>
      <ul>
        {items.map(item => <ItemRow key={item.id} item={item} />)}
      </ul>

      {/* Sentinel — always rendered, invisible */}
      <div ref={sentinelRef} aria-hidden />

      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && items.length > 0 && <p>You've reached the end</p>}
    </>
  )
}
```

**Always render the sentinel**, even when `!hasNextPage`. Conditionally rendering it causes the observer to reconnect/disconnect on every page fetch, which can misfire. Gate the `fetchNextPage` call in the observer callback instead.

---

## Scroll Position Preservation

The hardest problem in infinite scroll. When a user navigates away and back, the list rerenders from the top.

### React Query caches pages automatically

With `staleTime` set, React Query keeps all fetched pages in memory and restores them on remount — no extra work needed for in-memory navigation.

```ts
useInfiniteQuery({
  queryKey: ["items"],
  queryFn: ...,
  staleTime: 5 * 60 * 1000,  // pages stay fresh for 5 min, no refetch on remount
})
```

### Restoring scroll offset (SPA)

React Query restores data, but not the browser's scroll position. Store and restore it explicitly:

```ts
// On unmount, save scroll position keyed to the list
const scrollKey = "items-scroll"

useEffect(() => {
  const saved = sessionStorage.getItem(scrollKey)
  if (saved) window.scrollTo(0, parseInt(saved))

  return () => {
    sessionStorage.setItem(scrollKey, String(window.scrollY))
  }
}, [])
```

**`sessionStorage` over `localStorage`**: scroll position is session-specific and should not persist across browser sessions or tabs.

### The layout shift problem

Restoring scroll before images/dynamic content loads produces a wrong final position. Use `content-visibility: auto` on list items to reserve height before render:

```css
.item-row {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;  /* estimated height — browser reserves this space */
}
```

This lets the browser know the approximate height before layout, so scroll restoration lands close to the right position even before images load.

---

## New Items at the Top (Feed Pattern)

When items can be inserted at the top while the user is mid-scroll, appending them immediately pushes existing content down — jarring UX.

**Pattern: buffer new items, show "N new items" banner**

```ts
const [buffered, setBuffered] = useState<Item[]>([])

// On real-time event (SSE/WebSocket):
onNewItem((item) => setBuffered(prev => [item, ...prev]))

// Banner click flushes buffer to top of list:
const flushBuffer = () => {
  queryClient.setQueryData(["items"], (old) => ({
    ...old,
    pages: [{ items: buffered, nextCursor: old.pages[0].nextCursor }, ...old.pages],
  }))
  setBuffered([])
  window.scrollTo({ top: 0, behavior: "smooth" })
}
```

```tsx
{buffered.length > 0 && (
  <button onClick={flushBuffer}>
    ↑ {buffered.length} new {buffered.length === 1 ? "item" : "items"}
  </button>
)}
```

---

## Loading States

| State | What to show |
|---|---|
| Initial load (`status === "pending"`) | Skeleton list — same shape as real items |
| Fetching next page (`isFetchingNextPage`) | Spinner below last item |
| End of list (`!hasNextPage`) | "You've reached the end" — always show, absence is confusing |
| Empty (`status === "success"` and 0 items) | Empty state — separate from "end of list" |
| Error on initial load | Full error state with retry |
| Error fetching next page | Inline error with retry button below the list |

React Query exposes `isError` and `error` per mutation/query — surface next-page errors inline rather than replacing the whole list.

---

## Output Checklist

- [ ] Cursor encodes both sort key and `id` for tie-breaking
- [ ] Cursor is base64-opaque; callers never construct or parse it
- [ ] Backend uses `take: limit + 1` to detect next page without COUNT query
- [ ] `nextCursor: null` signals end of list (no separate `hasNextPage` field)
- [ ] `getNextPageParam` returns `undefined` (not `null`) to signal no next page
- [ ] Pages flattened with `.flatMap` before rendering
- [ ] Sentinel always rendered; page guard is inside observer callback
- [ ] `rootMargin` set to pre-fetch before user reaches bottom
- [ ] `staleTime` set so back-navigation restores pages without refetch
- [ ] Scroll offset saved to `sessionStorage` on unmount, restored on mount
- [ ] `content-visibility: auto` with `contain-intrinsic-size` on list items
- [ ] Feed with top-insertions uses buffer + banner pattern
- [ ] Next-page errors shown inline, not as full-page error