---
name: test-fixtures-factory
description: Create typed fixture and factory systems for test data with sensible defaults and per-test overrides. Use when tests repeat large objects or need reusable data builders.
category: "Testing"
---

# Test Fixtures Factory

Generates a typed builder-pattern factory system for test entities — defaults that read like real data, overrides that don't require reconstructing the whole object.

---

## Phase 1: Discovery

Extract before writing:

1. **Entity shapes** — what types/interfaces need factories? Get their definitions.
2. **Language + test runner** — TypeScript with Jest/Vitest, or plain JS?
3. **ORM/persistence** — does the factory need a `.create()` method that persists to DB, or just `.build()` for in-memory objects?
4. **Related entities** — do any entities reference others (e.g., `Post` has an `author: User`)? Need to know nesting depth.
5. **Existing tooling** — is `fishery`, `@anatine/zod-mock`, or similar already installed? Or building from scratch?
6. **Unique field constraints** — are any fields unique in the DB (email, slug)? These need sequence counters.

If the user pastes entity types, extract shape and relationships directly.

---

## Phase 2: Architecture Decisions

### Builder pattern vs plain function

| Approach | Non-obvious tradeoff |
|---|---|
| Plain function `makeUser(overrides?)` | Simplest; breaks down when you need method chaining or trait composition |
| Builder class with `.with*()` methods | Reads well; generates type noise; hard to compose traits across entities |
| `fishery` library | Best for complex graphs; `.associations` handles related entities cleanly; requires install |
| Functional builder (this skill's default) | Single `build(overrides?)` function with `traits` object; no class overhead; TypeScript-friendly |

**Default:** functional builder pattern — no external dependency, full TypeScript inference, trait support via named presets.

### Unique field strategy

Fields like `email` and `slug` that must be unique per-test row need a sequence counter. Do NOT use `Math.random()` — it produces unreadable test output and non-reproducible failures.

```ts
let _seq = 0
const seq = () => ++_seq

export const userFactory = {
  build: (overrides: Partial<User> = {}): User => ({
    id: `user-${seq()}`,
    email: `user-${seq()}@example.com`,
    name: 'Alice Example',
    role: 'member',
    createdAt: new Date('2024-01-01'),
    ...overrides,
  }),
}
```

**Non-obvious:** the sequence counter is module-scoped. If test files import the same factory, they share the counter across runs — which is fine for uniqueness but means `id` values are non-deterministic across test files. If tests assert on specific IDs, use `overrides` to pin them explicitly.

### Realistic defaults matter

Defaults should look like real data, not `'string'` or `'test'`. Why:
- Assertions that accidentally pass against placeholder values hide bugs
- `email: 'test'` (not a valid email) can cause validation failures in integration tests

Use plausible values: `'Alice Example'`, `'alice@example.com'`, realistic dates, valid enum values. Don't use faker for defaults — it produces non-reproducible failures. Use faker only for fields where the specific value is irrelevant AND the test doesn't assert on it.

---

## Phase 3: Factory Implementation

### Single entity factory

```ts
// factories/user.factory.ts

import type { User } from '../types'

let _seq = 0

export const userFactory = {
  build(overrides: Partial<User> = {}): User {
    const n = ++_seq
    return {
      id: `user-${n}`,
      email: `user-${n}@example.com`,
      name: 'Alice Example',
      role: 'member',
      emailVerified: true,
      createdAt: new Date('2024-01-01T00:00:00Z'),
      updatedAt: new Date('2024-01-01T00:00:00Z'),
      ...overrides,
    }
  },

  // Traits — named presets for common variants
  traits: {
    admin: (overrides: Partial<User> = {}): User =>
      userFactory.build({ role: 'admin', ...overrides }),

    unverified: (overrides: Partial<User> = {}): User =>
      userFactory.build({ emailVerified: false, ...overrides }),
  },
}
```

Usage:
```ts
const user = userFactory.build()                          // default
const admin = userFactory.traits.admin()                  // trait
const specificUser = userFactory.build({ name: 'Bob' })   // override
const adminNamed = userFactory.traits.admin({ name: 'Bob' }) // trait + override
```

### Related entity handling

**Non-obvious:** do not auto-build nested entities in defaults. It creates implicit coupling between factories and makes debugging factory output confusing.

```ts
// Wrong — auto-building post.author hides which user is being used
export const postFactory = {
  build(overrides: Partial<Post> = {}): Post {
    return {
      id: `post-1`,
      authorId: userFactory.build().id, // creates a phantom user
      ...overrides,
    }
  }
}

// Right — use a stable default ID; let tests wire relationships explicitly
export const postFactory = {
  build(overrides: Partial<Post> = {}): Post {
    const n = ++_seq
    return {
      id: `post-${n}`,
      authorId: 'user-1', // stable reference; test seeds the user separately
      title: 'Example Post',
      body: 'Post body content.',
      published: true,
      createdAt: new Date('2024-01-01T00:00:00Z'),
      ...overrides,
    }
  }
}
```

When a test needs a real relationship, wire it explicitly:

```ts
const author = userFactory.build()
const post = postFactory.build({ authorId: author.id })
```

### Persistence factory (`.create()`)

When tests need rows in the DB:

```ts
export const userFactory = {
  build(overrides: Partial<User> = {}): User { /* ... */ },

  async create(db: Db, overrides: Partial<User> = {}): Promise<User> {
    const data = userFactory.build(overrides)
    await db.insert(users).values(data)
    return data
  },
}
```

**Non-obvious:** `.create()` should call `.build()` internally — not duplicate the defaults. If a test needs the built object back (to use its generated ID), `.create()` returns it.

Pass `db` as a parameter rather than importing it at module level — this keeps the factory usable in both unit tests (`.build()` only, no DB) and integration tests (`.create()` with a real connection).

### List factory

```ts
buildList(count: number, overrides: Partial<User> = {}): User[] {
  return Array.from({ length: count }, () => userFactory.build(overrides))
}
```

**Non-obvious:** `overrides` apply to every item. If the test needs distinct values per item, use index-based overrides:

```ts
buildListWith(items: Partial<User>[]): User[] {
  return items.map(overrides => userFactory.build(overrides))
}
```

---

## Phase 4: Factory Organization

```
factories/
  index.ts          // re-exports all factories
  user.factory.ts
  post.factory.ts
  comment.factory.ts
```

**Reset sequences between test files** (if IDs are asserted anywhere):

```ts
// jest.setup.ts
import * as factories from './factories'
beforeEach(() => factories.resetSequences())
```

Expose a `resetSequences()` from each factory:
```ts
export const resetSequence = () => { _seq = 0 }
```

**Non-obvious:** Jest runs each test file in a separate worker, so module-level sequence counters reset automatically between files. You only need `resetSequences()` in `beforeEach` if tests within the same file assert on specific IDs.

---

## Phase 5: Output

Produce:

1. **Factory file(s)** — one per entity, typed, with sequence counter, traits, and `buildList`
2. **`.create()` method** if the user has DB integration tests
3. **`index.ts`** re-export if generating multiple factories
4. **Usage examples** — one test snippet per pattern (default, trait, override, list, relationship)

### Output notes

- Sequence counter per factory file, not global — avoids cross-factory coupling
- `createdAt`/`updatedAt` as fixed dates, not `new Date()` — prevents snapshot churn
- Traits call `build()` internally; they are not standalone objects
- Do not use class syntax unless the user's codebase already uses it — functional factories are simpler to extend
- If `fishery` is already installed, generate using its `Factory.define()` API instead; the `.associations` field handles related entities more cleanly than manual wiring