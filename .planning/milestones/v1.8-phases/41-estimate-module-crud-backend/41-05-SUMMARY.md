---
phase: 41-estimate-module-crud-backend
plan: 05
subsystem: estimate
tags: [repository, service, policy, mongodb, state-machine, range-math]
dependency_graph:
  requires: [41-01, 41-04]
  provides: [EstimateRepository, EstimateLineItemRepository, EstimateNumberGenerator, EstimateTotalsCalculator, EstimateTransitionService, EstimatePolicy, EstimateLineItemPolicy]
  affects: [41-06, 41-07, 41-08]
tech_stack:
  added: []
  patterns: [pagination-via-native-driver, atomic-counter, contingency-range-math, partial-unique-index]
key_files:
  created:
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
    - trade-flow-api/src/estimate/services/estimate-number-generator.service.ts
    - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
    - trade-flow-api/src/estimate/services/estimate-transition.service.ts
    - trade-flow-api/src/estimate/policies/estimate.policy.ts
    - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
    - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
    - trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-number-generator.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-totals-calculator.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts
    - trade-flow-api/src/estimate/test/policies/estimate.policy.spec.ts
    - trade-flow-api/src/estimate/test/policies/estimate-line-item.policy.spec.ts
  modified:
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
    - trade-flow-api/src/estimate/entities/estimate.entity.ts
decisions:
  - Pagination via native MongoDB driver (MongoConnectionService.getDb) rather than MongoDbFetcher, since MongoDbFetcher uses page-based not offset-based pagination
  - Added response/uncertainty fields to IEstimateDto that entity had but DTO lacked (Rule 2 deviation)
  - Entity deletedAt typed as Date | null to allow MongoDB filter {deletedAt: null} without type assertion (Rule 3 deviation)
metrics:
  duration: ~39 minutes
  completed: 2026-04-12
  tasks_completed: 4
  tasks_total: 4
  test_count: 96
  files_created: 14
  files_modified: 2
---

# Phase 41 Plan 05: Estimate Repositories and Stateless Services Summary

Repositories (CRUD + pagination + indexes), atomic counter, range-math totals calculator, state-machine transition service, and business-ownership policies for the estimate module -- all with companion specs.

## What Was Built

### EstimateRepository (estimate.repository.ts)
- COLLECTION = "estimates", implements OnModuleInit for index bootstrap
- CRUD: create, findByIdOrFail, update with $set of all mutable fields
- findAllByBusinessId (unpaginated convenience for internal use)
- findPaginatedByBusinessId via native MongoDB driver with offset/limit/sort/countDocuments
- 5 indexes: businessId+createdAt, jobId+createdAt, businessId+number (partial unique with deletedAt null + isCurrent true), rootEstimateId+isCurrent (partial unique), rootEstimateId+revisionNumber
- toEntity/toDto round-trip all fields; firstViewedAt only written when defined (Pitfall 8)

### EstimateLineItemRepository (estimate-line-item.repository.ts)
- COLLECTION = "estimatelineitems" (no underscore per D-ENT-02)
- CRUD: create, findByIdOrFail, update (partial field updates), findByEstimateId
- softDelete and softDeleteByParentLineItemId
- Money fields stored as major units matching QuoteLineItemRepository pattern

### EstimateNumberGenerator (estimate-number-generator.service.ts)
- Atomic findOneAndUpdate with $inc + upsert on "estimate_counters" collection
- Returns E-YYYY-NNN format (year from DateTime.now())

### EstimateTotalsCalculator (estimate-totals-calculator.service.ts)
- Iterates lineItems, skips parent-scoped and DELETED items
- Computes subTotal, taxTotal, total
- Builds priceRange with low = base totals, high = base * (1 + contingencyPercent/100)
- Carries displayMode through from input estimate

### EstimateTransitionService (estimate-transition.service.ts)
- State machine enforcer using isValidTransition from estimate-transitions
- Sets correct timestamp per target status (sentAt, firstViewedAt, respondedAt, convertedAt, declinedAt, lostAt, deletedAt)
- Pitfall 9: firstViewedAt preserved when already set
- Authorization via AccessControllerFactory + EstimatePolicy

### EstimatePolicy / EstimateLineItemPolicy
- Business-ownership guard mirroring QuotePolicy pattern
- canDelete always returns false (soft-delete via transition)
- Support user bypass via isSupportUser utility

## Test Coverage

- 31 repository tests (round-trip, collection names, CRUD, pagination, indexes)
- 39 number generator + totals calculator tests (parameterised: 7 contingencies x 4 subtotals penny-exact)
- 26 transition service + policy tests (invalid transitions, terminal states, timestamp setting, Pitfall 9, ownership)
- Total: 96 tests, all passing

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Fields] Added response/uncertainty fields to IEstimateDto**
- **Found during:** Task 1
- **Issue:** IEstimateDto (from Plan 04) lacked lastResponseType, lastResponseAt, lastResponseMessage, declineReason, siteVisitAvailability, uncertaintyReasons, uncertaintyNotes fields that exist on IEstimateEntity and are needed for correct round-tripping
- **Fix:** Added the missing fields to IEstimateDto interface
- **Files modified:** trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
- **Commit:** 43c7817

**2. [Rule 3 - Blocking Type Error] Entity deletedAt typed to allow null**
- **Found during:** Task 1
- **Issue:** MongoDB filter {deletedAt: null} caused TS2322 because entity typed deletedAt as Date | undefined (not null)
- **Fix:** Changed entity deletedAt to Date | null | undefined to allow MongoDB null-filter without type assertion
- **Files modified:** trade-flow-api/src/estimate/entities/estimate.entity.ts
- **Commit:** 43c7817

## Commits

| # | Hash | Message |
|---|------|---------|
| 1 | 43c7817 | feat(41-05): add EstimateRepository and EstimateLineItemRepository with specs |
| 2 | 1116d9a | feat(41-05): add EstimateNumberGenerator and EstimateTotalsCalculator with parameterised specs |
| 3 | 81dd787 | feat(41-05): add EstimateTransitionService and policies with specs |
| 4 | fab61c1 | chore(41-05): prettier fixes and remove unused import for estimate wave 4 |

## Self-Check: PASSED

All 14 created files verified present. All 4 commit hashes verified in git log.
