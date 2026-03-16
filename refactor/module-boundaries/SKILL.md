---
name: module-boundaries
description: Identify overly large or tightly coupled modules and split them into well-bounded units with clear public interfaces. Use when the user asks to reorganize modules, fix circular dependencies, separate concerns, or define module boundaries. Also trigger when shared code structure shows god modules, cross-cutting imports, or tight coupling.
category: "Refactoring"
---

# Module Boundaries

Diagnose and fix modules that have grown beyond a single clear purpose, or that have
become so entangled with other modules that changes ripple unpredictably. The goal:
each module owns one domain of logic, exposes only what callers need, and can be read,
tested, and changed without understanding the rest of the system.

---

## When to Apply This Skill

Trigger on any of:
- A single file/module exceeds ~300 lines and handles multiple distinct concerns
- Circular imports between two or more modules
- A module importing from many unrelated parts of the codebase
- Changes to one module routinely break unrelated modules
- "Utils", "helpers", "common", or "shared" modules that have become dumping grounds
- No clear distinction between what a module exposes publicly vs. uses internally
- Multiple modules duplicating the same logic because there's no clear owner

---

## Step 1 — Map the Current State

Before proposing any structure, understand what exists. Gather:

**1. The import graph.** Who imports whom? Look for:
- Modules imported by many others (high fan-in) — potential shared kernel or over-coupled util
- Modules that import many others (high fan-out) — potential god module or orchestration layer
- Cycles — A imports B imports A

Run this to get a quick picture (Node.js example):
```bash
# Show what each file imports
grep -r "^import\|^const.*require" src/ --include="*.js" --include="*.ts" -l | \
  xargs -I{} sh -c 'echo "=== {} ==="; grep -E "^import|require\(" {}'
```

**2. The responsibility inventory.** For each module, write one sentence: *"This module is responsible for ___."* If you need "and" more than once, it's doing too much.

**3. Coupling hotspots.** Note any module where:
- The import list is longer than ~8 entries
- It imports from both `domain/` and `infrastructure/` layers (mixed abstraction)
- It's imported in both UI and backend code (mixed context)

---

## Step 2 — Diagnose the Coupling Type

Different coupling problems need different fixes:

| Problem | Symptom | Fix |
|---|---|---|
| **God module** | One module does everything: types, logic, DB access, formatting | Split by domain concern |
| **Dumping ground** | `utils.ts` or `helpers/` with 30 unrelated functions | Redistribute to owning modules; create domain-specific util files |
| **Circular dependency** | A → B → A (often caused by shared types or event handlers) | Extract shared types to a neutral module; invert one dependency |
| **Leaky internals** | Callers import and use private implementation details | Define an explicit public interface (`index.ts` barrel or `__init__.py`) |
| **Cross-layer imports** | UI importing DB models directly; domain importing HTTP request objects | Introduce a boundary type / DTO at each layer edge |
| **Premature unification** | Two features merged into one module because they seemed similar | Split by feature, not by type (prefer vertical slices) |

---

## Step 3 — Design the New Boundaries

### The core principle: cohesion over convenience

A module boundary should reflect a *domain boundary*, not a *file-size limit*. Ask:
- Would a new team member know where to find this logic without a search?
- Can this module be tested without instantiating anything from another module?
- If this module changes, is the set of things that need to change clearly scoped?

### Boundary archetypes

**Feature module (vertical slice)**
```
orders/
├── index.ts          ← public interface (only export from here)
├── order.model.ts    ← domain types
├── order.service.ts  ← business logic
├── order.repo.ts     ← data access
└── order.schema.ts   ← validation
```
Everything about `orders` lives together. Other modules import only from `orders/index.ts`.

**Shared kernel (horizontal slice — use sparingly)**
```
shared/
├── types/            ← domain primitives used across features (UserId, Money, etc.)
├── errors/           ← base error classes
└── events/           ← event bus types (not implementations)
```
The shared kernel should be *minimal and stable*. If it changes often, it's not a shared kernel — it's a coupling hotspot.

**Infrastructure module**
```
infra/
├── db/               ← DB client, connection, base repo
├── email/            ← email transport abstraction
└── cache/            ← cache client wrapper
```
Domain modules depend on interfaces, not on `infra/` directly. Wiring happens at the app root.

### Defining the public interface

Every module should have exactly one entry point for external consumers:

**TypeScript/JavaScript — barrel file:**
```ts
// orders/index.ts — only export what callers should use
export { OrderService } from './order.service';
export type { Order, OrderStatus } from './order.model';
// OrderRepository is NOT exported — it's an internal implementation detail
```

**Python — `__init__.py`:**
```python
# orders/__init__.py
from .service import OrderService
from .models import Order, OrderStatus
# Do NOT expose Repository, internal helpers, etc.
```

