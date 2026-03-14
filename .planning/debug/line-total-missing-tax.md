---
status: diagnosed
phase: 14
trigger: "Line totals do not include the tax amount. The lineTotal displayed to the user does not factor in tax."
created: 2026-03-14T00:00:00Z
updated: 2026-03-14T00:00:00Z
---

## Current Focus

hypothesis: lineTotal is intentionally defined as tax-exclusive (qty * unitPrice), while the UI displays it as if it were the full tax-inclusive amount the customer pays per line
test: traced DTO comment, factory implementations, totals calculator
expecting: confirmed - lineTotal does NOT include tax anywhere in the pipeline
next_action: n/a - diagnosis complete

## Symptoms

expected: Line Total column shows the amount including tax (qty * unitPrice * (1 + taxRate/100))
actual: Line Total column shows qty * unitPrice only, with no tax component
errors: none (no error, just wrong value)
reproduction: open any quote with a non-zero tax rate; the Line Total for each row equals qty * unitPrice, tax is excluded
started: by design / from initial implementation

## Eliminated

- hypothesis: UI calculates or transforms lineTotal before display
  evidence: QuoteLineItemsTable.tsx line 248 calls `formatDecimal(lineItem.lineTotal)` directly with no transformation. The raw lineTotal value from the API is rendered as-is.
  timestamp: 2026-03-14

- hypothesis: lineTotal is set correctly by the totals calculator and overwritten back into each line item
  evidence: QuoteTotalsCalculator.calculateTotals() does NOT write any value back into individual lineItems. It only aggregates a quote-level subTotal, taxTotal, and total. It never updates lineItem.lineTotal.
  timestamp: 2026-03-14

- hypothesis: the bundle factory computes lineTotal differently (tax-inclusive)
  evidence: QuoteBundleLineItemFactory.buildComponentLineItem() sets lineTotal = resolvedDiscountedTotal (line 114), which is the result of qty * unitPrice after bundle discount, with no tax. buildParentLineItem() sets lineTotal = pricingPlan.targetTotal (line 149), also pre-tax.
  timestamp: 2026-03-14

## Evidence

- timestamp: 2026-03-14
  checked: IQuoteLineItemDto JSDoc comment on lineTotal field
  found: "The total price for this line item (unitPrice * quantity, after discounts, before tax)."
  implication: The DTO itself documents lineTotal as tax-exclusive. This is the authoritative type contract.

- timestamp: 2026-03-14
  checked: QuoteStandardLineItemFactory.create() (quote-standard-line-item-factory.service.ts, line 13)
  found: `const lineTotal = unitPrice.multiply(quantity);`
  implication: Standard (non-bundle) items compute lineTotal as qty * unitPrice with no tax factor applied. Tax-inclusive total is never stored per-line.

- timestamp: 2026-03-14
  checked: QuoteBundleLineItemFactory.buildComponentLineItem() (line 114) and buildParentLineItem() (line 149)
  found: Both set lineTotal to the discounted/target price with no tax multiplication. Tax rate is stored as a separate taxRate field but never folded into lineTotal.
  implication: Bundle line items follow the same tax-exclusive convention.

- timestamp: 2026-03-14
  checked: QuoteTotalsCalculator.calculateTotals() (lines 22-26)
  found: lineItemTax = lineItemTotal.percentage(lineItem.taxRate) is computed locally but only accumulated into a quote-level taxTotal. Individual lineItem.lineTotal is never updated.
  implication: The only place in the system that knows the per-line tax amount does not write it back to the line item. Tax is deliberately kept separate for the quote summary totals, not the line-level display.

- timestamp: 2026-03-14
  checked: QuoteLineItemsTable.tsx line 248
  found: `{formatDecimal(lineItem.lineTotal)}`
  implication: The UI renders lineTotal without any client-side augmentation. Whatever the API sends is what the user sees.

## Resolution

root_cause: |
  lineTotal is defined and computed throughout the system as a tax-exclusive amount: qty * unitPrice (after discounts).
  This is an intentional design choice documented in the DTO JSDoc. Tax is stored separately per-line as taxRate (a percentage),
  and the tax amount is only materialised at the quote level (IQuoteTotalsDto.taxTotal) by QuoteTotalsCalculator.
  The UI renders lineTotal directly. There is no layer that produces a tax-inclusive line total.

  The gap is that the user expects the "Line Total" column to represent what they will charge the customer for that line
  (i.e. the tax-inclusive amount), but the system stores and displays only the pre-tax subtotal per line.

fix: |
  TWO valid approaches exist - the choice depends on product intent:

  Option A - Compute tax-inclusive lineTotal at creation time (changes the stored value):
    In QuoteStandardLineItemFactory.create() change:
      const lineTotal = unitPrice.multiply(quantity);
    to:
      const lineTaxExclTotal = unitPrice.multiply(quantity);
      const lineTax = lineTaxExclTotal.percentage(taxRate);
      const lineTotal = lineTaxExclTotal.add(lineTax);
    Apply the same pattern in QuoteBundleLineItemFactory for component and parent line items.
    Update the DTO JSDoc to reflect "after discounts, including tax".
    NOTE: This would also affect the totals calculator, which currently sums lineTotal into subTotal -
    subTotal would then become tax-inclusive and the tax calculation would double-count. The totals
    calculator would need reworking to avoid that.

  Option B - Keep lineTotal as pre-tax, add a computed lineTotalIncTax field for display only:
    Add a derived field to the response DTO/serializer:
      lineTotalIncTax = lineTotal * (1 + taxRate / 100)
    Display lineTotalIncTax in QuoteLineItemsTable instead of lineTotal.
    This is the lower-risk change - it does not alter the stored lineTotal, which the totals
    calculator relies upon correctly.

  Recommended: Option B. It preserves the current totals calculation (which is correct) and is a
  smaller, safer change. The column label "Line Total" in the UI can remain as-is or be updated
  to "Line Total (inc. tax)" for clarity.

  Exact files to change for Option B:
    - trade-flow-api: add lineTotalIncTax to the serializer / response mapper
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx line 248:
        change `formatDecimal(lineItem.lineTotal)` to `formatDecimal(lineItem.lineTotalIncTax)`
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx line 310 (component rows):
        same change for the bundle component display

verification: n/a - diagnosis only, no fix applied
files_changed: []
