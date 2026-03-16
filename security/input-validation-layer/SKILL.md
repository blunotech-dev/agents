---
name: input-validation-layer
description: Add a centralized input validation layer at all system entry points. Use this skill when the user mentions input validation, schema validation, request validation, Zod, Joi, Yup, sanitizing API input, validating webhooks, form submissions, or asks "how do I validate incoming data?" or "where should validation live?".
category: "Security"
---

# Input Validation Layer

Validation belongs at every entry point — API routes, webhooks, queue consumers, cron job inputs — before any business logic runs. One missed entry point is all an attacker needs.

---

## Principles

- **Validate at the boundary** — before the data touches your app logic or DB
- **Allowlist, not blocklist** — define the exact shape you accept; reject everything else
- **Never trust** — headers, query params, URL params, request body, cookies, webhook payloads
- **Fail closed** — unknown/extra fields stripped or rejected; never passed through silently
- **Single schema = source of truth** — same schema used for validation, TypeScript types, and docs

---

## Schema Design (Zod, used throughout)

```ts
import { z } from 'zod';

// Define once — derive types from it
export const CreateInvoiceSchema = z.object({
  amount:      z.number().int().positive().max(1_000_000_00),  // cents, no float drift
  currency:    z.enum(['USD', 'EUR', 'GBP']),
  description: z.string().min(1).max(500).trim(),
  due_date:    z.string().datetime(),                           // ISO 8601 only
  recipient:   z.object({
    email: z.string().email().toLowerCase(),
    name:  z.string().min(1).max(100).trim(),
  }),
});

// TypeScript type derived — no duplication
export type CreateInvoice = z.infer<typeof CreateInvoiceSchema>;
```

**Strip unknown fields** — never let extra fields through to the ORM:
```ts
// .strip() is Zod's default — unknown keys silently dropped
// .strict() rejects unknown keys entirely (good for internal APIs)
const StrictSchema = CreateInvoiceSchema.strict();
```

---

## Centralizing Validation: Middleware Pattern

Write once, apply to any route.

```ts
// middleware/validate.ts
import { z, ZodSchema } from 'zod';
import { Request, Response, NextFunction } from 'express';

type Target = 'body' | 'query' | 'params';

export function validate(schema: ZodSchema, target: Target = 'body') {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req[target]);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        issues: result.error.issues.map(i => ({
          field: i.path.join('.'),
          message: i.message,
        })),
      });
    }
    req[target] = result.data; // replace with parsed/coerced/stripped data
    next();
  };
}

// Usage
app.post('/invoices',
  requireAuth,
  validate(CreateInvoiceSchema, 'body'),
  validate(z.object({ org_id: z.string().uuid() }), 'params'),
  createInvoiceHandler,
);
```

---

## Entry Points to Cover

### API routes (above pattern)

### Query params — often forgotten
```ts
const PaginationSchema = z.object({
  page:  z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort:  z.enum(['created_at', 'amount', 'due_date']).default('created_at'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

// z.coerce converts "?page=2" (string) → number automatically
app.get('/invoices', requireAuth, validate(PaginationSchema, 'query'), listInvoices);
```

### Webhooks — validate payload + verify signature
```ts
const StripeWebhookSchema = z.object({
  type: z.string(),
  data: z.object({ object: z.record(z.unknown()) }),
});

app.post('/webhooks/stripe',
  validateStripeSignature,     // verify HMAC before parsing — rejects spoofed payloads
  validate(StripeWebhookSchema),
  handleStripeWebhook,
);

function validateStripeSignature(req, res, next) {
  try {
    // stripe.webhooks.constructEvent needs raw body — use express.raw() on this route
    stripe.webhooks.constructEvent(req.rawBody, req.headers['stripe-signature'], WEBHOOK_SECRET);
    next();
  } catch { res.status(400).send('Invalid signature'); }
}
```

### Queue / worker consumers
```ts
// Don't trust queue messages any more than HTTP requests
async function processJob(rawPayload: unknown) {
  const payload = SendEmailSchema.parse(rawPayload); // throws on invalid
  await sendEmail(payload);
}
```

### Environment / config on startup
```ts
const EnvSchema = z.object({
  DATABASE_URL:    z.string().url(),
  JWT_PRIVATE_KEY: z.string().min(1),
  STRIPE_SECRET:   z.string().startsWith('sk_'),
  PORT:            z.coerce.number().default(3000),
});

const env = EnvSchema.parse(process.env); // fail fast at startup, not at runtime
export default env;
```

---

## Non-Obvious Things to Validate

| Input | What to enforce |
|---|---|
| Numeric IDs from URL | `.uuid()` or regex — never trust `parseInt` alone |
| Enum fields | `.enum([...])` — not just `z.string()` |
| Dates | `.datetime()` or `.coerce.date()` — not raw strings used in queries |
| Currency/money | Integer cents — reject floats to avoid drift |
| Redirect URLs | Validate scheme + origin (see xss-prevention / idor-fix skills) |
| File uploads | MIME type, extension, size limit — all three; MIME alone is spoofable |
| Free-text fields | `.max()` always — unbounded strings can fill DB / trigger DoS |
| Arrays | `.array().min(1).max(50)` — unbounded arrays = DoS vector |

---

## Validation vs Sanitization

| | Validation | Sanitization |
|---|---|---|
| **Purpose** | Reject invalid input | Transform input to safe form |
| **When** | Always, at entry point | Only when format must be preserved |
| **Examples** | Reject non-email strings | `.trim()`, `.toLowerCase()`, strip control chars |

Prefer **rejection over transformation** for security-sensitive fields. Transforming input silently (e.g., truncating to `max`) hides bugs and can introduce unexpected behavior.

`.trim()` in schema is fine for UX. Never silently truncate IDs, tokens, or enum values.

---

## Error Response Shape

Consistent validation error shape across all endpoints:
```json
{
  "error": "Validation failed",
  "issues": [
    { "field": "recipient.email", "message": "Invalid email" },
    { "field": "amount", "message": "Number must be positive" }
  ]
}
```

Don't leak internal details — schema paths are fine; stack traces are not.

---

## Audit Checklist

- [ ] Every route has a schema for body, query params, and URL params
- [ ] Unknown/extra fields stripped or rejected — not passed to ORM
- [ ] Query param numbers use `z.coerce` — no raw string-to-number conversion in handlers
- [ ] Webhook routes verify signature before parsing payload
- [ ] Queue/worker consumers validate payloads with same rigor as HTTP routes
- [ ] Env vars validated at startup with a schema
- [ ] All string fields have `.max()` — no unbounded strings
- [ ] All arrays have `.max()` — no unbounded arrays
- [ ] Enum fields use `.enum()` — not just `z.string()`
- [ ] Money/currency stored as integers — no floats
- [ ] Validation errors return consistent shape without stack traces