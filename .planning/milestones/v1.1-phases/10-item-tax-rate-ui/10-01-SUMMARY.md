---
phase: 10-item-tax-rate-ui
plan: 01
subsystem: ui
tags: [react, valibot, rtk-query, typescript, forms]

requires:
  - phase: 09-item-tax-rate-api
    provides: "Item API with taxRateId field replacing defaultTaxRate"
provides:
  - "Item, CreateItemRequest, UpdateItemRequest types with taxRateId field"
  - "Valibot schema with taxRateId optional string field"
  - "useItemForm hook with tax rate fetching, default selection, and submission mapping"
affects: [10-item-tax-rate-ui]

tech-stack:
  added: []
  patterns:
    - "Tax rate fetching in form hooks via useGetTaxRatesQuery"
    - "Active tax rate filtering for dropdowns (status === enabled)"

key-files:
  created: []
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/lib/forms/schemas/item.schema.ts
    - trade-flow-ui/src/features/items/components/forms/shared/useItemForm.ts

key-decisions:
  - "Filter tax rates to active-only for dropdown options in the hook, not in the component"
  - "Pre-select default tax rate on create using taxRates.find(tr => tr.isDefault)"

patterns-established:
  - "Tax rate selection: useItemForm returns activeTaxRates and isTaxRatesLoading for form components"

requirements-completed: [ITMUI-01, ITMUI-02]

duration: 2min
completed: 2026-03-08
---

# Phase 10 Plan 01: Item Data Contracts Summary

**Item types, form schema, and useItemForm hook updated from defaultTaxRate (number) to taxRateId (string) with tax rate fetching and default selection**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-08T14:10:46Z
- **Completed:** 2026-03-08T14:13:00Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Item, CreateItemRequest, and UpdateItemRequest types use taxRateId (string) instead of defaultTaxRate (number)
- Valibot schema simplified to optional string for taxRateId (removed numeric validation)
- useItemForm hook fetches tax rates, pre-selects default on create, maps taxRateId on submit
- Hook returns activeTaxRates and isTaxRatesLoading for downstream form components

## Task Commits

Each task was committed atomically:

1. **Task 1: Update Item types to use taxRateId** - `d0651a3` (feat)
2. **Task 2: Update item form schema and useItemForm hook for taxRateId** - `53aa835` (feat)

## Files Created/Modified
- `trade-flow-ui/src/types/api.types.ts` - Item, CreateItemRequest, UpdateItemRequest with taxRateId field
- `trade-flow-ui/src/lib/forms/schemas/item.schema.ts` - Valibot schema with taxRateId optional string
- `trade-flow-ui/src/features/items/components/forms/shared/useItemForm.ts` - Form hook with tax rate fetching and submission mapping

## Decisions Made
- Filter tax rates to active-only (status === "enabled") in the hook rather than in each form component
- Pre-select default tax rate on create using `taxRates.find(tr => tr.isDefault)?.id`
- Removed `bundleConfig: null` from useItemForm submit mapping since bundles use a separate form

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Data contracts are ready for Plan 02 to build the tax rate dropdown UI components
- Form components (MaterialItemForm, LabourItemForm, FeeItemForm) will consume activeTaxRates from useItemForm

---
*Phase: 10-item-tax-rate-ui*
*Completed: 2026-03-08*
