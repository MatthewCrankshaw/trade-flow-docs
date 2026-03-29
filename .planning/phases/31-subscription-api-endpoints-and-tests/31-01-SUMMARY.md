---
phase: 31-subscription-api-endpoints-and-tests
plan: 01
subsystem: api
tags: [stripe, nestjs, subscription, guard, billing-portal, cancel]

# Dependency graph
requires:
  - phase: 29-subscription-module-foundation
    provides: SubscriptionModule with repository, entity, Stripe provider, checkout endpoint
  - phase: 30-stripe-checkout-and-webhooks
    provides: Webhook controller, queue producer, stripe-webhook processor
provides:
  - GET /v1/subscription endpoint returning subscription status for authenticated user
  - DELETE /v1/subscription endpoint cancelling subscription at period end via Stripe API
  - POST /v1/subscription/portal endpoint returning Stripe Billing Portal URL
  - Global SubscriptionGuard enforcing write-operation gating on all routes
  - "@SkipSubscriptionCheck() decorator for route exemptions"
  - SubscriptionRetriever.findByUser, SubscriptionUpdater.cancelAtPeriodEnd, SubscriptionCreator.createPortalSession
affects: [31-02-subscription-tests, 32-subscription-ui]

# Tech tracking
tech-stack:
  added: []
  patterns: [global-guard-with-decorator-bypass, item-level-period-end-for-stripe-v21]

key-files:
  created:
    - trade-flow-api/src/subscription/services/subscription-updater.service.ts
    - trade-flow-api/src/subscription/guards/subscription.guard.ts
    - trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts
  modified:
    - trade-flow-api/src/subscription/services/subscription-retriever.service.ts
    - trade-flow-api/src/subscription/services/subscription-creator.service.ts
    - trade-flow-api/src/subscription/controllers/subscription.controller.ts
    - trade-flow-api/src/subscription/subscription.module.ts
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
    - trade-flow-api/package.json

key-decisions:
  - "Stripe v21 current_period_end accessed at items.data[0] level, not subscription level"
  - "SUBSCRIPTION_NOT_FOUND, SUBSCRIPTION_NO_ACTIVE, SUBSCRIPTION_REQUIRED error codes added to ErrorCodes enum"
  - "SubscriptionGuard checks supportRoles.length for support user bypass (DtoCollection has .length getter)"

patterns-established:
  - "Global guard with @SkipSubscriptionCheck() decorator bypass: class-level on WebhookController, method-level on subscription routes"
  - "Stripe v21 item-level period access: subscription.items.data[0].current_period_end instead of subscription.current_period_end"

requirements-completed: [BILL-01, BILL-02, BILL-03]

# Metrics
duration: 5min
completed: 2026-03-29
---

# Phase 31 Plan 01: Subscription API Endpoints and Guard Summary

**Three subscription management endpoints (GET status, DELETE cancel, POST portal) with global SubscriptionGuard blocking writes for unsubscribed users**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-29T15:56:52Z
- **Completed:** 2026-03-29T16:02:45Z
- **Tasks:** 2
- **Files modified:** 11

## Accomplishments
- GET /v1/subscription returns subscription status with 404 on missing record
- DELETE /v1/subscription calls Stripe subscriptions.update with cancel_at_period_end: true (not cancel)
- POST /v1/subscription/portal returns Stripe Billing Portal URL with /settings?tab=billing return_url
- Global SubscriptionGuard registered as APP_GUARD blocks write operations for users without trialing/active subscription
- @SkipSubscriptionCheck() decorator exempts subscription management routes, checkout, verify-session, and WebhookController
- Support users bypass guard via supportRoles.length check
- GET/HEAD requests always allowed regardless of subscription status
- All 313 existing tests pass

## Task Commits

Each task was committed atomically:

1. **Task 1: Create SubscriptionRetriever findByUser, SubscriptionUpdater cancel, SubscriptionCreator portal** - `22f8375` (feat)
2. **Task 2: Create SubscriptionGuard, decorator, controller routes, module registration** - `e5a2d9c` (feat)

