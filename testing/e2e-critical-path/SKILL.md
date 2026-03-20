---
name: e2e-critical-path
description: Identify and write E2E tests for the most critical user flows (top 3–5) that impact revenue or retention. Use when defining smoke tests or deciding what to test first.
category: "Testing"
---

# E2E Critical Path Skill

## Discovery

Before writing a single test, identify what "critical" means for this specific app. Ask or infer:

- **Revenue model** — SaaS subscription, e-commerce checkout, marketplace transaction, ad-driven?
- **Core value action** — the one thing users must be able to do for the product to have delivered value (send a message, publish a post, complete a booking, run a report)
- **Retention hooks** — what brings users back? (notifications, saved state, collaboration, history)
- **Known fragile integrations** — payments, email, file upload, real-time sync — these break disproportionately

If the user can't answer "what would make you wake up at 3am if it broke", that's the question to ask first.

---

## Flow Identification Framework

Score candidate flows on two axes only — likelihood of breakage × impact of breakage. Don't test flows that score low on both.

| Flow type | Almost always critical | Often not critical to E2E |
|---|---|---|
| Monetization | Checkout, subscription upgrade, trial activation | Coupon edge cases, invoice PDF download |
| Core value | Primary create/publish/send action | Settings tweaks, profile updates |
| Onboarding | First-time signup → first value moment | Step 3 of 7 in an optional wizard |
| Auth | Login, session persistence | Password strength indicator UI |
| Data integrity | Save → reload → data still there | Sort/filter preferences |

Cap at 5 flows. More than 5 critical path tests signals scope creep — if everything is critical, nothing is.

---

## Non-Obvious Patterns

### 1. Test the value delivery moment, not just navigation

The critical action isn't "user can reach the checkout page" — it's "user completes a purchase and receives confirmation":

```ts
test('user completes purchase and receives order confirmation', async ({ page }) => {
  await addItemToCart(page, productId);
  await proceedToCheckout(page);
  await fillPayment(page, testCard);
  await page.click('[data-testid=place-order]');

  // The value moment — not just a redirect
  await expect(page.locator('[data-testid=order-confirmation]')).toBeVisible();
  await expect(page.locator('[data-testid=order-number]')).toContainText(/ORD-\d+/);

  // Side effect — order actually exists
  const orderNum = await page.locator('[data-testid=order-number]').textContent();
  const order = await db.findOrder(orderNum);
  expect(order.status).toBe('confirmed');
});
```

---

### 2. Seed state via API/DB, never via UI setup steps

UI setup steps (clicking through a wizard to create prerequisite data) are the #1 source of flaky critical path tests. A test for "user can share a document" should not start by UI-navigating to create a document first:

```ts
// WRONG — UI setup couples two flows, doubles flake surface
test('user can share document', async ({ page }) => {
  await page.goto('/documents/new');
  await page.fill('[name=title]', 'Test Doc');
  await page.click('[data-testid=save]');
  // now test sharing...
});

// CORRECT — seed the prerequisite, test only what you're testing
test('user can share document', async ({ page }) => {
  const doc = await api.createDocument(authToken, { title: 'Test Doc' });
  await page.goto(`/documents/${doc.id}`);
  // test sharing directly
});
```

---

### 3. Assert the downstream effect, not just the UI confirmation

A "success" toast or redirect is a UI assertion. The critical path test must also verify the system state that makes the success real:

```ts
// Checkout: assert payment was captured, not just "thank you" shown
const charge = await stripe.charges.retrieve(chargeId);
expect(charge.status).toBe('succeeded');

// Message sent: assert it appears for the recipient, not just the sender
await recipientPage.goto('/inbox');
await expect(recipientPage.locator('[data-testid=message-item]')).toContainText(messageText);

// File upload: assert the file is retrievable, not just that upload UI closed
const res = await fetch(uploadedFileUrl);
expect(res.status).toBe(200);
```

---

### 4. Include the "returning user" variant for retention flows

First-time flows and returning-user flows have different code paths. A broken returning user flow is a silent churn driver:

```ts
// First visit — onboarding flow
test('new user completes onboarding and reaches dashboard', async ({ page }) => { ... });

// Return visit — must land in the right place with state intact
test('returning user session resumes with previous work intact', async ({ page }) => {
  // Use saved auth state
  await page.goto('/');
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('[data-testid=recent-items]')).not.toBeEmpty();
});
```

---

### 5. Mobile viewport for flows with high mobile traffic

If the app has significant mobile usage, run critical path tests at mobile viewport — not as separate tests, but as a parameterized variant:

```ts
// playwright.config.ts
projects: [
  { name: 'desktop', use: { ...devices['Desktop Chrome'] } },
  { name: 'mobile', use: { ...devices['Pixel 7'] } },
]
// critical path tests run in both projects automatically
```

A checkout flow that works on desktop but breaks on mobile viewport is a critical failure hiding in plain sight.

---

### 6. Capture the failure context, not just the assertion

Critical path test failures in CI should immediately tell you *what* broke, not just *that* something broke:

```ts
// playwright.config.ts
use: {
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
  trace: 'on-first-retry',
}
```

Without this, diagnosing a flaky critical path failure in CI requires a re-run with added logging — which wastes time during an incident.

---

### 7. Test the graceful degradation of each critical flow

Every critical path has a failure mode. Test the UX when it fails, not just when it succeeds:

```ts
// Payment fails — user must see a recoverable error, not a blank screen
test('payment failure shows actionable error message', async ({ page }) => {
  await fillPayment(page, declineCard); // use Stripe's test decline card
  await page.click('[data-testid=place-order]');
  await expect(page.locator('[data-testid=payment-error]')).toContainText(/declined|try again/i);
  // Cart must still be intact — user can retry
  await expect(page.locator('[data-testid=cart-items]')).not.toBeEmpty();
});
```

A critical path that fails ungracefully (blank page, infinite spinner, data loss) is worse than a flow that was never built.

---

## Suite Structure

```
e2e/
  critical-path/
    checkout.spec.ts         — add to cart → payment → confirmation
    core-action.spec.ts      — the primary value-delivery action
    onboarding.spec.ts       — signup → first value moment
    auth.spec.ts             — login, session, protected access (or use e2e-auth-flow skill)
    returning-user.spec.ts   — session resume, saved state, re-engagement
```

Each file: one flow, 2–4 tests (happy path + graceful failure + returning user variant where applicable). No more.

---

## CI Integration

Critical path tests should run on every deploy, not nightly. Configure them as a required check:

```yaml
# Run only critical path on PR — full suite on merge
- name: Critical path tests
  run: npx playwright test e2e/critical-path/
  env:
    BASE_URL: ${{ vars.STAGING_URL }}
```

Separate the critical path suite from the full E2E suite. The critical path suite must finish in under 3 minutes — if it doesn't, it will get skipped under pressure.

---

## What Not to Do

- **Don't test UI aesthetics in critical path tests** — colors, spacing, animations are not critical; functionality is
- **Don't add more than 5 flows** — if a flow isn't wake-you-up-at-3am critical, it belongs in a broader regression suite
- **Don't use fixed waits (`waitForTimeout`)** — wait for elements, URLs, or network idle; fixed waits make the suite slow and still flaky
- **Don't couple test order** — each test seeds its own state and runs independently; parallel execution must be safe
- **Don't skip the failure mode test** — a graceful failure is part of the critical path contract

---

## Output Format

One spec file per flow. Each file has a `describe` block named after the user journey (not the technical operation). Tests are ordered: happy path first, graceful failure second, returning user variant third if applicable. Setup is entirely via API/DB, never via UI navigation.