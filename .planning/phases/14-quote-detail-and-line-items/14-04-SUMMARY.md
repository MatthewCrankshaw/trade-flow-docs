---
phase: 14-quote-detail-and-line-items
plan: 04
subsystem: ui, api
tags: [bundle-pricing, popover-width, css-grid, price-strategy, rtk-query]

requires:
  - phase: 14-quote-detail-and-line-items
    provides: Quote line item CRUD, soft delete, totals calculator, bundle line item factory

provides:
  - Wider SearchableItemPicker popover for better readability
  - Grid-aligned bundle component rows matching parent table columns
  - User-selectable pricing strategy (component-based vs fixed) when adding bundles to quotes
  - priceStrategy parameter threaded from UI through API to QuoteBundleLineItemFactory

affects: [quote-detail, line-items, bundles, item-picker]

tech-stack:
  added: []
  patterns: [css-grid-column-alignment, two-step-bundle-add-flow]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/components/SearchableItemPicker.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
    - trade-flow-api/src/quote/requests/create-quote-line-item.request.ts
    - trade-flow-api/src/quote/services/quote-updater.service.ts
    - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
    - trade-flow-api/src/quote/controllers/quote.controller.ts
    - trade-flow-ui/src/features/quotes/api/quoteApi.ts
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx

key-decisions:
  - "CSS grid with gridTemplateColumns for bundle component alignment instead of flex"
  - "Two-step add flow for bundles overrides locked immediate-add decision per UAT gap diagnosis"
  - "priceStrategy override in factory uses spread to merge with bundleConfig"

patterns-established:
  - "Two-step bundle add: select pricing strategy then confirm (bundles only, non-bundles immediate)"

requirements-completed: [QUOT-03, QLIT-01, QLIT-03]

duration: 3min
completed: 2026-03-14
---

# Phase 14 Plan 04: UI Polish and Bundle Pricing Strategy Summary

**Widened item picker popover, grid-aligned bundle component rows, and added pricing strategy selector for bundle items**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-14T20:29:43Z
- **Completed:** 2026-03-14T20:33:39Z
- **Tasks:** 3
- **Files modified:** 8

## Accomplishments
- Widened SearchableItemPicker popover from 320px to 384px for better item name/badge/price display
- Replaced flex layout with CSS grid for bundle component rows, achieving uniform column alignment with parent table
- Added optional priceStrategy parameter to addLineItem API endpoint, threaded through controller/service/factory
- Built pricing strategy selector UI for bundle items with component-based and fixed price options

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix SearchableItemPicker width and bundle component column alignment** - `bcce165` (fix) [trade-flow-ui]
2. **Task 2: Add priceStrategy parameter to addLineItem API endpoint** - `6c2e436` (feat) [trade-flow-api]
3. **Task 3: Add pricing strategy selector to QuoteLineItemsCard for bundle items** - `ff318e7` (feat) [trade-flow-ui]

## Files Created/Modified
- `trade-flow-ui/src/components/SearchableItemPicker.tsx` - Widened popover from w-80 to w-96
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` - CSS grid layout for bundle component rows
- `trade-flow-api/src/quote/requests/create-quote-line-item.request.ts` - Added optional priceStrategy field
- `trade-flow-api/src/quote/services/quote-updater.service.ts` - Threaded priceStrategy through addLineItem
- `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` - priceStrategy override in params and effective config
- `trade-flow-api/src/quote/controllers/quote.controller.ts` - Passed priceStrategy from request to service
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` - Added priceStrategy to addLineItem mutation type
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` - Bundle pricing strategy selector UI

## Decisions Made
- CSS grid with explicit gridTemplateColumns for bundle component alignment (better than flex for matching parent table column widths)
- Two-step add flow for bundles intentionally overrides CONTEXT.md locked immediate-add decision per UAT gap diagnosis -- bundles require user input on pricing mode
- priceStrategy override uses object spread to merge with bundleConfig, keeping default behavior when no override

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- Source code lives in separate git repos (trade-flow-ui, trade-flow-api) from the planning repo -- commits made in respective repos
- Prettier formatting required single-line ternary for effectiveConfig assignment (auto-fixed during Task 2)

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All UAT gap closure plans (14-01 through 14-04) complete
- Bundle pricing strategy selection operational end-to-end
- UI cosmetic issues resolved

---
*Phase: 14-quote-detail-and-line-items*
*Completed: 2026-03-14*
