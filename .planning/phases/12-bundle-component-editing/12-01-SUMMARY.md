---
phase: 12-bundle-component-editing
plan: 01
subsystem: api
tags: [nestjs, class-validator, bundle, validation, typescript]

requires:
  - phase: 11-bundle-bug-fix-and-foundation
    provides: Bundle creation with component validation in ItemCreatorService
provides:
  - Bundle component editing via PATCH /items/:id with full replacement semantics
  - UpdateBundleConfigRequest with optional components array
  - ItemUpdaterService bundle validation (count, quantity, active, non-bundle)
  - Frontend UpdateItemRequest type with components in bundleConfig
affects: [12-bundle-component-editing]

tech-stack:
  added: []
  patterns: [reuse CreateBundleComponentRequest in update path, full replacement for nested arrays]

key-files:
  created: []
  modified:
    - trade-flow-api/src/item/requests/update-item.request.ts
    - trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts
    - trade-flow-api/src/item/services/item-updater.service.ts
    - trade-flow-api/src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts
    - trade-flow-api/src/item/test/services/item-updater.service.spec.ts
    - trade-flow-ui/src/types/api.types.ts

key-decisions:
  - "Reused CreateBundleComponentRequest for update validation (same shape, same rules)"
  - "Always validate bundle components on update (consistent with taxRateId always-validate pattern)"

patterns-established:
  - "Full replacement semantics: nested arrays in update requests replace entirely, not merge"
  - "Replicate validation from creator service into updater service for bundle items"

requirements-completed: [BNDL-02]

duration: 4min
completed: 2026-03-08
---

# Phase 12 Plan 01: Bundle Component Update API Summary

**API update endpoint extended with bundle component array, full replacement merge, and validation mirroring creation rules**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-08T19:30:46Z
- **Completed:** 2026-03-08T19:35:08Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- UpdateBundleConfigRequest accepts optional components array with nested validation via CreateBundleComponentRequest
- mergeBundleConfig handles components with full replacement when provided, keeps existing when undefined
- ItemUpdaterService validates bundle components on update (count 1-100, quantity > 0, active items, non-bundle)
- Frontend UpdateItemRequest type updated to include components in bundleConfig

## Task Commits

Each task was committed atomically:

1. **Task 1: Extend API update request, merge utility, and updater validation** - `d0aa11e` (test) + `9549b15` (feat)
2. **Task 2: Update frontend UpdateItemRequest type** - `2c79100` (feat)

_Note: Task 1 used TDD (RED: d0aa11e, GREEN: 9549b15)_

## Files Created/Modified
- `trade-flow-api/src/item/requests/update-item.request.ts` - Added optional components array to UpdateBundleConfigRequest
- `trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts` - Components merge with full replacement semantics
- `trade-flow-api/src/item/services/item-updater.service.ts` - Bundle validation (validateBundleItem, validateComponentQuantities, validateBundleComponents)
- `trade-flow-api/src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts` - 3 new merge tests for components
- `trade-flow-api/src/item/test/services/item-updater.service.spec.ts` - 6 new updater tests for bundle validation
- `trade-flow-ui/src/types/api.types.ts` - Added optional components to UpdateItemRequest.bundleConfig

## Decisions Made
- Reused CreateBundleComponentRequest for update validation rather than creating a separate class (same shape, same rules, per research recommendation)
- Always validate bundle components on update even when unchanged, consistent with existing taxRateId always-validate pattern from Phase 9

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed existing test mock for bundle item with new validation**
- **Found during:** Task 1 (GREEN phase)
- **Issue:** Existing test "should skip validation when updating a bundle item (taxRateId is null)" created a bundle DTO with components but itemRepository mock lacked findByIdOrFail, causing TypeError
- **Fix:** Added findByIdOrFail to itemRepository mock setup and provided component item mock for the existing bundle test
- **Files modified:** trade-flow-api/src/item/test/services/item-updater.service.spec.ts
- **Verification:** All 30 item tests pass
- **Committed in:** 9549b15 (Task 1 GREEN commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Necessary fix for test compatibility with new validation. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- API accepts and validates bundle components on update, ready for Plan 02 (UI bundle component editing form)
- Frontend type already aligned, BundleItemForm can send components through update mutation

## Self-Check: PASSED

All files verified present. All commits verified in git history:
- `d0aa11e` in trade-flow-api
- `9549b15` in trade-flow-api
- `2c79100` in trade-flow-ui

---
*Phase: 12-bundle-component-editing*
*Completed: 2026-03-08*
