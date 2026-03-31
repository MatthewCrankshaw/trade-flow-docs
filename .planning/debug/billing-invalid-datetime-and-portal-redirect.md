---
status: awaiting_human_verify
trigger: "Billing page shows 'Invalid DateTime' for renewal date and Manage Billing button redirects to dashboard instead of Stripe portal"
created: 2026-03-30T00:00:00Z
updated: 2026-03-30T00:00:00Z
---

## Current Focus

hypothesis: Both remaining issues fixed -- webhook handlers now populate date fields, portal mutation resets before each call
test: Typecheck + lint + full test suite
expecting: All pass
next_action: Await human verification that billing page shows dates and portal button works repeatedly

## Symptoms

expected: "Active -- renews 1 May 2026" (human-readable date) when subscribed/trial; clicking Manage Billing opens Stripe portal every time
actual: API response has no currentPeriodEnd/trialEnd/canceledAt fields at all; Manage Billing only works once per page load (second click redirects to dashboard)
errors: No console errors -- just missing date fields and unexpected redirect
reproduction: Go to Settings > Billing as subscribed user on trial; click Manage Billing, browser back, click again
started: After Phase 34 Luxon DateTime migration (2026-03-30)

## Eliminated

- hypothesis: Luxon DateTime serialization produces non-ISO output
  evidence: Verified via Node.js that DateTime.fromJSDate(date).toJSON() produces valid ISO strings like "2026-05-01T00:00:00.000Z" and JSON.stringify correctly calls toJSON()
  timestamp: 2026-03-30

- hypothesis: DateTime.fromISO() can't parse Luxon-generated ISO strings
  evidence: Verified via Node.js that DateTime.fromISO("2026-05-01T00:00:00.000Z") produces valid DateTime with correct formatting
  timestamp: 2026-03-30

- hypothesis: class-transformer or interceptors modify serialization
  evidence: No ClassSerializerInterceptor or APP_INTERCEPTOR in main.ts or any module. Only global ValidationPipe is applied.
  timestamp: 2026-03-30

- hypothesis: Portal URL field name mismatch (portalUrl vs url)
  evidence: Already fixed in previous iteration -- usePortalSession.ts now uses result.url matching API's { url: string }
  timestamp: 2026-03-30

## Evidence

- timestamp: 2026-03-30
  checked: API portal endpoint return type
  found: Controller line 72 returns IResponse<{ url: string }>. Service createPortalSession returns { url: session.url }. Field name is "url".
  implication: API response field is "url", not "portalUrl"

- timestamp: 2026-03-30
  checked: UI portal mutation expected type
  found: subscriptionApi.ts line 54 declares return type { portalUrl: string }. usePortalSession.ts line 11 accesses result.portalUrl.
  implication: UI expects "portalUrl" but API sends "url" -- field name mismatch causes result.portalUrl to be undefined

- timestamp: 2026-03-30
  checked: Effect of window.location.href = undefined
  found: Setting location.href to undefined coerces to string "undefined", navigating to relative URL that router catches and redirects to dashboard
  implication: This explains the redirect-to-dashboard behavior with 201 response

- timestamp: 2026-03-30
  checked: SubscriptionStatusCard getStatusDisplay logic
  found: For active status with cancelAtPeriodEnd=false, calls formatBillingDate(subscription.currentPeriodEnd!) which calls DateTime.fromISO(value)
  implication: If currentPeriodEnd is undefined/null, DateTime.fromISO(undefined) produces invalid DateTime -> "Invalid DateTime" text

- timestamp: 2026-03-30
  checked: Whether currentPeriodEnd could be missing
  found: checkout.session.completed handler does NOT set currentPeriodEnd. Only customer.subscription.updated sets it. If that webhook hasn't fired yet, or if Stripe item data lacks current_period_end, the field would be absent.
  implication: Data gap is possible -- component lacks null guard

