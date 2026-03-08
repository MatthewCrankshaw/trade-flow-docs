---
status: complete
phase: 11-bundle-bug-fix-and-foundation
source: 11-01-SUMMARY.md
started: 2026-03-08T18:25:00Z
updated: 2026-03-08T18:35:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Create a Bundle Item
expected: Navigate to Items. Click "New Item". Select type "Bundle". Fill in name, add at least one component. Submit the form. The bundle should be created successfully without any validation errors or API rejections.
result: pass

### 2. Picker Scrolling
expected: Open the item picker when adding a component to a bundle. The item list should be scrollable with the mouse wheel. You should be able to reach all items in the list. Also verify scrolling works in other pickers (e.g., Create Job dialog customer/type pickers).
result: pass

### 3. Add Component in Edit Mode
expected: Open an existing bundle item for editing. Click "Add component". The searchable item picker should open and allow you to add new components, just like in create mode.
result: issue
reported: "I can open the existing bundle item, I can add a component but then when it saves the item disappears. I think the API does not support adding or updating bundle components yet."
severity: major

### 4. Long Item Names
expected: Open the item picker. Items with long names should show a tooltip on hover revealing the full name. The dropdown should be wide enough to show most names comfortably.
result: pass

### 5. Search Filters Items
expected: With the item picker open, type a search term. The item list should filter in real-time across all groups. If nothing matches, an empty state message should show.
result: pass

### 6. Select Item and Auto-Close
expected: Click on an item in the picker. The dropdown should close automatically. A new component row should appear with that item pre-selected. The search field should be cleared for next use.
result: pass

### 7. Already-Added Items Hidden
expected: After adding an item to the bundle, open the picker again. The item you just added should NOT appear in the picker list.
result: pass

## Summary

total: 7
passed: 6
issues: 1
pending: 0
skipped: 0

## Gaps

- truth: "Bundle component changes are persisted when editing an existing bundle"
  status: failed
  reason: "User reported: I can open the existing bundle item, I can add a component but then when it saves the item disappears. I think the API does not support adding or updating bundle components yet."
  severity: major
  test: 3
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
