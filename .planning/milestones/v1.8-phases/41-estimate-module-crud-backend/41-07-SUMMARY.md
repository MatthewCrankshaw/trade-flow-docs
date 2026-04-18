---
phase: 41-estimate-module-crud-backend
plan: 07
subsystem: estimate-crud-services
tags: [estimate, crud, services, backend]
dependency_graph:
  requires: [41-04, 41-05, 41-06]
  provides: [estimate-creator, estimate-retriever, estimate-updater, estimate-deleter]
  affects: [41-08]
tech_stack:
  added: []
  patterns: [draft-only-enforcement, re-fetch-before-write, transition-service-delegation, authorized-creator-factory]
key_files:
  created:
    - trade-flow-api/src/estimate/services/estimate-creator.service.ts
    - trade-flow-api/src/estimate/services/estimate-retriever.service.ts
    - trade-flow-api/src/estimate/services/estimate-updater.service.ts
    - trade-flow-api/src/estimate/services/estimate-deleter.service.ts
    - trade-flow-api/src/estimate/test/services/estimate-creator.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-updater.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts
  modified:
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
decisions:
  - Mirrored QuoteCreator pattern exactly for EstimateCreator with estimate-specific defaults
  - Used EstimateTransitionService for soft delete instead of direct repository update
  - Paginated list does not load line items per estimate (totals calculated from empty collections for list view)
  - rootEstimateId set via two-step write pattern (insert then update) per Phase 42 D-CHAIN-03/04
metrics:
  duration: 899s
  completed: 2026-04-12T13:42:26Z
  tasks: 5/5
  files_created: 8
  files_modified: 2
  tests_added: 33
  tests_total: 606
---

# Phase 41 Plan 07: Estimate CRUD Services Summary

Four CRUD orchestration services for the estimate module with Draft-only enforcement, totals attachment, and 33 unit tests across all service files.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | EstimateCreator with defaults, number gen, totals | ecd4e20 | estimate-creator.service.ts, spec |
| 2 | EstimateRetriever with findByIdOrFail and findPaginated | ed754a1 | estimate-retriever.service.ts, spec |
| 3 | EstimateUpdater with Draft-only enforcement and line-item CRUD | f7a5270 | estimate-updater.service.ts, spec |
| 4 | EstimateDeleter with Draft-only soft delete via transition | 1fd044e | estimate-deleter.service.ts, spec |
| 5 | Full CI gate (formatting fixes) | 7d9489b | 4 files reformatted |

## Implementation Details

### EstimateCreator
- Validates customer (must be active) and job (must exist) before creation
- Generates E-YYYY-NNN number via EstimateNumberGenerator
- Applies defaults: contingencyPercent=10, displayMode=RANGE, revisionNumber=1, isCurrent=true, parentEstimateId=null
- Two-step root write: insert via AuthorizedCreatorFactory, then setRootEstimateId(created.id) for chain identity
- Attaches totals and priceRange via EstimateTotalsCalculator before returning
- 8 specs

### EstimateRetriever
- findByIdOrFail: loads estimate, checks access via AccessControllerFactory + EstimatePolicy, loads line items, attaches totals
- findPaginated: defaults limit=20, offset=0, supports status tab filtering, applies access control and totals to each result
- 8 specs

### EstimateUpdater
- update(): re-fetches from repository (Pitfall 5), asserts DRAFT, applies whitelisted patch fields, persists, reloads with totals
- addLineItem(): re-fetches, asserts DRAFT, resolves item, delegates to factory + creator (standard or bundle)
- updateLineItem(): re-fetches, asserts DRAFT, applies quantity/unitPrice patch to line item
- deleteLineItem(): re-fetches, asserts DRAFT, handles bundle parent+children soft-delete, prevents component-only deletion
- 12 specs

### EstimateDeleter
- softDelete(): re-fetches, asserts DRAFT, delegates to EstimateTransitionService.transition(DELETED)
- Does NOT inject or touch line items (D-DRAFT-03 history preservation)
- 5 specs

### Draft-only Enforcement Test Coverage
All non-DRAFT statuses tested for rejection: SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED, CONVERTED, DECLINED, EXPIRED, LOST, DELETED.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added setRootEstimateId to EstimateRepository**
- **Found during:** Task 1
- **Issue:** Plan specifies EstimateCreator calls estimateRepository.setRootEstimateId() but the method did not exist (Phase 42 D-CHAIN-04 fold-forward)
- **Fix:** Added `setRootEstimateId(id: string): Promise<void>` to EstimateRepository
- **Files modified:** trade-flow-api/src/estimate/repositories/estimate.repository.ts
- **Commit:** ecd4e20

**2. [Rule 3 - Blocking] Added missing estimate error codes to errors-map**
- **Found during:** Task 1
- **Issue:** ErrorCodes enum had estimate codes but errors-map.constant.ts lacked corresponding entries, which would cause "Error code unknown" messages at runtime
- **Fix:** Added 8 estimate error code entries to ERRORS_MAP
- **Files modified:** trade-flow-api/src/core/errors/errors-map.constant.ts
- **Commit:** ecd4e20

**3. [Rule 1 - Bug] InvalidRequestError takes 2 params not 3**
- **Found during:** Task 3
- **Issue:** Plan skeleton showed 3-arg `InvalidRequestError(code, message, { currentStatus })` but the class constructor only accepts `(code, details)`
- **Fix:** Used correct 2-arg constructor throughout
- **Files modified:** All four service files

## CI Gate Results

```
Test Suites: 79 passed, 79 total
Tests:       606 passed, 606 total
Lint: 0 errors, 27 warnings (all pre-existing in other modules)
Formatting: All matched files use Prettier code style
Typecheck: Pass
```

## Self-Check: PASSED
