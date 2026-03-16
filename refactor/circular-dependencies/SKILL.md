---
name: circular-dependencies
description: Detect and resolve circular import chains that cause undefined values, runtime errors, or build/test warnings. Use when the user reports circular dependency warnings, undefined imports, or order-dependent test failures. Also trigger when shared imports suggest a dependency cycle.
category: "Refactoring"

---

# Circular Dependencies

Find and break import cycles — the class of bugs where module A imports B which
imports A (directly or through a chain), causing one module to be partially
initialized when the other loads. The goal: a directed acyclic import graph where
every module's dependencies are fully initialized before it runs.

---

## When to Apply This Skill

Trigger on any of:
- `undefined` value at import time that is defined correctly inside the module
- `ReferenceError: Cannot access 'X' before initialization`
- Jest/Vitest warning: `Jest has detected the following N circular dependencies`
- webpack warning: `WARNING in Circular dependency detected`
- Test behavior that changes depending on which test file runs first
- `TypeError: Class extends value undefined` (classic circular class inheritance)
- An import that works in isolation but fails inside the full application

---

## Step 1 — Detect and Map the Cycle

### Automated detection (always run this first)

**Node.js / TypeScript:**
```bash
# madge is the standard tool for this
npx madge --circular --extensions ts,tsx,js,jsx src/
# With a visual graph (requires graphviz):
npx madge --circular --image graph.svg src/
```

**Python:**
```bash
pip install pydeps
pydeps src/mypackage --show-cycles
```

**Java / Kotlin:**
```bash
# jdeps (ships with JDK)
jdeps -verbose:class -filter:archive build/libs/app.jar | grep "circular"
```

**Webpack bundle:**
```bash
# circular-dependency-plugin in webpack config, or:
npx webpack --stats-all 2>&1 | grep -i circular
```

