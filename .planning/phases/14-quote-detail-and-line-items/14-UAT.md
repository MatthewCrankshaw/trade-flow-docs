---
status: diagnosed
phase: 14-quote-detail-and-line-items
source: [14-01-SUMMARY.md, 14-02-SUMMARY.md]
started: 2026-03-14T20:00:00Z
updated: 2026-03-14T20:18:00Z
---

## Current Test

## Current Test

[testing complete]

## Tests

### 1. View Quote Line Items
expected: Navigate to a quote detail page that has line items. A line items table displays with columns for item name, quantity, unit price, tax, discount, and line total.
result: pass

### 2. Add Standard Item
expected: Click "Add Item" button, SearchableItemPicker opens. Select a standard (non-bundle) item. Item appears as a new row in the line items table with correct name, quantity, price, and calculated total.
result: issue
reported: "The searchable item picker's width is too small. Otherwise pass"
severity: minor

### 3. Add Bundle Item
expected: Click "Add Item", SearchableItemPicker shows bundles alongside standard items. Select a bundle. Bundle appears as a row in the line items table with dashes for unit price and tax columns.
result: issue
reported: "The bundle component aren't layed out in a very visually pleasing way, the amount and unit, price and total are horizontally aligned in a way that is not uniform when there are lots of components."
severity: cosmetic

### 4. Expand Bundle Row
expected: Click the expand chevron on a bundle row. Component items display nested underneath the bundle parent. Components show their individual quantities, prices, and totals.
result: pass

### 5. Inline Edit Quantity
expected: Click on a line item's quantity field — it becomes editable. Change the value and press Enter or blur. Quantity updates and line total recalculates server-side.
result: pass

### 6. Inline Edit Price
expected: Click on a line item's unit price field — it becomes editable. Change the value and press Enter or blur. Price updates and line total recalculates. Pressing Escape cancels the edit.
result: pass

### 7. Delete Line Item
expected: Click delete on a standard line item. Item is immediately removed from the table (no confirmation dialog). Quote totals update.
result: issue
reported: "The item is immediatly removed and the quote totals update, but the item is hard deleted from the database. Instead I want the item's status to be updated to be \"deleted\" but I don't want to hard delete the item, but exclude it from the quote. (Soft Delete)"
severity: major

### 8. Delete Bundle (Cascade)
expected: Click delete on a bundle parent row. The bundle and all its component items are removed together. Cannot delete individual bundle components.
result: pass

### 9. Read-Only Mode
expected: View a quote with status Accepted or Rejected. No "Add Item" button visible. No inline edit fields or delete controls on line items.
result: pass

### 10. Responsive Mobile Layout
expected: Narrow browser to mobile width. Line items switch from table layout to card layout. Cards show same information and support same actions (edit, delete, expand bundles).
result: issue
reported: "tax is not being calculated for component based bundles."
severity: major

### 11. Empty State
expected: View a quote with no line items. Shows an empty state message with a call-to-action to add the first item.
result: pass

## Summary

total: 11
passed: 7
issues: 5
pending: 0
skipped: 0

## Gaps

- truth: "SearchableItemPicker opens at appropriate width for selecting items"
  status: failed
  reason: "User reported: The searchable item picker's width is too small. Otherwise pass"
  severity: minor
  test: 2
  root_cause: "PopoverContent in SearchableItemPicker.tsx (line 116) has hardcoded w-80 (320px) width class. Too narrow for item names, type badges, and prices, especially with bundles enabled."
  artifacts:
    - path: "trade-flow-ui/src/components/SearchableItemPicker.tsx"
      issue: "Hardcoded w-80 on PopoverContent is too narrow"
  missing:
    - "Increase width class (e.g., w-96 or w-[28rem]) or accept optional width prop"
  debug_session: ".planning/debug/picker-width-too-small.md"
