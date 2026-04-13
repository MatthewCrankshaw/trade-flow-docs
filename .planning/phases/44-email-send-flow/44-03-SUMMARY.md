---
phase: 44-email-send-flow
plan: "03"
subsystem: trade-flow-api
tags:
  - backend
  - nestjs
  - estimate
  - email
  - send-flow
  - audit
dependency_graph:
  requires:
    - 44-01 (EstimateSettingsModule, EstimateSettingsRetriever)
    - 44-02 (EstimateEmailRenderer, EstimateEmailSendCreator, DocumentTokenRepository extensions)
    - Phase 42 (ESTIMATE_FOLLOWUP_CANCELLER interface, EstimateReviser)
  provides:
    - POST /v1/estimates/:id/send endpoint
    - EstimateEmailSender orchestrator service
    - SendEstimateRequest with @Matches subject validation
    - EstimateEmailSendRepository (write-only audit)
    - EstimateEmailSendCreator service
    - lastSentAt/lastSentTo fields on entity, DTO, response
  affects:
    - 44-04 (frontend SendEstimateDialog consumes POST /v1/estimates/:id/send)
    - Phase 45 (public customer page uses existing token set by sender)
    - Phase 46 (followup queue cancellation via IEstimateFollowupCanceller)
tech_stack:
  added: []
  patterns:
    - Token reuse on re-send (findActiveByDocumentId + extendExpiry) with degraded-path new-token creation on expiry
    - Separation-over-DRY sanitizeHtml copied from QuoteEmailSender (D-SND-04)
    - Post-send error handling: email delivered before audit/transition writes; failure after delivery logs and re-throws as 500
    - Write-only audit repository pattern (no update/delete methods per D-AUDIT-01)
key_files:
  created:
    - trade-flow-api/src/estimate/services/estimate-email-sender.service.ts
    - trade-flow-api/src/estimate/requests/send-estimate.request.ts
    - trade-flow-api/src/estimate/test/services/estimate-email-sender.service.spec.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-email-send.dto.ts
    - trade-flow-api/src/estimate/entities/estimate-email-send.entity.ts
    - trade-flow-api/src/estimate/enums/estimate-email-send-type.enum.ts
    - trade-flow-api/src/estimate/repositories/estimate-email-send.repository.ts
    - trade-flow-api/src/estimate/services/estimate-email-send-creator.service.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-email.mock-generator.ts
    - trade-flow-api/src/estimate/test/services/estimate-email-send-creator.service.spec.ts
  modified:
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts (send endpoint + lastSentAt/lastSentTo in mapToResponse)
    - trade-flow-api/src/estimate/estimate.module.ts (EmailModule, EstimateSettingsModule, DocumentTokenModule imports; new providers)
    - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts (EstimateEmailSender mock; D-TXN-05 test updated)
    - trade-flow-api/src/estimate/entities/estimate.entity.ts (lastSentAt, lastSentTo fields)
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts (lastSentAt, lastSentTo fields)
    - trade-flow-api/src/estimate/responses/estimate.responses.ts (lastSentAt, lastSentTo fields)
    - trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts (VIEWED/RESPONDED/SITE_VISIT_REQUESTED -> SENT tests)
    - trade-flow-api/src/estimate/enums/estimate-transitions.ts (re-send transitions already present, confirmed)
    - trade-flow-api/src/document-token/repositories/document-token.repository.ts (findActiveByDocumentId, extendExpiry)
    - trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts (new method tests)
    - trade-flow-api/src/core/errors/error-codes.enum.ts (ESTIMATE_NOT_SENDABLE, DOCUMENT_TOKEN_NOT_FOUND)
    - trade-flow-api/src/email/services/estimate-email-renderer.service.ts (already implemented by parallel wave)
    - trade-flow-api/openapi.yaml (send endpoint, estimate-settings endpoints, lastSentAt/lastSentTo schema fields)
    - .planning/milestones/v1.8-ROADMAP.md (Templates -> Documents tab rename)
    - .planning/REQUIREMENTS.md (SND-02 Templates -> Documents tab rename)
