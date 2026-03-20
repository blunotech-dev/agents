---
name: third-party-integration-test
description: Write tests for third-party integrations using mocks, fixtures, or sandboxes to verify API calls, webhooks, retries, and failure handling without hitting live services.
category: "Testing"
---

# Third-Party Integration Test Skill

## Discovery

Before writing tests, identify:

- **Service and SDK** — Stripe SDK vs raw fetch to Stripe API are tested differently
- **Test strategy available** — does the service offer a sandbox/test mode (Stripe, Twilio), or must responses be recorded/mocked?
- **Interaction type** — outbound API calls, inbound webhooks, polling, or streaming
- **What can fail** — rate limits, network errors, malformed responses, auth failures, service-specific errors (e.g. `card_declined`, `invalid_template_id`)
- **Secrets handling** — test keys must never appear in fixtures or committed code

---

## Strategy Selection

Choose based on what the service provides:

| Service has... | Use |
|---|---|
| Sandbox / test mode (Stripe, Twilio, PayPal) | Sandbox with real test credentials in CI secrets |
| Official mock server (Stripe CLI, Prism) | Local mock server in CI |
| No sandbox | `msw` (HTTP) or `nock` to intercept and replay |
| Webhook delivery | Local interceptor + signature verification test |
| SDK wrapping HTTP | Mock at the SDK client level, not at `fetch` |

Never mock at `fetch`/`axios` when an SDK is involved — SDK internals can transform requests in ways that matter (retry logic, auth header injection, payload serialization).

---

## Non-Obvious Patterns

### 1. Record real responses once, replay forever

Use `msw` handlers or `nock` recordings from actual sandbox calls. Don't hand-write fake response payloads — they drift from the real API shape over time:

```ts
// Record once (run against sandbox, save to fixture)
// fixtures/stripe-customer-create.json — real response from Stripe test mode

// Replay in tests
server.use(
  http.post('https://api.stripe.com/v1/customers', () =>
    HttpResponse.json(fixture('stripe-customer-create'))
  )
);
```

Fixture files should be the raw API response, not a trimmed version — trimming hides fields your code might silently start depending on.

---

### 2. Test the error *codes*, not just that an error occurred

Third-party services return structured errors. Assert the code, not just the message:

```ts
// Stripe card decline
server.use(
  http.post('https://api.stripe.com/v1/payment_intents', () =>
    HttpResponse.json(
      { error: { type: 'card_error', code: 'card_declined', decline_code: 'insufficient_funds' } },
      { status: 402 }
    )
  )
);

const result = await chargeCard(paymentMethod);
expect(result.error.code).toBe('card_declined');
expect(result.error.decline_code).toBe('insufficient_funds');
// Don't just expect(result.success).toBe(false)
```

---

### 3. Webhook signature verification is not optional to test

If the service signs webhooks, test that your handler rejects tampered payloads — this is a security boundary:

```ts
// Valid signature — should process
const validSig = stripe.webhooks.generateTestHeaderString({
  payload: body, secret: webhookSecret
});
const res = await handler(body, validSig);
expect(res.status).toBe(200);

// Tampered payload — must reject
const res2 = await handler(body + 'x', validSig);
expect(res2.status).toBe(400);

// Missing signature header — must reject
const res3 = await handler(body, '');
expect(res3.status).toBe(400);
```

---

### 4. Idempotency keys — test that retries don't double-charge

For payment/billing services that support idempotency:

```ts
let callCount = 0;
server.use(
  http.post('https://api.stripe.com/v1/charges', ({ request }) => {
    callCount++;
    // Return same response for same idempotency key
    return HttpResponse.json(fixture('charge-success'));
  })
);

// Simulate retry (e.g. network blip on first attempt)
await createCharge(amount, { idempotencyKey: 'order-123' });
await createCharge(amount, { idempotencyKey: 'order-123' }); // retry

// Your code must send the same idempotency key on retry
// Verify by inspecting request headers
```

