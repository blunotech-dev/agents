---
name: subscription-fullstack
description: Implement subscription billing end-to-end with plan selection UI, Stripe checkout, webhook lifecycle handling, plan-based access control, trials, and upgrade/downgrade/cancellation flows. Use when building or managing subscriptions, handling Stripe webhooks, enforcing plan limits, or syncing billing state with your app. Trigger on subscription billing, Stripe subscriptions, checkout sessions, webhooks, trial periods, billing portals, or subscription status changes.
category: "Fullstack"
---

# Subscription Fullstack

Covers the non-obvious parts of subscription billing: why your DB — not Stripe — is the access gate, the exact webhook events that matter and which to ignore, idempotency in webhook handlers, and the trial-to-paid transition edge cases most teams discover in production. Skips Stripe account setup — assumes keys exist and a product/price catalog is configured.

---

## Discovery

Before writing anything, answer:

1. **Billing provider**: Stripe (assumed below), Paddle, Lemon Squeezy?
2. **Plan structure**: Single plan with tiers, or multiple distinct products?
3. **Trial period**: Free trial before payment, or paid from day one?
4. **Cancellation behavior**: Cancel immediately, or access until period end?
5. **Seat-based or flat-rate**: Per-user pricing changes metering logic significantly.
6. **Customer portal**: Use Stripe's hosted portal, or build custom upgrade/cancel UI?

---

## Core Patterns

### 1. Your DB Is the Access Gate — Not Stripe

**The trap**: checking `stripe.subscriptions.retrieve()` on every request to verify access.

This is slow, expensive (Stripe API rate limits), and fails open when Stripe is down.

**Fix**: sync subscription state into your DB via webhooks and gate access against your own DB.

```ts
// DB schema — the subscription state you own and query
model Subscription {
  id                 String   @id
  userId             String   @unique
  stripeCustomerId   String   @unique
  stripeSubId        String?  @unique
  status             String   // 'trialing' | 'active' | 'past_due' | 'canceled' | 'unpaid'
  plan               String   // 'free' | 'pro' | 'enterprise'
  currentPeriodEnd   DateTime
  cancelAtPeriodEnd  Boolean  @default(false)
  trialEnd           DateTime?
}

// Access check — fast, no Stripe call
async function hasActiveSubscription(userId: string): Promise<boolean> {
  const sub = await db.subscription.findUnique({ where: { userId } });
  if (!sub) return false;

  const accessStatuses = new Set(['trialing', 'active']);
  // Non-obvious: 'past_due' — grace period, still grant access while retrying payment
  // 'unpaid' — retries exhausted, revoke access
  return accessStatuses.has(sub.status);
}
```

---

### 2. Checkout Session Creation

