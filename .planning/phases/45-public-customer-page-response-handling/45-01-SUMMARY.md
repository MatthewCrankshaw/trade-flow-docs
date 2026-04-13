---
phase: 45-public-customer-page-response-handling
plan: "01"
subsystem: estimate-enums-entity-repository
tags: [estimates, enums, entity, repository, response-handling, 3-action-model]
dependency_graph:
  requires: [phase-41-estimate-crud, phase-42-revisions]
  provides: [estimate-response-type-enum, estimate-decline-reason-enum, estimate-responses-array, push-response-method]
  affects: [phase-45-02, phase-45-03, phase-45-04, phase-46-followup]
tech_stack:
  added: []
  patterns: [mongodb-set-over-push-for-typed-arrays]
key_files:
  created: []
  modified:
    - trade-flow-api/src/estimate/enums/estimate-status.enum.ts
    - trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts
    - trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts
    - trade-flow-api/src/estimate/enums/estimate-transitions.ts
    - trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts
    - trade-flow-api/src/estimate/entities/estimate.entity.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/estimate/responses/estimate.responses.ts
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts
    - trade-flow-api/src/estimate/services/estimate-reviser.service.ts
    - trade-flow-api/src/estimate/services/estimate-deleter.service.ts
    - trade-flow-api/src/estimate/services/estimate-email-sender.service.ts
    - trade-flow-api/src/estimate/services/estimate-transition.service.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts
    - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts
    - .planning/REQUIREMENTS.md
    - .planning/ROADMAP.md
    - .planning/milestones/v1.8-ROADMAP.md
decisions:
  - "pushResponse uses $set with full appended array instead of $push due to MongoDB 7.0 PushOperator TypeScript type limitation on entities extending Record<string, unknown>"
  - "EstimateDeclineReason enum updated from 6 values (with GOING_WITH_ANOTHER_TRADESPERSON and OTHER) to 5 canonical values (GOING_WITH_SOMEONE_ELSE replaces GOING_WITH_ANOTHER_TRADESPERSON, OTHER removed)"
  - "DECLINED transitions expanded to include LOST and DELETED (non-terminal) per 3-action model"
  - "RESPONDED transitions expanded to include DECLINED and SENT (re-send) per service continuity"
metrics:
  duration: 45min
  completed: "2026-04-13"
  tasks: 3
  files: 23
---

# Phase 45 Plan 01: Foundation — Enum, Entity, and Docs Updates Summary

Remove SITE_VISIT_REQUESTED from estimate status lifecycle and replace the 4-button response model with the 3-action conversational CTA model (proceed/message/decline), adding responses[] array to entity/DTO/repository and updating planning documentation.

## What Was Built

### Task 1: Estimate Enums and Transition Map

- **EstimateStatus** (`estimate-status.enum.ts`): Removed `SITE_VISIT_REQUESTED = "site_visit_requested"`. Enum now has exactly 9 values.
- **EstimateResponseType** (`estimate-response-type.enum.ts`): Replaced 4-value enum (SITE_VISIT_REQUEST, SEND_ME_QUOTE, QUESTION, DECLINE) with 3-action model: `PROCEED = "proceed"`, `MESSAGE = "message"`, `DECLINE = "decline"`.
- **EstimateDeclineReason** (`estimate-decline-reason.enum.ts`): Updated to 5 canonical values: TOO_EXPENSIVE, GOING_WITH_SOMEONE_ELSE, DECIDED_NOT_TO_DO_WORK, JUST_GETTING_IDEA, TIMING_NOT_RIGHT (removed OTHER and GOING_WITH_ANOTHER_TRADESPERSON).
- **ALLOWED_TRANSITIONS** (`estimate-transitions.ts`): Removed SITE_VISIT_REQUESTED key and all references. Added DECLINED as valid from VIEWED and RESPONDED. DECLINED allows LOST and DELETED transitions (not terminal). Maintained SENT re-send from VIEWED and RESPONDED.
- **Transition spec** (`estimate-transitions.spec.ts`): Rewrote to remove all SITE_VISIT_REQUESTED test cases. Added VIEWED->DECLINED, RESPONDED->DECLINED, DECLINED->LOST, DECLINED->DELETED as allowed. Added RESPONDED->VIEWED as disallowed. Confirmed 9-key map size. All 30 tests pass.
- **EstimateMockGenerator**: Added `generateEstimateResponse()` static method returning `IEstimateResponseEntity` with default PROCEED type. Updated `createEstimateEntity()` to include required `responses: []` field.

### Task 2: Responses Array in Entity, DTO, and Repository

- **IEstimateResponseEntity interface** (new, in `estimate.entity.ts`): `{ type: string; message?: string | null; reason?: string | null; respondedAt: Date; }`. Field `responses: IEstimateResponseEntity[]` added to `IEstimateEntity` as required (non-optional to satisfy MongoDB TypeScript types). Removed `siteVisitAvailability`, `lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason` from entity.
- **IEstimateResponseDto interface** (new, in `estimate.dto.ts`): `{ type: string; message: string | null; reason: string | null; respondedAt: string; }`. Field `responses?: IEstimateResponseDto[]` added to `IEstimateDto`. Removed deprecated individual response fields.
- **IEstimateResponseSummaryDto** (`estimate-response-summary.dto.ts`): Rewritten to `{ lastResponseType: string | null; lastResponseAt: string | null; lastResponseMessage: string | null; declineReason: string | null; }`. Removed `siteVisitAvailability`.
- **EstimateRepository** (`estimate.repository.ts`): Added `pushResponse(estimateId, response)` method using `$set` with full array append (see Decisions). Updated `toDto()` to map `entity.responses` to `IEstimateResponseDto[]`. Updated `toEntity()` to map `dto.responses` to `IEstimateResponseEntity[]`.
- **Cascading fixes**: Removed SITE_VISIT_REQUESTED from `REVISABLE_STATUSES` in estimate-reviser, estimate-deleter, estimate-email-sender, and estimate-transition services. Updated `mapResponseSummary` in controller to use new summary fields. Updated all related test specs.

