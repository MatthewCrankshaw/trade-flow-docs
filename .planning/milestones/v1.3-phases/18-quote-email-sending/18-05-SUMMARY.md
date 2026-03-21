---
phase: 18-quote-email-sending
plan: 05
subsystem: api
tags: [nestjs, mongodb, quote-settings, separation-of-concerns]

# Dependency graph
requires:
  - phase: 18-quote-email-sending
    provides: QuoteEmailSender service with business.quoteEmailSubject/quoteEmailBody fields
provides:
  - QuoteSettingsModule with entity, repository, retriever, updater, controller
  - GET /v1/business/:businessId/quote-settings endpoint
  - PATCH /v1/business/:businessId/quote-settings endpoint
  - QuoteSettingsRetriever exported for cross-module use
affects: [18-quote-email-sending, trade-flow-ui quote settings UI]

# Tech tracking
tech-stack:
  added: []
  patterns: [upsert-with-setOnInsert for settings-style entities]

key-files:
  created:
    - trade-flow-api/src/quote-settings/quote-settings.module.ts
    - trade-flow-api/src/quote-settings/entities/quote-settings.entity.ts
    - trade-flow-api/src/quote-settings/data-transfer-objects/quote-settings.dto.ts
    - trade-flow-api/src/quote-settings/repositories/quote-settings.repository.ts
    - trade-flow-api/src/quote-settings/services/quote-settings-retriever.service.ts
    - trade-flow-api/src/quote-settings/services/quote-settings-updater.service.ts
    - trade-flow-api/src/quote-settings/controllers/quote-settings.controller.ts
    - trade-flow-api/src/quote-settings/requests/update-quote-settings.request.ts
    - trade-flow-api/src/quote-settings/responses/quote-settings.response.ts
    - trade-flow-api/src/quote-settings/policies/quote-settings.policy.ts
  modified:
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json
    - trade-flow-api/src/business/entities/business.entity.ts
    - trade-flow-api/src/business/data-transfer-objects/business.dto.ts
    - trade-flow-api/src/business/repositories/business.repository.ts
    - trade-flow-api/src/business/services/business-updater.service.ts
    - trade-flow-api/src/business/responses/business.response.ts
    - trade-flow-api/src/business/controllers/business.controller.ts
    - trade-flow-api/src/business/request/update-business.request.ts
    - trade-flow-api/src/quote/services/quote-email-sender.service.ts
    - trade-flow-api/src/quote/quote.module.ts
    - trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts
    - trade-flow-api/openapi.yaml

key-decisions:
  - "Upsert pattern for quote settings (create-on-first-update via $setOnInsert)"
  - "Business access verification via BusinessRetriever.findByIdOrFail in updater (reuses existing auth)"
  - "GET returns empty defaults when no settings exist (no 404)"

patterns-established:
  - "Settings-style entity with upsert: use findOneAndUpdate with $setOnInsert for first-create, $set for updates"

requirements-completed: [DLVR-02, DLVR-03]

# Metrics
duration: 4min
completed: 2026-03-21
---

# Phase 18 Plan 05: Quote Settings Separation Summary

**Separate QuoteSettings module with GET/PATCH API, quote_settings MongoDB collection, and Business entity cleaned of email template fields**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-21T09:24:53Z
- **Completed:** 2026-03-21T09:29:20Z
- **Tasks:** 2
- **Files modified:** 24

## Accomplishments
- New QuoteSettings module with full CRUD (entity, DTO, repository, retriever, updater, policy, controller, request, response)
- GET/PATCH endpoints at /v1/business/:businessId/quote-settings with upsert semantics
- Business entity cleaned of quoteEmailSubject and quoteEmailBody fields across all layers
- QuoteEmailSender rewired with QuoteSettingsRetriever injection
- All 284 existing tests passing, typecheck clean

## Task Commits

Each task was committed atomically:

1. **Task 1: Create QuoteSettings module with entity, DTO, repository, services, policy, and controller** - `50e4a87` (feat)
2. **Task 2: Remove quoteEmail fields from Business entity and rewire QuoteEmailSender** - `f930dac` (feat)

