---
phase: 35-no-card-trial-api-endpoint
plan: 01
subsystem: api
tags: [stripe, nestjs, subscription, trial]

# Dependency graph
requires:
  - phase: 29-subscription-module-foundation
    provides: SubscriptionRepository, ISubscriptionDto, SubscriptionStatus enum, STRIPE_CLIENT provider
  - phase: 30-stripe-checkout-and-webhooks
    provides: SubscriptionController, SubscriptionModule, upsertByStripeCustomerId repo method
provides:
  - POST /v1/subscription/trial endpoint for no-card 30-day Stripe trial
  - SubscriptionTrialCreator service with duplicate guard
affects: [35-02 (webhook compatibility), 37 (onboarding wizard trial activation)]

# Tech tracking
tech-stack:
  added: []
  patterns: [Stripe v21+ item-level current_period_end for subscription creation]

key-files:
  created:
    - trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts
    - trade-flow-api/src/subscription/test/services/subscription-trial-creator.service.spec.ts
  modified:
    - trade-flow-api/src/subscription/controllers/subscription.controller.ts
    - trade-flow-api/src/subscription/subscription.module.ts

key-decisions:
  - "Used SUBSCRIPTION_ALREADY_ACTIVE error code for trial duplicate guard (reuses existing error mapping)"
  - "Used DateTime.fromSeconds for Stripe timestamps (Luxon standard per CLAUDE.md)"
  - "Followed Stripe v21+ pattern: current_period_end at item level, not subscription level"

patterns-established:
  - "Trial creation pattern: findByUserId guard -> Stripe customer.create -> subscription.create -> upsertByStripeCustomerId"

requirements-completed: [TRIAL-01, TRIAL-04]

# Metrics
duration: 5min
completed: 2026-04-02
---

# Phase 35 Plan 01: No-Card Trial API Endpoint Summary

**POST /v1/subscription/trial endpoint with SubscriptionTrialCreator service creating 30-day Stripe trial (no card required, auto-cancel on missing payment method)**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-02T11:04:23Z
- **Completed:** 2026-04-02T11:09:00Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- SubscriptionTrialCreator service creates Stripe customer + subscription with 30-day trial and missing_payment_method: cancel
- Duplicate guard rejects with HTTP 422 (InvalidRequestError) if any subscription record exists for the user
- POST /v1/subscription/trial wired into JwtAuthGuard-protected controller, returns standard IResponse format
- 6 unit tests covering success path, duplicate rejection, Stripe params, and PRICE_ID injection
- Full test suite green (376 tests pass)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create SubscriptionTrialCreator service with unit tests** - `3cb5f86` (feat) -- TDD: test + implementation
2. **Task 2: Wire trial endpoint into controller and module** - `c01e0b2` (feat)

## Files Created/Modified
- `trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts` - Trial creation business logic with Stripe customer/subscription creation, duplicate guard, and local record upsert
- `trade-flow-api/src/subscription/test/services/subscription-trial-creator.service.spec.ts` - 6 unit tests for SubscriptionTrialCreator
- `trade-flow-api/src/subscription/controllers/subscription.controller.ts` - Added @Post("trial") route with HttpStatus.CREATED
- `trade-flow-api/src/subscription/subscription.module.ts` - Registered SubscriptionTrialCreator as provider

## Decisions Made
- Used `ErrorCodes.SUBSCRIPTION_ALREADY_ACTIVE` for the duplicate guard rather than creating a new error code -- the existing mapping provides appropriate messaging and the semantics match (user already has a subscription)
- Used `DateTime.fromSeconds()` instead of `new Date(ts * 1000)` per Luxon standard enforced in CLAUDE.md
- Followed Stripe v21+ pattern where `current_period_end` is at item level (`subscription.items.data[0].current_period_end`), consistent with existing webhook processor and updater service

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Stripe v21+ current_period_end location**
- **Found during:** Task 1 (SubscriptionTrialCreator implementation)
- **Issue:** Plan specified `subscription.current_period_end` but Stripe v21+ SDK types place this at item level
- **Fix:** Used `subscription.items.data[0]?.current_period_end` pattern matching existing codebase (subscription-updater.service.ts, stripe-webhook.processor.ts)
- **Files modified:** subscription-trial-creator.service.ts, subscription-trial-creator.service.spec.ts
- **Verification:** TypeScript compilation passes, tests pass
- **Committed in:** 3cb5f86

**2. [Rule 1 - Bug] DateTime.fromSeconds instead of new Date()**
- **Found during:** Task 1 (SubscriptionTrialCreator implementation)
- **Issue:** Plan specified `new Date(subscription.current_period_end * 1000)` but CLAUDE.md mandates Luxon DateTime in all DTOs
- **Fix:** Used `DateTime.fromSeconds(timestamp, { zone: "utc" })` per Stripe timestamp convention in CLAUDE.md
- **Files modified:** subscription-trial-creator.service.ts
- **Verification:** TypeScript compilation passes, DTO types satisfied
- **Committed in:** 3cb5f86

**3. [Rule 3 - Blocking] Repository findByUserId already existed**
- **Found during:** Task 1 (reading existing code)
- **Issue:** Plan instructed adding findByUserId to SubscriptionRepository but it already existed from Phase 29
- **Fix:** Skipped adding duplicate method, used existing one directly
- **Files modified:** None (no change needed)
- **Committed in:** N/A

---

**Total deviations:** 3 auto-fixed (2 bug fixes, 1 blocking)
**Impact on plan:** All deviations ensured correctness and codebase consistency. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required. STRIPE_PRICE_ID environment variable already configured from Phase 30.

## Next Phase Readiness
- Trial endpoint ready for Phase 37 onboarding wizard integration
- Plan 02 (webhook compatibility) can proceed to ensure `customer.subscription.created` webhook handles both Checkout-created and API-created subscriptions

---
*Phase: 35-no-card-trial-api-endpoint*
*Completed: 2026-04-02*
