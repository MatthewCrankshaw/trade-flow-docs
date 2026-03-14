---
status: diagnosed
phase: 14
trigger: "pricing strategy selector appears in quote add flow instead of on the item/bundle configuration"
created: 2026-03-14T00:00:00.000Z
updated: 2026-03-14T00:00:00.000Z
---

## Current Focus

hypothesis: Plan 14-04 added a two-step bundle add flow to QuoteLineItemsCard that is entirely self-contained — removing it cleanly restores the original single-step behavior because the factory already reads priceStrategy from bundleConfig on the item itself.
test: n/a — diagnosis complete
expecting: n/a
next_action: handoff to fix phase — remove all quote-side priceStrategy plumbing

## Symptoms

expected: Selecting a bundle in the quote add flow adds it immediately (same as a standard item), using the pricing strategy already stored on the bundle item's bundleConfig.
actual: When a bundle is selected a two-step confirmation panel appears in the quote line items card, letting the user choose "Component-based" or "Fixed price" before confirming the add.
errors: none — functional bug, not a runtime error
reproduction: Open any draft quote, click "Add Item...", select a bundle item.
started: Introduced by plan 14-04 in phase 14.

## Eliminated

- hypothesis: priceStrategy is not stored on the item at all and must come from the quote flow
  evidence: IItemBundleConfigEntity (item.entity.ts line 14) has priceStrategy: PriceStrategy as a required field. IBundleConfigDto (bundle-config.dto.ts line 6) also has it required. The factory already uses bundleConfig.priceStrategy as the base value.
  timestamp: 2026-03-14

- hypothesis: The factory has no way to read priceStrategy without it being passed from the UI
  evidence: QuoteBundleLineItemFactory.createLineItems (line 48) builds effectiveConfig as: `priceStrategy ? { ...bundleConfig, priceStrategy } : bundleConfig`. When priceStrategy is undefined it falls back to bundleConfig directly — which already contains the item's stored priceStrategy. The override only takes effect when a value is explicitly passed.
  timestamp: 2026-03-14

## Evidence

- timestamp: 2026-03-14
  checked: QuoteLineItemsCard.tsx (lines 32–68, 97–136)
  found: Three pieces of state and logic added by 14-04:
    1. `pendingBundleItem` state (line 32) — holds the selected bundle while the user makes a strategy choice
    2. `selectedStrategy` state (line 33-34) — defaults to "component_based", overridden from item's bundleConfig on selection (line 45-47)
    3. `handleConfirmBundle` (line 58-68) — fires addLineItem with priceStrategy passed explicitly
    The confirmation panel UI (lines 97-136) is entirely new markup from 14-04.
  implication: The entire two-step flow is contained in this single component. Removing the three state variables, the two handlers (handleConfirmBundle, handleCancelBundle), and the panel JSX, and changing handleAddItem to call addLineItem directly for bundles, fully removes the feature.

- timestamp: 2026-03-14
  checked: quoteApi.ts (lines 73-98), addLineItem mutation
  found: priceStrategy is an optional field on the mutation args (line 80) and is spread into the request body conditionally (line 86): `...(priceStrategy && { priceStrategy })`. When undefined it is simply omitted from the POST body.
  implication: The API call itself does not need changing. Omitting priceStrategy is already valid — the backend handles the undefined case correctly.

- timestamp: 2026-03-14
  checked: CreateQuoteLineItemRequest (create-quote-line-item.request.ts)
  found: priceStrategy is @IsOptional() @IsString() — it is optional at the validation layer. The field was added by 14-04 to support the quote-side override.
  implication: Removing the UI selector means priceStrategy will never be sent. The field on the request class is harmless to leave or can be removed. It does not affect normal operation either way.

- timestamp: 2026-03-14
  checked: quote.controller.ts (lines 98-105), addLineItem handler
  found: lineItem.priceStrategy is forwarded as the 6th argument to quoteUpdater.addLineItem. When priceStrategy is absent from the request body it will be undefined — which is a valid call signature.
  implication: No change needed in the controller for the normal path. The priceStrategy parameter plumbing in the controller can remain or be removed as cleanup.

