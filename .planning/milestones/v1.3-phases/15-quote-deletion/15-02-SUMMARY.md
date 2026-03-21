---
phase: 15-quote-deletion
plan: 02
subsystem: ui
tags: [react, rtk-query, optimistic-update, dropdown-menu, alert-dialog, soft-delete]

# Dependency graph
requires:
  - phase: 15-01
    provides: DELETED status enum, DRAFT->DELETED transition, deletedAt timestamp, list exclusion filter
provides:
  - deleteQuote RTK Query mutation with optimistic cache removal
  - Delete button on quote detail page action strip (draft only)
  - Row-level three-dot dropdown on quote list table and card views (draft only)
  - Confirmation dialog with destructive styling for delete actions
  - Post-delete navigation (detail page) and optimistic row removal (list page)
affects: [quote-ui, quote-detail, quote-list]

# Tech tracking
tech-stack:
  added: []
  patterns: [optimistic-delete-with-rollback, row-level-dropdown-menu, parent-owned-confirmation-dialog]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/types/quote.ts
    - trade-flow-ui/src/features/quotes/api/quoteApi.ts
    - trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
    - trade-flow-ui/src/features/quotes/components/QuotesTable.tsx
    - trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx
    - trade-flow-ui/src/features/quotes/components/QuotesDataView.tsx
    - trade-flow-ui/src/pages/QuotesPage.tsx
    - trade-flow-ui/src/pages/QuoteDetailPage.tsx
    - trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx

key-decisions:
  - "Confirmation dialog owned by parent (QuotesPage) for list deletes, by QuoteActionStrip for detail page deletes"
  - "Used buttonVariants({ variant: 'destructive' }) for red delete button in AlertDialogAction"

patterns-established:
  - "Row-level dropdown: DropdownMenu with stopPropagation on trigger and menu items to prevent row click navigation"
  - "Optimistic delete: RTK Query onQueryStarted filters item from cache, rolls back on error"

requirements-completed: [QMGT-01]

# Metrics
duration: 4min
completed: 2026-03-15
---

# Phase 15 Plan 02: Quote Deletion UI Summary

**Delete UI for draft quotes with detail page button, list row dropdown, confirmation dialog, optimistic removal, and toast notifications**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-15T17:54:17Z
- **Completed:** 2026-03-15T17:58:48Z
- **Tasks:** 2
- **Files modified:** 9

## Accomplishments
- deleteQuote RTK Query mutation with optimistic cache removal and rollback on API error
- Delete button on detail page action strip (draft only) with confirmation dialog and post-delete navigation
- Row-level three-dot dropdown on both table and card list views with delete option for draft quotes
- Parent-level confirmation dialog on QuotesPage shared between table and card list delete flows
- "deleted" added defensively to all statusColors maps across the app (4 locations)

## Task Commits

Each task was committed atomically:

1. **Task 1: Add deleteQuote mutation and update QuoteStatus type** - `ac2148c` (feat)
2. **Task 2: Add delete action to detail page and row-level dropdown to list** - `bc8336d` (feat)

## Files Created/Modified
- `trade-flow-ui/src/types/quote.ts` - Added "deleted" to QuoteStatus, deletedAt to Quote interface
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` - deleteQuote mutation with optimistic cache update
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` - Delete button and confirmation dialog for detail page
- `trade-flow-ui/src/features/quotes/components/QuotesTable.tsx` - Row-level dropdown with delete option
- `trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx` - Card-level dropdown with delete option
- `trade-flow-ui/src/features/quotes/components/QuotesDataView.tsx` - Pass onDeleteQuote callback through
- `trade-flow-ui/src/pages/QuotesPage.tsx` - Parent delete confirmation dialog and mutation wiring
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` - Added "deleted" to statusColors map
- `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` - Added "deleted" to quoteStatusColors map

## Decisions Made
- Confirmation dialog owned by parent (QuotesPage) for list deletes to share between table and card views; owned by QuoteActionStrip for detail page deletes since navigation occurs after
- Used `buttonVariants({ variant: "destructive" })` className on AlertDialogAction for red delete button styling

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Added "deleted" to JobDetailTabs quoteStatusColors map**
- **Found during:** Task 1 (typecheck)
- **Issue:** JobDetailTabs.tsx had a `Record<QuoteStatus, ...>` statusColors map missing the new "deleted" entry, causing TypeScript error
- **Fix:** Added `deleted: "destructive"` to the quoteStatusColors map
- **Files modified:** trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx
- **Verification:** typecheck passes
- **Committed in:** ac2148c (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential for TypeScript correctness after type union expansion. No scope creep.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Quote deletion feature complete end-to-end (API + UI)
- Ready for any additional quote management features
- All lint and typecheck passing on both UI and API

---
*Phase: 15-quote-deletion*
*Completed: 2026-03-15*