- timestamp: 2026-03-30
  checked: Luxon DateTime.toJSON() with invalid input (fromJSDate of null/undefined/string)
  found: fromJSDate with non-Date input creates invalid DateTime whose toJSON() returns null. API would send currentPeriodEnd: null.
  implication: If MongoDB has corrupt/missing date data, API sends null, UI parses null as invalid DateTime

- timestamp: 2026-03-30
  checked: All webhook handlers for date field population
  found: |
    (a) checkout.session.completed: sets status=TRIALING, stripeSubscriptionId, stripeLatestCheckoutSessionId -- NO date fields
    (b) customer.subscription.updated: sets currentPeriodEnd (from items.data[0].current_period_end), trialEnd, cancelAtPeriodEnd -- ONLY handler that sets dates
    (c) customer.subscription.deleted: sets status=CANCELED, canceledAt -- no currentPeriodEnd/trialEnd
    (d) invoice.payment_succeeded: sets status=ACTIVE only -- NO date fields
    (e) invoice.payment_failed: sets status=PAST_DUE only -- NO date fields
  implication: If customer.subscription.updated doesn't fire or fires before checkout.session.completed creates the record, dates are never populated. User's actual data confirms: status=active, no dates.

- timestamp: 2026-03-30
  checked: Portal redirect-to-dashboard on second click (after bfcache restore)
  found: After window.location.href navigates to Stripe portal and user presses back, bfcache restores page with stale RTK Query mutation state. Without resetting, mutation may not cleanly re-fire.
  implication: Need mutation reset() before each new portal call

## Resolution

root_cause: |
  Issue 1 (Missing date fields): The checkout.session.completed and invoice.payment_succeeded webhook handlers never populated currentPeriodEnd, trialEnd, or cancelAtPeriodEnd. The only handler that set dates was customer.subscription.updated, but if it didn't fire (or fired before the local record existed), dates remained absent in MongoDB. The user's API response confirmed: status=active with zero date fields.

  Issue 2 (Portal redirect on second click): After navigating to Stripe portal and pressing browser back, bfcache restores the page with stale RTK Query mutation state. Without resetting the mutation before each call, the second invocation may not properly fire or may resolve with stale data.

fix: |
  Issue 1 (API): Injected Stripe client into StripeWebhookProcessor. In checkout.session.completed, now retrieves the full subscription from Stripe API to get authoritative status and dates (currentPeriodEnd, trialEnd, cancelAtPeriodEnd). In invoice.payment_succeeded, also retrieves the subscription to sync dates alongside the status update. Extracted shared helper extractSubscriptionDates() and refactored customer.subscription.updated to use it too.

  Issue 1 (UI defensive): Updated SubscriptionStatusCard to gracefully handle missing date fields -- shows "Active subscription" instead of "Active -- renews N/A", "Trial period active" instead of "Trial -- null days remaining", and "Cancellation pending" instead of "Cancels on N/A".

  Issue 2 (UI): Added reset() call in usePortalSession before each createPortal() invocation to clear stale mutation state. Also added a defensive check for missing result.url before setting window.location.href.

verification: |
  - TypeScript typecheck passes on both API and UI repos
  - Full API test suite: 364 tests pass across 55 suites, zero failures
  - Webhook processor tests updated to verify Stripe subscription retrieval and date field population
  - Pre-existing lint issues only (API ESLint config crash, UI e2e test lint errors) -- no new issues

files_changed:
  - trade-flow-api/src/worker/processors/stripe-webhook.processor.ts
  - trade-flow-api/src/subscription/test/services/stripe-webhook.processor.spec.ts
  - trade-flow-ui/src/features/subscription/hooks/usePortalSession.ts
  - trade-flow-ui/src/features/subscription/components/SubscriptionStatusCard.tsx
  - trade-flow-ui/src/features/subscription/api/subscriptionApi.ts
  - trade-flow-ui/src/features/subscription/utils/subscription-helpers.ts
