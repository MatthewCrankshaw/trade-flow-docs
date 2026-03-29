# Requirements: Trade Flow v1.6 — Stripe Subscription Billing

**Defined:** 2026-03-28
**Core Value:** A job is the centre of the business — Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.6 Requirements

### Subscription Acquisition

- [x] **ACQ-01**: User can initiate a 30-day free trial by completing Stripe Checkout (card required at trial start)
- [x] **ACQ-02**: API creates a Stripe Checkout Session with `trial_period_days: 30` and returns the hosted checkout URL
- [x] **ACQ-03**: User is redirected to `/subscribe/success` after completing Stripe Checkout
- [ ] **ACQ-04**: User is redirected to `/subscribe/cancel` if they abandon Stripe Checkout, with a "Try again" option
- [x] **ACQ-05**: `/subscribe/success` page polls for subscription status and verifies the Checkout Session server-side (race condition safety net)

### Webhook & Status Sync

- [x] **WBHK-01**: Stripe webhook endpoint (`POST /v1/webhooks/stripe`) verifies request signature using raw body before processing any event
- [x] **WBHK-02**: `checkout.session.completed` event creates the local subscription record in MongoDB with status `trialing`
- [x] **WBHK-03**: `customer.subscription.updated` event syncs status, `currentPeriodEnd`, `trialEnd`, and `cancelAtPeriodEnd` to the local record
- [x] **WBHK-04**: `customer.subscription.deleted` event sets local record status to `canceled` and records `canceledAt`
- [x] **WBHK-05**: `invoice.payment_succeeded` event sets local record status to `active`
- [x] **WBHK-06**: `invoice.payment_failed` event sets local record status to `past_due`
- [x] **WBHK-07**: Webhook handler is idempotent — processing the same event twice produces no duplicate records (upsert by `stripeSubscriptionId`)

### Access Enforcement

- [x] **GATE-01**: All business routes (dashboard, customers, jobs, quotes, items, business) are wrapped by `SubscriptionGate` and redirect to `/subscribe` when subscription status is `past_due`, `canceled`, `incomplete`, or when no subscription record exists
- [ ] **GATE-02**: Settings page is always accessible regardless of subscription status (so users can resubscribe)
- [ ] **GATE-03**: `/subscribe`, `/subscribe/success`, and `/subscribe/cancel` pages are accessible without an active subscription
- [x] **GATE-04**: Users with a support role bypass `SubscriptionGate` entirely and have full access without a subscription
- [x] **GATE-05**: `SubscriptionGate` shows a loading skeleton during the initial subscription status fetch (no flash of the subscribe page for active users)

### Trial Experience

- [x] **TRIAL-01**: A persistent banner is shown to trialing users displaying the number of days remaining in their free trial and a "Subscribe" CTA
- [x] **TRIAL-02**: The trial banner is not shown when subscription status is `active`

### Billing Management

- [x] **BILL-01**: `GET /v1/subscription` returns the current user's subscription status, `currentPeriodEnd`, `trialEnd`, and `cancelAtPeriodEnd`
- [x] **BILL-02**: `DELETE /v1/subscription` cancels the subscription at period end (`cancel_at_period_end: true`) — access continues until `currentPeriodEnd`
- [x] **BILL-03**: `POST /v1/subscription/portal` creates a Stripe Billing Portal session and returns the portal URL for redirect
- [x] **BILL-04**: Settings page has a Billing tab showing the user's current subscription status, next billing date (or trial end date), and appropriate CTAs (Subscribe / Manage Billing / Cancel)
- [x] **BILL-05**: When `cancelAtPeriodEnd` is true, the Billing tab shows "Cancels on [date]" rather than "Active" so the user knows when access ends

### Testing & Quality

- [x] **TEST-01**: `SubscriptionCreatorService` has unit tests covering Checkout Session creation (Stripe SDK mocked)
- [x] **TEST-02**: `SubscriptionUpdaterService` has unit tests covering each webhook event type, including invalid signature rejection
- [x] **TEST-03**: `SubscriptionRetrieverService` has unit tests covering found and not-found subscription scenarios
- [x] **TEST-04**: `SubscriptionRepository` has unit tests covering upsert, findByUserId, and findByStripeSubscriptionId operations

---

## Future Requirements

<!-- Deferred from v1.6 -- candidate for future milestone -->

- BullMQ-based scheduled trial reminder emails (Stripe handles natively in v1.6 via Stripe-managed trial emails)
- Annual billing / discounted annual plan
- Team/seat-based billing
- Invoice PDF generation in-app (Stripe Billing Portal provides invoice history)
- Proration display when resubscribing mid-period
- Multiple pricing tiers

---

## Out of Scope

<!-- Explicit exclusions with reasoning -->

- **Custom card input form (Stripe Elements)** — hosted Checkout handles PCI compliance, SCA, 3DS automatically; building custom forms adds compliance risk and weeks of work
- **Stripe.js / React Stripe.js on frontend** — not needed for hosted Checkout flow
- **Custom dunning emails** — Stripe's built-in dunning handles payment failure retry emails; building custom via BullMQ is premature
- **Stripe webhook queue via BullMQ** — webhooks are synchronous HTTP events processed inline; no background queuing needed
- **Multiple plans or annual billing** — single £6/month plan only; no plan selection UI needed

---

## Traceability

<!-- Filled by roadmapper: maps each phase to its REQ-IDs -->

| Phase | REQ-IDs |
|-------|---------|
| Phase 29: Subscription Module Foundation | ACQ-02, WBHK-01 |
| Phase 30: Stripe Checkout and Webhooks | ACQ-01, ACQ-03, ACQ-04, ACQ-05, WBHK-02, WBHK-03, WBHK-04, WBHK-05, WBHK-06, WBHK-07 |
| Phase 31: Subscription API Endpoints and Tests | BILL-01, BILL-02, BILL-03, TEST-01, TEST-02, TEST-03, TEST-04 |
| Phase 32: Subscription Gate and Subscribe Pages | GATE-01, GATE-02, GATE-03, GATE-04, GATE-05 |
| Phase 33: Trial Banner and Billing Settings Tab | TRIAL-01, TRIAL-02, BILL-04, BILL-05 |
