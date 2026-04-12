---
phase: 41-estimate-module-crud-backend
plan: 03
subsystem: document-token
tags: [rename, refactor, security, token-type-confusion]
dependency_graph:
  requires: [41-01]
  provides: [document-token-module, document-type-discriminator, token-type-assertion]
  affects: [app.module, quote.module, quote.controller, quote-transition.service, quote-email-sender.service]
tech_stack:
  added: []
  patterns: [document-type-discriminator, token-type-confusion-mitigation]
key_files:
  created:
    - trade-flow-api/src/document-token/document-token.module.ts
    - trade-flow-api/src/document-token/entities/document-token.entity.ts
    - trade-flow-api/src/document-token/data-transfer-objects/document-token.dto.ts
    - trade-flow-api/src/document-token/repositories/document-token.repository.ts
    - trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
    - trade-flow-api/src/document-token/services/document-token-creator.service.ts
    - trade-flow-api/src/document-token/services/document-token-retriever.service.ts
    - trade-flow-api/src/document-token/services/document-token-revoker.service.ts
    - trade-flow-api/src/document-token/services/public-quote-retriever.service.ts
    - trade-flow-api/src/document-token/services/quote-response-handler.service.ts
    - trade-flow-api/src/document-token/controllers/public-quote.controller.ts
    - trade-flow-api/src/document-token/requests/decline-quote.request.ts
    - trade-flow-api/src/document-token/responses/public-quote.response.ts
    - trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts
    - trade-flow-api/src/document-token/test/guards/document-session-auth.guard.spec.ts
    - trade-flow-api/src/document-token/test/controllers/public-quote.controller.spec.ts
    - trade-flow-api/src/document-token/test/services/document-token-creator.service.spec.ts
    - trade-flow-api/src/document-token/test/services/document-token-retriever.service.spec.ts
    - trade-flow-api/src/document-token/test/services/document-token-revoker.service.spec.ts
    - trade-flow-api/src/document-token/test/services/public-quote-retriever.service.spec.ts
    - trade-flow-api/src/document-token/test/services/quote-response-handler.service.spec.ts
    - trade-flow-api/src/document-token/test/mocks/document-token-mock-generator.ts
  modified:
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/src/quote/quote.module.ts
    - trade-flow-api/src/quote/controllers/quote.controller.ts
    - trade-flow-api/src/quote/services/quote-transition.service.ts
    - trade-flow-api/src/quote/services/quote-email-sender.service.ts
    - trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts
    - trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json
    - trade-flow-api/CLAUDE.md
  deleted:
    - trade-flow-api/src/quote-token/ (entire directory - 21 files)
decisions:
  - Renamed revokeAllForQuote to revokeAllForDocument (dropped documentType filter param since existing callers only revoke by documentId)
  - Renamed findLatestNonRevokedForQuote to findLatestNonRevokedForDocument on both repository and retriever
  - DocumentTokenCreator.create takes documentType as required second parameter (no default) for explicitness
  - DocumentTokenMockGenerator defaults documentType to "quote" for backward compatibility with existing callers
  - Updated CLAUDE.md Session Authentication section to reflect new guard and DTO names
metrics:
  duration: ~22 minutes
  completed: 2026-04-12
  tasks_completed: 4
  tasks_total: 4
  files_created: 22
  files_modified: 10
  files_deleted: 21
  tests_added: 45
  tests_total: 421
---

# Phase 41 Plan 03: Document Token Rename Summary

End-to-end code rename of src/quote-token/ to src/document-token/ with documentType discriminator and token-type-confusion mitigation.

## One-liner

Pure rename of quote-token module to document-token with documentId/documentType discriminator fields and controller-level type assertion preventing cross-document token confusion.

## Tasks Completed

### Task 1: Create src/document-token/ with renamed entities, DTOs, repository, guard, and module

Created the full document-token directory structure with renamed files:

| FROM (quote-token) | TO (document-token) |
|-----|-----|
| IQuoteTokenEntity | IDocumentTokenEntity |
| IQuoteTokenDto | IDocumentTokenDto |
| QuoteTokenRepository (quote_tokens) | DocumentTokenRepository (document_tokens) |
| QuoteSessionAuthGuard | DocumentSessionAuthGuard |
| QuoteTokenModule | DocumentTokenModule |
| QuoteTokenMockGenerator | DocumentTokenMockGenerator |

Field-level renames:
- `quoteId: ObjectId` -> `documentId: ObjectId` (entity)
- `quoteId: string` -> `documentId: string` (DTO)
- Added `documentType: "quote" | "estimate"` to both entity and DTO
- `request.quoteToken` -> `request.documentToken` (guard)

Guard error codes updated:
- 404: `ErrorCodes.DOCUMENT_TOKEN_NOT_FOUND`
- 410 expired: `ErrorCodes.DOCUMENT_TOKEN_EXPIRED`
- 410 revoked: `ErrorCodes.DOCUMENT_TOKEN_REVOKED`

Repository create omits `firstViewedAt` when not provided (Pitfall 8 mitigation).

### Task 2: Rename and relocate services and PublicQuoteController with type assertion

Service class renames:
- `QuoteTokenCreator` -> `DocumentTokenCreator` (now accepts `documentType` parameter)
- `QuoteTokenRetriever` -> `DocumentTokenRetriever`
- `QuoteTokenRevoker` -> `DocumentTokenRevoker`

Preserved names (quote-specific by purpose):
- `PublicQuoteController`, `PublicQuoteRetriever`, `QuoteResponseHandler`
- `IPublicQuoteResponse`, `DeclineQuoteRequest`

