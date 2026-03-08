# Phase 10: Item Tax Rate UI - Context

**Gathered:** 2026-03-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Users select tax rates from a dropdown when creating or editing items, and see linked tax rate details when viewing items. Tax rate dropdown replaces the numeric tax input on Material, Labour, and Fee item forms. No changes to item list/card display -- tax rate is only visible in the edit form.

</domain>

<decisions>
## Implementation Decisions

### Dropdown format
- Display format: "Name (rate%)" -- e.g., "Standard (20%)", "Reduced (5%)", "Zero Rated (0%)"
- No tax rate type badge (percentage vs fixed) -- just name and rate
- Only show active tax rates in the dropdown options
- If an item references an archived tax rate, show it as the selected value but don't include it in the dropdown options list

### Default selection
- Create forms: pre-select the business's default tax rate (the one with `isDefault: true`)
- Edit forms: show the item's current tax rate pre-selected
- Saves a click for the common case on create

### Tax rate in lists
- No tax rate column in the items table (desktop)
- No tax rate display on item cards (mobile)
- Tax rate is only visible when editing an item -- it's a secondary detail
- No changes to ItemsTable or ItemsCardList components

### Form layout
- Keep the existing 2-column side-by-side grid: Price input on left, Tax Rate dropdown on right
- Label changes from "Tax" to "Tax Rate"
- Identical dropdown treatment across all three forms: Material, Labour, and Fee
- Bundle form: no tax rate field (hidden) -- bundles have taxRateId: null, tax computed from component items at quote time

### Missing tax rates
- If no active tax rates exist, show the dropdown as empty/disabled with an inline warning: "No tax rates found. Create one in Settings." with a link to tax rates settings page
- Same warning treatment whether tax rates never existed or all have been archived (dropdown filters to active only)
- Form can still be submitted for bundles (which don't require a tax rate)

### Claude's Discretion
- Exact Select component implementation (shadcn Select vs Combobox)
- Form validation schema updates (Valibot schema changes for taxRateId replacing defaultTaxRate)
- TypeScript type updates for Item, CreateItemRequest, UpdateItemRequest
- How to fetch and cache tax rates in the form (likely useGetTaxRatesQuery from existing taxRateApi)
- RTK Query cache strategy for resolving tax rate details

</decisions>

<specifics>
## Specific Ideas

- Dropdown format matches how tax rates appear in the tax rates settings page -- consistent terminology
- Pre-selecting the default tax rate follows the same pattern as onboarding default items (Phase 9 decision)

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `useGetTaxRatesQuery(businessId)`: Already exists in `taxRateApi.ts` -- returns `TaxRate[]` with name, rate, type, isDefault
- shadcn `Select` component: Available in `components/ui/select.tsx` -- standard dropdown pattern
- `FormField` component: Existing form field wrapper -- may need a `FormSelect` equivalent or inline Select usage
- `useItemForm` hook: Shared form logic in `forms/shared/useItemForm.ts` -- needs taxRateId support

### Established Patterns
- Item forms use `react-hook-form` with `FormProvider` and Valibot schemas
- Forms use `FormField<ItemFormValues>` for typed field rendering
- RTK Query hooks used directly in components for data fetching
- Price and Tax currently in `grid grid-cols-2 gap-4` layout

### Integration Points
- `MaterialItemForm.tsx`, `LabourItemForm.tsx`, `FeeItemForm.tsx`: Replace numeric `defaultTaxRate` FormField with tax rate Select dropdown
- `ItemFormDialog.tsx`: `handleSubmit` currently maps `defaultTaxRate` -- needs to map `taxRateId` instead
- `itemSchema.ts`: Valibot schema needs `taxRateId` (string) replacing `defaultTaxRate` (number)
- `api.types.ts` or `@/types`: Item type needs `taxRateId` field, remove `defaultTaxRate`
- `CreateItemRequest` / `UpdateItemRequest`: Field name change to `taxRateId`

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 10-item-tax-rate-ui*
*Context gathered: 2026-03-08*
