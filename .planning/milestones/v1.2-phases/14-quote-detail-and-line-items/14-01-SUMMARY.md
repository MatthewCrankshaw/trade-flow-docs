---
phase: 14-quote-detail-and-line-items
plan: 01
subsystem: api, ui
tags: [nestjs, rtk-query, mongodb, quote-line-items, searchable-picker]

requires:
  - phase: 13-quote-api-integration
    provides: Quote CRUD API, quote types, RTK Query base endpoints
provides:
  - PATCH and DELETE API endpoints for quote line items
  - QuoteLineItem UI type matching API response shape
  - RTK Query mutations for addLineItem, updateLineItem, deleteLineItem
  - SearchableItemPicker bundle support via includeBundles prop
  - MongoDbWriter deleteOne/deleteMany methods
affects: [14-02-quote-detail-ui]

tech-stack:
  added: []
  patterns: [partial-update-dto, cascade-delete-bundle-components, backward-compatible-prop-extension]

key-files:
  created:
    - trade-flow-api/src/quote/requests/update-quote-line-item.request.ts
  modified:
    - trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts
    - trade-flow-api/src/quote/repositories/quote-line-item.repository.ts
    - trade-flow-api/src/quote/services/quote-updater.service.ts
    - trade-flow-api/src/quote/controllers/quote.controller.ts
    - trade-flow-ui/src/types/quote.ts
    - trade-flow-ui/src/types/index.ts
    - trade-flow-ui/src/features/quotes/api/quoteApi.ts
    - trade-flow-ui/src/components/SearchableItemPicker.tsx

key-decisions:
  - "Totals not persisted in quote entity -- recalculated from line items on every read (existing pattern)"
  - "Bundle component deletion cascades from parent -- cannot delete components individually"
  - "Line item update recalculates lineTotal and discountAmount server-side from quantity and unitPrice"

patterns-established:
  - "Cascade delete: bundle parent deletion triggers deleteByParentLineItemId before parent delete"
  - "Backward-compatible prop extension: includeBundles defaults to false preserving existing behavior"

requirements-completed: [QLIT-01, QLIT-02, QLIT-04]

duration: 5min
completed: 2026-03-14
---

# Phase 14 Plan 01: API and Data Foundation Summary

**Quote line item CRUD endpoints (POST/PATCH/DELETE) with cascade bundle deletion, typed QuoteLineItem interface, RTK Query mutation hooks, and bundle-aware SearchableItemPicker**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-14T19:34:22Z
- **Completed:** 2026-03-14T19:39:11Z
- **Tasks:** 2
- **Files modified:** 9

## Accomplishments
- PATCH and DELETE API endpoints for quote line items with access control and totals recalculation
- MongoDbWriter extended with deleteOne/deleteMany for cascade bundle component deletion
- QuoteLineItem type exported from UI types barrel, Quote interface includes lineItems array
- Three RTK Query mutation hooks: useAddLineItemMutation, useUpdateLineItemMutation, useDeleteLineItemMutation
- SearchableItemPicker shows bundles when includeBundles={true}, backward compatible

## Task Commits

Each task was committed atomically:

1. **Task 1: Add update and delete line item API endpoints** - `d194c56` (feat) [trade-flow-api]
2. **Task 2: Add UI types, RTK Query mutations, and SearchableItemPicker bundle support** - `9cfbae0` (feat) [trade-flow-ui]

## Files Created/Modified
- `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` - Added deleteOne/deleteMany methods
- `trade-flow-api/src/quote/requests/update-quote-line-item.request.ts` - Validation DTO for partial line item updates
- `trade-flow-api/src/quote/repositories/quote-line-item.repository.ts` - Added findByIdOrFail, update, delete, deleteByParentLineItemId
- `trade-flow-api/src/quote/services/quote-updater.service.ts` - Added updateLineItem and deleteLineItem methods
- `trade-flow-api/src/quote/controllers/quote.controller.ts` - Added PATCH and DELETE endpoints
- `trade-flow-ui/src/types/quote.ts` - Added QuoteLineItem interface and lineItems field on Quote
- `trade-flow-ui/src/types/index.ts` - Exported QuoteLineItem type
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` - Added addLineItem, updateLineItem, deleteLineItem mutations
- `trade-flow-ui/src/components/SearchableItemPicker.tsx` - Added includeBundles prop and bundle group support

## Decisions Made
- Totals not persisted in quote entity -- recalculated from line items on every read (consistent with existing addLineItem pattern)
- Bundle component deletion cascades from parent -- cannot delete components individually (per CONTEXT: components are locked)
- Line item update recalculates lineTotal and discountAmount server-side from quantity and unitPrice

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- API endpoints ready for Plan 02 UI consumption
- RTK Query hooks exported and ready for QuoteDetail page integration
- SearchableItemPicker bundle support ready for line item add dialog

---
*Phase: 14-quote-detail-and-line-items*
*Completed: 2026-03-14*
