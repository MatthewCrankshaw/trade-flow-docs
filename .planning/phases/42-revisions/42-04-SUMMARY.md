---
phase: 42
plan: 04
subsystem: estimate-reviser
tags: [service, reviser, concurrency, phase-42]
dependency_graph:
  requires: [42-02, 42-03]
  provides: [EstimateReviser service with two-write revise flow]
  affects: [42-06]
tech_stack:
  added: []
  patterns: [compensating-rollback, two-pass-line-item-clone, atomic-filter-gate]
key_files:
  created:
    - trade-flow-api/src/estimate/services/estimate-reviser.service.ts
    - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts
  modified: []
decisions:
  - Used undefined instead of null for optional DateTime fields in buildNewRevisionDto (IEstimateDto optional fields are typed as `?: DateTime`, not `DateTime | null`)
  - Adapted to actual method name EstimateLineItemMockGenerator.createEstimateLineItemDto (plan referenced non-existent createLineItemDto)
  - Used ItemMockGenerator.createUserDto for auth user in specs (no UserMockGenerator exists)
metrics:
  duration: 5m 26s
  completed: 2026-04-12T18:36:14Z
  tasks_completed: 2
  tasks_total: 2
  files_created: 3
  files_modified: 0
  test_count: 18
  test_pass: 18
---

# Phase 42 Plan 04: EstimateReviser Service Summary

EstimateReviser service implementing the two-write clone-only revise flow with atomic downgradeCurrent filter gate, two-pass line-item clone preserving bundle parent/child structure, and compensating rollback on failure.

## Completed Tasks

| # | Task | Commit | Key Files |
|---|------|--------|-----------|
| 1 | EstimateRevisionMockGenerator fixture builder | `0aa955b` (api) | `estimate-revision-mock-generator.ts` |
| 2 | EstimateReviser service + spec (18 tests) | `258e391` (api) | `estimate-reviser.service.ts`, `estimate-reviser.service.spec.ts` |

## Implementation Details

### EstimateReviser.revise() Flow

1. **Step 0:** Load source via EstimateRetriever.findByIdOrFail + authorize via AccessControllerFactory
2. **Step 1:** Atomic downgradeCurrent with filter on isCurrent + REVISABLE_STATUSES (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED); throws ConflictError on filter miss
3. **Step 2:** Insert new revision row (DRAFT, isCurrent: true, revisionNumber + 1)
4. **Step 3:** Two-pass line-item clone: roots first (captures new IDs), then children with remapped parentLineItemId
5. **Step 5:** Return fresh DTO via retriever so totals + priceRange are recalculated

### Compensating Rollback

On failure after step 1 succeeds:
- If new row was inserted: soft-delete it via `estimateRepository.softDeleteRow()` FIRST (frees partial unique index)
- Then restore source's isCurrent via `restoreCurrent()`
- D-REV-06 ordering verified via `invocationCallOrder` assertion in spec

### D-HOOK-03 Compliance

`followupCanceller` is injected via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` but never called from the reviser. Phase 44 owns the call at the DRAFT-to-SENT transition. Spec has explicit `expect(cancelSpy).not.toHaveBeenCalled()` assertion.

### Test Coverage (18 tests)

- Happy path: SENT source creates DRAFT with revisionNumber + 1
- 4 allowed statuses via `it.each`
- 6 disallowed statuses via `it.each` (ConflictError on filter miss)
- D-HOOK-03 non-call assertion
- Compensating rollback on insertRevision failure (restoreCurrent only)
- Compensating rollback on line-item clone failure (softDeleteRow + restoreCurrent with ordering)
- Bundle parent/child line-item clone (2 roots, 2 children, parentLineItemId remapped)
- Revision number monotonicity (N=5 creates N=6)
- Atomic filter shape verification
- DI injection resolution

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed mock generator createRoot overriding caller's revisionNumber**
- **Found during:** Task 2 (monotonicity test failing)
- **Issue:** `createRoot` spread `...overrides` before hardcoded `revisionNumber: 1`, so caller's override was always overwritten
- **Fix:** Changed to `revisionNumber: overrides.revisionNumber ?? 1` with `...overrides` spread first, structural invariants after
- **Files modified:** `estimate-revision-mock-generator.ts`
- **Commit:** `258e391`

**2. [Rule 3 - Blocking] Adapted to actual mock generator method name**
- **Found during:** Task 1
- **Issue:** Plan referenced `EstimateLineItemMockGenerator.createLineItemDto` but actual method is `createEstimateLineItemDto`
- **Fix:** Used correct method name throughout
- **Files modified:** `estimate-revision-mock-generator.ts`

**3. [Rule 3 - Blocking] Used undefined instead of null for optional DTO fields**
- **Found during:** Task 2
- **Issue:** Plan skeleton used `null` for fields like `sentAt`, `firstViewedAt`, etc. but `IEstimateDto` types them as `?: DateTime` (not `DateTime | null`)
- **Fix:** Used `undefined` for all optional DateTime/enum/string fields in `buildNewRevisionDto`
- **Files modified:** `estimate-reviser.service.ts`

## Self-Check: PASSED

All 3 created files verified on disk. Both commits (0aa955b, 258e391) verified in trade-flow-api git log.
