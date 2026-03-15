---
phase: 14-quote-detail-and-line-items
plan: 05
subsystem: ui
tags: [react, table-layout, tax-calculation, shadcn]

requires:
  - phase: 14-04
    provides: Bundle component display in QuoteLineItemsTable with CSS grid layout
provides:
  - Aligned bundle component sub-rows using native table layout
  - Tax-inclusive line total display across desktop and mobile views
affects: [quote-detail]

tech-stack:
  added: []
  patterns: [lineTotalIncTax helper for consistent tax-inclusive display]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx

key-decisions:
  - "Replaced CSS grid colSpan approach with native TableRow sub-rows for automatic column alignment"
  - "lineTotalIncTax helper centralises tax-inclusive calculation to avoid duplication"

patterns-established:
  - "lineTotalIncTax helper: item.lineTotal * (1 + item.taxRate / 100) for display"

requirements-completed: [QLIT-03, QLIT-04]

duration: 2min
completed: 2026-03-14
---

# Phase 14 Plan 05: Bundle Alignment and Tax-Inclusive Totals Summary

**Native table sub-rows for bundle component column alignment and lineTotalIncTax helper for tax-inclusive line totals in desktop and mobile views**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-14T21:03:31Z
- **Completed:** 2026-03-14T21:05:14Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments
- Bundle component rows now render as proper TableRow elements, aligning with parent table header columns via native table layout
- All line total display sites use tax-inclusive amounts via lineTotalIncTax helper
- Column header updated to "Total (inc. tax)" for clarity
- Mobile CardList view also displays tax-inclusive totals

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix bundle component alignment with table sub-rows and add tax-inclusive line totals** - `b3ef6ec` (fix)

## Files Created/Modified
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` - Replaced CSS grid component rows with TableRow sub-rows; added lineTotalIncTax helper; updated column header
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` - Added lineTotalIncTax helper for tax-inclusive display in mobile cards

## Decisions Made
- Replaced CSS grid inside colSpan with actual TableRow elements -- native table layout algorithm handles column alignment automatically without maintaining grid column widths
- Created lineTotalIncTax helper in each component rather than a shared utility since it is a simple one-liner used in only two files

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All gap closure plans for Phase 14 are now complete
- Bundle component alignment and tax-inclusive totals verified via lint and typecheck

---
*Phase: 14-quote-detail-and-line-items*
*Completed: 2026-03-14*
