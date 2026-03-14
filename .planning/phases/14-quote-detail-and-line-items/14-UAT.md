---
status: complete
phase: 14-quote-detail-and-line-items
source: [14-01-SUMMARY.md, 14-02-SUMMARY.md]
started: 2026-03-14T20:00:00Z
updated: 2026-03-14T20:14:00Z
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
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "Deleting a line item soft-deletes it (status set to deleted) rather than hard-deleting from the database"
  status: failed
  reason: "User reported: The item is immediatly removed and the quote totals update, but the item is hard deleted from the database. Instead I want the item's status to be updated to be \"deleted\" but I don't want to hard delete the item, but exclude it from the quote. (Soft Delete)"
  severity: major
  test: 7
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "Bundle component rows display with uniform column alignment matching the parent table"
  status: failed
  reason: "User reported: The bundle component aren't layed out in a very visually pleasing way, the amount and unit, price and total are horizontally aligned in a way that is not uniform when there are lots of components."
  severity: cosmetic
  test: 3
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "Tax is calculated for component-based bundles"
  status: failed
  reason: "User reported: tax is not being calculated for component based bundles."
  severity: major
  test: 10
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
- truth: "When editing or creating bundle line items, user can switch between component-based pricing and fixed pricing"
  status: failed
  reason: "User reported: there is missing functionality where editing and creating bundle items I cannot change between component based pricing and fixed pricing on bundles."
  severity: major
  test: 11
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
