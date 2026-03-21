---
phase: 19-customer-response
plan: 01
subsystem: api
tags: [nestjs, email, quote-response, notification, public-api]

# Dependency graph
requires:
  - phase: 17-customer-quote-page
    provides: PublicQuoteController, QuoteSessionAuthGuard, PublicQuoteRetriever, public quote page
  - phase: 18-quote-email-sending
    provides: EmailSenderService, QuoteEmailRenderer, Maizzle inlining pattern, Resend integration
provides:
  - POST /v1/public/quote/:token/accept endpoint
  - POST /v1/public/quote/:token/decline endpoint
  - QuoteResponseHandler orchestration service
  - NotificationEmailRenderer for tradesperson notifications
  - notification-email.html template
  - declineReason field persisted end-to-end (entity/DTO/repository/public response)
  - publicTransition method with idempotency on QuoteTransitionService
  - QUOTE_EXPIRED error code for expired quote validation
affects: [19-customer-response, quote-management, email-notifications]

# Tech tracking
tech-stack:
  added: []
  patterns: [public-endpoint-mutation-pattern, failure-tolerant-email-notification, idempotent-status-transition]

key-files:
  created:
    - trade-flow-api/src/quote-token/services/quote-response-handler.service.ts
    - trade-flow-api/src/email/services/notification-email-renderer.service.ts
    - trade-flow-api/src/email/templates/notification-email.html
    - trade-flow-api/src/quote-token/requests/decline-quote.request.ts
  modified:
    - trade-flow-api/src/quote/entities/quote.entity.ts
    - trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts
    - trade-flow-api/src/quote/repositories/quote.repository.ts
    - trade-flow-api/src/quote/services/quote-transition.service.ts
    - trade-flow-api/src/quote-token/responses/public-quote.response.ts
    - trade-flow-api/src/quote-token/services/public-quote-retriever.service.ts
    - trade-flow-api/src/quote-token/controllers/public-quote.controller.ts
    - trade-flow-api/src/quote-token/quote-token.module.ts
    - trade-flow-api/src/email/email.module.ts
    - trade-flow-api/src/business/repositories/business-user.repository.ts
    - trade-flow-api/src/business/business.module.ts
    - trade-flow-api/src/quote/quote.module.ts
    - trade-flow-api/src/core/errors/error-codes.enum.ts

key-decisions:
  - "UserRepository.findById returns null (not findByIdOrFail), so QuoteResponseHandler gracefully exits notification if user not found"
  - "QuoteTransitionService exported from QuoteModule for reuse by QuoteResponseHandler in QuoteTokenModule"
  - "UserModule imported with forwardRef in QuoteTokenModule to resolve circular dependency"

patterns-established:
  - "Public mutation endpoints use stricter rate limiting (10/min) vs GET endpoints (60/min)"
  - "Notification emails are failure-tolerant: try/catch around email send, log error but do not throw"
  - "publicTransition method handles idempotency by returning existing quote when status matches target"

requirements-completed: [RESP-02, RESP-03, RESP-04, AUTO-02, AUTO-03, NOTF-01, NOTF-02]

# Metrics
duration: 4min
completed: 2026-03-21
---

# Phase 19 Plan 01: Customer Response Backend Summary

**Accept/decline POST endpoints on public controller with QuoteResponseHandler orchestrating status transitions, decline reason persistence, and failure-tolerant tradesperson notification emails**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-21T18:26:25Z
- **Completed:** 2026-03-21T18:30:11Z
- **Tasks:** 2
- **Files modified:** 17

## Accomplishments
- Two new POST endpoints (accept/decline) on PublicQuoteController with session auth and rate limiting
- QuoteResponseHandler orchestrates: validate expiry -> transition status -> send notification email -> return updated public response
- declineReason field persisted end-to-end through entity, DTO, repository, and public response
- publicTransition method on QuoteTransitionService with idempotent double-click handling
- Notification email template and renderer for tradesperson notifications with business branding
- Expired quote validation prevents responses on quotes past validUntil date

