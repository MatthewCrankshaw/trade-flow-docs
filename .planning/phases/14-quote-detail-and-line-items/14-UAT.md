---
status: diagnosed
phase: 14-quote-detail-and-line-items
source: [14-03-SUMMARY.md, 14-04-SUMMARY.md]
started: 2026-03-14T20:40:00Z
updated: 2026-03-14T21:05:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Bundle Tax Calculation
expected: Navigate to a quote with a component-based bundle. Expand the bundle row. Component items should show their individual tax rates. The parent bundle row should display a non-zero effective tax rate (weighted average of component taxes), not 0%.
result: issue
reported: "There are a few issues: 1. In desktop view the type, quantity, unit, price, tax rate, total do not line up with the columns of the table. 2. The line totals do not include the tax amount. 3. Users should not be asked about component vs fixed pricing when adding a bundle item to a quote, the component vs fixed pricing should be set on the item in the item's screen / item dialog of the item screen."
severity: major

### 2. Soft Delete Line Item
expected: Delete a standard line item from a quote. Item disappears from the table and quote totals update. Check the database — the line item record should still exist with status "DELETED", not removed from the collection.
result: skipped

### 3. Soft Delete Bundle (Cascade)
expected: Delete a bundle from a quote. Both the parent bundle and all child components should have status set to "DELETED" in the database. They should no longer appear in the quote line items list or affect totals.
result: pass

### 4. SearchableItemPicker Width
expected: Click "Add Item" to open the SearchableItemPicker. The popover should be wide enough (384px) to comfortably display item names, type badges, and prices without truncation or overflow.
result: pass

### 5. Bundle Component Column Alignment
expected: View a quote with a component-based bundle and expand it. Component rows should have uniform column alignment — numeric columns (quantity, unit price, tax%, line total) should be right-aligned and vertically consistent across all component rows, matching the parent table structure.
result: skipped
reason: Already reported as issue in test 1

### 6. Bundle Pricing Strategy Selector
expected: Click "Add Item" and select a bundle item. Instead of being added immediately, a pricing strategy selector appears with two options: "Component-based" and "Fixed price". The default matches the bundle's configured strategy. Clicking "Add to quote" adds the bundle with the selected strategy. Clicking "Cancel" dismisses without adding.
result: skipped
reason: Already reported as issue in test 1 — selector belongs on item screen, not quote add flow

### 7. Non-Bundle Items Add Immediately
expected: Click "Add Item" and select a standard (non-bundle) item. The item should be added to the quote immediately without any pricing strategy prompt — the two-step flow only applies to bundles.
result: pass

## Summary

total: 7
passed: 3
issues: 1
pending: 0
skipped: 3

## Gaps

- truth: "Bundle component rows align with parent table columns and line totals include tax"
  status: failed
  reason: "User reported: In desktop view the type, quantity, unit, price, tax rate, total do not line up with the columns of the table."
  severity: cosmetic
  test: 1
  root_cause: "Two compounding issues: (1) Qty and Unit are merged into one 5rem grid track with an empty spacer that collapses, shifting everything leftward. (2) The ml-6 indent on the grid container displaces the entire grid 24px right of the table's content edge. CSS grid with static widths cannot reliably match the browser's runtime table layout algorithm."
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      issue: "Component row CSS grid has merged qty/unit track and ml-6 indent offset vs parent table columns"
  missing:
    - "Separate qty and unit into distinct grid tracks matching parent table"
    - "Account for ml-6 indent offset or use table-native layout for component rows"
  debug_session: ".planning/debug/bundle-component-alignment-v2.md"
- truth: "Line totals include tax amount in the calculation"
  status: failed
  reason: "User reported: The line totals do not include the tax amount."
  severity: major
  test: 1
  root_cause: "lineTotal is defined as tax-exclusive (qty * unitPrice before tax) across all layers — DTO JSDoc documents it as 'before tax', both factories compute it without tax, and the UI renders it raw. The totals calculator computes per-line tax but only accumulates into quote-level taxTotal, never writes back to lineItem. No layer produces a tax-inclusive per-line total."
  artifacts:
    - path: "trade-flow-api/src/quote/services/quote-standard-line-item-factory.service.ts"
      issue: "lineTotal = unitPrice.multiply(quantity) — no tax"
    - path: "trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts"
      issue: "lineTotal set to pre-tax discounted amount"
    - path: "trade-flow-api/src/quote/data-transfer-objects/quote-line-item.dto.ts"
      issue: "DTO contract defines lineTotal as before-tax"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      issue: "Renders lineItem.lineTotal raw at lines 248 and 310"
  missing:
    - "Add derived lineTotalIncTax field to API response (lineTotal * (1 + taxRate/100)) or compute in UI"
    - "Display tax-inclusive total in QuoteLineItemsTable instead of raw lineTotal"
  debug_session: ".planning/debug/line-total-missing-tax.md"
- truth: "Bundle pricing strategy (component-based vs fixed) is configured on the item itself, not when adding to a quote"
  status: failed
  reason: "User reported: Users should not be asked about component vs fixed pricing when adding a bundle item to a quote, the component vs fixed pricing should be set on the item in the item's screen / item dialog of the item screen."
  severity: major
  test: 1
  root_cause: "Plan 14-04 added a quote-side pricing strategy selector as a workaround because BundleItemForm doesn't expose priceStrategy as editable. The factory already reads priceStrategy from bundleConfig correctly — the override mechanism falls back gracefully when no override is provided. The fix is to remove the quote-side selector (restoring immediate-add for bundles) and add a priceStrategy field to BundleItemForm/bundleItemFormSchema instead."
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx"
      issue: "Contains entire two-step bundle selector (pendingBundleItem state, strategy picker UI) — needs removal"
    - path: "trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx"
      issue: "Hard-codes priceStrategy on submit line 85, no user-facing control"
    - path: "trade-flow-ui/src/lib/forms/schemas/item.schema.ts"
      issue: "bundleItemFormSchema missing priceStrategy field"
  missing:
    - "Remove two-step selector from QuoteLineItemsCard, restore immediate-add for bundles"
    - "Add priceStrategy radio/select to BundleItemForm"
    - "Add priceStrategy to bundleItemFormSchema"
  debug_session: ".planning/debug/pricing-strategy-wrong-location.md"