## Files Created/Modified
- `src/quote-settings/quote-settings.module.ts` - NestJS module importing CoreModule and BusinessModule
- `src/quote-settings/entities/quote-settings.entity.ts` - MongoDB entity with businessId, quoteEmailSubject, quoteEmailBody
- `src/quote-settings/data-transfer-objects/quote-settings.dto.ts` - DTO interface extending IBaseResourceDto
- `src/quote-settings/repositories/quote-settings.repository.ts` - Repository with findByBusinessId and upsert ($setOnInsert pattern)
- `src/quote-settings/services/quote-settings-retriever.service.ts` - Read service delegating to repository
- `src/quote-settings/services/quote-settings-updater.service.ts` - Write service with business access verification
- `src/quote-settings/controllers/quote-settings.controller.ts` - GET and PATCH endpoints
- `src/quote-settings/requests/update-quote-settings.request.ts` - Validation with class-validator
- `src/quote-settings/responses/quote-settings.response.ts` - API response interface
- `src/quote-settings/policies/quote-settings.policy.ts` - Business-scoped access control policy
- `src/business/entities/business.entity.ts` - Removed quoteEmail fields
- `src/business/data-transfer-objects/business.dto.ts` - Removed quoteEmail fields
- `src/business/repositories/business.repository.ts` - Removed quoteEmail from create/update/mapToDto
- `src/business/services/business-updater.service.ts` - Removed quoteEmail from ensurePermittedUpdates
- `src/business/responses/business.response.ts` - Removed quoteEmail fields
- `src/business/controllers/business.controller.ts` - Removed quoteEmail from update and mapToResponse
- `src/business/request/update-business.request.ts` - Removed quoteEmail properties and decorators
- `src/quote/services/quote-email-sender.service.ts` - Added QuoteSettingsRetriever injection
- `src/quote/quote.module.ts` - Added QuoteSettingsModule import
- `openapi.yaml` - Added QuoteSettings endpoints and schemas

## Decisions Made
- Upsert pattern for quote settings: $setOnInsert creates _id and createdAt on first write, $set updates fields on subsequent writes
- Business access verified by delegating to BusinessRetriever.findByIdOrFail (reuses existing BusinessPolicy auth check)
- GET endpoint returns empty defaults (id: '', businessId, undefined fields) when no settings exist -- avoids 404 for unconfigured businesses

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added @quote-settings Jest path alias to package.json**
- **Found during:** Task 2 (rewiring QuoteEmailSender)
- **Issue:** Jest could not resolve @quote-settings/* imports -- test suite failed to run
- **Fix:** Added `"^@quote-settings/(.*)$": "<rootDir>/quote-settings/$1"` to moduleNameMapper in package.json
- **Files modified:** package.json
- **Verification:** quote-email-sender tests pass (6/6)
- **Committed in:** f930dac (Task 2 commit)

**2. [Rule 3 - Blocking] Added QuoteSettingsRetriever mock to quote-email-sender test**
- **Found during:** Task 2 (rewiring QuoteEmailSender)
- **Issue:** Test module missing the new QuoteSettingsRetriever provider -- NestJS TestingModule failed to compile
- **Fix:** Added mock provider `{ provide: QuoteSettingsRetriever, useValue: { findByBusinessId: jest.fn().mockResolvedValue(null) } }`
- **Files modified:** src/quote/test/services/quote-email-sender.service.spec.ts
- **Verification:** All 6 existing tests pass unchanged
- **Committed in:** f930dac (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (2 blocking)
**Impact on plan:** Both fixes required for test suite to run. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- QuoteSettings API ready for frontend integration
- Frontend can fetch email template defaults via GET /v1/business/:businessId/quote-settings
- Frontend can save email template via PATCH /v1/business/:businessId/quote-settings
- QuoteSettingsRetriever exported and available for future direct usage in QuoteEmailSender send logic

## Self-Check: PASSED

All 10 created files verified. Both task commits (50e4a87, f930dac) found on stage branch in trade-flow-api.

---
*Phase: 18-quote-email-sending*
*Completed: 2026-03-21*
