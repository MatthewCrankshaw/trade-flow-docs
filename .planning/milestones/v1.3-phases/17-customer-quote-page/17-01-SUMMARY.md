---
phase: 17-customer-quote-page
plan: 01
subsystem: api
tags: [nestjs, mongodb, quote-token, view-tracking]

requires:
  - phase: 16-send-quote-flow
    provides: quote-token entity, repository, public controller, QuoteTokenModule

provides:
  - firstViewedAt field on quote-token entity/DTO/repository
  - updateFirstViewedAt atomic method (update-if-null)
  - findLatestNonRevokedForQuote repository method
  - Enriched expired/revoked error responses with business name
  - viewedAt field on authenticated quote response
  - viewedAt field on frontend Quote type

affects: [17-customer-quote-page, customer-quote-ui]

tech-stack:
  added: []
  patterns:
    - "Atomic update-if-null for first-event recording ($exists: false filter)"
    - "Cross-module repository access via forwardRef and module exports"
    - "Error response enrichment with business context for customer-facing pages"

key-files:
  created: []
  modified:
    - trade-flow-api/src/quote-token/entities/quote-token.entity.ts
    - trade-flow-api/src/quote-token/data-transfer-objects/quote-token.dto.ts
    - trade-flow-api/src/quote-token/repositories/quote-token.repository.ts
    - trade-flow-api/src/quote-token/controllers/public-quote.controller.ts
    - trade-flow-api/src/quote-token/quote-token.module.ts
    - trade-flow-api/src/quote/responses/quote.responses.ts
    - trade-flow-api/src/quote/controllers/quote.controller.ts
    - trade-flow-ui/src/types/quote.ts
    - trade-flow-api/src/quote-token/test/controllers/public-quote.controller.spec.ts

key-decisions:
  - "Used $exists: false filter for atomic first-view-only update (no race conditions)"
  - "Business name lookup in error path wrapped in try/catch to avoid cascading failures"
  - "viewedAt sourced from latest non-revoked token sorted by createdAt descending"

patterns-established:
  - "Atomic update-if-null: use { field: { $exists: false } } filter for first-event recording"

requirements-completed: [DLVR-05]

duration: 4min
completed: 2026-03-20
---

# Phase 17 Plan 01: Backend View Tracking Summary

**firstViewedAt tracking on quote tokens with enriched error responses and viewedAt on authenticated quote API**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-20T19:16:02Z
- **Completed:** 2026-03-20T19:19:36Z
- **Tasks:** 2
- **Files modified:** 9

## Accomplishments
- firstViewedAt field persisted atomically on first public quote view (update-if-null semantics)
- Expired/revoked token error responses enriched with business name for customer-friendly messaging
- Authenticated quote response includes viewedAt from latest non-revoked token
- Frontend Quote type updated with viewedAt optional field
- All 263 existing tests pass with updated test mocks

## Task Commits

Each task was committed atomically:

1. **Task 1: Add firstViewedAt to entity/DTO/repository and update public controller** - `e180977` (feat)
2. **Task 2: Expose viewedAt on authenticated quote response** - `85134dc` (feat, API) + `8187492` (feat, UI)

**Test fix:** `6fb4583` (test - update controller spec for new dependency)

## Files Created/Modified
- `trade-flow-api/src/quote-token/entities/quote-token.entity.ts` - Added firstViewedAt optional Date field
- `trade-flow-api/src/quote-token/data-transfer-objects/quote-token.dto.ts` - Added firstViewedAt optional DateTime field
- `trade-flow-api/src/quote-token/repositories/quote-token.repository.ts` - Added updateFirstViewedAt and findLatestNonRevokedForQuote methods, updated toDto
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` - First view recording, enriched error responses with business name
- `trade-flow-api/src/quote-token/quote-token.module.ts` - Exported QuoteTokenRepository for cross-module access
- `trade-flow-api/src/quote/responses/quote.responses.ts` - Added viewedAt optional string field
- `trade-flow-api/src/quote/controllers/quote.controller.ts` - Injected QuoteTokenRepository, added viewedAt lookup
- `trade-flow-ui/src/types/quote.ts` - Added viewedAt optional string field
- `trade-flow-api/src/quote-token/test/controllers/public-quote.controller.spec.ts` - Updated mocks and assertions

## Decisions Made
- Used `$exists: false` filter for atomic first-view-only update to prevent race conditions on concurrent views
- Business name lookup in expired/revoked error path wrapped in try/catch to avoid cascading failures if quote/business not found
- viewedAt sourced from latest non-revoked token sorted by `createdAt: -1` to reflect the most recent send action

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated existing controller tests for new constructor dependency**
- **Found during:** Verification (post-Task 2)
- **Issue:** PublicQuoteController tests failed because QuoteTokenRepository was added as constructor dependency but not provided in test module
- **Fix:** Added QuoteTokenRepository mock to test providers, updated expired/revoked assertions for new error codes and business name details
- **Files modified:** trade-flow-api/src/quote-token/test/controllers/public-quote.controller.spec.ts
- **Verification:** All 5 controller tests pass, full suite 263/263 pass
- **Committed in:** 6fb4583

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Test update necessary for correctness. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Backend view tracking complete, ready for customer-facing quote page UI (Plan 02)
- viewedAt field available for tradesperson UI "Viewed" badge display
- Business name available in error responses for customer-friendly expired/revoked pages

---
*Phase: 17-customer-quote-page*
*Completed: 2026-03-20*