decisions:
  - EstimateEmailSender uses findActiveByDocumentId on rootEstimateId for token reuse; creates new token if expired (degraded re-send per D-TKN-05)
  - sanitizeHtml allowlist (p, br, strong, em, a, ul, ol, li) copied verbatim from QuoteEmailSender — no shared utility per D-SND-04
  - EstimateModule uses forwardRef(() => EstimateSettingsModule) to avoid circular dependency with BusinessModule
  - D-TXN-05 test updated from Phase 41's "no transition endpoints" assertion to Phase 44's "send endpoint present, no unplanned endpoints" assertion
  - Post-send steps (audit, transition, updateSendFields, extendExpiry, cancelFollowups) are wrapped in try/catch; failure logs at error level and re-throws since email is already delivered
metrics:
  duration: ~25min
  completed_date: "2026-04-13"
  tasks: 2
  files: 21
---

# Phase 44 Plan 03: EstimateEmailSender Orchestrator and Send Endpoint Summary

EstimateEmailSender orchestrator wiring all Phase 44 building blocks into the complete backend send pipeline — token reuse, sanitized HTML rendering, Resend delivery, write-only audit persistence, status transition, and followup cancellation on revised sends — plus the POST /v1/estimates/:id/send controller endpoint, module wiring, and OpenAPI documentation.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | EstimateEmailSender service + SendEstimateRequest + audit layer + entity/DTO/response extensions | 4796c54 | 20 files (service, request, spec, audit repo/creator/dto/entity/enum, mock generator, entity/DTO/response extensions, document-token extensions, error codes, transition spec) |
| 2 | Controller endpoint + module wiring + OpenAPI + planning doc updates | 594374c, 96c7c0f, 7aee5f5 | 4 files (controller, module, controller spec, openapi.yaml) + planning docs |

## What Was Built

### EstimateEmailSender (`src/estimate/services/estimate-email-sender.service.ts`)

Orchestrator service with full send pipeline:

1. **Load and authorize** — `estimateRetriever.findByIdOrFail()` + `estimatePolicy.canUpdate()` gate (throws ForbiddenError on non-owner)
2. **Determine send path** — DRAFT rev 1 = INITIAL, DRAFT rev > 1 = INITIAL + shouldCancelFollowups, SENT/VIEWED/RESPONDED/SITE_VISIT_REQUESTED = RESEND, terminal = throw ESTIMATE_NOT_SENDABLE
3. **Resolve or create token** — `findActiveByDocumentId(rootId, "estimate")`; reuse if active, create new if null (degraded re-send per D-TKN-05)
4. **Load settings and render** — `estimateSettingsRetriever.findByBusinessId()`, `formatRange()` for price display, `sanitizeHtml()` allowlist, `estimateEmailRenderer.render()`
5. **Send via Resend** — `emailSenderService.sendEmail()` before any write operations
6. **Post-send writes** (try/catch, re-throw on failure):
   - Audit: `estimateEmailSendCreator.create()` with full rendered HTML
   - Transition: `estimateTransitionService.transition()` to SENT
   - Update: `estimateRepository.updateSendFields()` with lastSentAt/lastSentTo
   - Extend token: `documentTokenRepository.extendExpiry()` +30 days
   - Followups: `followupCanceller.cancelAllFollowups()` on previous revision (revised send only)
   - SaveEmail: `customerUpdater.update()` if request.saveEmail === true

### SendEstimateRequest (`src/estimate/requests/send-estimate.request.ts`)

- `@IsEmail()` on `to` field
- `@Matches(/estimate/i)` on `subject` — rejects subjects missing "Estimate" keyword (T-44-11 mitigation)
- `@IsString() @IsNotEmpty()` on `message`
- `@IsOptional() @IsBoolean()` on `saveEmail`

### Entity/DTO/Response Extensions

- `IEstimateEntity`: `lastSentAt?: Date`, `lastSentTo?: string`
- `IEstimateDto`: `lastSentAt?: DateTime`, `lastSentTo?: string`
- `IEstimateResponse`: `lastSentAt: string | null`, `lastSentTo: string | null`
- Controller `mapToResponse()` updated to include both fields

### Audit Layer

