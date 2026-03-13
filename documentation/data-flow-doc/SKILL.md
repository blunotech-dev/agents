---
name: data-flow-doc
description: Document how data flows through a feature or system from user action to API to state to UI — producing a step-by-step trace and Mermaid diagram. Use whenever a user wants to map, trace, or document data movement - "how does X work end-to-end", "document this feature", "trace what happens when a user does Y", or when they share code and ask how it works.
category: "Documentation"
---

# Data Flow Documentation Skill

This skill produces structured, human-readable data-flow documents that trace how data moves
through a system — from the moment a user takes an action all the way through APIs, services,
state management, and UI updates.

The output is always both **readable prose** (for understanding) and **diagram-friendly structure**
(for Mermaid, sequence diagrams, or hand-off to design tools).

---

## What a Data Flow Doc covers

A complete data-flow document answers these questions:

1. **Trigger** — What initiates the flow? (user gesture, timer, webhook, etc.)
2. **Propagation** — How does the event/data travel? (function calls, network requests, queues)
3. **Transformation** — What shape does the data take at each step?
4. **State changes** — What gets written where? (local state, cache, DB, URL)
5. **UI consequence** — What does the user see / feel as a result?
6. **Error paths** — What happens when something goes wrong?

---

## Output Format

Always produce **three sections** in this order:

### 1. Flow Summary (1–3 sentences)
Plain English. What is this flow, and why does it exist?

### 2. Step-by-Step Trace
Numbered steps. Each step has:
- **Layer label** — one of: `User`, `UI Component`, `Hook/Store`, `Service/Util`, `API`, `Database`, `External`
- **Action** — what happens (imperative verb phrase)
- **Data shape** — what the data looks like at this point (inline TypeScript-ish type, JSON snippet, or plain English)
- **Notes** — optional: side-effects, async behavior, error handling

Format each step as:

```
N. [Layer] → Action
   Data: <shape or example>
   Notes: <optional>
```

### 3. Diagram
A Mermaid `sequenceDiagram` (preferred for request/response flows) or `flowchart TD`
(preferred for branching/decision flows). Choose based on the flow type.

Include the diagram as a fenced code block:

```mermaid
sequenceDiagram
  ...
```

Optionally add a **4th section** — **Error & Edge Cases** — as a table when there are notable
failure modes the reader should understand.

---

## Layered Architecture Vocabulary

Use consistent layer names so documents across a codebase are uniform:

| Layer | Meaning |
|---|---|
| `User` | Physical gesture or intent (click, type, navigate) |
| `UI Component` | React/Vue/HTML component that receives the event |
| `Hook / Store` | Client-side state layer (useState, Redux, Zustand, Pinia…) |
| `Router` | Client or server routing (React Router, Next.js, etc.) |
| `Service / Util` | Business logic, helper functions, formatters |
| `API Client` | Fetch/Axios/tRPC/GraphQL call layer |
| `API Route / Controller` | Server-side endpoint handler |
| `Middleware` | Auth, rate limiting, logging interceptors |
| `Service Layer` | Server-side business logic / use-case handlers |
| `Repository / ORM` | Database abstraction (Prisma, TypeORM, raw SQL) |
| `Database` | Actual persistence (Postgres, Redis, S3…) |
| `External` | Third-party APIs, webhooks, queues |
| `Cache` | Redis, CDN, in-memory LRU |

Only include layers that are actually present in the flow. Don't pad with empty layers.

---

## Step-by-Step Process

### Step 1 — Identify the flow entry point

Ask (or infer from code/description):
- What is the **triggering event**? (onClick, form submit, page load, cron, webhook…)
- What **component or file** handles it first?

If you have code, scan for: event handlers, `useEffect` hooks, route handlers, message queue
consumers, or scheduled jobs — these are almost always entry points.

### Step 2 — Trace forward through layers

Follow the data forward. At each layer, note:
- What function/method is called?
- What data is passed in?
- What data comes out?
- Is the call synchronous or async?
- Does it branch? (auth check, feature flag, error)

If you don't have code, ask the user to describe or paste the relevant pieces.

### Step 3 — Identify state mutations and side effects

Look for:
- `setState`, `dispatch`, store mutations
- `localStorage` / `sessionStorage` writes
- Cookie or URL param changes
- Cache invalidation
- Background jobs triggered
- Events emitted (websocket, analytics, logging)

These are often the "invisible" parts of a flow that cause bugs — make them explicit.

### Step 4 — Trace to the UI consequence

End the flow at what the user sees:
- Component re-render with new data
- Navigation to new route
- Toast / modal / error message
- No visible change (background sync)

### Step 5 — Write the document

Use the three-section format above. Be concise: each step should be 1–3 lines.
Avoid explaining what a generic library does (don't explain how React's useState works);
focus on **this specific flow in this specific codebase**.

### Step 6 — Draw the diagram

Choose diagram type:
- **sequenceDiagram** — use when there is clear request/response or message passing between
  parties (most API flows, most async flows)
- **flowchart TD** — use when there are significant branches or decision trees
- **stateDiagram-v2** — use only when the primary concern is state machine transitions

Keep diagrams to ≤ 20 nodes. If the flow is longer, split into sub-flows and document each
separately, then show a high-level overview diagram that links them.

---

## Input Modes

This skill works with multiple types of input:

| Input | How to handle |
|---|---|
| **Code files / paste** | Read the code directly; trace actual function calls |
| **Verbal description** | Ask clarifying questions; produce a draft based on description |
| **Partial info** | Produce what you can; annotate unknowns with `[?]` placeholders |
| **Existing docs** | Augment or reformat into the standard structure |
| **Architecture diagram / image** | Use visual cues to infer layers; ask about missing pieces |

