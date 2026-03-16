---
name: graphql-security
description: Audit a GraphQL API for security issues. Use this skill when the user mentions GraphQL, query depth, introspection, batching abuse, field-level authorization, resolver security, N+1 in security context, or asks "is my GraphQL API secure?" or "how do I lock down GraphQL?".
category: "Security"
---

# GraphQL Security

GraphQL's flexibility — arbitrary queries, nested traversal, batched operations — creates attack surfaces that don't exist in REST. Each must be explicitly mitigated.

---

## 1. Disable Introspection in Production

Introspection exposes the full schema: every type, field, mutation, and their relationships. It's a free recon tool for attackers.

```ts
// Apollo Server
new ApolloServer({
  introspection: process.env.NODE_ENV !== 'production',
});

// GraphQL Yoga / envelop
import { useDisableIntrospection } from '@envelop/disable-introspection';
plugins: [useDisableIntrospection()];

// graphql-js (manual)
const NO_INTROSPECTION_RULE = (context) => ({
  Field({ name }) {
    if (name.value === '__schema' || name.value === '__type') {
      context.reportError(new GraphQLError('Introspection disabled'));
    }
  }
});
validationRules: [NO_INTROSPECTION_RULE]
```

**Exceptions:** keep introspection on in staging behind auth, or use `@apollo/server` persisted queries to lock the schema surface entirely.

---

## 2. Query Depth Limiting

Deeply nested queries can exhaust memory before hitting a timeout.

```graphql
# Attack: exponential traversal
{ user { friends { friends { friends { friends { posts { comments { author { friends { ... } } } } } } } } } }
```

```ts
import depthLimit from 'graphql-depth-limit';

new ApolloServer({
  validationRules: [depthLimit(7)], // 5–10 is reasonable for most schemas
});
```

**Pick depth based on your schema's legitimate max.** Map your deepest real query first — then add 2 levels of headroom.

---

## 3. Query Complexity Limiting

Depth alone doesn't catch wide queries. Complexity assigns cost per field and rejects queries over a budget.

```ts
import { createComplexityRule } from 'graphql-query-complexity';

validationRules: [
  createComplexityRule({
    maximumComplexity: 1000,
    variables: {},
    onComplete(complexity) {
      console.log('Query complexity:', complexity);
    },
    estimators: [
      // Lists cost more than scalars
      fieldExtensionsEstimator(),
      simpleEstimator({ defaultComplexity: 1 }),
    ],
  }),
]
```

**Annotate expensive fields in the schema:**
```graphql
type Query {
  users(limit: Int): [User] @complexity(value: 10, multipliers: ["limit"])
  user(id: ID!): User      @complexity(value: 1)
}
```

---

## 4. Batching Abuse

GraphQL allows multiple operations in one request. Without limits, one request can become thousands of DB queries.

```json
// Single HTTP request, 500 queries
[
  { "query": "{ user(id: \"1\") { email } }" },
  { "query": "{ user(id: \"2\") { email } }" },
  ...499 more
]
```

```ts
// Apollo Server — disable or limit batching
new ApolloServer({
  allowBatchedHttpRequests: false, // disable entirely if not needed
});

// Or limit batch size with a custom plugin
const batchLimitPlugin = {
  requestDidStart() {
    return {
      didResolveOperation({ requestContext }) {
        if (Array.isArray(requestContext.request) &&
            requestContext.request.length > 10) {
          throw new GraphQLError('Batch size exceeds limit');
        }
      }
    };
  }
};
```

---

## 5. Field-Level Authorization

GraphQL's biggest REST-unlike footgun: route-level auth isn't enough. Each resolver returns data independently — a user might be authorized to query `user` but not `user.salary`.

```ts
// ❌ Auth only on the root resolver — nested fields unprotected
const resolvers = {
  Query: {
    user: requireAuth((_, { id }, ctx) => db.users.findById(id)),
  },
  User: {
    salary: (user) => user.salary,     // anyone who can query User gets salary
    ssn:    (user) => user.ssn,        // same
  },
};

// ✅ Auth at each sensitive field resolver
const resolvers = {
  User: {
    salary: (user, _, ctx) => {
      if (ctx.user.role !== 'admin' && ctx.user.id !== user.id) return null;
      return user.salary;
    },
    ssn: (user, _, ctx) => {
      if (ctx.user.role !== 'admin') return null;
      return user.ssn;
    },
  },
};
```

### Shield for declarative field-level auth

```ts
import { shield, rule, and } from 'graphql-shield';

const isAdmin = rule()((_, __, ctx) => ctx.user?.role === 'admin');
const isOwner = rule()((parent, _, ctx) => parent.id === ctx.user?.id);

export const permissions = shield({
  Query: { user: isAuthenticated },
  User: {
    salary: and(isAuthenticated, isOwner),  // own record only
    ssn:    isAdmin,
  },
  Mutation: {
    deleteUser: isAdmin,
  },
});
```

---

## 6. N+1 and Data Exposure via DataLoader

Without DataLoader, nested queries cause N+1 DB hits — also a DoS vector.

```ts
// ❌ N+1: 1 query for posts + N queries for each author
Post: {
  author: (post) => db.users.findById(post.author_id)
}

// ✅ DataLoader batches into 1 query
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids: string[]) =>
  db.users.findMany({ where: { id: { in: ids } } })
    .then(users => ids.map(id => users.find(u => u.id === id)))
);

Post: {
  author: (post, _, ctx) => ctx.loaders.user.load(post.author_id)
}
```

DataLoader is a correctness fix but also a security fix — it prevents N+1 from being weaponized.

---

## 7. Non-Obvious Vectors

| Vector | Issue | Fix |
|---|---|---|
| `__typename` in depth count | Often excluded from depth limit plugins — still counts | Confirm your depth plugin counts it |
| Aliases bypassing rate limits | `{ a: user(id:1) { email } b: user(id:2) { email } }` — one request, many operations | Alias count limit or complexity covers this |
| Mutations in subscriptions | Schema allows it; should be blocked | Validate operation type at subscription handler |
| Error messages leaking schema | Default errors expose resolver internals | Mask errors in production: `formatError: maskErrors` |
| File upload endpoints | `multipart/form-data` bypasses query validation | Apply same depth/complexity rules to upload mutations |
| Persisted queries not enforced | Full arbitrary query surface open | Use persisted queries in production to whitelist operations |

### Mask errors in production

```ts
import { maskErrors } from 'graphql-errors'; // or Apollo's built-in

new ApolloServer({
  formatError(err) {
    if (process.env.NODE_ENV === 'production') {
      // Return safe message; log full error internally
      console.error(err.originalError);
      return new GraphQLError('Internal server error');
    }
    return err;
  },
});
```

---

## Audit Checklist

- [ ] Introspection disabled in production
- [ ] Query depth limit set (based on schema's legitimate max + 2)
- [ ] Query complexity limit set; expensive list fields annotated with multipliers
- [ ] Batching disabled or limited to a small N
- [ ] Field-level auth on sensitive fields — not just root Query/Mutation resolvers
- [ ] DataLoader used for all relation resolvers (no N+1)
- [ ] Aliases counted in complexity — not just top-level fields
- [ ] Errors masked in production; full errors logged internally only
- [ ] Rate limiting applied at HTTP level (GraphQL is one POST endpoint)
- [ ] Persisted queries enforced in production (if security-critical)