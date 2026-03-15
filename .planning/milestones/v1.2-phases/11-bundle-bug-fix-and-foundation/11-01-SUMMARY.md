---
phase: 11-bundle-bug-fix-and-foundation
plan: 01
subsystem: ui
tags: [react, shadcn, combobox, popover, command, bundle]

requires:
  - phase: 10-item-tax-rate-linkage
    provides: Items feature with BundleItemForm and BundleComponentsList
provides:
  - Bundle creation bug fix (unit: "bundle" in submit data)
  - SearchableItemPicker reusable component (Popover+Command combobox)
  - BundleComponentsList refactored to use SearchableItemPicker
affects: [12-bundle-component-editing, 13-quote-builder]

tech-stack:
  added: []
  patterns: [Popover+Command combobox with shouldFilter=false and manual useMemo filtering]

key-files:
  created:
    - trade-flow-ui/src/components/SearchableItemPicker.tsx
  modified:
    - trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
    - trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx

key-decisions:
  - "Items pre-selected via picker eliminates empty itemId intermediate state"
  - "SearchableItemPicker placed in src/components/ for cross-feature reuse"

patterns-established:
  - "SearchableItemPicker: Popover+Command combobox with grouped items, type badges, and price display"
  - "Item exclusion pattern: excludeItemIds prop filters already-selected items"

requirements-completed: [BNDL-01, BNDL-03]

duration: 2min
completed: 2026-03-08
---

# Phase 11 Plan 01: Bundle Bug Fix and SearchableItemPicker Summary

**Fixed bundle creation API rejection by adding unit field, built SearchableItemPicker with grouped Popover+Command combobox pattern**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-08T17:11:55Z
- **Completed:** 2026-03-08T17:14:17Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Fixed bundle creation bug: API was rejecting bundle items due to missing `unit: "bundle"` field
- Built SearchableItemPicker component with Popover+Command pattern, items grouped by type (Materials, Labour, Fees), type badges, and formatted prices
- Integrated SearchableItemPicker into BundleComponentsList, replacing the Select dropdown with a searchable grouped combobox

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix bundle unit bug and create SearchableItemPicker** - `1b9bd6e` (feat)
2. **Task 2: Integrate SearchableItemPicker into BundleComponentsList** - `6b90d7e` (feat)

## Files Created/Modified
- `trade-flow-ui/src/components/SearchableItemPicker.tsx` - Reusable searchable item picker with grouped Popover+Command combobox
- `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` - Added unit: "bundle" to submit data; updated handleAddComponent to accept itemId
- `trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx` - Replaced Select with SearchableItemPicker; simplified create-mode component rows

## Decisions Made
- Items are pre-selected via the picker, eliminating the empty `itemId: ""` intermediate state that previously existed when adding a component row
- SearchableItemPicker placed in `src/components/` (not features/) for cross-feature reuse in quotes

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- SearchableItemPicker ready for reuse in quote builder (Phase 13)
- Bundle creation flow works end-to-end
- Phase 12 (bundle component editing) can build on updated BundleComponentsList

---
*Phase: 11-bundle-bug-fix-and-foundation*
*Completed: 2026-03-08*
