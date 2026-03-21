---
name: payment-status-sync
description: Handle edge cases in payment and subscription state sync beyond basic webhooks, including out-of-order event delivery, retries, and status reconciliation. Ensure accurate UI display for payment states, grace periods, and next actions. Use when debugging webhook ordering issues, syncing Stripe status with your system, or building reliable billing status pages. Trigger on out-of-order webhooks, stale or incorrect status, payment retries, subscription state mismatch, or billing UI inconsistencies.
category: "Fullstack"
---

# Payment Status Sync

Covers what breaks after basic webhook handling is in place: events arriving out of order, status that looks stale to the user mid-flow, and building a status display that accurately reflects every sub-state without confusing users. Assumes idempotency and basic event handling already exist (see `subscription-fullstack`).

---

## Discovery

Before writing anything, answer:

1. **Do you have a `webhookEvent` deduplication table?** If not, add one before this skill adds anything useful.
2. **What's the user-visible status page?** Account settings, a dedicated billing page, or a banner in the app header?
3. **Retry behavior**: How many payment retry attempts does Stripe make before marking `unpaid`? (Configurable in Stripe Dashboard — default is 4 over ~4 weeks.)
4. **Dunning emails**: Sent by Stripe Smart Retries, your own system, or both?

---

## Core Patterns

### 1. Out-of-Order Webhook Delivery

Stripe does not guarantee event ordering. A common real sequence:

```
Expected:  checkout.session.completed → invoice.payment_succeeded → customer.subscription.updated
Actual:    customer.subscription.updated → checkout.session.completed → invoice.payment_succeeded
```

**The bug**: if `customer.subscription.updated` arrives first, your handler looks up `stripeSubId` in the DB to find the subscription — but `checkout.session.completed` hasn't run yet, so `stripeSubId` is null. The update silently no-ops or throws.

**Fix: look up by `stripeCustomerId`, not `stripeSubId`**

```ts
// FRAGILE — stripeSubId may not exist yet when this event arrives
await db.subscription.update({
  where: { stripeSubId: sub.id },
  data: { status: sub.status },
});

// ROBUST — stripeCustomerId is written at customer creation, always exists
await db.subscription.update({
  where: { stripeCustomerId: sub.customer as string },
  data: {
    stripeSubId:       sub.id,          // write it here too, idempotently
    status:            sub.status,
    currentPeriodEnd:  new Date(sub.current_period_end * 1000),
    cancelAtPeriodEnd: sub.cancel_at_period_end,
  },
});
```

Apply this pattern to every webhook handler — always look up by `stripeCustomerId`.

---

### 2. Timestamp-Guarded Updates

Out-of-order events also mean a stale event can overwrite a newer one. An `invoice.payment_failed` arriving after `invoice.payment_succeeded` would incorrectly set `status: 'past_due'`.

**Fix: only write if the incoming event is newer than what's stored**

```ts
case 'customer.subscription.updated': {
  const sub = event.data.object as Stripe.Subscription;

  // event.created is unix seconds — the timestamp of when Stripe generated the event
  await db.subscription.updateMany({
    where: {
      stripeCustomerId: sub.customer as string,
      lastWebhookAt: { lt: new Date(event.created * 1000) }, // only if older
    },
    data: {
      status:           sub.status,
      currentPeriodEnd: new Date(sub.current_period_end * 1000),
      lastWebhookAt:    new Date(event.created * 1000),
    },
  });
  // Non-obvious: updateMany with a where clause — if the record was already
  // updated by a newer event, this silently no-ops instead of overwriting.
  break;
}
```

Add `lastWebhookAt DateTime?` to your subscription model. Initialize it on creation.

---

### 3. Reconciliation Job — Catch What Webhooks Miss

Webhooks fail silently when: your server is down during delivery, a handler throws after the 200 was already sent, or Stripe's retry window (72 hours) expires without a successful ack.

**Fix: a periodic reconciliation job that syncs from Stripe directly**