Capture the outgoing request and assert the `Idempotency-Key` header was sent and is stable across retries.

---

### 5. Rate limiting — test backoff behavior, not just the error

```ts
let attempts = 0;
server.use(
  http.get('https://api.sendgrid.com/v3/messages', () => {
    attempts++;
    if (attempts < 3) {
      return new HttpResponse(null, {
        status: 429,
        headers: { 'Retry-After': '1' }
      });
    }
    return HttpResponse.json(fixture('messages-list'));
  })
);

const result = await emailClient.listMessages();
expect(attempts).toBe(3);          // retried twice before succeeding
expect(result).toBeDefined();      // eventually got a result
```

Without asserting `attempts`, you're only testing the happy path that happens to follow a 429.

---

### 6. S3 / object storage — test key construction and metadata, not just upload success

```ts
const uploadSpy = jest.spyOn(s3Client, 'send');

await uploadDocument(userId, file);

const putCommand = uploadSpy.mock.calls[0][0];
expect(putCommand.input.Key).toBe(`users/${userId}/documents/${file.name}`);
expect(putCommand.input.ContentType).toBe(file.type);
expect(putCommand.input.ServerSideEncryption).toBe('AES256');
// Bucket name, key prefix, encryption — all should be asserted
```

An upload test that only checks "it didn't throw" won't catch a wrong bucket, missing encryption, or a key collision bug.

---

### 7. Pagination — test cursor/token exhaustion

Many APIs paginate with cursors. Test the full traversal, not just page 1:

```ts
server.use(
  http.get('https://api.stripe.com/v1/charges', ({ request }) => {
    const url = new URL(request.url);
    const cursor = url.searchParams.get('starting_after');
    if (!cursor) return HttpResponse.json({ data: [charge1, charge2], has_more: true, last_id: 'ch_2' });
    return HttpResponse.json({ data: [charge3], has_more: false });
  })
);

const all = await fetchAllCharges();
expect(all).toHaveLength(3); // traversed both pages
```

---

### 8. Timeout and network failure — verify your client doesn't hang

```ts
server.use(
  http.post('https://api.twilio.com/2010-04-01/Messages.json', async () => {
    await delay(10_000); // simulate hung connection
    return HttpResponse.json({});
  })
);

// Your client must timeout and not hang the test suite
await expect(sendSms('+15550001234', 'hello')).rejects.toThrow(/timeout/i);
```

If this test hangs, the production code has no timeout configured.

---

## Fixture Management

### Name fixtures by service, endpoint, and scenario

```
fixtures/
  stripe/
    customer-create-success.json
    customer-create-duplicate-email.json
    payment-intent-card-declined.json
    payment-intent-insufficient-funds.json
  sendgrid/
    send-email-success.json
    send-email-invalid-to.json
  s3/
    put-object-success.json
    put-object-access-denied.json
```

### Scrub before committing

Strip PII and live keys from recorded responses before committing. Fields to always scrub: `email`, `phone`, `name`, `address`, `ip`, `api_key`, `secret`, `token`, `livemode: true`.

Add a lint step or fixture validation script that fails CI if `livemode: true` appears in any fixture file.

---

## What Not to Do

- **Don't hit live production APIs in tests** — even accidentally; assert `baseURL` points to sandbox/mock in test env
- **Don't hand-write fixture payloads** — record from the real sandbox API; hand-written fixtures miss fields and drift
- **Don't mock at `fetch` when an SDK is in use** — mock at the SDK client or use an interceptor at the network layer
- **Don't skip webhook signature tests** — unsigned webhook handlers are a real attack surface
- **Don't assert only on the happy path** — every service has at least 3-5 named error codes that should have dedicated test cases

---

## Output Format

Group tests by interaction type: outbound calls, error responses, webhooks, retries. Each `it` covers one service scenario (one status code, one error code, one webhook event type). Include a comment on each fixture reference explaining what scenario it represents.