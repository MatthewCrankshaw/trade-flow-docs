---
status: complete
phase: 11-bundle-bug-fix-and-foundation
source: 11-01-SUMMARY.md
started: 2026-03-08T18:00:00Z
updated: 2026-03-08T18:10:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Create a Bundle Item
expected: Navigate to Items. Click "New Item". Select type "Bundle". Fill in name, add at least one component. Submit the form. The bundle should be created successfully without any validation errors or API rejections.
result: issue
reported: "I am unable to scroll down in the searchable item picker list, I have noticed this with other dropdown picker / searchable pickers as well. Apart from that it seems to be working fine."
severity: major

### 2. Searchable Item Picker Opens
expected: When creating or editing a bundle, click "Add component" (or equivalent). A searchable dropdown should appear with a search input at the top. Items should be listed below, grouped under headings: Materials, Labour, Fees.
result: issue
reported: "When creating a bundle I can click the add component button but when editing a bundle I cannot. The dropdown seems to work fine apart from scrolling."
severity: major

### 3. Search Filters Items
expected: With the item picker open, type a search term. The item list should filter in real-time across all groups simultaneously. Only items matching the search appear. If nothing matches, an empty state message should show (e.g., "No items found").
result: pass

### 4. Item Display in Picker
expected: Each item in the picker dropdown should show: the item name, a type badge (Material/Labour/Fee), and the default price formatted correctly (e.g., "$15.00" not "1500").
result: pass

### 5. Select Item and Auto-Close
expected: Click on an item in the picker. The dropdown should close automatically. A new component row should appear in the bundle with that item pre-selected. The search field should be cleared for next use.
result: pass

### 6. Already-Added Items Hidden
expected: After adding an item to the bundle, open the picker again. The item you just added should NOT appear in the picker list (hidden, not grayed out). This prevents duplicate components.
result: issue
reported: "Yes the already selected item is hidden, there is an issue where items with longer names have their names cut off and there is no way to see what the full name is."
severity: minor

## Summary

total: 6
passed: 3
issues: 3
pending: 0
skipped: 0

## Gaps

- truth: "Searchable item picker list is scrollable to reach all items"
  status: failed
  reason: "User reported: I am unable to scroll down in the searchable item picker list, I have noticed this with other dropdown picker / searchable pickers as well."
  severity: major
  test: 1
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""

- truth: "Add component button works in both bundle creation and editing contexts"
  status: failed
  reason: "User reported: When creating a bundle I can click the add component button but when editing a bundle I cannot."
  severity: major
  test: 2
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""

- truth: "Items with long names are fully visible or have a way to view the full name"
  status: failed
  reason: "User reported: items with longer names have their names cut off and there is no way to see what the full name is."
  severity: minor
  test: 6
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