```ts
// Run nightly via cron or a queue job
async function reconcileSubscriptions() {
  // Find subscriptions that haven't received a webhook in >24 hours
  // and are in a non-terminal state
  const stale = await db.subscription.findMany({
    where: {
      status: { in: ['active', 'trialing', 'past_due'] },
      lastWebhookAt: { lt: new Date(Date.now() - 24 * 60 * 60 * 1000) },
      stripeSubId: { not: null },
    },
  });

  for (const sub of stale) {
    try {
      const stripeSub = await stripe.subscriptions.retrieve(sub.stripeSubId!);
      await db.subscription.update({
        where: { id: sub.id },
        data: {
          status:           stripeSub.status,
          currentPeriodEnd: new Date(stripeSub.current_period_end * 1000),
          lastWebhookAt:    new Date(), // reset so it won't be flagged again immediately
        },
      });
    } catch (err) {
      // Log but continue — don't let one bad record abort the whole job
      console.error('Reconcile failed for', sub.id, err);
    }
  }
}
```

**Non-obvious**: this job also catches the case where a user cancels their subscription directly in the Stripe Dashboard (bypassing your cancel endpoint). Without reconciliation, your DB stays `active` indefinitely.

---

### 4. Status Display — Every Sub-State the User Can See

Raw Stripe statuses map poorly to user-facing copy. Define the mapping once:

```ts
// lib/subscriptionDisplay.ts
interface StatusDisplay {
  label:       string;
  description: string;
  severity:    'success' | 'warning' | 'error' | 'neutral';
  action?:     { label: string; href: string };
}

export function getStatusDisplay(sub: Subscription): StatusDisplay {
  const portalHref = '/api/billing/portal'; // redirects to Stripe Billing Portal

  // Non-obvious: cancelAtPeriodEnd is not a Stripe status — it's a flag on an
  // 'active' subscription. Check it before checking status or you'll never show it.
  if (sub.status === 'active' && sub.cancelAtPeriodEnd) {
    const date = sub.currentPeriodEnd.toLocaleDateString();
    return {
      label:       'Cancels ' + date,
      description: `Your plan is active until ${date}. You won't be charged again.`,
      severity:    'warning',
      action:      { label: 'Resume subscription', href: portalHref },
    };
  }

  const map: Record<string, StatusDisplay> = {
    trialing: {
      label:       'Free trial',
      description: `Trial ends ${sub.trialEnd?.toLocaleDateString() ?? 'soon'}.`,
      severity:    'success',
      action:      { label: 'Add payment method', href: portalHref },
    },
    active: {
      label:       'Active',
      description: `Renews ${sub.currentPeriodEnd.toLocaleDateString()}.`,
      severity:    'success',
    },
    past_due: {
      label:       'Payment failed',
      description: 'We couldn\'t charge your card. We\'ll retry automatically.',
      severity:    'warning',
      action:      { label: 'Update payment method', href: portalHref },
    },
    unpaid: {
      label:       'Access suspended',
      description: 'Payment retries exhausted. Update your payment method to restore access.',
      severity:    'error',
      action:      { label: 'Update payment method', href: portalHref },
    },
    canceled: {
      label:       'Canceled',
      description: 'Your subscription has ended.',
      severity:    'neutral',
      action:      { label: 'Resubscribe', href: '/billing/plans' },
    },
    incomplete: {
      label:       'Payment pending',
      description: 'Complete your payment to activate your subscription.',
      severity:    'warning',
      action:      { label: 'Complete payment', href: portalHref },
    },
    incomplete_expired: {
      label:       'Subscription expired',
      description: 'Your trial ended without a payment method on file.',
      severity:    'error',
      action:      { label: 'Start a new plan', href: '/billing/plans' },
    },
  };

  return map[sub.status] ?? {
    label: 'Unknown', description: '', severity: 'neutral',
  };
}
```

---

### 5. Optimistic Status Update During Checkout

After `checkout.session.completed`, the webhook typically arrives 1–10 seconds after the redirect. During that window, the user sees their old plan — causing confusion.

**Fix: optimistic status on the success page, reconcile with DB after**

```ts
// success page — /billing/success?session_id=xxx
async function BillingSuccessPage({ searchParams }) {
  const sessionId = searchParams.session_id;

  // Fetch session from Stripe directly on the success page — fast path
  // Non-obvious: this is the ONE place it's acceptable to call Stripe directly,
  // because the webhook may not have arrived yet
  const session = await stripe.checkout.sessions.retrieve(sessionId);

  if (session.payment_status === 'paid') {
    // Show success UI immediately without waiting for webhook
    return <SuccessBanner plan={getPlanFromSession(session)} />;
  }

  // Payment still processing — show pending state
  return <PendingBanner />;
}

