# Feature Landscape

**Domain:** Stripe subscription billing for a sole-tradesperson SaaS app (£6/month)
**Researched:** 2026-03-28

---

## Table Stakes (must ship in v1.6)

### Subscription Acquisition
- **Stripe Checkout flow** — hosted card entry with 30-day trial; user redirected to `checkout.stripe.com`, returns to `/subscribe/success`
- **Subscribe page (`/subscribe`)** — shows plan name, price (£6/month), feature list, and single "Start Free Trial" CTA
- **Success callback page (`/subscribe/success`)** — confirms subscription started, shows next billing date, redirects to dashboard
- **Cancel callback page (`/subscribe/cancel`)** — user abandoned Checkout; shows "Try again" CTA, no penalty

### Access Enforcement
- **SubscriptionGate** — React component wrapping all business routes; redirects to `/subscribe` when status is `past_due`, `canceled`, or trial expired
- **Read-only mode** — all pages load but create/edit/delete actions disabled; banner prompts to subscribe
- **Settings accessible without subscription** — user can always reach billing settings to resubscribe

### Trial Experience
- **Trial banner** — persistent top banner during trial showing "X days left in your free trial" + "Subscribe" CTA
- **Banner disappears** when subscription becomes `active`
- **Trial expiry** — on `trialEndsAt` date: status transitions to `past_due` or `canceled` depending on card outcome; SubscriptionGate enforces read-only

### Billing Management (Settings > Billing tab)
- **SubscriptionStatusCard** — shows current status (Trialing / Active / Past Due / Canceled), current period end, trial end date
- **"Manage Billing" button** — opens Stripe Billing Portal (update card, cancel, view invoice history)
- **Cancel flow** — sets `cancelAtPeriodEnd: true`; access continues until period end; status shows "Cancels on [date]"
- **Resubscribe** — if canceled, show "Resubscribe" CTA back to Stripe Checkout

### Webhook-Driven Status Sync
- **`customer.subscription.created`** — creates local subscription record, status = `trialing`
- **`customer.subscription.updated`** — syncs status, `currentPeriodEnd`, `trialEndsAt`, `cancelAtPeriodEnd`
- **`customer.subscription.deleted`** — sets status = `canceled`, `canceledAt`
- **`invoice.payment_succeeded`** — sets status = `active`
- **`invoice.payment_failed`** — sets status = `past_due`

---

## Differentiators (worth adding in v1.6 if low effort)

- **Support role exemption** — users with `supportRoles` bypass SubscriptionGate entirely; allows admin access without subscription
- **Proration messaging** — if user cancels and resubscribes mid-period, Stripe handles proration automatically; no custom logic needed
- **"Cancels on [date]" indicator** — show `cancelAtPeriodEnd` state clearly so user knows when access ends

---

## Anti-Features (do NOT build in v1.6)

| Feature | Why not |
|---------|---------|
| Custom in-app card form (Stripe Elements) | Hosted Checkout handles PCI compliance, SCA, 3DS — building custom forms adds weeks of work and compliance risk |
| Email reminders for trial ending | Stripe handles this natively; building custom via BullMQ is premature |
| Multiple pricing tiers / plans | Single £6/month plan — no need for plan selection UI |
| Annual billing / discounts | Adds pricing logic complexity; not in scope |
| Team/seat-based billing | Solo operator only; per-user is correct |
| Invoice PDF generation in-app | Stripe Billing Portal provides invoice history already |
| Dunning emails (custom) | Stripe's built-in dunning handles payment failure retries |
| In-app upgrade prompts per feature | Single SubscriptionGate is sufficient at this scale |

---

## User Flows

### New User Subscribe Flow
1. Completes onboarding → prompted to start free trial
2. Lands on `/subscribe` → sees plan details + "Start Free Trial"
3. Clicks CTA → API creates Stripe Checkout Session with `trial_period_days: 30`
4. Redirected to `checkout.stripe.com` → enters card
5. Stripe redirects to `/subscribe/success`
6. Webhook fires `customer.subscription.created` → local record created (status=`trialing`)
7. Frontend polls/refetches subscription status → trial banner appears

### Trial Expiry Flow
1. `trialEndsAt` reached → Stripe attempts to charge card
2a. **Payment succeeds** → `invoice.payment_succeeded` webhook → status = `active`, banner disappears
2b. **Payment fails** → `invoice.payment_failed` webhook → status = `past_due`, SubscriptionGate enforces read-only
3. User prompted to update card via Stripe Billing Portal

### Manage/Cancel Flow
1. User goes to Settings > Billing
2. Sees current status and billing date
3. Clicks "Manage Billing" → API creates Billing Portal session → redirect to `billing.stripe.com`
4. User cancels or updates card in portal
5. Webhook fires → local record updated
