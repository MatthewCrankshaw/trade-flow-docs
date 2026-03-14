---
status: complete
phase: 14-quote-detail-and-line-items
source: [14-03-SUMMARY.md, 14-04-SUMMARY.md]
started: 2026-03-14T20:40:00Z
updated: 2026-03-14T20:55:00Z
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
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "Line totals include tax amount in the calculation"
  status: failed
  reason: "User reported: The line totals do not include the tax amount."
  severity: major
  test: 1
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "Bundle pricing strategy (component-based vs fixed) is configured on the item itself, not when adding to a quote"
  status: failed
  reason: "User reported: Users should not be asked about component vs fixed pricing when adding a bundle item to a quote, the component vs fixed pricing should be set on the item in the item's screen / item dialog of the item screen."
  severity: major
  test: 1
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
