---
phase: 41-estimate-module-crud-backend
plan: 08
title: "Controller, Module Wiring, and OpenAPI Documentation"
subsystem: estimate
tags: [controller, module, openapi, wiring, phase-final]
dependency_graph:
  requires: [41-04, 41-05, 41-06, 41-07]
  provides: [estimate-rest-api, estimate-module-registration, openapi-estimate-endpoints]
  affects: [trade-flow-api/src/estimate/, trade-flow-api/src/app.module.ts, trade-flow-api/openapi.yaml]
tech_stack:
  added: []
  patterns: [controller-service-repository, nestjs-module-wiring, openapi-spec]
key_files:
  created:
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts
    - trade-flow-api/src/estimate/estimate.module.ts
    - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
  modified:
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/openapi.yaml
decisions:
  - "Dropped DocumentTokenModule from EstimateModule imports since no estimate send flow exists yet (Phase 44)"
  - "Exported 6 services from EstimateModule for downstream phases: EstimateRepository, EstimateLineItemRepository, EstimateTransitionService, EstimateTotalsCalculator, EstimateCreator, EstimateRetriever"
metrics:
  duration: "21 minutes"
  completed: "2026-04-12T15:21:00Z"
  tasks_completed: 4
  tasks_total: 5
  tests_added: 11
  tests_total: 617
  files_created: 3
  files_modified: 2
---

# Phase 41 Plan 08: Controller, Module Wiring, and OpenAPI Documentation Summary

Wire EstimateController with 8 REST endpoints (5 CRUD + 3 line-item), assemble EstimateModule with all 15 providers from Plans 05-07, register in AppModule, and update openapi.yaml with full endpoint + schema documentation. Final CI gate green.

## Commits

| Task | Commit | Message |
|------|--------|---------|
| 1 | `08f98d6` | feat(41-08): add EstimateController with 8 authenticated CRUD + line-item endpoints |
| 2 | `9b1a40e` | feat(41-08): wire EstimateModule and register in AppModule |
| 3 | `04b0bcf` | docs(41-08): add estimate endpoints and DTOs to openapi.yaml |
| 5 | `8bf58f5` | chore(41-08): prettier fixes for estimate controller and module |

## 8 Endpoints

| Method | Path | OperationId | Handler |
|--------|------|-------------|---------|
| POST | /v1/estimates | createEstimate | create |
| GET | /v1/estimates | listEstimates | findAll |
| GET | /v1/estimates/:estimateId | getEstimate | findOne |
| PATCH | /v1/estimates/:estimateId | updateEstimate | update |
| DELETE | /v1/estimates/:estimateId | deleteEstimate | softDelete |
| POST | /v1/estimates/:estimateId/line-item | addEstimateLineItem | addLineItem |
| PATCH | /v1/estimates/:estimateId/line-item/:lineItemId | updateEstimateLineItem | updateLineItem |
| DELETE | /v1/estimates/:estimateId/line-item/:lineItemId | deleteEstimateLineItem | deleteLineItem |

All 8 handlers: JwtAuthGuard + try/catch with createHttpError + createResponse.

## Module Providers (15)

**Repositories:** EstimateRepository, EstimateLineItemRepository
**Policies:** EstimatePolicy, EstimateLineItemPolicy
**CRUD Services:** EstimateCreator, EstimateRetriever, EstimateUpdater, EstimateDeleter
**Stateless Services:** EstimateNumberGenerator, EstimateTotalsCalculator, EstimateTransitionService
**Line-Item Factories:** EstimateStandardLineItemFactory, EstimateBundleLineItemFactory, EstimateLineItemCreator, EstimateLineItemRetriever

**Exports (6):** EstimateRepository, EstimateLineItemRepository, EstimateTransitionService, EstimateTotalsCalculator, EstimateCreator, EstimateRetriever

## OpenAPI Schemas Added (10)

CreateEstimateRequest, UpdateEstimateRequest, CreateEstimateLineItemRequest, UpdateEstimateLineItemRequest, EstimateResponse, EstimateResponseEnvelope, EstimateListResponseEnvelope, EstimateTotals, EstimatePriceRange, EstimateResponseSummary, EstimatePriceRangeBound, EstimateLineItemResponse, MoneyValue

## CI Gate Results

- Tests: 617 passed, 0 failed (80 suites)
- Lint: 0 errors, 27 warnings (all pre-existing in other modules)
- Format: All files use Prettier code style
- Typecheck: 0 errors

## Deviations from Plan

### Skipped Task

**Task 4: ROADMAP.md / v1.8-ROADMAP.md / STATE.md rewrite per D-TKN-07**
- **Reason:** Orchestrator instructions explicitly state "Do NOT update STATE.md or ROADMAP.md -- the orchestrator owns those writes after all worktree agents in the wave complete."
- **Action needed:** The orchestrator should apply the D-TKN-07 rewrite (success criterion #5) and the D-ENT-02 `estimatelineitems` collection name fix in ROADMAP.md Phase 41 prose when it runs state updates.
- **Locked D-TKN-07 wording:** "The `quote-token` module is fully replaced by the new `document-token` module at the code level. The existing `/v1/public/quote/:token` endpoint continues to resolve tokens with `documentType === 'quote'` via the new guard. No data migration is required because no production data exists."

### Auto-fixed Issues

**1. [Rule 3 - Blocking] JwtAuthGuard DI resolution in test module**
- **Found during:** Task 1
- **Issue:** TestingModule could not resolve JwtAuthGuard dependencies (UserCreator, UserRetriever)
- **Fix:** Added `.overrideGuard(JwtAuthGuard).useValue({ canActivate: jest.fn(() => true) })` to mirror the BusinessController spec pattern
- **Files modified:** estimate.controller.spec.ts
- **Commit:** `08f98d6`

**2. [Rule 3 - Blocking] IUserDto mock missing required fields**
- **Found during:** Task 1
- **Issue:** Test authUser mock was missing `externalAuthUserId`, `supportRoleIds`, `businessRoleIds`, `supportRoles`, `businessRoles` required by IUserDto
- **Fix:** Added all required fields following the pattern in estimate-creator.service.spec.ts
- **Files modified:** estimate.controller.spec.ts
- **Commit:** `08f98d6`

**3. [Rule 1 - Bug] Prettier formatting violations**
- **Found during:** Task 5
- **Issue:** 12 prettier errors in controller, module, and spec files
- **Fix:** Ran `npx prettier --write` on the 3 affected files
- **Files modified:** estimate.controller.ts, estimate.module.ts, estimate.controller.spec.ts
- **Commit:** `8bf58f5`

## Known Stubs

None. All endpoints are fully wired to real services.

## Threat Flags

None. All endpoints require JwtAuthGuard authentication. No new public/unauthenticated surface introduced.