## Files Created/Modified
- `trade-flow-api/src/subscription/services/subscription-retriever.service.ts` - Added findByUser method returning 404 on missing subscription
- `trade-flow-api/src/subscription/services/subscription-updater.service.ts` - New service with cancelAtPeriodEnd using Stripe subscriptions.update
- `trade-flow-api/src/subscription/services/subscription-creator.service.ts` - Added createPortalSession returning Stripe Billing Portal URL
- `trade-flow-api/src/subscription/guards/subscription.guard.ts` - Global guard enforcing subscription status on write operations
- `trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts` - SetMetadata decorator for guard bypass
- `trade-flow-api/src/subscription/controllers/subscription.controller.ts` - Added GET, DELETE, POST portal routes with @SkipSubscriptionCheck
- `trade-flow-api/src/subscription/subscription.module.ts` - Added SubscriptionUpdater, SubscriptionGuard to providers/exports
- `trade-flow-api/src/app.module.ts` - Registered SubscriptionGuard as APP_GUARD
- `trade-flow-api/src/core/errors/error-codes.enum.ts` - Added SUBSCRIPTION_NOT_FOUND, SUBSCRIPTION_NO_ACTIVE, SUBSCRIPTION_REQUIRED
- `trade-flow-api/src/core/errors/errors-map.constant.ts` - Added error code message mappings
- `trade-flow-api/package.json` - Added @subscription and @subscription-test jest moduleNameMapper aliases

## Decisions Made
- Stripe v21 moved current_period_end to item level (subscription.items.data[0].current_period_end) -- aligned with existing webhook processor pattern
- Added three new ErrorCodes (SUBSCRIPTION_NOT_FOUND, SUBSCRIPTION_NO_ACTIVE, SUBSCRIPTION_REQUIRED) since plan referenced non-existent INVALID_REQUEST/FORBIDDEN codes
- DtoCollection.length getter used for support role check (matches actual IUserDto.supportRoles type)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed Stripe v21 current_period_end access**
- **Found during:** Task 1 (SubscriptionUpdater)
- **Issue:** Plan referenced `updated.current_period_end` but Stripe v21 removed this field from Subscription top-level; it exists at `items.data[0].current_period_end`
- **Fix:** Used `updated.items.data[0]?.current_period_end` matching existing webhook processor pattern
- **Files modified:** trade-flow-api/src/subscription/services/subscription-updater.service.ts
- **Verification:** TypeScript strict check passes
- **Committed in:** e5a2d9c (Task 2 commit, included with updater fix)

**2. [Rule 3 - Blocking] Added missing ErrorCodes for subscription operations**
- **Found during:** Task 1
- **Issue:** Plan referenced ErrorCodes.INVALID_REQUEST and ErrorCodes.FORBIDDEN which don't exist in the enum
- **Fix:** Added SUBSCRIPTION_NOT_FOUND, SUBSCRIPTION_NO_ACTIVE, SUBSCRIPTION_REQUIRED to ErrorCodes enum and ERRORS_MAP
- **Files modified:** trade-flow-api/src/core/errors/error-codes.enum.ts, trade-flow-api/src/core/errors/errors-map.constant.ts
- **Verification:** TypeScript compiles, error constructors resolve correctly
- **Committed in:** 22f8375 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 blocking)
**Impact on plan:** Both fixes necessary for correctness. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - all endpoints are fully wired to Stripe SDK and local repository.

## Next Phase Readiness
- All three service methods ready for unit testing in Plan 02
- SubscriptionGuard ready for unit testing in Plan 02
- Jest @subscription path alias configured for test imports
- 313 existing tests continue to pass

## Self-Check: PASSED

- All 3 created files exist on disk
- Both commit hashes (22f8375, e5a2d9c) found in git log
- 313/313 tests passing
- TypeScript strict check has zero subscription-related errors

---
*Phase: 31-subscription-api-endpoints-and-tests*
*Completed: 2026-03-29*