- timestamp: 2026-03-14
  checked: quote-updater.service.ts (lines 36-63), addLineItem method
  found: priceStrategy is accepted as an optional 6th parameter (line 42) and forwarded to addBundleLineItems (line 60) as `priceStrategy: priceStrategy as PriceStrategy | undefined`. When undefined the factory receives undefined.
  implication: No behavior change when priceStrategy is absent. Cleanup could remove the parameter, but it is not breaking.

- timestamp: 2026-03-14
  checked: quote-bundle-line-item-factory.service.ts (lines 39-59), createLineItems method
  found: Line 48 is the key decision point: `const effectiveConfig = priceStrategy ? { ...bundleConfig, priceStrategy } : bundleConfig`. When priceStrategy is undefined the item's own bundleConfig is used as-is — which is exactly the desired behavior.
  implication: The factory already implements the correct behavior when no override is supplied. No backend changes are needed to restore correct behavior.

- timestamp: 2026-03-14
  checked: BundleItemForm.tsx (lines 84-88), handleSubmit
  found: When submitting the form, priceStrategy is hard-coded to fall back: `priceStrategy: item?.bundleConfig?.priceStrategy ?? "component_based"`. This means on create it always sets component_based; on edit it preserves the existing value. The form schema (bundleItemFormSchema in item.schema.ts) has no priceStrategy field — it is not a user-editable field in the form at all.
  implication: This is the secondary gap: the item configuration screen does not currently expose priceStrategy as a UI control. The user's intent is that pricing strategy should be configurable here. The fix for the quote side removes the selector from the quote flow; a future task should add the selector to BundleItemForm so users can actually set it per-item.

- timestamp: 2026-03-14
  checked: item.entity.ts and bundle-config.dto.ts
  found: priceStrategy is a required field on both IItemBundleConfigEntity and IBundleConfigDto. The enum PriceStrategy has two values: FIXED = "fixed" and COMPONENT_BASED = "component_based". The field is stored in MongoDB as part of the item document.
  implication: The data model fully supports per-item pricing strategy. The infrastructure is correct. Only the UI edit surface is missing.

## Resolution

root_cause: Plan 14-04 added a quote-side pricing strategy selector because the item configuration form (BundleItemForm) does not expose priceStrategy as a user-editable field, so there was no way to change it per-item. The workaround — intercepting the add-to-quote flow — was placed in the wrong layer. The correct location per the user's intent is the item edit dialog.

fix: Two distinct changes are needed:

  PART 1 — Remove from quote flow (the reported bug):
  In QuoteLineItemsCard.tsx:
    - Remove state: pendingBundleItem, selectedStrategy
    - Remove handlers: handleConfirmBundle, handleCancelBundle
    - Simplify handleAddItem: remove the bundle branch; call addLineItem directly for all item types
    - Remove the pendingBundleItem panel JSX (lines 97-136)
  In quoteApi.ts: priceStrategy on the addLineItem mutation args can remain (harmless) or be removed as cleanup.
  In CreateQuoteLineItemRequest: priceStrategy field can remain (harmless) or be removed as cleanup.
  In QuoteUpdater.addLineItem and QuoteBundleLineItemFactory: the priceStrategy parameter plumbing can remain (harmless, already handles undefined correctly) or be removed as cleanup. The factory's fallback-to-bundleConfig behavior is correct and should be kept.

  PART 2 — Add to item form (the underlying gap, separate task):
  In BundleItemForm.tsx: add a pricing strategy selector (component_based / fixed) to the form UI and include priceStrategy in the form schema and submitted itemData.
  In item.schema.ts / bundleItemFormSchema: add priceStrategy as a picklist field.
  This unblocks users from configuring per-bundle pricing strategy in the item screen, which is the intended UX.

verification: not yet applied — diagnosis only
files_changed: []