## Task Commits

Each task was committed atomically:

1. **Task 1: Data layer changes** - `73bada1` (feat)
2. **Task 2: Notification email, response handler, controller endpoints, module wiring** - `8e54f82` (feat)

## Files Created/Modified
- `trade-flow-api/src/quote-token/services/quote-response-handler.service.ts` - Orchestration service for accept/decline flow
- `trade-flow-api/src/email/services/notification-email-renderer.service.ts` - HTML notification email renderer with Maizzle inlining
- `trade-flow-api/src/email/templates/notification-email.html` - Lighter notification template with business header, message, CTA button
- `trade-flow-api/src/quote-token/requests/decline-quote.request.ts` - Request validation for decline with optional reason (max 500 chars)
- `trade-flow-api/src/quote/entities/quote.entity.ts` - Added declineReason field
- `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts` - Added declineReason field
- `trade-flow-api/src/quote/repositories/quote.repository.ts` - Added declineReason to $set, toDto, toEntity
- `trade-flow-api/src/quote/services/quote-transition.service.ts` - Added publicTransition method with idempotency
- `trade-flow-api/src/quote-token/responses/public-quote.response.ts` - Added acceptedAt, rejectedAt, declineReason
- `trade-flow-api/src/quote-token/services/public-quote-retriever.service.ts` - Added timestamp/reason mapping
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` - Added accept/decline POST endpoints
- `trade-flow-api/src/quote-token/quote-token.module.ts` - Added EmailModule, UserModule imports and QuoteResponseHandler provider
- `trade-flow-api/src/email/email.module.ts` - Added NotificationEmailRenderer to providers/exports
- `trade-flow-api/src/business/repositories/business-user.repository.ts` - Added findByBusinessId method
- `trade-flow-api/src/business/business.module.ts` - Exported BusinessUserRepository
- `trade-flow-api/src/quote/quote.module.ts` - Exported QuoteTransitionService
- `trade-flow-api/src/core/errors/error-codes.enum.ts` - Added QUOTE_EXPIRED error code

## Decisions Made
- UserRepository uses `findById` (nullable) not `findByIdOrFail`, so QuoteResponseHandler gracefully exits notification if user not found
- QuoteTransitionService exported from QuoteModule for public transition reuse
- UserModule imported with forwardRef in QuoteTokenModule to handle circular dependency chain

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] UserRepository.findByIdOrFail does not exist**
- **Found during:** Task 2 (QuoteResponseHandler)
- **Issue:** Plan specified `this.userRepository.findByIdOrFail()` but UserRepository only has `findById()` returning nullable
- **Fix:** Used `findById()` with null check and early return from notification (graceful degradation)
- **Files modified:** trade-flow-api/src/quote-token/services/quote-response-handler.service.ts
- **Verification:** TypeScript compiles without errors
- **Committed in:** 8e54f82 (Task 2 commit)

**2. [Rule 3 - Blocking] QuoteTransitionService not exported from QuoteModule**
- **Found during:** Task 2 (QuoteResponseHandler needs QuoteTransitionService)
- **Issue:** QuoteModule only exported QuoteRepository, not QuoteTransitionService needed by QuoteResponseHandler
- **Fix:** Added QuoteTransitionService to QuoteModule exports array
- **Files modified:** trade-flow-api/src/quote/quote.module.ts
- **Verification:** TypeScript compiles, NestJS DI graph resolves
- **Committed in:** 8e54f82 (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 blocking)
**Impact on plan:** Both fixes necessary for correct compilation and runtime DI. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Backend endpoints ready for Plan 02 (frontend accept/decline UI)
- PublicQuoteResponse now includes acceptedAt, rejectedAt, declineReason for frontend consumption
- Notification emails will be sent to tradesperson on customer response

---
*Phase: 19-customer-response*
*Completed: 2026-03-21*
