---
status: diagnosed
trigger: "When editing or creating bundle line items, there is no way to switch between component-based pricing and fixed pricing on bundles"
created: 2026-03-14T21:00:00Z
updated: 2026-03-14T21:05:00Z
---

## Current Focus

hypothesis: CONFIRMED - Missing feature: no UI control or API endpoint exists to toggle bundle pricing mode at the quote line item level
test: n/a - root cause confirmed
expecting: n/a
next_action: Return diagnosis

## Symptoms

expected: When editing or creating bundle line items, user can switch between component-based pricing and fixed pricing
actual: There is missing functionality where editing and creating bundle items the user cannot change between component based pricing and fixed pricing on bundles
errors: None reported
reproduction: Test 11 in UAT - try to change pricing mode on a bundle line item
started: Discovered during UAT for phase 14

## Eliminated

## Evidence

- timestamp: 2026-03-14T21:01:00Z
  checked: Data model - PriceStrategy enum and IBundleConfigDto
  found: PriceStrategy enum has FIXED and COMPONENT_BASED values. IBundleConfigDto holds priceStrategy and bundlePrice fields. The data model fully supports both pricing modes.
  implication: The backend data model is complete - this is not a schema gap.

- timestamp: 2026-03-14T21:02:00Z
  checked: BundleItemForm.tsx (item management form, not quote line item)
  found: Line 85 hard-codes priceStrategy from the existing item config with no toggle: `priceStrategy: item?.bundleConfig?.priceStrategy ?? "component_based"`. No UI control (radio, select, toggle) for changing pricing mode exists in the form.
  implication: Even the item-level bundle form lacks a pricing mode selector - it silently preserves existing value or defaults to component_based.

- timestamp: 2026-03-14T21:03:00Z
  checked: QuoteLineItemsTable.tsx and quoteApi.ts (quote line item UI and API layer)
  found: The addLineItem mutation sends only { itemId, quantity }. The updateLineItem mutation sends only { quantity, unitPrice }. Neither endpoint accepts a priceStrategy override. The UI has no pricing mode control for bundle rows.
  implication: The quote line item flow inherits whatever priceStrategy the Item has in its bundleConfig. There is no way to override pricing mode per-quote.

- timestamp: 2026-03-14T21:04:00Z
  checked: QuoteBundleLineItemFactory (API service)
  found: Line 44 reads bundleConfig from the item entity directly via bundleConfigValidator.requireValidConfig(bundleItem). The pricing plan is derived from the item's priceStrategy. No parameter exists to override pricing mode when creating bundle line items.
  implication: Confirms the API service has no mechanism to accept a per-quote pricing mode override.

- timestamp: 2026-03-14T21:04:30Z
  checked: BundlePricingPlanner service
  found: The planner correctly branches on bundleConfig.priceStrategy (line 25), returning component sums for COMPONENT_BASED and allocated fixed pricing for FIXED. The pricing logic itself works - it just always uses the item-level config.
  implication: The calculation engine is ready for both modes. The gap is purely in the input/control layer.

## Resolution

root_cause: Missing feature across three layers - (1) BundleItemForm.tsx has no UI control to toggle priceStrategy, hard-coding it from existing item or defaulting to "component_based" (line 85); (2) The quote addLineItem API endpoint accepts no priceStrategy parameter; (3) QuoteBundleLineItemFactory reads pricing mode solely from the item entity's bundleConfig with no override mechanism. The data model and pricing calculation engine both support dual pricing modes, but there is no user-facing control or API path to switch between them.
fix:
verification:
files_changed: []