- `EstimateEmailSendRepository` — write-only; `create()`, `findAllByEstimateId()`, `findAllByRootEstimateId()`; 3 indexes; no update/delete per D-AUDIT-01
- `EstimateEmailSendCreator` — thin service delegating to repository
- `EstimateEmailSendType` enum — INITIAL, RESEND, FOLLOWUP_3D, FOLLOWUP_10D, FOLLOWUP_21D

### DocumentTokenRepository Extensions

- `findActiveByDocumentId(documentId, documentType)` — returns active non-expired token or null
- `extendExpiry(tokenId, newExpiresAt)` — extends token expiry; throws ResourceNotFoundError if not found

### POST /v1/estimates/:id/send Controller Endpoint

```typescript
@UseGuards(JwtAuthGuard)
@Post("estimates/:id/send")
public async send(
  @Req() request: { user: IUserDto; params: { id: string } },
  @Body() body: SendEstimateRequest,
): Promise<IResponse<IEstimateResponse>>
```

### EstimateModule Wiring

New imports: `EmailModule`, `forwardRef(() => EstimateSettingsModule)`, `DocumentTokenModule`
New providers: `EstimateEmailSendRepository`, `EstimateEmailSendCreator`, `EstimateEmailSender`

### OpenAPI Updates

- `POST /v1/estimates/{id}/send` with `SendEstimateRequest` schema
- `GET /PATCH /v1/business/{businessId}/estimate-settings` with `EstimateSettingsResponse` schema
- `lastSentAt` (nullable datetime) and `lastSentTo` (nullable string) added to `EstimateResponse` schema
- New schemas: `SendEstimateRequest`, `UpdateEstimateSettingsRequest`, `EstimateSettingsResponse`, `EstimateSettingsResponseEnvelope`

### Planning Doc Updates

- `v1.8-ROADMAP.md`: All 3 occurrences of "Business > Templates tab" updated to "Business > Documents tab"
- `REQUIREMENTS.md` SND-02: "Templates tab" updated to "Documents tab"

## Verification

All tests pass:

