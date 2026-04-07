# Phase 35: No-Card Trial API Endpoint - Context

**Gathered:** 2026-04-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Backend endpoint (`POST /v1/subscription/trial`) that creates a Stripe subscription with a 30-day trial period requiring no payment method. The endpoint writes a local MongoDB subscription record synchronously (belt-and-suspenders), and the webhook infrastructure is extended with a `customer.subscription.created` handler to ensure compatibility with both Checkout-originated and API-originated subscriptions. When the trial ends without a card on file, Stripe auto-cancels the subscription.

</domain>

<decisions>
## Implementation Decisions

### Trial Creation Flow
- **D-01:** Reuse existing Stripe Customer if one exists for the user. Look up `stripeCustomerId` on the local subscription record by `userId`. Only create a new Stripe Customer if no record exists. One Customer per user in Stripe.
- **D-02:** Write the local subscription record synchronously in the endpoint response (status=trialing, stripeSubscriptionId, trialEnd, currentPeriodEnd). User sees trial active immediately without waiting for webhook.
- **D-03:** Use the same `STRIPE_PRICE_ID` env var that the Checkout flow reads. One price for both flows.
- **D-04:** Return the full local subscription record in the response body (status, trialEnd, currentPeriodEnd, etc.). Matches existing `GET /v1/subscription` response shape.
- **D-05:** Set `trial_settings.end_behavior.missing_payment_method: 'cancel'` on the `stripe.subscriptions.create()` call. Explicit in code, version-controlled, no manual Stripe Dashboard configuration per environment.

### Webhook Compatibility
- **D-06:** Add `customer.subscription.created` as a new event type in `StripeWebhookProcessor`. It upserts by `stripeCustomerId` — if a record already exists (from sync write), it updates; if not (Checkout flow or race condition), it creates.
- **D-07:** Update `checkout.session.completed` handler to upsert instead of insert-only. Handles edge cases where a user starts trial, cancels, then goes through Checkout. Both webhook paths become resilient to pre-existing records.
- **D-08:** Only add `customer.subscription.created` — do NOT add `customer.subscription.trial_will_end`. Trial nudge emails (NUDGE-01/02) are explicitly deferred to a future milestone.

### Subscription Guard Rules
- **D-09:** Trial endpoint rejects if ANY local subscription record exists for the userId (any status: trialing, active, past_due, canceled). Trial is a one-time benefit for brand-new users only.
- **D-10:** No re-trial after cancellation. Users with canceled subscriptions must resubscribe via Checkout/Billing Portal with a card.
- **D-11:** Return HTTP 422 (`InvalidRequestError`) with message like "Trial already used. Manage your subscription in Billing." Consistent with Phase 30's duplicate checkout guard pattern.

### Auto-Cancel Behavior
- **D-12:** When trial ends without a payment method, Stripe fires `customer.subscription.deleted`. The existing Phase 30 handler already sets `status=canceled` and `canceledAt`. No additional handling needed.
- **D-13:** Phase 38's differentiated paywall messaging can infer "trial expired" vs "user canceled" by comparing `trialEnd` and `canceledAt` timestamps on the existing entity fields. No new schema fields required.

### Claude's Discretion
- Exact `stripe.subscriptions.create()` parameter structure beyond what's specified above
- Repository method naming for the new lookup patterns (e.g., `findByUserId`, `findByStripeCustomerId`)
- How the trial creator service is structured (new service class vs extending existing)
- Unit test structure and scope
- Stripe webhook registration configuration (which events to register in Stripe Dashboard — documented in README or deploy notes)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.7 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.7 requirements; Phase 35 covers TRIAL-01, TRIAL-04

### Roadmap
- `.planning/ROADMAP.md` §Phase 35 — Phase goal, success criteria, dependency chain

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/milestones/v1.6-phases/29-subscription-module-foundation/29-CONTEXT.md` — Stripe Customer creation, entity schema, module structure, env vars, SubscriptionStatus enum
- `.planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-CONTEXT.md` — Webhook event handling, reconciliation by stripeCustomerId, upsert idempotency, duplicate checkout guard, verify-session endpoint

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — NestJS naming patterns, error handling, logging
- `.planning/codebase/ARCHITECTURE.md` — Module structure, controller->service->repo layering

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SubscriptionModule`: existing NestJS module with `STRIPE_CLIENT` injection token, repository, entity, DTOs
- `SubscriptionRepository`: existing repo with upsert patterns by `stripeCustomerId` and `stripeSubscriptionId`
- `StripeWebhookProcessor`: existing BullMQ processor handling 5 event types — add `customer.subscription.created` here
- `QueueProducer` + `STRIPE_WEBHOOKS` queue: existing BullMQ infrastructure for async webhook processing
- `InvalidRequestError`: existing error class for HTTP 422 responses
- `@Public()` decorator: marks routes as public (no JwtAuthGuard) — webhook route already uses this
- `ConfigService`: existing pattern for reading env vars (STRIPE_SECRET_KEY, STRIPE_PRICE_ID, FRONTEND_URL)

### Established Patterns
- Checkout flow: create Stripe Customer -> create Checkout Session -> sync write local record -> webhook updates later
- Webhook handler: validate signature -> enqueue on BullMQ -> processor upserts local record by stripeCustomerId
- Duplicate guard: check for existing trialing/active subscription before creating new one (Phase 30 D-14)
- Response format: `{ data: T[], pagination?, errors? }` standard response wrapper

### Integration Points
- `SubscriptionController`: add new `POST /v1/subscription/trial` route alongside existing checkout and manage endpoints
- `StripeWebhookProcessor`: add `customer.subscription.created` case to event type switch/handler
- `checkout.session.completed` handler: update from insert to upsert
- Stripe Dashboard: register `customer.subscription.created` webhook event (deploy note)

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing subscription module patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 35-no-card-trial-api-endpoint*
*Context gathered: 2026-04-01*
