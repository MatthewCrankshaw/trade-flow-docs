# Quick Task: Remove Type Assertions and Use Custom Errors in Stripe Webhook Processor

**Commit:** `e399c93` in trade-flow-api
**Duration:** ~5 minutes
**Tasks:** 5/5 completed

## One-liner

Replaced `as` type casts and generic `Error` throws with runtime-validated mapping utility and domain-specific error classes in stripe-webhook processor.

## Changes Made

### Error Codes (Task 1)
- Added 4 new error codes in `ErrorCodes` enum: `WEBHOOK_MISSING_CUSTOMER_ID`, `WEBHOOK_MISSING_SUBSCRIPTION_ID`, `WEBHOOK_SUBSCRIPTION_NOT_FOUND`, `WEBHOOK_SUBSCRIPTION_INCOMPLETE`
- Added corresponding entries in `ERRORS_MAP` with descriptive messages

### New Utility (Task 2)
- Created `toSubscriptionStatus()` in `src/subscription/utilities/to-subscription-status.utility.ts`
- Maps Stripe status strings to `SubscriptionStatus` enum with runtime validation
- Throws `InternalServerError` for unknown statuses

### Processor Refactor (Task 3)
- Replaced 3 `status as SubscriptionStatus` casts with `toSubscriptionStatus()` calls
- Removed `as Stripe.Subscription` cast on subscription ID extraction (unnecessary after typeof narrowing)
- Replaced 8 `throw new Error(...)` with appropriate `InternalServerError` or `ResourceNotFoundError`
- Kept `as` casts on `event.data.object` in switch/case (justified -- Stripe SDK limitation)

### Test Updates (Task 4)
- Updated 3 error assertions from string matching to error class matching (`InternalServerError`, `ResourceNotFoundError`)
- All 9 tests pass

### CLAUDE.md Updates (Task 5)
- Added "Type Assertions" section under TypeScript Standards
- Added type assertion rule to ESLint Rules section

## Files Modified

| File | Change |
|------|--------|
| `src/core/errors/error-codes.enum.ts` | Added 4 webhook error codes |
| `src/core/errors/errors-map.constant.ts` | Added 4 error map entries |
| `src/subscription/utilities/to-subscription-status.utility.ts` | NEW -- status mapping utility |
| `src/worker/processors/stripe-webhook.processor.ts` | Replaced casts and generic errors |
| `src/subscription/test/services/stripe-webhook.processor.spec.ts` | Updated error assertions |
| `CLAUDE.md` | Added type assertion guidelines |

## Verification

- Tests: 9 passed, 0 failed
- TypeScript typecheck: clean (no errors)

## Deviations from Plan

None -- plan executed exactly as written.

## Self-Check: PASSED
