---
status: awaiting_human_verify
trigger: "UI displays 'Active' instead of 'Trial' when trialEnd is set and in the future"
created: 2026-03-30T00:00:00.000Z
updated: 2026-03-30T00:00:00.000Z
---

## Current Focus

hypothesis: CONFIRMED - handleInvoicePaymentSucceeded hard-codes status to ACTIVE, overwriting the correct "trialing" status
test: n/a - root cause confirmed via code reading
expecting: n/a
next_action: Fix handleInvoicePaymentSucceeded to use stripeSubscription.status instead of hard-coded ACTIVE

## Symptoms

expected: UI should show subscription status as "Trial" or "Free Trial" when trialEnd is set and in the future
actual: UI shows subscription status as "Active", ignoring the trialEnd field
errors: No errors - just incorrect display
reproduction: Subscribe with a trial period, observe the subscription status in the UI
started: Likely since trial support was added

## Eliminated

- hypothesis: UI does not have trial-aware display logic
  evidence: SubscriptionStatusCard.tsx line 44 checks for status === "trialing" and displays "Trial" badge correctly. TrialChip.tsx also checks for "trialing" status. The UI logic is correct.
  timestamp: 2026-03-30T00:01:00.000Z

- hypothesis: API does not return trialEnd field
  evidence: subscription.response.ts and subscription.dto.ts include trialEnd. The webhook processor extracts trial_end from Stripe correctly in extractSubscriptionDates().
  timestamp: 2026-03-30T00:01:00.000Z

## Evidence

- timestamp: 2026-03-30T00:01:00.000Z
  checked: SubscriptionStatusCard.tsx getStatusDisplay() function
  found: Line 44 checks subscription.status === "trialing" and shows Trial badge. The UI logic is correct and would display Trial if the status were "trialing".
  implication: The problem is upstream - the status stored in DB is wrong.

- timestamp: 2026-03-30T00:02:00.000Z
  checked: stripe-webhook.processor.ts handleCheckoutSessionCompleted()
  found: Line 72 correctly passes stripeSubscription.status as SubscriptionStatus (which would be "trialing" for trial subs).
  implication: Status is set correctly during checkout completion.

- timestamp: 2026-03-30T00:03:00.000Z
  checked: stripe-webhook.processor.ts handleInvoicePaymentSucceeded()
  found: Line 165 HARD-CODES status to SubscriptionStatus.ACTIVE, ignoring the actual Stripe subscription status. Line 161 retrieves the Stripe subscription (which has the real status) but doesn't use it for the status field.
  implication: When Stripe fires invoice.payment_succeeded for the $0 trial invoice, this handler overwrites the correct "trialing" status with "active". This is the root cause.

- timestamp: 2026-03-30T00:03:30.000Z
  checked: Stripe event ordering for trial subscriptions
  found: When a checkout with trial_period_days completes, Stripe fires checkout.session.completed (sets "trialing"), then invoice.payment_succeeded for the $0 invoice (overwrites to "active"). The last writer wins.
  implication: Race condition where invoice handler clobbers the correct trial status.

## Resolution

root_cause: In stripe-webhook.processor.ts, handleInvoicePaymentSucceeded() hard-codes status to SubscriptionStatus.ACTIVE (line 165) instead of using the actual Stripe subscription status. When a trial subscription is created, Stripe fires invoice.payment_succeeded for the $0 trial invoice, and this handler overwrites the correct "trialing" status with "active".
fix: Change line 165 to use stripeSubscription.status (already retrieved on line 161) instead of hard-coded SubscriptionStatus.ACTIVE
verification: All 9 tests pass (including new regression test for trialing status preservation). Typecheck clean.
files_changed: [trade-flow-api/src/worker/processors/stripe-webhook.processor.ts, trade-flow-api/src/subscription/test/services/stripe-webhook.processor.spec.ts]
