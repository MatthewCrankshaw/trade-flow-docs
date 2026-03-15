---
status: resolved
trigger: "Add component button doesn't work when editing a bundle, but works when creating one"
created: 2026-03-08T00:00:00Z
updated: 2026-03-08T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED - BundleComponentsList hardcodes `isReadOnly = mode === "edit"` which hides the Add button and renders read-only display
test: Direct code inspection
expecting: The mode prop controls whether SearchableItemPicker renders
next_action: Fix applied and verified

## Symptoms

expected: "Add component" button (SearchableItemPicker) should be visible and functional when editing a bundle
actual: Button is hidden in edit mode; components render as read-only text with no add/remove/edit capability
errors: None (silent behavior difference)
reproduction: Open any bundle item in edit mode - components section is read-only
started: By design - this was intentionally coded as read-only but is now a bug/limitation

## Eliminated

(none - root cause found on first inspection)

## Evidence

- timestamp: 2026-03-08
  checked: BundleComponentsList.tsx line 47
  found: `const isReadOnly = mode === "edit";` - this single line gates ALL interactivity
  implication: When mode="edit", three things happen:
    1. Line 48-51: `canAddMore` is always false (because `!isReadOnly` is false)
    2. Line 57: `!isReadOnly && onAddComponent` condition hides SearchableItemPicker entirely
    3. Line 76: Components render in read-only display (no quantity input, no remove button)

- timestamp: 2026-03-08
  checked: BundleItemForm.tsx line 119
  found: `mode={isEditMode ? "edit" : "create"}` - parent always passes "edit" when item exists
  implication: The mode prop is the sole mechanism controlling read-only behavior

- timestamp: 2026-03-08
  checked: BundleItemForm.tsx lines 84-91
  found: `handleSubmit` skips bundleConfig entirely in edit mode (`if (!isEditMode)` guard on line 84)
  implication: TWO bugs - even if the UI allowed editing components, the submit handler would discard the changes

- timestamp: 2026-03-08
  checked: BundleComponentsList.tsx line 145-148
  found: Explicit message "Component editing is not yet available. This feature is coming soon."
  implication: This was a deliberate deferral, not an accidental bug

## Resolution

root_cause: |
  Two coordinated blocks prevent bundle component editing:

  1. **UI block** (BundleComponentsList.tsx:47): `const isReadOnly = mode === "edit"` hides the
     SearchableItemPicker and renders components as read-only text. The `!isReadOnly` guard on
     line 57 prevents the Add button from rendering at all.

  2. **Data block** (BundleItemForm.tsx:84-91): The submit handler wraps `bundleConfig` in
     `if (!isEditMode)`, so even if components were editable in the UI, changes would never
     be sent to the API.

fix: |
  1. Remove the `isReadOnly` gating in BundleComponentsList - always allow interaction
     (remove line 47, update canAddMore/rendering to not use isReadOnly)
  2. Include bundleConfig in the edit-mode submit payload in BundleItemForm
  3. Keep the mode prop if needed for future display differences, but don't use it to block editing

verification: Typecheck passes. Lint passes.
files_changed:
  - trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx
  - trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