```
estimate-email-sender spec: 8 tests (7 send cases + 1 sanitizeHtml)
estimate-transition spec: 14 tests (includes 3 new re-send transitions)
estimate-email-send-creator spec: 2 tests
document-token.repository spec: 11 tests
estimate.controller spec: 21 tests (updated D-TXN-05, new send endpoint test)
All other estimate specs: unchanged, all pass

Total: 730 tests across 88 suites — all PASS
CI: 730/730 tests pass, lint clean (0 errors), format clean, typecheck clean
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Controller mapToResponse missing lastSentAt/lastSentTo (TypeScript error TS2739)**
- **Found during:** Running tests after Task 1 commit
- **Issue:** `IEstimateResponse` interface (updated in 44-02) requires `lastSentAt: string | null` and `lastSentTo: string | null` but controller's `mapToResponse()` didn't include them
- **Fix:** Added `lastSentAt: estimate.lastSentAt?.toISO() ?? null` and `lastSentTo: estimate.lastSentTo ?? null` to `mapToResponse()`
- **Files modified:** `src/estimate/controllers/estimate.controller.ts`
- **Commit:** 594374c

**2. [Rule 1 - Bug] Controller spec missing EstimateEmailSender mock provider**
- **Found during:** Running tests after adding EstimateEmailSender to controller constructor
- **Issue:** Existing `estimate.controller.spec.ts` did not provide `EstimateEmailSender` in the test module, causing all 19 tests to fail with DI resolution error
- **Fix:** Added `{ provide: EstimateEmailSender, useValue: mockEmailSender }` to the test module providers
- **Files modified:** `src/estimate/test/controllers/estimate.controller.spec.ts`
- **Commit:** 594374c

**3. [Rule 1 - Bug] D-TXN-05 test incompatible with Phase 44 send endpoint**
- **Found during:** Running tests after adding `send` method to controller
- **Issue:** Phase 41's D-TXN-05 test checked that no methods matching `/send|respond|convert|markLost|transition/i` existed — this now fails because Phase 44 adds `send()`
- **Fix:** Updated the test description and regex to exclude `send` (which is intentional in Phase 44) while still asserting that `respond`, `convert`, `markLost`, and `transition` are not present. Added a positive assertion that `send` exists.
- **Files modified:** `src/estimate/test/controllers/estimate.controller.spec.ts`
- **Commit:** 594374c

**4. [Rule 1 - Bug] Prettier formatting violations in estimate-email-sender.service.ts, estimate.repository.ts, and estimate-email-sender.service.spec.ts**
- **Found during:** CI lint:check run after Task 2 commit
- **Issue:** Multi-line expressions exceeded Prettier's 125-char print width in wrong positions
- **Fix:** `npx prettier --write` applied to affected files
- **Files modified:** `src/estimate/services/estimate-email-sender.service.ts`, `src/estimate/repositories/estimate.repository.ts`, `src/estimate/test/services/estimate-email-sender.service.spec.ts`
- **Commit:** 7aee5f5

**5. [Rule 2 - Missing Critical Functionality] Estimate-email-send audit and document-token extension files were uncommitted from 44-02 parallel wave**
- **Found during:** Task 1 setup — git status showed untracked/modified 44-02 files
- **Context:** The parallel 44-02 wave agent created the files but did not commit them all to the API repo. Only the Maizzle template was committed.
- **Fix:** Staged and committed all 44-02 remaining artifacts as part of the 44-03 Task 1 commit (4796c54) with appropriate attribution
- **Files affected:** estimate-email-send.dto.ts, estimate-email-send.entity.ts, estimate-email-send-type.enum.ts, estimate-email-send.repository.ts, estimate-email-send-creator.service.ts, and related specs

## Known Stubs

None — all pipeline steps are fully wired. The `IEstimateFollowupCanceller` binding resolves to `NoopEstimateFollowupCanceller` (returns immediately without scheduling) which is the correct v1.8 Phase 44 behavior — Phase 46 replaces this with the real BullMQ implementation.

## Threat Flags

No new threat surface beyond what is documented in the plan's threat model. All mitigations applied:
- T-44-09: `JwtAuthGuard` on POST /v1/estimates/:id/send
- T-44-10: `sanitizeHtml()` allowlist (p, br, strong, em, a, ul, ol, li) strips all other tags
- T-44-11: `@Matches(/estimate/i)` rejects subjects missing "Estimate" keyword
- T-44-12: `@IsEmail()` validates recipient format; `EstimatePolicy.canUpdate()` ensures business ownership
- T-44-13: `estimatePolicy.canUpdate(authUser, estimate)` checked before any processing

## Self-Check: PASSED

All key files verified present:
- `trade-flow-api/src/estimate/services/estimate-email-sender.service.ts` — contains `findActiveByDocumentId`, `extendExpiry`, `cancelAllFollowups`, `sanitizeHtml`, `ESTIMATE_FOLLOWUP_CANCELLER`, `estimatePolicy`, `import { formatRange }`
- `trade-flow-api/src/estimate/requests/send-estimate.request.ts` — contains `@Matches(/estimate/i`
- `trade-flow-api/src/estimate/controllers/estimate.controller.ts` — contains `v1/estimates/:id/send`, `estimateEmailSender.send`, `lastSentAt`, `lastSentTo`
- `trade-flow-api/src/estimate/estimate.module.ts` — contains `EstimateEmailSender`, `EstimateEmailSendRepository`, `EstimateSettingsModule`
- `trade-flow-api/openapi.yaml` — contains `/v1/estimates/{id}/send`, `/v1/business/{businessId}/estimate-settings`, `lastSentAt`
- `.planning/milestones/v1.8-ROADMAP.md` — all "Templates tab" changed to "Documents tab"

All commits verified in API git history:
- 4796c54: feat(44-03): add EstimateEmailSender service, SendEstimateRequest, audit layer, and transition extensions
- 594374c: feat(44-03): add POST /v1/estimates/:id/send endpoint, module wiring, and controller mapToResponse fields
- 96c7c0f: docs(44-03): update OpenAPI spec with send endpoint, estimate-settings endpoints, and lastSentAt/lastSentTo fields
- 7aee5f5: fix(44-03): fix prettier formatting in estimate-email-sender and repository files
