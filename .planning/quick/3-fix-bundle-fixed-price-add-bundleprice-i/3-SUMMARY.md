---
phase: quick
plan: 3
subsystem: ui, api
tags: [valibot, react-hook-form, class-validator, bundle-pricing]

requires:
  - phase: v1.2 Bundles & Quotes
    provides: Bundle item form and API request DTOs

provides:
  - Conditional bundlePrice input field in BundleItemForm
  - Valibot schema validation for bundlePrice when priceStrategy is fixed
  - Backend @IsDefined enforcement for bundlePrice when FIXED

affects: [bundles, quotes, items]

tech-stack:
  added: []
  patterns:
    - "Conditional form field visibility driven by useWatch on priceStrategy"
    - "Forward validation check in Valibot for cross-field dependencies"

key-files:
  created: []
  modified:
    - trade-flow-ui/src/lib/forms/schemas/item.schema.ts
    - trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
    - trade-flow-api/src/item/requests/create-item.request.ts
    - trade-flow-api/src/item/requests/update-item.request.ts

key-decisions:
  - "Used v.forward check to validate bundlePrice conditionally rather than custom pipe"

patterns-established: []

requirements-completed: [QUICK-3]

duration: 9min
completed: 2026-03-15
---

# Quick Task 3: Fix Bundle Fixed-Price -- Add bundlePrice Input Summary

**Conditional bundlePrice input in BundleItemForm with Valibot cross-field validation and backend @IsDefined enforcement**

## Performance

- **Duration:** 9 min
- **Started:** 2026-03-15T13:15:45Z
- **Completed:** 2026-03-15T13:25:00Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Bundle item form now shows a price input field when "Fixed price" strategy is selected
- Valibot schema enforces bundlePrice is a valid positive number when priceStrategy is "fixed"
- Backend rejects null bundlePrice for FIXED strategy via @IsDefined decorator
- Editing existing fixed-price bundles pre-populates the price field via useCurrencyConversion

## Task Commits

Each task was committed atomically:

1. **Task 1: Add bundlePrice to schema and form UI** - `ef4ec77` (feat) [trade-flow-ui repo]
2. **Task 2: Enforce non-null bundlePrice in backend validation** - `fcbae3c` (fix) [trade-flow-api repo]

## Files Created/Modified
- `trade-flow-ui/src/lib/forms/schemas/item.schema.ts` - Added bundlePrice field and forward validation check
- `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` - Added conditional price input, currency conversion, useWatch for priceStrategy
- `trade-flow-api/src/item/requests/create-item.request.ts` - Added @IsDefined to bundlePrice in CreateBundleConfigRequest
- `trade-flow-api/src/item/requests/update-item.request.ts` - Added @IsDefined to bundlePrice in UpdateBundleConfigRequest

## Decisions Made
- Used `v.forward` check pattern (consistent with existing component checks) for cross-field bundlePrice validation
- Kept `number | null` type on backend bundlePrice to allow null when component_based (ValidateIf skips validation)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

---
*Quick Task: 3*
*Completed: 2026-03-15*