**Manual grep (quick scan when tools aren't available):**
```bash
# Find all imports in a file, then chase the chain
grep -E "^import|^from|require\(" src/orders/order.service.ts
```

### Trace the full cycle

Once a cycle is flagged, trace every hop manually. A two-module cycle is obvious;
a five-module chain (`A → B → C → D → E → A`) requires tracing. Write it out:

```
orders/order.service.ts
  → imports users/user.model.ts
      → imports auth/auth.service.ts
          → imports orders/order.service.ts   ← CYCLE CLOSES HERE
```

Identify the exact import line that closes the cycle — that's the break point.

---

## Step 2 — Classify the Cycle

Not all cycles are equal. The fix depends on *why* the cycle exists:

| Cycle type | Symptom | Root cause | Fix |
|---|---|---|---|
| **Type-only cycle** | Cycle exists only for TypeScript `type` / `interface` imports | Types placed in implementation files | Extract types to a neutral types file |
| **Shared constant cycle** | Module A needs a constant defined in B, B needs one from A | Constants co-located with logic | Extract constants to a dedicated file |
| **Mutual service dependency** | Service A calls a method on Service B and vice versa | Two services are actually one concern, or one should emit events | Merge, or introduce event/mediator |
| **Parent–child confusion** | Parent module imports child, child imports parent for context | Child reaching up for something it should receive | Dependency injection / pass as argument |
| **Initialization order cycle** | No logical cycle, but module-level code runs in wrong order | Side effects at module scope | Lazy initialization / factory functions |
| **Re-export cycle** | Barrel `index.ts` re-exports everything including files that import the barrel | Barrel files importing from within the same barrel | Remove internal barrel imports |

---

## Step 3 — Break the Cycle

### Technique A: Extract shared types to a neutral module

The most common fix. If the cycle exists because two modules share a type defined
in one of them, move that type out of both.

**Before:**
```
user.model.ts  ←→  order.model.ts
(Order imports User for ownership; User imports Order for order history)
```

**After:**
```
shared/types/domain.ts   ← UserId, OrderId, and other primitives live here
user.model.ts  → shared/types/domain.ts
order.model.ts → shared/types/domain.ts   (no cycle)
```

**Rule:** The neutral module must import from *nothing in the app* — it is a leaf node.
If it starts importing from feature modules, it becomes the new coupling hotspot.

---

### Technique B: Invert the dependency (event / callback)

When two services genuinely need to react to each other, the cycle usually reveals
that one should *emit* rather than *call*.

**Before:**
```ts
// order.service.ts
import { NotificationService } from '../notifications/notification.service';
// notification.service.ts
import { OrderService } from '../orders/order.service';  // cycle
```

**After:**
```ts
// order.service.ts — emits an event, knows nothing about notifications
this.eventBus.emit('order.placed', { orderId, userId });

// notification.service.ts — subscribes, no import of OrderService
this.eventBus.on('order.placed', ({ orderId, userId }) => {
  this.sendConfirmation(userId);
});
```

The dependency now flows: both modules → eventBus. No cycle.

---

### Technique C: Dependency injection (break parent→child→parent)

When a child module imports its parent to access context or configuration,
pass the dependency in instead of importing it.

**Before:**
```ts
// config.ts imports logger.ts for startup logging
// logger.ts imports config.ts for log level   ← cycle
```

**After:**
```ts
// logger.ts accepts logLevel as a constructor argument
class Logger {
  constructor(private logLevel: string) {}
}
// config.ts creates logger and passes its own value
const logger = new Logger(config.logLevel);
```

The child (Logger) no longer imports the parent (config). Cycle broken.

---

### Technique D: Lazy initialization / dynamic import

When the cycle is caused by module-level side effects or class definitions that
reference each other at load time, defer the import until it's actually needed.

**Before:**
```ts
// plugin-registry.ts
import { CoreEngine } from './core-engine';  // cycle: core-engine imports plugin-registry
```

**After:**
```ts
// plugin-registry.ts — import deferred to call time
async function getEngine() {
  const { CoreEngine } = await import('./core-engine');
  return new CoreEngine();
}
```

Use sparingly — dynamic imports make the dependency graph harder to analyze statically
and can introduce async complexity. Prefer structural fixes (A, B, C) first.

---

### Technique E: Fix barrel file self-cycles

Barrel `index.ts` files that re-export everything in a folder often create
hidden cycles when files in that folder import from the barrel.

**Before:**
```ts
// features/orders/index.ts
export * from './order.service';
export * from './order.repo';

// features/orders/order.repo.ts
import { OrderService } from '.';  // imports from the barrel — cycle
```

**After:**
```ts
// features/orders/order.repo.ts
import { OrderService } from './order.service';  // direct import, not via barrel
```

**Rule:** Files *inside* a module should use direct relative imports. The barrel is
only for *external* consumers.

---

### Technique F: Merge if the cycle reveals cohesion

If two modules import each other for many different reasons, they may actually
be one module. The cycle is the system telling you the boundary is wrong.

```
Before: order-item.ts ↔ order.ts  (6 mutual imports)
After:  orders/  (single module — order-item was never really separate)
```

This is the right fix when: the cycle has many crossing points, the modules always
change together, and no clean interface can be defined between them.

---

## Step 4 — Verify the Fix

After breaking the cycle:

**1. Re-run the cycle detector:**
```bash
npx madge --circular --extensions ts,tsx src/
# Should output: No circular dependency found!
```

**2. Check for hidden runtime undefined errors:**
```ts
// Add a temporary guard to catch initialization-order bugs
import { OrderService } from './order.service';
console.assert(OrderService !== undefined, 'OrderService is undefined — possible cycle');
```

**3. Run tests in random order (Jest):**
```bash
# Jest runs in a fixed order by default — randomize to catch order-dependent failures
npx jest --randomize
```

**4. Check bundle output** (if webpack/rollup):
```bash
npx webpack --display-modules | grep -i "circular\|cycle"
```

---

## Step 5 — Document and Enforce

Once cycles are eliminated, prevent them from returning:

**ESLint (TypeScript/JavaScript):**
```json
// .eslintrc
{
  "plugins": ["import"],
  "rules": {
    "import/no-cycle": ["error", { "maxDepth": 3 }]
  }
}
```

**Python (import-linter):**
```ini
# setup.cfg
[importlinter]
root_package = myapp

[importlinter:contract:no-cycles]
name = No circular imports
type = forbidden
source_modules =
    myapp.orders
    myapp.users
forbidden_modules =
    myapp.orders: myapp.users
    myapp.users: myapp.orders
```

**CI check:**
```bash
# Add to CI pipeline
npx madge --circular --extensions ts,tsx src/ && echo "No cycles" || exit 1
```

---

## Common Scenarios

### "It works in the app but fails in tests"
Jest module registry is fresh per test file. If module A and B form a cycle,
which one gets a partially-initialized version depends on which test imported first.
Fix: break the cycle; don't work around it with `jest.resetModules()`.

### "The value is `undefined` only sometimes"
Classic symptom of an initialization-order cycle. One module is loaded before its
dependency finishes evaluating. The value exists later (after full initialization)
but is `undefined` at import time. Fix: Technique C (DI) or D (lazy init).

### "Removing one import makes it work, but I need that import"
This is the cycle making itself known. The import isn't wrong — the structure is.
Trace the full cycle and apply a structural fix rather than removing the import.

### "TypeScript compiles fine but runtime breaks"
TypeScript's type imports (`import type { Foo }`) are erased at compile time and
never cause runtime cycles. If TS compiles but runtime breaks, the cycle is
in value imports (`import { Foo }`), not type imports. Check for
`import type` vs `import` distinctions and convert what you can.

---

## Language-Specific Notes

### TypeScript
- `import type { Foo }` is erased at compile time — use it for type-only imports to eliminate a whole class of false-positive cycles.
- `verbatimModuleSyntax: true` in `tsconfig.json` forces explicit `import type` where appropriate.
- Barrel files (`index.ts`) are the #1 source of hidden cycles in TS projects — audit them first.

### Python
- Cycles in Python are often caused by module-level code (class definitions, `@decorator` calls at import time). Moving imports inside functions is a valid Python idiom.
- `TYPE_CHECKING` guard: `if TYPE_CHECKING: from .orders import Order` — runs only during type checking, not at runtime.
- `__init__.py` files that import from submodules can create cycles; keep them minimal.

### Go
- Go's compiler rejects import cycles outright — no runtime surprise. The fix is always structural.
- `internal/` packages enforce one-way dependencies at the compiler level.

### Java
- Java allows circular class references (both classes can import each other). The cycle is usually logical, not a runtime error.
- Circular Spring bean dependencies (`@Autowired` cycle) cause `BeanCurrentlyInCreationException` — fix with constructor injection and restructured ownership, not `@Lazy`.

---

## Quick Checklist Before Delivering

- [ ] Cycle fully traced and written out hop-by-hop
- [ ] Cycle type classified (type-only, shared constant, mutual service, etc.)
- [ ] Correct technique chosen for the cycle type — not just the first technique
- [ ] Re-run of cycle detector shows zero cycles
- [ ] `import type` used for type-only imports (TypeScript)
- [ ] Barrel self-imports checked and fixed if present
- [ ] ESLint `import/no-cycle` or equivalent linting rule added
- [ ] CI enforcement step recommended
- [ ] Any ambiguous ownership decisions surfaced explicitly