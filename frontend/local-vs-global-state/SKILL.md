---
name: local-vs-global-state
description: Audit a React component tree and classify state as local, shared, or server state, then recommend a refactored structure. Use when users ask about state management, prop drilling, global stores, or where state should live, especially when reviewing component trees or state architecture.
category: "Frontend"
---

# Local vs Global State Auditor

Audit a component tree and classify every piece of state, then produce a concrete refactored structure with rationale.

---

## Overview

When a user shares components, a file tree, or describes their app's state, follow this process:

1. **Extract** all state declarations (`useState`, `useReducer`, stores, context values, server fetches)
2. **Classify** each piece of state into one of three categories
3. **Identify** problems (prop drilling, redundant global state, missing cache layer, etc.)
4. **Output** a refactored structure with code snippets

---

## Step 1: Extract State

Scan for:
- `useState` / `useReducer` hooks
- Context providers and their values
- Global store slices (Redux, Zustand, Jotai, etc.)
- Data fetching (useEffect + fetch, React Query, SWR, tRPC)
- URL/router state (params, search params)
- Form state (react-hook-form, Formik, uncontrolled)
- Derived/computed state

If the user hasn't shared code, ask them to share the component tree or describe what state exists and where.

---

## Step 2: Classify Each Piece of State

Use the decision tree below for each state item.

### Classification Decision Tree

```
Is this state used by only ONE component?
  └─ YES → LOCAL STATE
     (useState inside that component)

Is this state used by MULTIPLE components in the same subtree?
  └─ YES → Is the subtree small (2–4 components)?
       └─ YES → SHARED LOCAL STATE (lift to nearest common ancestor)
       └─ NO  → SHARED STATE via Context or lightweight store

Does this state come from / sync with a server/API/DB?
  └─ YES → SERVER STATE
     (React Query, SWR, tRPC — not useState + useEffect)

Is this state needed across MANY unrelated subtrees OR persists across navigation?
  └─ YES → GLOBAL STATE (Zustand, Redux, Jotai atom)

Is this state better represented in the URL?
  └─ YES → URL STATE (useSearchParams, router params)
```

### Category Definitions

| Category | Definition | Recommended Tool |
|---|---|---|
| **Local** | Used only within one component or its direct children | `useState`, `useReducer` |
| **Shared (lifted)** | Used by siblings or cousins in a small subtree | Lift to parent, pass as props |
| **Shared (context)** | Used across a medium subtree, avoid prop-drilling | React Context + `useContext` |
| **Global** | App-wide, cross-route, or persisted | Zustand / Jotai / Redux |
| **Server** | Async data from an API, cache + invalidation needed | React Query / SWR / tRPC |
| **URL** | Shareable, bookmarkable, browser-back-compatible | `useSearchParams` / router |
| **Form** | Ephemeral input state before submission | `react-hook-form` / local |

---

## Step 3: Identify Problems

Flag these anti-patterns:

- **Prop drilling** — state passed through 3+ levels of components that don't use it
- **Redundant global state** — state in a store that is only read by one component
- **useState for server data** — fetching in useEffect and storing in useState (use React Query instead)
- **Stale closure / sync issues** — derived state stored in useState instead of computed
- **God context** — a single Context that holds everything, causing re-renders
- **Missing memoization** — expensive derived values recomputed on every render
- **URL-worthy state in memory** — filters, pagination, tabs that break on refresh

---

## Step 4: Output Format

Always produce these sections:

### 4a. State Audit Table

```
| State Item         | Current Location    | Current Type | Recommended Type | Reason |
|--------------------|---------------------|--------------|------------------|--------|
| isModalOpen        | App.tsx             | useState     | Local (Modal.tsx) | Only used in Modal |
| currentUser        | multiple useEffects | useState     | Server (React Query) | Async, needs cache |
| selectedFilters    | SearchPage          | useState     | URL state        | Shareable/bookmarkable |
```

### 4b. Problems Found

Bulleted list of anti-patterns detected, with the component name and state variable.

### 4c. Recommended Architecture Diagram

ASCII or description of the new component tree with state ownership annotated:

```
App
├── AuthProvider (currentUser — React Query)
├── Layout
│   ├── Sidebar
│   │   └── navOpen: local useState       ← was prop-drilled from App
│   └── Main
│       ├── SearchPage
│       │   └── filters: URL (useSearchParams) ← was useState, lost on refresh
│       └── Modal (isOpen: local useState)   ← was in App, now co-located
```

### 4d. Refactored Code Snippets

Provide before/after code for the most impactful changes. Prioritize:
1. Removing prop drilling
2. Moving server state to React Query/SWR
3. Moving local state down (co-location)
4. Replacing useState for URL-worthy state

Use concise, copy-pasteable snippets. Prefer showing the hook/store definition + one consumer.

### 4e. Summary Recommendation

One paragraph summarizing: what changed, why it improves the codebase, and any caveats (e.g., "if you add authentication later, promote `currentUser` to Zustand").

---

## Heuristics & Rules of Thumb

- **Co-locate state as close to where it's used as possible.** Only lift or globalize when there's a real need.
- **Server state is not application state.** Avoid mirroring API responses into a Redux store — use a cache layer.
- **URL state is underused.** Filters, tabs, pagination, and modal IDs usually belong in the URL.
- **Context is not a performance solution.** Large Context values cause all consumers to re-render. Use Zustand or split contexts for high-frequency updates.
- **The best global state is less global state.** Before adding to a store, ask: can this be derived? Can it be local?

---

## Handling Ambiguity

If the user shares partial code or describes their app verbally:
- Make reasonable assumptions and state them explicitly
- Ask 1–2 focused clarifying questions if critical info is missing (e.g., "Is `currentUser` shared across routes, or only used in the header?")
- Still produce a recommendation — a directional answer is more useful than waiting for perfect information

If the user names a framework other than React (Vue, Svelte, Angular):
- Apply the same classification logic
- Adapt tool recommendations: Pinia for Vue, stores for Svelte, services/signals for Angular
- See `references/non-react-frameworks.md` for framework-specific guidance

---

## Example Classification

**Input state description:**
> `userData` fetched with useEffect in `App.tsx`, passed down to `Header`, `Profile`, and `Settings` pages via props. `modalOpen` boolean in `App.tsx` passed to `Modal`.

**Output:**
- `userData` → **Server State** (React Query in a `useCurrentUser` hook, no prop drilling)
- `modalOpen` → **Local State** (move into `Modal.tsx` itself, or the component that triggers it)

**Problem flagged:** Prop drilling `userData` through 3 components; `useState` for async server data.