### Task 3: Planning Documentation Updates

- **REQUIREMENTS.md**: Rewrote RESP-01..04 for the 3-action model: RESP-01 = "Happy to Proceed" (RESPONDED), RESP-02 = "Message [Tradesperson First Name]" (RESPONDED + message), RESP-03 = "Not right for me" (DECLINED + 5 structured reasons + optional freeform), RESP-04 = inline success confirmation + read-only terminal view.
- **ROADMAP.md Phase 45**: Updated SC #3 to "three response actions" description, SC #4 to remove SiteVisitRequested from transition list. Phase 45 summary line updated.
- **v1.8-ROADMAP.md Phase 45**: Same SC #3/#4 rewrites, goal updated from "four structured buttons" to "three conversational actions", summary line updated. Phase 46 SC #2 updated from "four response buttons" to "three response actions".

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] MongoDB PushOperator type incompatibility with $push on Record<string, unknown> entities**
- **Found during:** Task 2
- **Issue:** `IBaseEntity extends Record<string, unknown>` causes MongoDB 7.0's `PushOperator<IEstimateEntity>` to place `responses` in `NotAcceptedFields` (due to index signature ambiguity), making `$push: { responses: response }` a TypeScript error even with a properly typed array field.
- **Fix:** `pushResponse()` uses `$set: { responses: [...currentResponses, response] }` — fetches current responses, appends the new entry, and sets the full array. Functionally equivalent for append semantics. Avoids any `as` assertion.
- **Files modified:** `estimate.repository.ts`
- **Commit:** `7edbfc9`

**2. [Rule 2 - Missing transitions] DECLINED required non-terminal transitions; RESPONDED->DECLINED and SENT->CONVERTED missing from initial transition map**
- **Found during:** Task 1 (test failures)
- **Issue:** The 3-action model requires DECLINED to be reachable from VIEWED and RESPONDED, not just a terminal state. Also SENT->CONVERTED and re-send (SENT from VIEWED/RESPONDED) were in existing tests.
- **Fix:** Expanded ALLOWED_TRANSITIONS: DECLINED allows LOST and DELETED; RESPONDED allows DECLINED, CONVERTED, SENT; VIEWED allows CONVERTED and SENT.
- **Files modified:** `estimate-transitions.ts`, `estimate-transitions.spec.ts`
- **Commit:** `174edb6` (with subsequent refinement)

**3. [Rule 1 - Bug] Cascading SITE_VISIT_REQUESTED references in services not in plan scope**
- **Found during:** Task 2
- **Issue:** estimate-reviser, estimate-deleter, estimate-email-sender, estimate-transition services all referenced `EstimateStatus.SITE_VISIT_REQUESTED` which no longer exists.
- **Fix:** Removed SITE_VISIT_REQUESTED from REVISABLE_STATUSES in all four services and their associated test specs.
- **Files modified:** 4 service files + 3 spec files
- **Commit:** `7edbfc9`

## Known Stubs

None — no data flows to UI rendering in this plan. The `responseSummary: null` in `toDto()` is intentional — Phase 45-03 will implement the response summary derivation from the responses array.

## Threat Flags

None — this plan modifies internal enums, entity types, and planning docs only. No new network endpoints, auth paths, or trust boundary changes.

## Self-Check: PASSED

- [x] `estimate-status.enum.ts` has no SITE_VISIT_REQUESTED
- [x] `estimate-response-type.enum.ts` has PROCEED/MESSAGE/DECLINE (3 values)
- [x] `estimate-decline-reason.enum.ts` has exactly 5 values
- [x] `estimate-transitions.ts` has VIEWED->RESPONDED and VIEWED->DECLINED
- [x] `estimate.entity.ts` has `responses: IEstimateResponseEntity[]`
- [x] `estimate.repository.ts` has `pushResponse` method
- [x] All 303 estimate tests pass
- [x] TypeScript typecheck: zero errors
- [x] grep SITE_VISIT_REQUESTED in src/estimate/: zero matches
- [x] grep siteVisitAvailability in src/estimate/: zero matches
- [x] REQUIREMENTS.md RESP-01 contains "Happy to Proceed"
- [x] REQUIREMENTS.md RESP-02 contains "Message [Tradesperson First Name]"
- [x] REQUIREMENTS.md RESP-03 contains "Not right for me"
- [x] REQUIREMENTS.md RESP-04 contains "inline success confirmation"
- [x] ROADMAP.md Phase 45 contains "three response actions"
- [x] ROADMAP.md Phase 45 does not contain "SiteVisitRequested" (Phase 41 historical reference excluded)

## Commits

**trade-flow-api repo:**
- `174edb6`: feat(45-01): update estimate enums and transition map for 3-action CTA model
- `7edbfc9`: feat(45-01): add responses[] array to estimate entity, DTO, and repository

**trade-flow-docs repo:**
- `71f8097`: docs(45-01): rewrite RESP-01..04 for 3-action CTA model, update Phase 45 success criteria