```ts
// Backend: create Stripe Checkout session
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

router.post('/api/billing/checkout', async (req, res) => {
  const { priceId } = req.body;
  const userId = req.user.id;

  // Upsert Stripe customer — one customer per user, reuse across sessions
  let sub = await db.subscription.findUnique({ where: { userId } });
  let customerId = sub?.stripeCustomerId;

  if (!customerId) {
    const customer = await stripe.customers.create({
      email: req.user.email,
      metadata: { userId },   // critical — lets you identify the user in webhooks
    });
    customerId = customer.id;
    await db.subscription.create({
      data: { id: customerId, userId, stripeCustomerId: customerId,
              status: 'free', plan: 'free',
              currentPeriodEnd: new Date() },
    });
  }

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    line_items: [{ price: priceId, quantity: 1 }],
    // Non-obvious: include the userId in metadata so the webhook can find the user
    // even if the customer lookup fails
    subscription_data: {
      metadata: { userId },
      trial_period_days: 14,  // omit if no trial
    },
    success_url: `${process.env.APP_URL}/billing/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url:  `${process.env.APP_URL}/billing/plans`,
  });

  res.json({ url: session.url });
});
```

**Non-obvious**: always attach `metadata: { userId }` to both the Stripe customer AND the subscription. Webhooks arrive with no session context — you'll need to look up the user from the Stripe object's metadata.

---

### 3. Webhook Handler — The Only Events That Matter

Most Stripe webhook tutorials handle every event. In practice, subscriptions only need these:

| Event | What to do |
|---|---|
| `checkout.session.completed` | Provision access; write `stripeSubId`, set `status: 'trialing'` or `'active'` |
| `customer.subscription.updated` | Sync `status`, `plan`, `currentPeriodEnd`, `cancelAtPeriodEnd` |
| `customer.subscription.deleted` | Set `status: 'canceled'` |
| `invoice.payment_failed` | Set `status: 'past_due'`; send dunning email |
| `invoice.payment_succeeded` | Reset `status: 'active'`; update `currentPeriodEnd` |

Everything else (`invoice.created`, `customer.updated`, etc.) can be ignored unless you have a specific reason.

```ts
// Webhook endpoint
router.post('/api/webhooks/stripe',
  express.raw({ type: 'application/json' }), // must be raw bytes for signature verification
  async (req, res) => {
    const sig = req.headers['stripe-signature']!;
    let event: Stripe.Event;

    try {
      event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
    } catch {
      return res.status(400).send('Webhook signature verification failed');
    }

    // Idempotency: Stripe retries webhooks on non-2xx — your handler must be idempotent
    // Use the event.id to deduplicate
    const alreadyProcessed = await db.webhookEvent.findUnique({ where: { id: event.id } });
    if (alreadyProcessed) return res.json({ received: true }); // acknowledge, don't reprocess

    await db.webhookEvent.create({ data: { id: event.id, type: event.type } });

    try {
      await handleStripeEvent(event);
    } catch (err) {
      // Log but still return 200 — returning 4xx/5xx causes Stripe to retry indefinitely
      console.error('Webhook handler error', event.id, err);
    }

    res.json({ received: true });
  }
);
```

**Non-obvious**: return `200` even when your handler throws. If you return a 5xx, Stripe retries the webhook up to 3 days — which can cause duplicate processing or spam your error logs. Catch handler errors, log them, still return 200, and alert separately.

---

### 4. Webhook Event Handlers

```ts
async function handleStripeEvent(event: Stripe.Event) {
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;
      if (session.mode !== 'subscription') break; // ignore one-time checkouts

      const sub = await stripe.subscriptions.retrieve(session.subscription as string);
      const userId = sub.metadata.userId; // from subscription_data.metadata

      await db.subscription.update({
        where: { userId },
        data: {
          stripeSubId:       sub.id,
          status:            sub.status,            // 'trialing' or 'active'
          plan:              getPlanFromPriceId(sub.items.data[0].price.id),
          currentPeriodEnd:  new Date(sub.current_period_end * 1000), // unix → Date
          trialEnd:          sub.trial_end ? new Date(sub.trial_end * 1000) : null,
        },
      });
      break;
    }

    case 'customer.subscription.updated': {
      const sub = event.data.object as Stripe.Subscription;
      const userId = sub.metadata.userId;

      await db.subscription.update({
        where: { userId },
        data: {
          status:           sub.status,
          plan:             getPlanFromPriceId(sub.items.data[0].price.id),
          currentPeriodEnd: new Date(sub.current_period_end * 1000),
          cancelAtPeriodEnd: sub.cancel_at_period_end,
        },
      });
      break;
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object as Stripe.Subscription;
      await db.subscription.update({
        where: { userId: sub.metadata.userId },
        data: { status: 'canceled', plan: 'free' },
      });
      break;
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice;
      const sub = await stripe.subscriptions.retrieve(invoice.subscription as string);
      await db.subscription.update({
        where: { userId: sub.metadata.userId },
        data: { status: 'past_due' },
      });
      // Trigger dunning email here
      break;
    }

    case 'invoice.payment_succeeded': {
      const invoice = event.data.object as Stripe.Invoice;
      if (invoice.billing_reason === 'subscription_create') break; // handled by checkout.session.completed
      const sub = await stripe.subscriptions.retrieve(invoice.subscription as string);
      await db.subscription.update({
        where: { userId: sub.metadata.userId },
        data: { status: 'active', currentPeriodEnd: new Date(sub.current_period_end * 1000) },
      });
      break;
    }
  }
}

