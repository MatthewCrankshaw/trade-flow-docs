---
phase: 42-revisions
plan: 05
subsystem: api
tags: [nestjs, estimate, retriever, deleter, revision-chain, service]

requires:
  - phase: 42-03
    provides: "Repository methods: findCurrentInChainByRootId, findRevisionsByRootId, restoreCurrent, downgradeCurrent"
  - phase: 42-02
    provides: "ConflictError class for error handling"
  - phase: 41
    provides: "Phase 41 EstimateRetriever and EstimateDeleter base implementations"
provides:
  - "EstimateRetriever.findByIdOrFail with D-DET-01 non-current-to-current chain resolution"
  - "EstimateRetriever.findRevisionsByIdOrFail for revision history endpoint support"
  - "EstimateDeleter.softDelete with D-REV-05/06 predecessor restoration for non-root revisions"
affects: [42-06-controller-module-openapi]

tech-stack:
  added: []
  patterns:
    - "Single-step chain resolution (no recursion) with zero-current-window fallback"
    - "Two-step guarded write with compensation on failure for delete-rollback"

key-files:
  created: []
  modified:
    - "trade-flow-api/src/estimate/services/estimate-retriever.service.ts"
    - "trade-flow-api/src/estimate/services/estimate-deleter.service.ts"
    - "trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts"
    - "trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts"

key-decisions:
  - "Used rootEstimateId ?? target.id fallback in findRevisionsByIdOrFail for null/undefined rootEstimateId"
  - "Adapted to existing softDelete method name (plan assumed 'delete') and transition service signature (passes id string, not DTO)"
  - "Defense-in-depth: canRead re-applied on resolved current row in D-DET-01 path"

patterns-established:
  - "Non-recursive chain resolution: single findCurrentInChainByRootId call with null fallback"
  - "Predecessor restoration pattern: restoreCurrent before soft-delete with downgradeCurrent compensation"

requirements-completed: [REV-02, REV-04]

duration: 6min
completed: 2026-04-12
---

# Phase 42 Plan 05: Retriever and Deleter Extensions Summary

**EstimateRetriever gains non-current-to-current chain resolution and revision history lookup; EstimateDeleter gains predecessor restoration on non-root Draft delete with two-step guarded write**

## Performance

- **Duration:** 6 min
- **Started:** 2026-04-12T18:30:10Z
- **Completed:** 2026-04-12T18:36:06Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- EstimateRetriever.findByIdOrFail resolves non-current :id to chain's current row via single-step lookup (D-DET-01), with zero-current-window fallback returning target as-is
- EstimateRetriever.findRevisionsByIdOrFail resolves any chain member to full revision history ordered by revisionNumber ascending, with policy enforcement and totals calculation
- EstimateDeleter.softDelete handles non-root Draft revisions via restore-then-soft-delete with compensation on soft-delete failure (D-REV-05/06)
- D-DET-02 list filter verified as repository-level pass-through (no post-filtering in retriever)
- 31 total tests (20 retriever, 11 deleter) all green

## Task Commits

Each task was committed atomically:

1. **Task 1: Extend EstimateRetriever with D-DET-01, D-DET-02, and findRevisionsByIdOrFail** - `0c5481d` (feat)
2. **Task 2: Extend EstimateDeleter with D-REV-05/06 predecessor restoration** - `5466320` (feat)

_Note: TDD tasks committed as single RED+GREEN commits since implementation was straightforward_

## Files Created/Modified
- `trade-flow-api/src/estimate/services/estimate-retriever.service.ts` - Extended findByIdOrFail with D-DET-01 chain resolution; added findRevisionsByIdOrFail
- `trade-flow-api/src/estimate/services/estimate-deleter.service.ts` - Extended softDelete with D-REV-05/06 predecessor restoration for non-root revisions
- `trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` - 12 new Phase 42 spec cases (D-DET-01, D-DET-02, D-HIST-01)
- `trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` - 6 new Phase 42 spec cases (D-REV-05, root regression, parentEstimateId coercion)

## Decisions Made
- Adapted to existing `softDelete` method name rather than renaming to `delete` (plan assumed `delete` but Phase 41 produced `softDelete`)
- Used `rootEstimateId ?? target.id` fallback in findRevisionsByIdOrFail since rootEstimateId is typed as `string | null | undefined`
- Used `parentEstimateId ?? null` coercion to normalize null/undefined for root detection (per research Pitfall section 8)
- Defense-in-depth: canRead is re-applied on the resolved current row (not just the target) in D-DET-01 path

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Type narrowing for rootEstimateId**
- **Found during:** Task 1 (EstimateRetriever implementation)
- **Issue:** `rootEstimateId` is typed `string | null | undefined` in IEstimateDto, causing TS2345 when passed to repository methods expecting `string`
- **Fix:** Added truthiness guard `target.rootEstimateId` before calling `findCurrentInChainByRootId`; used `?? target.id` fallback in `findRevisionsByIdOrFail`
- **Files modified:** trade-flow-api/src/estimate/services/estimate-retriever.service.ts
- **Verification:** TypeScript compilation passes, tests green
- **Committed in:** 0c5481d

**2. [Rule 1 - Bug] Prettier formatting in retriever spec**
- **Found during:** Task 2 (lint check)
- **Issue:** Multi-line argument formatting violated prettier/prettier rule (125 char print width)
- **Fix:** Collapsed to single-line arguments
- **Files modified:** trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts
- **Verification:** eslint passes with zero errors
- **Committed in:** 5466320

---

**Total deviations:** 2 auto-fixed (1 blocking, 1 bug)
**Impact on plan:** Both auto-fixes necessary for type safety and code style compliance. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Verification Results

- `grep -c "findRevisionsByIdOrFail"` in retriever service: confirmed present
- `grep -c "findCurrentInChainByRootId"` in retriever service: confirmed present
- No recursion: no `this.findByIdOrFail` self-call in retriever service
- `grep -c "restoreCurrent"` in deleter service: confirmed present
- `grep -c "downgradeCurrent"` in deleter service: confirmed present (compensation path)
- No `as` type assertions in production code
- No eslint-disable / ts-ignore / ts-expect-error in any modified file
- 31 tests pass across both suites

## Next Phase Readiness
- EstimateRetriever and EstimateDeleter are revision-aware, ready for plan 42-06 (controller + module wiring)
- findRevisionsByIdOrFail provides the service layer for the GET /revisions endpoint
- Delete-rollback path is complete for the DELETE endpoint's non-root revision handling

## Self-Check: PASSED

- [x] 42-05-SUMMARY.md exists
- [x] estimate-retriever.service.ts exists on disk (gitignored trade-flow-api)
- [x] estimate-deleter.service.ts exists on disk (gitignored trade-flow-api)
- [x] estimate-retriever.service.spec.ts exists on disk
- [x] estimate-deleter.service.spec.ts exists on disk
- [x] 20 retriever tests pass, 11 deleter tests pass (31 total)
- [x] npm run ci exits 0

---
*Phase: 42-revisions*
*Completed: 2026-04-12*