Token-type-confusion mitigation (T-41-03-01):
Every handler in PublicQuoteController asserts `documentType === "quote"` and throws 404 with `DOCUMENT_TOKEN_TYPE_MISMATCH` on mismatch.

Method renames:
- `revokeAllForQuote` -> `revokeAllForDocument`
- `findLatestNonRevokedForQuote` -> `findLatestNonRevokedForDocument`

### Task 3: Delete src/quote-token/, update AppModule, QuoteController, and QuoteTransitionService

- `AppModule`: `QuoteTokenModule` -> `DocumentTokenModule`
- `QuoteModule`: `QuoteTokenModule` -> `DocumentTokenModule`
- `QuoteController`: `QuoteTokenRetriever` -> `DocumentTokenRetriever`, callsite updated
- `QuoteTransitionService`: `QuoteTokenRevoker` -> `DocumentTokenRevoker`, callsite updated
- `QuoteEmailSender`: `QuoteTokenCreator` -> `DocumentTokenCreator`, callsite updated with `"quote"` parameter
- All test specs updated with new import paths and class names
- `@quote-token/*` aliases removed from tsconfig.json and jest moduleNameMapper
- `src/quote-token/` directory deleted (21 files)
- CLAUDE.md Session Authentication section updated

### Task 4: Full CI gate verification

`npm run ci` exits 0:
- 63 test suites, 421 tests passed
- 0 lint errors (25 pre-existing warnings)
- Prettier format check passed
- TypeScript typecheck passed

## New Tests Added

| Test | File | Description |
|------|------|-------------|
| Collection constant | document-token.repository.spec.ts | Asserts COLLECTION === "document_tokens" |
| Create with documentType | document-token.repository.spec.ts | Verifies documentType written to entity |
| Omit firstViewedAt | document-token.repository.spec.ts | Asserts entity has no firstViewedAt on create |
| Guard 404 code | document-session-auth.guard.spec.ts | Asserts DOCUMENT_TOKEN_NOT_FOUND error code |
| Guard 410 expired code | document-session-auth.guard.spec.ts | Asserts DOCUMENT_TOKEN_EXPIRED error code |
| Guard 410 revoked code | document-session-auth.guard.spec.ts | Asserts DOCUMENT_TOKEN_REVOKED error code |
| Guard attaches documentToken | document-session-auth.guard.spec.ts | Asserts request.documentToken is set |
| Type mismatch 404 | public-quote.controller.spec.ts | documentType "estimate" returns 404 |
| Creator sets documentType | document-token-creator.service.spec.ts | Verifies documentType plumbed to repository |
| Creator estimate type | document-token-creator.service.spec.ts | Verifies "estimate" documentType passes through |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added jest moduleNameMapper entries for @document-token**
- **Found during:** Task 1
- **Issue:** Plan 01 added tsconfig path aliases but not jest moduleNameMapper entries
- **Fix:** Added `@document-token/*` and `@document-token-test/*` to package.json jest config
- **Files modified:** trade-flow-api/package.json

**2. [Rule 2 - Critical] Updated QuoteEmailSender and its spec**
- **Found during:** Task 3
- **Issue:** QuoteEmailSender also imports QuoteTokenCreator; not listed in plan Task 3 files
- **Fix:** Updated import, constructor, and callsite to DocumentTokenCreator with "quote" documentType
- **Files modified:** trade-flow-api/src/quote/services/quote-email-sender.service.ts, trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts

**3. [Rule 2 - Critical] Updated CLAUDE.md Session Authentication documentation**
- **Found during:** Task 3
- **Issue:** CLAUDE.md referenced QuoteSessionAuthGuard, IQuoteTokenDto, request.quoteToken
- **Fix:** Updated entire Session Authentication section with new class names and documentType assertion pattern
- **Files modified:** trade-flow-api/CLAUDE.md

**4. [Rule 1 - Bug] Prettier formatting fixes**
- **Found during:** Task 4
- **Issue:** 6 prettier errors in public-quote.controller.ts and document-token.module.ts
- **Fix:** Ran npm run format
- **Files modified:** trade-flow-api/src/document-token/controllers/public-quote.controller.ts, trade-flow-api/src/document-token/document-token.module.ts

## Threat Mitigation Status

| Threat ID | Status | Notes |
|-----------|--------|-------|
| T-41-03-01 | Mitigated | documentType assertion on all 3 controller handlers + spec test |
| T-41-03-02 | Mitigated | Plan 01 pre-check confirmed empty collection |
| T-41-03-03 | Accepted | Same 410/404 distinction preserved from quote-token flow |
| T-41-03-04 | Mitigated | Repository toEntity never includes documentType in update paths |
| T-41-03-05 | Mitigated | @Throttle decorators preserved on all handlers |
| T-41-03-06 | Mitigated | Same string parameter pattern preserved |
| T-41-03-07 | Mitigated | Repository create omits firstViewedAt; spec asserts undefined |

## Known Stubs

None - all functionality is fully wired.

## Commit Status

All code changes are complete and verified (npm run ci passes with 0 errors). Changes need to be committed in the trade-flow-api repository. The git operations were blocked by the sandbox permission system during execution.

Files to stage and commit in trade-flow-api:
- `git add src/document-token/ src/app.module.ts src/quote/ package.json tsconfig.json CLAUDE.md`
- `git rm -r src/quote-token/` (already deleted from disk)

## Self-Check: PENDING

Commits could not be created due to sandbox restrictions on git write operations in the trade-flow-api sub-repository. All files exist on disk and CI passes.