When information is missing, **do not invent** implementation details. Use `[?]` or
`[to be confirmed]` markers, and note what the author needs to fill in.

---

## Quality Checklist

Before delivering the document, verify:

- [ ] Every layer that handles the data is represented
- [ ] Data shapes are specified at each major transformation point
- [ ] Async steps are clearly marked (e.g., `await`, `Promise`, `callback`)
- [ ] At least one error/failure path is documented
- [ ] The Mermaid diagram compiles (mentally validate syntax)
- [ ] No layer is named too generically — "Backend" is bad, "Express Route Handler" is good
- [ ] The summary is accurate and jargon-free

---

## Full Example

> User submits a login form in a React + Node.js app.

---

### Flow Summary

The user submits their email and password via the login form. The client validates the form
locally, POSTs credentials to the auth API, receives a JWT, stores it in memory and
`httpOnly` cookie, then redirects the user to the dashboard.

---

### Step-by-Step Trace

```
1. [User] → Clicks "Sign In" button
   Data: { email: string, password: string } (from controlled form inputs)

2. [UI Component: LoginForm] → Calls onSubmit handler; runs client-side validation
   Data: same — checks email format, password non-empty
   Notes: If invalid, sets local error state; flow ends here

3. [Hook: useAuth] → Calls authService.login(email, password)
   Data: { email: string, password: string }

4. [Service: authService] → POST /api/auth/login via Axios
   Data: Request body { email, password }; expects Response { token: JWT, user: UserDTO }

5. [API Route: POST /api/auth/login] → Validates body with Zod schema
   Data: { email: string, password: string }
   Notes: Returns 400 if schema fails

6. [Middleware: rateLimiter] → Checks IP-based rate limit (10 req/min)
   Notes: Returns 429 if exceeded

7. [Service Layer: AuthService.login()] → Looks up user by email; compares bcrypt hash
   Data: User record from DB
   Notes: Returns generic 401 if user not found OR password wrong (no user enumeration)

8. [Repository: UserRepository.findByEmail()] → SELECT * FROM users WHERE email = ?
   Data: User | null

9. [Service Layer] → Signs JWT with user.id and role; sets httpOnly cookie
   Data: JWT string (exp: 7 days)

10. [API Route] → Returns { token, user: { id, email, role, name } }
    Data: 200 OK with JSON body

11. [Service: authService] → Receives response; returns user + token to hook
    Data: { token: string, user: UserDTO }

12. [Hook: useAuth] → Stores user in Zustand store; stores token in memory variable
    Notes: Token NOT stored in localStorage (XSS risk); cookie handles persistence

13. [UI Component: LoginForm] → Receives success; calls router.push('/dashboard')
    Data: none

14. [Router] → Navigates to /dashboard
    Notes: Route guard re-reads Zustand store; confirms auth before rendering
```

---

### Diagram

```mermaid
sequenceDiagram
  actor User
  participant Form as LoginForm
  participant Hook as useAuth (Zustand)
  participant Svc as authService
  participant API as POST /api/auth/login
  participant DB as Database

  User->>Form: Click "Sign In"
  Form->>Form: Client-side validation
  alt Invalid
    Form-->>User: Show field errors
  else Valid
    Form->>Hook: onSubmit(email, password)
    Hook->>Svc: authService.login()
    Svc->>API: POST { email, password }
    API->>API: Zod validation + rate limit
    API->>DB: SELECT user WHERE email = ?
    DB-->>API: User record
    API->>API: bcrypt.compare(password, hash)
    alt Wrong credentials
      API-->>Svc: 401 Unauthorized
      Svc-->>Hook: throw AuthError
      Hook-->>Form: error state
      Form-->>User: "Invalid email or password"
    else Success
      API->>API: Sign JWT; set httpOnly cookie
      API-->>Svc: 200 { token, user }
      Svc-->>Hook: { token, user }
      Hook->>Hook: Update Zustand store
      Hook-->>Form: success
      Form->>Form: router.push('/dashboard')
    end
  end
```

---

### Error & Edge Cases

| Scenario | Layer | Response | User Experience |
|---|---|---|---|
| Empty / malformed email | UI Component | Form error, no request sent | Inline field error |
| Rate limit exceeded | Middleware | 429 Too Many Requests | "Too many attempts, try again in 1 min" |
| User not found | Service Layer | 401 (generic) | "Invalid email or password" |
| Wrong password | Service Layer | 401 (generic) | "Invalid email or password" |
| DB connection failure | Repository | 500 Internal Server Error | "Something went wrong, try again" |
| Network timeout | API Client | Axios timeout error | "Could not reach server" |

---

## Tips for Complex Systems

- **Break microservices into separate flows.** Document each service's internal flow
  independently, then create a high-level "orchestration" flow that shows how they connect.

- **Show data shapes at boundaries.** The most important moments to show a data shape are
  right before it crosses a network boundary (serialization) and right after (deserialization).

- **Name async boundaries explicitly.** Mark every `await`, queue enqueue/dequeue, and
  webhook callback. These are where timing bugs, race conditions, and lost messages hide.

- **Document the "happy path" first**, then add error branches. Trying to document everything
  at once produces unreadable flows.

- **For event-driven systems**, use `flowchart` with event names on edges rather than
  `sequenceDiagram` — it better represents fan-out.

- **Keep diagrams printable.** If a diagram has more than ~20 nodes, split it. A diagram
  that requires scrolling is a diagram no one reads.

---

## Reference Files

- `references/mermaid-cheatsheet.md` — Quick reference for Mermaid syntax (sequence, flowchart, state)
- `references/layer-patterns.md` — Layer naming patterns for common stacks (Next.js, Rails, Django, Go, etc.)