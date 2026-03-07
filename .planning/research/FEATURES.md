# Feature Landscape: Item-Tax Rate Linkage

**Domain:** Item/product tax rate referencing in business management and invoicing apps
**Researched:** 2026-03-07
**Competitors/References:** QuickBooks, Xero, Invoice Ninja, FreshBooks, Tradify

## Table Stakes

Features users expect. Missing = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Tax rate dropdown on item forms | Every invoicing app uses a dropdown to select from predefined tax rates, not manual numeric entry | Low | Replaces current free-text number input with a select populated from business tax rates |
| Display linked tax rate name + percentage on items | Users need to see "VAT (20%)" not just "20" -- the name provides semantic meaning | Low | Item list/detail views should show the tax rate name and percentage |
| Default tax rate pre-selection | When creating a new item, the business's default tax rate should be pre-selected in the dropdown | Low | Tax rate entity already has `isDefault` field -- use it |
| "No tax" / exempt option | Some items are tax-exempt; users need a way to explicitly set zero tax | Low | Provide a "No tax (0%)" option or allow clearing the selection |
| Disabled tax rates hidden from dropdown | Only enabled tax rates should appear in the item form dropdown | Low | Filter by `status: enabled` when populating the dropdown |

## Differentiators

Features that set product apart. Not expected, but valued.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Tax rate inline preview | Show the rate percentage next to dropdown options (e.g., "VAT - 20%", "Reduced Rate - 5%") so users can confirm without navigating away | Low | Minor UX polish that reduces errors |
| Quick-create tax rate from item form | If a needed tax rate doesn't exist, let users create one without leaving the item form | Medium | "Add new tax rate" option at bottom of dropdown -- common in QuickBooks/Xero. Defer to later milestone. |

## Anti-Features

Features to explicitly NOT build.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Multiple tax rates per item | Adds significant complexity (compound tax, tax-on-tax). Most trades businesses deal with a single tax rate per item (VAT or no VAT). Invoice Ninja supports up to 3 per item but it's overkill for sole traders. | Single tax rate reference per item. Revisit only if invoicing milestone demands compound tax. |
| Tax rate override at invoice/quote line level | The quote system already snapshots `taxRate` as a numeric value on the line item. Allowing per-line overrides at the item level adds confusion between "item default" and "actual applied rate". | Keep item tax rate as the default. Overrides happen at the quote/invoice line item level (already supported by existing `taxRate` field on quote line items). |
| Tax-inclusive pricing toggle per item | Some apps let each item specify whether its price includes or excludes tax. This is a business-wide setting, not per-item. | Handle at business level if needed later. Current system treats prices as exclusive. |
| Hard deletion of tax rates | QuickBooks does not allow deletion of tax rates -- only deactivation. Xero archives rather than deletes. Deleting a tax rate that items reference causes orphaned references. | Use existing `status: disabled` for soft-delete. Prevent disabling if items reference it, or allow it with clear warnings. |

## Feature Dependencies

```
Tax Rates (existing)     Items (existing)
     |                        |
     v                        v
Tax Rate ID Reference ------> Item Entity (replace defaultTaxRate: number with taxRateId: string)
     |                        |
     v                        v
Tax Rate Dropdown      Item Form (replace number input with select)
     |                        |
     v                        v
Display Tax Info       Item List/Detail (show tax rate name + percentage)
```

### Dependency Details

1. **API must change before UI** -- Item entity/DTO needs the `taxRateId` field before the UI can use a dropdown
2. **Tax rate list endpoint already exists** -- `GET /v1/business/:businessId/tax-rates` returns all tax rates with name, rate, rateType, status
3. **RTK Query hook already exists** -- `useGetTaxRatesQuery(businessId)` is ready to use in the item form
4. **Quote line item factory must update** -- `QuoteStandardLineItemFactory` currently reads `item.defaultTaxRate` (a number) and must be updated to resolve the tax rate from the referenced ID
5. **Bundle tax calculation must update** -- `BundleTaxRateCalculator` derives tax from component items, so it transitively depends on the new linkage

