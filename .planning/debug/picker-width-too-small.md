---
status: diagnosed
trigger: "SearchableItemPicker width is too small when used in the quote line items add flow"
created: 2026-03-14T21:00:00Z
updated: 2026-03-14T21:00:00Z
---

## Current Focus

hypothesis: PopoverContent has hardcoded w-80 (320px) which is too narrow for item names + badges + prices
test: Confirmed by reading component source
expecting: n/a - root cause confirmed
next_action: return diagnosis

## Symptoms

expected: SearchableItemPicker opens at appropriate width for selecting items
actual: The searchable item picker's width is too small
errors: None reported
reproduction: Test 2 in UAT - click Add Item on quote detail page
started: Discovered during UAT

## Eliminated

(none)

## Evidence

- timestamp: 2026-03-14T21:00:00Z
  checked: SearchableItemPicker.tsx line 116
  found: PopoverContent uses className="w-80 p-0" which sets width to 320px (20rem)
  implication: Each row displays item name (truncated) + type badge + price, all squeezed into 320px. This is the sole width constraint on the picker dropdown.

- timestamp: 2026-03-14T21:00:00Z
  checked: QuoteLineItemsCard.tsx usage
  found: SearchableItemPicker is used with includeBundles={true}, meaning it shows 4 item types. No width override is passed to the component.
  implication: The component does not accept a className or width prop, so consumers cannot override the width. The fix must be in the component itself.

## Resolution

root_cause: PopoverContent in SearchableItemPicker.tsx (line 116) has a hardcoded width of w-80 (320px). This is too narrow to display item names, type badges, and formatted prices without excessive truncation. The component also does not expose a way for consumers to override this width.
fix: ""
verification: ""
files_changed: []
