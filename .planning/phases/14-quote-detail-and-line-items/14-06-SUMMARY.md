---
phase: 14-quote-detail-and-line-items
plan: 06
subsystem: ui
tags: [react, valibot, shadcn, radio-group, bundle, pricing-strategy]

requires:
  - phase: 14-04-quote-detail-and-line-items
    provides: Bundle pricing strategy two-step add flow (now removed)
provides:
  - Immediate add for all item types including bundles in quote flow
  - priceStrategy radio group in BundleItemForm for item-level pricing config
affects: [quotes, items, bundles]

tech-stack:
  added: [shadcn radio-group]
  patterns: [Controller-based RadioGroup for valibot-validated forms]

key-files:
  created:
    - trade-flow-ui/src/components/ui/radio-group.tsx
  modified:
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx
    - trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
    - trade-flow-ui/src/lib/forms/schemas/item.schema.ts

key-decisions:
  - "Pricing strategy is an item-level setting, not a per-quote decision"
  - "API uses bundleConfig.priceStrategy when no override sent from quote UI"

patterns-established:
  - "Controller + RadioGroup pattern for valibot picklist fields in forms"

requirements-completed: [QUOT-03, QLIT-01]

duration: 2min
completed: 2026-03-14
---

# Phase 14 Plan 06: Bundle Pricing Strategy Relocation Summary

**Moved bundle pricing strategy from quote add-item flow to BundleItemForm with immediate-add restored for all item types**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-14T21:03:38Z
- **Completed:** 2026-03-14T21:05:39Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Removed two-step bundle add flow from QuoteLineItemsCard -- all items now add immediately
- Added priceStrategy radio group to BundleItemForm with component_based and fixed options
- Added priceStrategy picklist validation to bundleItemFormSchema

## Task Commits

Each task was committed atomically:

1. **Task 1: Remove bundle pricing selector from QuoteLineItemsCard** - `e5346d3` (feat)
2. **Task 2: Add priceStrategy field to BundleItemForm and schema** - `32b2d9c` (feat)

## Files Created/Modified
- `trade-flow-ui/src/components/ui/radio-group.tsx` - shadcn RadioGroup component (new)
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` - Removed bundle pricing selector, simplified handleAddItem
- `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` - Added priceStrategy defaultValue, RadioGroup UI, dynamic submit
- `trade-flow-ui/src/lib/forms/schemas/item.schema.ts` - Added priceStrategy picklist to bundleItemFormSchema

## Decisions Made
- Pricing strategy is an item-level setting, not a per-quote decision -- API reads from bundleConfig automatically
- Used Controller + RadioGroup pattern (consistent with shadcn/react-hook-form integration)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Installed missing shadcn radio-group component**
- **Found during:** Task 2 (BundleItemForm priceStrategy UI)
- **Issue:** radio-group.tsx did not exist in components/ui
- **Fix:** Ran `npx shadcn@latest add radio-group`
- **Files modified:** src/components/ui/radio-group.tsx (created)
- **Verification:** Import resolves, typecheck passes
- **Committed in:** 32b2d9c (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Expected dependency installation. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Bundle pricing strategy now fully configurable from item management screen
- Quote add-item flow is clean and immediate for all item types
- Gap closure plan complete

## Self-Check: PASSED

All files exist. All commits verified.

---
*Phase: 14-quote-detail-and-line-items*
*Completed: 2026-03-14*
