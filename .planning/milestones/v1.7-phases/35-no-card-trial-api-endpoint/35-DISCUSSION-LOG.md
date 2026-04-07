# Phase 35: No-Card Trial API Endpoint - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-01
**Phase:** 35-no-card-trial-api-endpoint
**Areas discussed:** Trial creation flow, Webhook compatibility, Subscription guard rules, Auto-cancel behavior

---

## Trial Creation Flow

### Stripe Customer Reuse

| Option | Description | Selected |
|--------|-------------|----------|
| Reuse if exists | Look up existing Stripe Customer by userId. Only create new if none exists. One Customer per user in Stripe. | ✓ |
| Always create new | Create fresh Stripe Customer every time. Simpler but produces orphaned Customer objects. | |
| You decide | Claude chooses based on existing patterns and Stripe best practices. | |

**User's choice:** Reuse if exists
**Notes:** None

### Synchronous Local Record Write

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, sync write | Write local record immediately with status=trialing. User sees trial active instantly. Matches success criteria #2. | ✓ |
| Webhook only | Let webhook create local record. Brief window where user has no local subscription. | |
| You decide | Claude chooses based on success criteria and existing patterns. | |

**User's choice:** Yes, sync write
**Notes:** None

### Price Configuration

| Option | Description | Selected |
|--------|-------------|----------|
| Same price, env var | Use existing STRIPE_PRICE_ID env var. One price for both flows. | ✓ |
| Hardcoded in trial endpoint | Hardcode price ID. Simple but diverges from checkout pattern. | |
| You decide | Claude follows existing env var patterns. | |

**User's choice:** Same price, env var
**Notes:** None

### Response Body

| Option | Description | Selected |
|--------|-------------|----------|
| Subscription status | Return full local subscription record. Matches existing GET /v1/subscription pattern. | ✓ |
| Minimal confirmation | Return just { success: true }. Caller may need follow-up GET. | |
| You decide | Claude decides based on existing response patterns. | |

**User's choice:** Subscription status
**Notes:** None

---

## Webhook Compatibility

### Deduplication Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Upsert by stripeCustomerId | Add customer.subscription.created to processor. Upserts — if record exists from sync write, updates; if not, creates. Same idempotency as Phase 30. | ✓ |
| Skip if record exists | Check if record with stripeSubscriptionId exists, log and skip. Simpler but misses webhook-only fields. | |
| You decide | Claude chooses deduplication strategy. | |

**User's choice:** Upsert by stripeCustomerId
**Notes:** None

### checkout.session.completed Update

| Option | Description | Selected |
|--------|-------------|----------|
| Make it upsert too | Update to upsert instead of insert-only. Handles trial->cancel->checkout edge case. Both paths resilient. | ✓ |
| Keep checkout as-is | Leave unchanged. Risk: duplicate key error if user had prior trial. | |
| You decide | Claude assesses existing handler. | |

**User's choice:** Make it upsert too
**Notes:** None

### New Event Types

| Option | Description | Selected |
|--------|-------------|----------|
| Only subscription.created | Minimal change. trial_will_end useful for nudges but deferred to future milestone. | ✓ |
| Add trial_will_end too | Register event now even if just logged. Prevents later Stripe config update. | |
| You decide | Claude decides based on scope boundaries. | |

**User's choice:** Only subscription.created
**Notes:** None

---

## Subscription Guard Rules

### Re-Trial Policy

| Option | Description | Selected |
|--------|-------------|----------|
| No re-trial — subscribe only | If user had any prior subscription (any status), reject trial. Direct to Checkout/Billing Portal. Trial is one-time benefit. | ✓ |
| Allow re-trial after cancel | Let canceled users start fresh trial. More lenient but risks abuse. | |
| You decide | Claude decides based on business model. | |

**User's choice:** No re-trial — subscribe only
**Notes:** None

### Guard Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Reject if any record exists | Any local subscription record for userId (any status) blocks trial. Trial only for brand-new users. | ✓ |
| Reject only trialing/active | Reuse Phase 30's guard logic. Allows canceled users to re-trial. | |
| You decide | Claude implements consistent with no-re-trial decision. | |

**User's choice:** Reject if any record exists
**Notes:** None

### Error Response

| Option | Description | Selected |
|--------|-------------|----------|
| 422 with redirect hint | HTTP 422 (InvalidRequestError) with "Trial already used" message. Consistent with Phase 30 pattern. | ✓ |
| 409 Conflict | HTTP 409. Semantically accurate but no existing error class for 409. | |
| You decide | Claude picks following existing error patterns. | |

**User's choice:** 422 with redirect hint
**Notes:** None

---

## Auto-Cancel Behavior

### Configuration Level

| Option | Description | Selected |
|--------|-------------|----------|
| In API code | Set trial_settings on stripe.subscriptions.create(). Explicit, version-controlled. | ✓ |
| In Stripe Dashboard | Configure on product/price in each Stripe account. Requires manual setup per env. | |
| You decide | Claude chooses based on maintainability. | |

**User's choice:** In API code
**Notes:** User asked for explanation of what trial_settings.end_behavior.missing_payment_method means. Explained cancel vs pause vs create_invoice options and why cancel is correct for no-card trial.

### Existing Handler Sufficiency

| Option | Description | Selected |
|--------|-------------|----------|
| Existing handler sufficient | customer.subscription.deleted sets status=canceled. Phase 38 can infer trial-expired vs user-canceled from timestamps. No schema change. | ✓ |
| Add cancelReason field | Add cancelReason enum to entity. Explicit but requires schema change. | |
| You decide | Claude decides based on Phase 38 needs. | |

**User's choice:** Existing handler sufficient
**Notes:** User asked about difference between canceling and pausing in Stripe. Explained: cancel = permanent end, pause = billing suspended but subscription alive. Cancel is correct because no payment method means nothing to resume.

---

## Claude's Discretion

- Exact stripe.subscriptions.create() parameter structure
- Repository method naming
- Trial creator service structure
- Unit test scope
- Stripe webhook registration documentation

## Deferred Ideas

None — discussion stayed within phase scope.