**Go — package-level exports:**
```go
// Unexported identifiers (lowercase) are the boundary mechanism.
// Only export what external packages need.
type OrderService struct { ... }   // exported
type orderRepo struct { ... }      // unexported — stays internal
```

**Rule:** If a caller has to import from a subpath (`orders/internal/repo`), your
public interface is leaking. Fix the barrel, not the caller.

---

## Step 4 — Break Circular Dependencies

Circular imports are always a design smell. Three techniques:

**A. Extract the shared type to a neutral module**
```
Before: auth → user (for User type), user → auth (for AuthToken type)
After:  auth → shared/types, user → shared/types (no cycle)
```

**B. Invert the dependency (dependency inversion)**
```
Before: orders → notifications (orders directly calls notification service)
After:  orders emits an event; notifications subscribes to it
        orders has zero knowledge of notifications
```

**C. Merge if the cycle reveals true cohesion**
```
Before: orderItem ↔ order (they import each other constantly)
After:  merge into a single orders module — the cycle was telling you they belong together
```

Do not resolve cycles by adding a third "glue" module that imports both — this just
moves the coupling.

---

## Step 5 — Write the Refactored Structure

Output:

1. **The proposed directory/file structure** — show the before and after tree.
2. **The public interface for each new module** — show the `index.ts` / `__init__.py` exports.
3. **The migration path** — in what order to make the changes to avoid breaking the build mid-refactor:
   - Create new module shells first
   - Move types before logic (types have fewer dependencies)
   - Update imports from outermost callers inward
   - Delete old paths last

4. **Any behavior questions** — flag cases where it's unclear which module should own a piece of logic.

---

## Step 6 — Verify and Annotate

After writing the structure, include:

**Boundary health summary:**

| Module | Before | After | Change |
|---|---|---|---|
| `utils.ts` (340 lines) | 22 unrelated exports | Deleted — redistributed to `orders/`, `users/`, `shared/types/` | Eliminated dumping ground |
| `orders.ts` (480 lines) | Mixed DB + logic + formatting | Split into `order.service`, `order.repo`, `order.formatter` | Single concern per file |
| Circular: `auth ↔ user` | Cycle via shared types | `shared/types/auth.ts` extracted | Cycle broken |

**Then call out:**
- **Ownership decisions** that were ambiguous — explain the reasoning
- **What the public interface deliberately excludes** — and why
- **Any logic that has no clear owner** — surface it rather than guess

---

## Language-Specific Notes

### TypeScript / JavaScript
- Enforce barrel imports with ESLint: `no-restricted-imports` to ban deep-path imports from other feature modules.
- `paths` in `tsconfig.json` for clean import aliases: `@orders/*` instead of `../../../orders/`.
- `"exports"` field in `package.json` for true encapsulation in monorepos (blocks subpath imports entirely).

### Python
- `__all__` in `__init__.py` to explicitly declare the public surface.
- Use relative imports (`from .service import OrderService`) within a package; absolute imports across packages.
- `src/` layout (`src/myapp/`) prevents accidental imports of the package directory itself.

### Java
- Package-private (no modifier) for internal classes — only `public` for the intended interface.
- Module system (`module-info.java`, Java 9+) for hard boundaries: `exports com.app.orders` exposes only what's declared.

### Go
- The package is the boundary — internal packages (`internal/`) are enforced by the compiler.
- Keep package names single-word, lowercase, and domain-describing (`orders`, not `orderManagement`).

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it's bad | Fix |
|---|---|---|
| Splitting by type (`models/`, `services/`, `repos/` at top level) | Forces you to touch 4 folders for any single feature change | Split by feature (vertical slices) |
| A `shared/` module that keeps growing | Becomes the new dumping ground | Shared kernel must be minimal and frozen; push new things to feature modules |
| Splitting a module before understanding its seams | Creates artificial boundaries that don't reflect the domain | Map responsibilities first; let the seams emerge |
| Public interface that exports everything | Defeats the purpose of the boundary | Default to not exporting; add exports only when a caller needs them |
| Resolving a cycle by making both modules import a third "bridge" | Adds indirection without removing coupling | Extract the shared *type*, not the shared *logic* |

---

## Quick Checklist Before Delivering

- [ ] Each module has a one-sentence responsibility statement with no "and"
- [ ] Every module has a single entry-point barrel/init that is the only import path for external consumers
- [ ] No circular dependencies remain
- [ ] `utils` / `helpers` / `common` dumping grounds have been redistributed or scoped
- [ ] Cross-layer imports (UI→DB, domain→HTTP) eliminated via boundary types
- [ ] Migration path is sequenced to avoid mid-refactor breakage
- [ ] Ambiguous ownership decisions surfaced and explained
- [ ] Before/after structure tree included