// Map Stripe price IDs to your internal plan names — keep this centralized
function getPlanFromPriceId(priceId: string): string {
  const map: Record<string, string> = {
    [process.env.STRIPE_PRICE_PRO_MONTHLY!]:    'pro',
    [process.env.STRIPE_PRICE_PRO_YEARLY!]:     'pro',
    [process.env.STRIPE_PRICE_ENTERPRISE!]:     'enterprise',
  };
  return map[priceId] ?? 'free';
}
```

---

### 5. Trial-to-Paid Transition Edge Cases

**Trial end without payment method**: if a user starts a trial without a card, Stripe moves the subscription to `incomplete_expired` when the trial ends — not `past_due`. Handle this:

```ts
case 'customer.subscription.updated': {
  const sub = event.data.object as Stripe.Subscription;
  // 'incomplete_expired' = trial ended, no payment method added
  const effectiveStatus = sub.status === 'incomplete_expired' ? 'canceled' : sub.status;
  await db.subscription.update({
    where: { userId: sub.metadata.userId },
    data: { status: effectiveStatus, ... },
  });
}
```

**Trial conversion prompt** — show a banner N days before trial ends:

```ts
function trialDaysRemaining(sub: Subscription): number | null {
  if (sub.status !== 'trialing' || !sub.trialEnd) return null;
  return Math.ceil((sub.trialEnd.getTime() - Date.now()) / (1000 * 60 * 60 * 24));
}

// In layout component
const daysLeft = trialDaysRemaining(subscription);
if (daysLeft !== null && daysLeft <= 7) {
  // Show upgrade banner
}
```

---

### 6. Access Gating by Plan

```ts
const PLAN_FEATURES = {
  free:       { maxProjects: 3,  teamMembers: 1,  apiAccess: false },
  pro:        { maxProjects: 50, teamMembers: 10, apiAccess: true  },
  enterprise: { maxProjects: Infinity, teamMembers: Infinity, apiAccess: true },
} satisfies Record<string, PlanFeatures>;

// Backend middleware — gate by feature, not plan name
function requireFeature(feature: keyof PlanFeatures) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const sub = await db.subscription.findUnique({ where: { userId: req.user.id } });
    const plan = (sub?.status === 'active' || sub?.status === 'trialing') ? sub.plan : 'free';
    const features = PLAN_FEATURES[plan as keyof typeof PLAN_FEATURES];

    if (!features[feature]) {
      return res.status(403).json({ error: 'Upgrade required', feature });
    }
    next();
  };
}

// Non-obvious: gate on feature, not plan name. This lets you add 'apiAccess' to 'free'
// later without hunting down every `plan === 'pro'` check in your codebase.
router.get('/api/v1/data', requireFeature('apiAccess'), getData);
```

---

### 7. Cancellation and Stripe Billing Portal

**Cancel at period end** (preferred — user keeps access until paid period ends):

```ts
router.post('/api/billing/cancel', async (req, res) => {
  const sub = await db.subscription.findUnique({ where: { userId: req.user.id } });
  if (!sub?.stripeSubId) return res.status(400).json({ error: 'No active subscription' });

  // cancel_at_period_end = true means Stripe won't charge again, access continues until end
  await stripe.subscriptions.update(sub.stripeSubId, { cancel_at_period_end: true });

  await db.subscription.update({
    where: { userId: req.user.id },
    data: { cancelAtPeriodEnd: true },
  });

  res.json({ cancelAtPeriodEnd: true, accessUntil: sub.currentPeriodEnd });
});
```

**Stripe Billing Portal** — for self-serve plan changes, payment method updates, and invoice history without building custom UI:

```ts
router.post('/api/billing/portal', async (req, res) => {
  const sub = await db.subscription.findUnique({ where: { userId: req.user.id } });

  const session = await stripe.billingPortal.sessions.create({
    customer: sub!.stripeCustomerId,
    return_url: `${process.env.APP_URL}/billing`,
  });

  res.json({ url: session.url });
});
```

**Non-obvious**: when a user changes plans via the Billing Portal, your app receives a `customer.subscription.updated` webhook — not a new checkout session. The webhook handler above covers this automatically. No special handling needed.

---

## Output

Produce:
- `api/billing.ts` — checkout session, portal session, cancel endpoints
- `api/webhooks/stripe.ts` — signature verification, idempotency check, event router
- `lib/subscription.ts` — `hasActiveSubscription`, `requireFeature`, `getPlanFromPriceId`, `trialDaysRemaining`
- `SubscriptionGate.tsx` — frontend component that reads plan from context and renders upgrade prompt or children

Flag clearly in comments:
- The `past_due` grace period decision (access granted vs revoked)
- Every unix timestamp → Date conversion (Stripe sends unix, not ISO)
- The `billing_reason === 'subscription_create'` guard in `invoice.payment_succeeded`
- Idempotency key usage and why 200 is always returned from the webhook endpoint