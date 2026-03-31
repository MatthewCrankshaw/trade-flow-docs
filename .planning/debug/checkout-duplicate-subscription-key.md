---
status: awaiting_human_verify
trigger: "Clicking 'Unlock full access' button triggers checkout endpoint which fails with E11000 duplicate key error on subscriptions collection userId_1 index."
created: 2026-03-29T00:00:00Z
updated: 2026-03-29T00:01:00Z
---

## Current Focus

hypothesis: CONFIRMED -- createCheckoutSession uses repository.create() (insertOne) instead of upsertByUserId when a subscription record already exists with non-active/non-trialing status (e.g., INCOMPLETE, CANCELED).
test: n/a -- root cause confirmed via code reading
expecting: n/a
next_action: Replace repository.create() with upsertByUserId() in createCheckoutSession so that re-checkout after a failed/canceled subscription updates the existing record instead of inserting a duplicate.

## Symptoms

expected: Checkout session is created and user is redirected to Stripe Checkout
actual: API returns 500 error with "Stripe checkout failed: E11000 duplicate key error collection: trade-flow-db.subscriptions index: userId_1 dup key: { userId: \"8mpADxPDNZQvk2zwP48QJAtmxB83\" }"
errors: E11000 duplicate key error — MongoDB unique index violation on subscriptions.userId
reproduction: Click "Unlock full access" button when user already has a subscription record in the database (e.g., from a previous webhook event or failed checkout attempt)
started: After Phase 30 webhook processing creates a subscription record, then Phase 32 checkout flow tries to create another one

## Eliminated

## Evidence

- timestamp: 2026-03-29T00:00:30Z
  checked: subscription-creator.service.ts lines 26-59
  found: The duplicate checkout guard (lines 28-37) only blocks TRIALING and ACTIVE statuses. For any other existing status (INCOMPLETE, CANCELED, PAST_DUE), the guard passes through, then line 59 calls `this.subscriptionRepository.create(subscriptionDto)` which uses `insertOne` -- this hits the unique index `userId_1` because a record already exists for that userId.
  implication: This is the root cause. The code assumes no record exists when the guard passes, but records with INCOMPLETE/CANCELED/PAST_DUE status DO exist.

- timestamp: 2026-03-29T00:00:45Z
  checked: subscription.repository.ts -- available methods
  found: Repository already has `upsertByUserId(userId, dto)` method (line 42) that performs a findOneAndUpdate with upsert:true. This is the safe alternative to `create()`.
  implication: The fix is straightforward -- use upsertByUserId instead of create.

- timestamp: 2026-03-29T00:00:50Z
  checked: stripe-webhook.processor.ts -- webhook handler
  found: handleCheckoutSessionCompleted (line 60) uses upsertByStripeCustomerId, handleSubscriptionUpdated uses upsertByStripeSubscriptionId. All webhook paths use upsert. Only the checkout creation path uses raw insertOne.
  implication: Confirms the pattern -- webhook handlers already handle idempotency via upserts, but the checkout creation path was missed.

## Resolution

root_cause: `SubscriptionCreator.createCheckoutSession()` calls `subscriptionRepository.create()` (which uses `insertOne`) at line 59 after a guard that only checks for ACTIVE/TRIALING status. When a subscription record already exists with INCOMPLETE, CANCELED, or PAST_DUE status, the guard passes but `insertOne` fails with E11000 duplicate key error on the `userId_1` unique index.
fix: Replace `subscriptionRepository.create()` with `subscriptionRepository.upsertByUserId()` so that an existing record is updated (resetting status to INCOMPLETE and setting the new stripeCustomerId) instead of trying to insert a duplicate.
verification: All 46 subscription tests pass. TypeScript compiles cleanly. Unit test updated to assert upsertByUserId instead of create.
files_changed: ["trade-flow-api/src/subscription/services/subscription-creator.service.ts", "trade-flow-api/src/subscription/test/services/subscription-creator.service.spec.ts"]