## Edge Cases: Reference Integrity

These are the critical scenarios that must be handled when items reference tax rates.

### When a referenced tax rate is disabled

**Industry standard:** QuickBooks deactivates but does not delete. Items keep the reference. The disabled tax rate still applies to existing items but cannot be selected for new items.

**Recommended approach:**
- Items that already reference a disabled tax rate continue to work -- the tax rate ID remains valid
- The dropdown for new items / editing items filters to `status: enabled` only
- When editing an item that references a disabled tax rate, show it as a special option: "VAT 20% (disabled)" so the user can see what's set but is encouraged to change it
- Do NOT automatically reassign items to a different tax rate -- that changes pricing silently

### When a tax rate's percentage changes

**Industry standard:** The change applies going forward. Historical documents (quotes, invoices) retain the rate that was in effect when created.

**Recommended approach:**
- Items reference a tax rate by ID, so they always get the current rate value
- The quote line item already snapshots the numeric `taxRate` value at creation time (existing behavior) -- this is correct and must be preserved
- No migration needed for historical quotes -- they already store the numeric value

### When all tax rates are disabled or none exist

**Recommended approach:**
- Item form should show an empty dropdown with a helpful message: "No tax rates available. Create one in Settings."
- Form validation should require a tax rate selection for non-bundle items (matching current behavior where `defaultTaxRate` is required)
- Alternatively, allow a "No tax" selection that stores `null` or a sentinel value

### Data migration for existing items

**Current state:** Items store `defaultTaxRate: number` (e.g., `20`)
**Target state:** Items store `taxRateId: string` referencing a tax rate document

**Migration approach:**
- Match existing items to tax rates by rate value within the same business
- Items with `defaultTaxRate: 0` can be set to null/exempt or matched to a 0% tax rate if one exists
- Items that cannot be matched (no tax rate with that percentage exists) should be flagged for manual resolution
- Keep `defaultTaxRate` field temporarily for backward compatibility during migration, remove after verification

## MVP Recommendation

Prioritize (all are table stakes, low complexity):
1. **Tax rate ID reference on item entity** -- the foundational data model change
2. **Tax rate dropdown on item create/edit forms** -- replaces number input, filtered to enabled rates, default pre-selected
3. **Display linked tax rate on item views** -- show name and percentage where items are displayed
4. **Reference integrity on disable** -- show disabled tax rate on existing items, filter from new selections

Defer:
- **Quick-create tax rate from item form** -- nice UX but not essential for v1.1; users can create tax rates in the existing tax rate management UI
- **Data migration script** -- may be needed depending on whether this is a breaking change or additive; assess during implementation planning
- **Quote line item factory update** -- must be updated but is a code-level concern, not a feature

## Sources

- [QuickBooks: Delete sales tax rates and agencies](https://quickbooks.intuit.com/learn-support/en-us/help-article/sales-taxes/delete-sales-tax-rates-agencies/L6JKrQWgQ_US_en_US) -- confirms QBO does not allow deletion, only deactivation
- [Invoice Ninja: Tax Settings](https://invoiceninja.github.io/docs/user-guide/taxes) -- documents per-item tax rate dropdown with 1-3 rate support
- [Invoice Ninja: Tax Setting Per Item or Invoice Total](https://www.invoiceninja.com/tax-setting-per-item-or-invoice-total/) -- per-item vs invoice-level tax selection patterns
- [FreshBooks: How do I create an invoice?](https://support.freshbooks.com/hc/en-us/articles/216631328-How-do-I-create-an-Invoice-) -- documents "Add Taxes" link pattern on line items
- [Xero: Suspend, archive or delete a tax return](https://central.xero.com/s/article/Suspend-archive-or-delete-a-tax-return) -- Xero archive vs delete approach
- Existing codebase: `trade-flow-api/src/item/data-transfer-objects/item.dto.ts`, `trade-flow-api/src/tax-rate/entities/tax-rate.entity.ts`, `trade-flow-ui/src/features/tax-rates/api/taxRateApi.ts`

---
*Research completed: 2026-03-07*