// After rendering, poll your own DB until the webhook lands
// Client-side:
function usePlanSyncPoll(expectedPlan: string) {
  const { data } = useQuery({
    queryKey: ['subscription'],
    queryFn: () => fetch('/api/billing/status').then(r => r.json()),
    refetchInterval: (data) =>
      data?.plan === expectedPlan ? false : 2000, // poll every 2s until synced, then stop
  });
  return data;
}
```

---

### 6. On-Demand Reconciliation — Fix It Now, Not Tonight

The nightly job catches drift eventually. When a user is actively blocked ("Stripe says I paid, why can't I access X?"), you need two things: a support endpoint to force-sync a single user, and a self-serve button so users can try themselves without filing a ticket.

```ts
// Support endpoint — internal only, gated by admin role
router.post('/api/admin/billing/reconcile/:userId', requireRole('admin'), async (req, res) => {
  const sub = await db.subscription.findUnique({ where: { userId: req.params.userId } });
  if (!sub?.stripeSubId) return res.status(404).json({ error: 'No subscription found' });

  const stripeSub = await stripe.subscriptions.retrieve(sub.stripeSubId);
  await db.subscription.update({
    where: { id: sub.id },
    data: {
      status:           stripeSub.status,
      currentPeriodEnd: new Date(stripeSub.current_period_end * 1000),
      cancelAtPeriodEnd: stripeSub.cancel_at_period_end,
      lastWebhookAt:    new Date(),
    },
  });

  res.json({ previous: sub.status, current: stripeSub.status });
});

// Self-serve endpoint — rate-limited so users can't hammer Stripe
const reconcileRateLimit = rateLimit({ windowMs: 60_000, max: 3 }); // 3 per minute

router.post('/api/billing/refresh', reconcileRateLimit, async (req, res) => {
  const sub = await db.subscription.findUnique({ where: { userId: req.user.id } });
  if (!sub?.stripeSubId) return res.json({ status: 'free' });

  const stripeSub = await stripe.subscriptions.retrieve(sub.stripeSubId);
  await db.subscription.update({
    where: { id: sub.id },
    data: {
      status:           stripeSub.status,
      currentPeriodEnd: new Date(stripeSub.current_period_end * 1000),
      lastWebhookAt:    new Date(),
    },
  });

  res.json({ status: stripeSub.status });
});
```

```tsx
// "Refresh billing status" button on the billing page
function BillingStatusRefresh() {
  const [refreshing, setRefreshing] = useState(false);
  const queryClient = useQueryClient();

  async function handleRefresh() {
    setRefreshing(true);
    await fetch('/api/billing/refresh', { method: 'POST' });
    await queryClient.invalidateQueries({ queryKey: ['subscription'] });
    setRefreshing(false);
  }

  return (
    <button onClick={handleRefresh} disabled={refreshing}>
      {refreshing ? 'Checking...' : 'Refresh billing status'}
    </button>
  );
}
```

**Non-obvious**: the self-serve endpoint must be rate-limited — without it, a confused user hitting refresh repeatedly generates a Stripe API call per click, which can exhaust rate limits during an incident when many users are affected simultaneously.

---

## Output

Produce:
- `lastWebhookAt` column added to subscription model + all handlers updated to use `stripeCustomerId` lookup and timestamp guard
- `lib/reconcile.ts` — nightly job with stale detection and per-record error isolation
- `api/billing/refresh.ts` — rate-limited self-serve sync endpoint + admin force-sync endpoint
- `lib/subscriptionDisplay.ts` — full status → display mapping including `cancelAtPeriodEnd` check
- `BillingSuccessPage` with optimistic Stripe session fetch + polling hook
- `BillingStatusRefresh` button component with query invalidation

Flag clearly in comments:
- Every `stripeCustomerId` lookup explaining why `stripeSubId` is unsafe for ordering reasons
- The `updateMany` + `lastWebhookAt` guard explaining the stale-overwrite risk
- The rate limit on the self-serve endpoint and why it matters during incidents
- The success page Stripe call as the only acceptable direct Stripe call outside of write operations