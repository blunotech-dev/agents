---
name: god-object-split
description: Refactor classes or modules with too many responsibilities into smaller, cohesive units that follow the Single Responsibility Principle (SRP). Use when the user asks to split a god object, fat class, or bloated module, or when a file clearly handles multiple concerns.
category: "Refactoring"
---

# God Object Split

A structured refactoring skill for decomposing oversized classes and modules into
focused, cohesive units — each with a single, well-defined responsibility.

---

## When to Use This Skill

- A class has 10+ methods spanning multiple unrelated concerns
- A module imports from 5+ different domains (DB, HTTP, email, auth, formatting…)
- Adding a feature requires touching the same file every time
- Unit testing the class requires mocking half the system
- The class name ends in `Manager`, `Service`, `Handler`, `Utils`, or `Helper` and does everything

---

## Phase 1 — Audit the God Object

Before splitting anything, map what the class actually does.

### 1.1 Extract a responsibility inventory

Read through every method and field. Group them by what they operate on or care about:

```
UserService (500 lines)
├── Auth          → login(), logout(), hashPassword(), validateToken()
├── Persistence   → save(), findById(), findByEmail(), delete()
├── Notifications → sendWelcomeEmail(), sendPasswordReset()
├── Formatting    → toJSON(), toDisplayName(), formatAddress()
└── Business      → applyDiscount(), checkSubscriptionStatus()
```

### 1.2 Identify coupling and shared state

Note which groups share instance variables or call each other — that determines split boundaries and what needs to be injected afterward.

### 1.3 Score cohesion

For each group, ask: *If I extracted this group into its own class, would it make sense in isolation?* If yes, it's a candidate for extraction.

---

## Phase 2 — Design the Target Structure

### 2.1 Name new units precisely

Avoid vague names. Each new class/module name should describe exactly what it owns:

| Vague (avoid)       | Precise (prefer)            |
|---------------------|-----------------------------|
| `UserHelper`        | `UserFormatter`             |
| `UserUtils`         | `UserRepository`            |
| `UserManager`       | `UserAuthenticator`         |
| `NotifService`      | `UserNotificationSender`    |

### 2.2 Define interfaces / contracts first

Before writing implementation, sketch the public API of each new unit. This surfaces dependency conflicts early.

```ts
// Before touching any code:
interface UserRepository { findById(id: string): Promise<User> }
interface UserAuthenticator { login(email: string, pw: string): Promise<Session> }
interface UserNotifier { sendWelcomeEmail(user: User): Promise<void> }
```

### 2.3 Map inter-unit dependencies

Draw a simple dependency graph. Cycles = a boundary is drawn in the wrong place.

```
UserController
    └── UserAuthenticator
            └── UserRepository
    └── UserNotifier
            └── EmailClient
```

---

## Phase 3 — Execute the Split

### 3.1 Extract one responsibility at a time

Never split everything in one commit. Use this order:

1. **Pure/stateless logic first** — formatters, validators, pure calculations. Zero risk.
2. **I/O boundaries next** — repositories, API clients. Easy to test with mocks.
3. **Orchestration last** — the original class often becomes a thin coordinator that delegates.

### 3.2 Move methods without changing behavior

For each extracted group:

1. Create the new file/class
2. Copy (don't cut) the methods across
3. Make the original class delegate to the new one
4. Run tests to confirm no regression
5. Remove the original methods only after tests pass

### 3.3 Handle shared state

If two groups read the same field, it belongs in one of them (inject it into the other), or in a shared value object passed between them. Avoid keeping shared mutable state in the original class as a "glue" layer.

---

## Phase 4 — Output Format

Deliver the refactoring as:

### Responsibility map (Markdown table)

| New Class/Module        | Responsibility                        | Methods Moved                          |
|-------------------------|---------------------------------------|----------------------------------------|
| `UserRepository`        | Persistence: read/write User to DB    | `save`, `findById`, `findByEmail`      |
| `UserAuthenticator`     | Auth: credential validation + sessions| `login`, `logout`, `hashPassword`      |
| `UserNotificationSender`| Email delivery for user lifecycle     | `sendWelcomeEmail`, `sendPasswordReset`|
| `UserFormatter`         | Presentation: serialise/display User  | `toJSON`, `toDisplayName`              |

### Refactored code

Produce each new file in full. Then show the slimmed-down original class delegating to them.

### Migration notes

- List any breaking changes to public APIs
- Note constructor signature changes (new injection points)
- Flag methods that were ambiguous about which unit they belong to

---

## Heuristics & Edge Cases

**"The method calls two groups"** → It's orchestration. Keep it in the coordinator or create an explicit use-case/service class for that workflow.

**Circular dependency after split** → The boundary is wrong. Merge the two units or introduce an interface they both depend on.

**Tiny extracted classes (1–2 methods)** → Fine if the responsibility is truly distinct. Don't merge just because it's small.

**Inheritance hierarchies** → Split responsibilities into collaborators (composition), not subclasses. Subclasses should represent subtypes, not feature collections.

**Framework-specific god objects** (Rails concerns, Django models with 50 methods) → Follow framework conventions for where things live, but still extract service objects and query objects for non-trivial logic.

**Tests for the original class** → After splitting, rewrite tests to target each new unit directly. Integration tests for the coordinator are fine but should not be the primary test surface.

---

## Checklist Before Calling It Done

- [ ] Every new class has exactly one reason to change
- [ ] No new class name ends in `Manager`, `Utils`, or `Helper`
- [ ] No circular dependencies between new units
- [ ] The original class (if kept) is a thin coordinator, not a residual god object
- [ ] Each new unit is independently testable without loading the others
- [ ] Public API changes are documented