- truth: "Deleting a line item soft-deletes it (status set to deleted) rather than hard-deleting from the database"
  status: failed
  reason: "User reported: The item is immediatly removed and the quote totals update, but the item is hard deleted from the database. Instead I want the item's status to be updated to be \"deleted\" but I don't want to hard delete the item, but exclude it from the quote. (Soft Delete)"
  severity: major
  test: 7
  root_cause: "Three-layer issue: (1) Repository delete()/deleteByParentLineItemId() call writer.deleteOne/deleteMany (hard delete), (2) QuoteLineItemStatus enum has no DELETED value, (3) findAllByQuoteId() has no status filter to exclude deleted items."
  artifacts:
    - path: "trade-flow-api/src/quote/repositories/quote-line-item.repository.ts"
      issue: "delete() and deleteByParentLineItemId() use hard delete operations"
    - path: "trade-flow-api/src/quote/enums/quote-line-item-status.enum.ts"
      issue: "Missing DELETED enum value"
    - path: "trade-flow-api/src/quote/services/quote-totals-calculator.service.ts"
      issue: "Does not filter out deleted items when summing"
  missing:
    - "Add DELETED to QuoteLineItemStatus enum"
    - "Replace deleteOne/deleteMany with status update to DELETED"
    - "Add status filter to findAllByQuoteId() to exclude DELETED items"
  debug_session: ".planning/debug/line-item-hard-delete.md"
- truth: "Bundle component rows display with uniform column alignment matching the parent table"
  status: failed
  reason: "User reported: The bundle component aren't layed out in a very visually pleasing way, the amount and unit, price and total are horizontally aligned in a way that is not uniform when there are lots of components."
  severity: cosmetic
  test: 3
  root_cause: "Component rows (lines 266-312 in QuoteLineItemsTable.tsx) render in a single colSpan cell using flex divs with gap-4 and no fixed widths. Parent table uses structured TableHead with explicit width classes, but component rows bypass this entirely."
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      issue: "Component rows use flex layout with no fixed widths instead of table-aligned columns"
  missing:
    - "Replace flex div layout with CSS grid using fixed column widths matching parent table"
  debug_session: ".planning/debug/bundle-component-alignment.md"
- truth: "Tax is calculated for component-based bundles"
  status: failed
  reason: "User reported: tax is not being calculated for component based bundles."
  severity: major
  test: 10
  root_cause: "Money.percentageOf() (lines 121-128 in money.value-object.ts) has a math bug: divides Dinero object by base.toMinorUnits() (e.g., 10000 for $100), causing integer division to return 0. BundleTaxRateCalculator correctly sums component taxes, but QuoteBundleLineItemFactory calls percentageOf() to get effective tax rate, which returns 0."
  artifacts:
    - path: "trade-flow-api/src/core/value-objects/money.value-object.ts"
      issue: "percentageOf divides Dinero by minor units integer — precision loss returns 0"
    - path: "trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts"
      issue: "Caller of the broken percentageOf method (line 135)"
  missing:
    - "Rewrite percentageOf to compute ratio using raw numeric values: (this.toMinorUnits() / base.toMinorUnits()) * 100"
  debug_session: ".planning/debug/bundle-tax-not-calculated.md"
- truth: "When editing or creating bundle line items, user can switch between component-based pricing and fixed pricing"
  status: failed
  reason: "User reported: there is missing functionality where editing and creating bundle items I cannot change between component based pricing and fixed pricing on bundles."
  severity: major
  test: 11
  root_cause: "Missing feature across three layers: (1) BundleItemForm.tsx hard-codes priceStrategy on line 85 with no toggle UI, (2) addLineItem/updateLineItem API mutations lack priceStrategy parameter, (3) QuoteBundleLineItemFactory reads pricing mode only from item entity bundleConfig with no override."
  artifacts:
    - path: "trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx"
      issue: "No pricing mode toggle control; hard-codes priceStrategy"
    - path: "trade-flow-ui/src/features/quotes/api/quoteApi.ts"
      issue: "addLineItem/updateLineItem mutations lack priceStrategy parameter"
    - path: "trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts"
      issue: "No override mechanism for pricing mode"
  missing:
    - "Add priceStrategy toggle (radio/select) to BundleItemForm"
    - "Add priceStrategy field to addLineItem API endpoint"
    - "Pass priceStrategy override through to QuoteBundleLineItemFactory"
  debug_session: ".planning/debug/bundle-pricing-mode-toggle.md"
