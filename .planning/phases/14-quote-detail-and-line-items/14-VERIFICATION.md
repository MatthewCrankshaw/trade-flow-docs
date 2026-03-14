---
phase: 14-quote-detail-and-line-items
verified: 2026-03-14T21:00:00Z
status: passed
score: 21/21 must-haves verified
re_verification:
  previous_status: passed
  previous_score: 16/16
  gaps_closed:
    - "Tax is calculated correctly for component-based bundles (non-zero effective tax rate on parent)"
    - "Deleting a line item sets status to DELETED rather than removing from database"
    - "Deleted line items are excluded from quote totals calculation"
    - "Deleted line items are excluded from findAllByQuoteId results"
    - "SearchableItemPicker popover is wide enough to display item names, badges, and prices without truncation"
    - "Bundle component rows align uniformly with parent table columns"
    - "When adding a bundle to a quote, user can choose between component-based and fixed pricing"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Bundle expansion interactive behavior — click expand chevron on a bundle row"
    expected: "Component rows appear beneath bundle row with correct names, quantities, prices, and line totals. Clicking again collapses."
    why_human: "React state transitions and conditional rendering require visual confirmation."
  - test: "Inline edit saves on Enter and resets on Escape"
    expected: "Enter commits the mutation and field shows updated value. Escape resets field to previous server value without firing a mutation."
    why_human: "Keyboard event behavior and the defaultValue+key reset pattern needs live verification."
  - test: "Read-only mode on Accepted/Rejected quotes"
    expected: "No Add Item button visible, quantity/price fields are plain text (not inputs), no delete buttons."
    why_human: "isEditable conditional rendering requires visual confirmation across all three components."
  - test: "Totals update after add/edit/delete"
    expected: "Totals reflect the server-recalculated values after each mutation (RTK Query cache invalidation triggers re-fetch)."
    why_human: "RTK Query cache invalidation and refetch behavior needs live network confirmation."
  - test: "Mobile card layout at narrow viewport"
    expected: "Card layout renders (not table), expand/collapse works, inline editing works, delete button visible on editable quotes."
    why_human: "Responsive breakpoint and card layout require visual/device verification."
  - test: "Bundle pricing strategy selector — two-step add flow"
    expected: "Selecting a bundle from the picker shows the pricing strategy selector (Component-based / Fixed price buttons). Choosing a strategy and clicking Add to quote sends priceStrategy to API. Non-bundle items add immediately."
    why_human: "Two-step UI flow with inline state requires visual confirmation of toggle/confirm interaction."
  - test: "Bundle tax rate correct after fix"
    expected: "After adding a bundle whose components have tax rates, the bundle parent row shows a non-zero tax rate and the quote totals include tax from that bundle."
    why_human: "Math correction in Money.percentageOf needs live data verification to confirm the API returns non-zero taxRate on bundle parent rows."
---

# Phase 14: Quote Detail and Line Items Verification Report

**Phase Goal:** Users can view full quote details with line items, add items and bundles to quotes, and see calculated totals
**Verified:** 2026-03-14T21:00:00Z
**Status:** PASSED
**Re-verification:** Yes — after gap closure (plans 03 and 04)

---

## Goal Achievement

### Observable Truths — Plans 01 and 02 (Regression Check)

All 16 truths from the initial verification were regression-checked against files modified by plans 03 and 04. No regressions found.

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | API accepts POST to add a line item and returns updated quote with recalculated totals | VERIFIED | `quote.controller.ts` L86-113: `@Post` endpoint calls `quoteUpdater.addLineItem`, including new `priceStrategy` param (L104) |
| 2 | API accepts PATCH to update a line item and returns updated quote with recalculated totals | VERIFIED | `quote.controller.ts` L116-143: `@Patch` calls `quoteUpdater.updateLineItem`; recalculates and returns enriched response |
| 3 | API accepts DELETE to remove a line item and returns updated quote | VERIFIED | `quote-updater.service.ts` L121-124: now calls `softDeleteByParentLineItemId` and `softDelete` instead of hard delete |
| 4 | UI Quote type includes lineItems array matching API response shape | VERIFIED | `trade-flow-ui/src/types/quote.ts`: `interface QuoteLineItem` with all fields; `lineItems: QuoteLineItem[]` on `Quote` |
| 5 | RTK Query mutations exist for addLineItem, updateLineItem, deleteLineItem with cache invalidation | VERIFIED | `quoteApi.ts` L73-146: all three mutations with `invalidatesTags`; `addLineItem` updated to include optional `priceStrategy` field |
| 6 | SearchableItemPicker shows bundle items when includeBundles prop is true | VERIFIED | `SearchableItemPicker.tsx` L30-51: `includeBundles` prop, `GROUP_CONFIG_WITH_BUNDLES` with bundle group |
| 7 | User can view a quote detail page showing header information and a list of line items | VERIFIED | `QuoteDetailPage.tsx` renders `QuoteLineItemsCard`; no placeholder text |
| 8 | User can add standard items (material, labour, fee) to a quote via SearchableItemPicker | VERIFIED | `QuoteLineItemsCard.tsx` L42-55: non-bundle items call `addLineItem` immediately |
| 9 | User can add bundle items to a quote and see them as a single rolled-up line | VERIFIED | `QuoteLineItemsCard.tsx` L43-68: bundles enter two-step flow; API's `addLineItem` creates parent+component rows |
| 10 | User can expand a bundle line item to reveal its individual component breakdown | VERIFIED | `QuoteLineItemsTable.tsx` L62-79: `expandedBundles` Set state, `toggleBundleExpansion`; L266-318: expanded row renders components using CSS grid |
| 11 | User can view quote totals (subtotal, tax, total) calculated from line items | VERIFIED | `QuoteTotalsCalculator` now skips `DELETED` items (L18-20); `QuoteDetailPage.tsx` renders `quote.totals` |
| 12 | User can edit line item quantity inline (saves on blur/Enter) | VERIFIED | `QuoteLineItemsTable.tsx`: `handleQuantityBlur` calls `updateLineItem`; Enter triggers blur, Escape resets |
| 13 | User can edit line item unit price inline | VERIFIED | `QuoteLineItemsTable.tsx`: `handlePriceBlur` calls `updateLineItem` |
| 14 | User can delete a line item from a quote | VERIFIED | `quote-updater.service.ts` L102-128: `deleteLineItem` now performs soft delete via `softDelete` / `softDeleteByParentLineItemId` |
| 15 | Adding/editing restricted to Draft and Sent; Accepted/Rejected are read-only | VERIFIED | `QuoteLineItemsCard.tsx` L26: `isEditable = quote.status === "draft" \|\| quote.status === "sent"` — unchanged |
| 16 | Mobile view shows card layout per line item with expandable bundles | VERIFIED | `QuoteLineItemsCard.tsx` L159-174: `isDesktop ? <QuoteLineItemsTable> : <QuoteLineItemsCardList>` |

### Observable Truths — Plans 03 and 04 (Gap Closures)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 17 | Tax is calculated correctly for component-based bundles (non-zero effective tax rate on parent) | VERIFIED | `money.value-object.ts` L121-127: `percentageOf` uses `(this.toMinorUnits() / base.toMinorUnits()) * 100` — pure numeric, no Dinero integer division |
| 18 | Deleting a line item sets status to DELETED rather than removing from database | VERIFIED | `quote-line-item.repository.ts` L84-94: `softDelete` calls `findOneAndUpdate` with `$set: { status: QuoteLineItemStatus.DELETED }` |
| 19 | Deleted line items are excluded from quote totals calculation | VERIFIED | `quote-totals-calculator.service.ts` L18-20: `if (lineItem.status === QuoteLineItemStatus.DELETED) { continue; }` |
| 20 | Deleted line items are excluded from findAllByQuoteId results | VERIFIED | `quote-line-item.repository.ts` L110-118: filter includes `status: { $ne: QuoteLineItemStatus.DELETED }` |
| 21 | SearchableItemPicker popover is wide enough to display item names, badges, and prices without truncation | VERIFIED | `SearchableItemPicker.tsx` L116: `className="w-96 p-0"` (384px) |

**Score:** 21/21 truths verified (16 original + 5 gap closures)

---

## Required Artifacts

### Plans 01 and 02 Artifacts (Regression Check)

| Artifact | Status | Notes |
|----------|--------|-------|
| `trade-flow-api/src/quote/requests/update-quote-line-item.request.ts` | VERIFIED | Unchanged |
| `trade-flow-ui/src/types/quote.ts` | VERIFIED | Unchanged |
| `trade-flow-ui/src/features/quotes/api/quoteApi.ts` | VERIFIED | Updated: `addLineItem` mutation includes optional `priceStrategy?: string` (L80) |
| `trade-flow-ui/src/components/SearchableItemPicker.tsx` | VERIFIED | Updated: `w-96` width (L116) |
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` | VERIFIED | Updated: two-step bundle add flow, `pendingBundleItem` state, pricing strategy selector UI |
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` | VERIFIED | Updated: CSS grid layout for bundle component rows (L284-288) |
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` | VERIFIED | Unchanged |
| `trade-flow-ui/src/pages/QuoteDetailPage.tsx` | VERIFIED | Unchanged |

### Plans 03 and 04 Artifacts (New)

| Artifact | Min Lines | Actual | Status | Notes |
|----------|-----------|--------|--------|-------|
| `trade-flow-api/src/core/value-objects/money.value-object.ts` | — | 128 | VERIFIED | `percentageOf` at L121-127 uses numeric division |
| `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` | — | 79 | VERIFIED | `updateMany` method added at L62-70 |
| `trade-flow-api/src/quote/enums/quote-line-item-status.enum.ts` | — | 6 | VERIFIED | `DELETED = "DELETED"` present at L5 |
| `trade-flow-api/src/quote/repositories/quote-line-item.repository.ts` | — | 165 | VERIFIED | `softDelete` L84-94, `softDeleteByParentLineItemId` L96-108, `findAllByQuoteId` filter L114 |
| `trade-flow-api/src/quote/services/quote-updater.service.ts` | — | 162 | VERIFIED | `priceStrategy` param L42, soft delete calls L121/124 |
| `trade-flow-api/src/quote/services/quote-totals-calculator.service.ts` | — | 48 | VERIFIED | DELETED filter L18-20 |
| `trade-flow-api/src/quote/requests/create-quote-line-item.request.ts` | — | 13 | VERIFIED | `priceStrategy?: string` at L12 |
| `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` | — | 159 | VERIFIED | `priceStrategy?: PriceStrategy` in params L26; `effectiveConfig` override L48 |
| `trade-flow-api/src/quote/controllers/quote.controller.ts` | — | 200+ | VERIFIED | `lineItem.priceStrategy` passed to `quoteUpdater.addLineItem` at L104 |

---

## Key Link Verification

| From | To | Via | Status | Evidence |
|------|----|-----|--------|----------|
| `quote.controller.ts` | `quote-updater.service.ts` | `addLineItem` with `priceStrategy` | WIRED | L98-105: `this.quoteUpdater.addLineItem(..., lineItem.priceStrategy)` |
| `quote-updater.service.ts` | `quote-bundle-line-item-factory.service.ts` | `priceStrategy` in `ICreateBundleLineItemsParams` | WIRED | L54-62: `priceStrategy: priceStrategy as PriceStrategy \| undefined` in params object |
| `quote-bundle-line-item-factory.service.ts` | `bundleConfig` | `effectiveConfig` spread override | WIRED | L48: `priceStrategy ? { ...bundleConfig, priceStrategy } : bundleConfig`; `effectiveConfig` used at L50/52 |
| `quote-updater.service.ts` | `quote-line-item.repository.ts` | `softDelete` / `softDeleteByParentLineItemId` | WIRED | L121: `softDeleteByParentLineItemId(lineItemId)`; L124: `softDelete(lineItemId)` |
| `quote-line-item.repository.ts` | `quote-line-item-status.enum.ts` | `QuoteLineItemStatus.DELETED` filter | WIRED | L114: `status: { $ne: QuoteLineItemStatus.DELETED }` in `findAllByQuoteId` |
| `quote-totals-calculator.service.ts` | `quote-line-item-status.enum.ts` | `QuoteLineItemStatus.DELETED` guard | WIRED | L18-20: `if (lineItem.status === QuoteLineItemStatus.DELETED) { continue; }` |
| `QuoteLineItemsCard.tsx` | `quoteApi.ts` | `addLineItem` with `priceStrategy` | WIRED | L60-67: `handleConfirmBundle` calls `addLineItem({ ..., priceStrategy: selectedStrategy })` |
| `SearchableItemPicker.tsx` | `PopoverContent` | `w-96` width class | WIRED | L116: `className="w-96 p-0"` |
| `QuoteLineItemsTable.tsx` | CSS grid layout | `gridTemplateColumns` for bundle component rows | WIRED | L284-288: `style={{ gridTemplateColumns: "1fr auto 5rem auto 7rem auto auto" }}` |

---

## Requirements Coverage

| Requirement | Source Plans | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| QUOT-03 | 14-02, 14-04 | User can view quote detail with line items and calculated totals | SATISFIED | `QuoteDetailPage` renders `QuoteLineItemsCard` and totals Card; totals now exclude soft-deleted items; pricing strategy selector improves bundle add UX |
| QLIT-01 | 14-01, 14-02, 14-04 | User can add standard items (material, labour, fee) to a quote | SATISFIED | Non-bundle items added immediately via `addLineItem`; picker width increased for readability |
| QLIT-02 | 14-01, 14-02, 14-03 | User can add bundle items to a quote (creates parent + component line items) | SATISFIED | Bundle add now uses two-step flow with pricing strategy selection; `addLineItem` API creates parent+component rows; soft delete cascades to children |
| QLIT-03 | 14-02, 14-04 | User can view bundle line items as a rolled-up line that expands to show individual components | SATISFIED | `expandedBundles` Set state, chevron toggle; component rows now use CSS grid for uniform column alignment |
| QLIT-04 | 14-01, 14-02, 14-03 | User can view quote totals (subtotal, tax, total) calculated from line items | SATISFIED | `Money.percentageOf` math bug fixed (correct bundle tax rates); `QuoteTotalsCalculator` excludes DELETED items; totals recalculated on every mutation |

All 5 requirements accounted for. No orphaned requirements detected for Phase 14.

---

## Anti-Patterns Found

None. No TODOs, FIXMEs, placeholders, empty implementations, or stub handlers found in any plans 03 or 04 files.

Note: `placeholder="Add Item..."` in `QuoteLineItemsCard.tsx` (lines 89, 152) is the `placeholder` prop on `SearchableItemPicker` — UI copy, not a code stub.

---

## Human Verification Required

### 1. Bundle expansion interactive behavior

**Test:** Navigate to a quote detail page with a bundle line item. Click the expand chevron.
**Expected:** Component rows appear beneath the bundle row with correct item names, quantities, unit prices, and line totals. Clicking again collapses.
**Why human:** React state transitions and conditional rendering require visual confirmation.

### 2. Inline edit saves on Enter and resets on Escape

**Test:** On a Draft quote, click into a quantity cell, change the value. Press Enter. Then change a price, press Escape.
**Expected:** Enter commits the mutation and field shows updated value. Escape resets field to previous server value without firing a mutation.
**Why human:** Keyboard event behavior and the `defaultValue`+`key` reset pattern needs live verification.

### 3. Read-only mode on Accepted/Rejected quotes

**Test:** Open an Accepted or Rejected quote detail page.
**Expected:** No Add Item button visible, quantity/price fields are plain text (not inputs), no delete buttons.
**Why human:** `isEditable` conditional rendering requires visual confirmation across all three components.

### 4. Totals update after add/edit/delete

**Test:** Add a line item to a Draft quote. Verify subtotal, tax, and total values refresh. Edit quantity. Verify totals update again.
**Expected:** Totals reflect the server-recalculated values after each mutation (RTK Query cache invalidation triggers re-fetch).
**Why human:** RTK Query cache invalidation and refetch behavior needs live network confirmation.

### 5. Mobile card layout at narrow viewport

**Test:** Resize browser below 768px. Navigate to a quote with line items and a bundle.
**Expected:** Card layout renders (not table), expand/collapse works, inline editing works, delete button visible on editable quotes.
**Why human:** Responsive breakpoint and card layout require visual/device verification.

### 6. Bundle pricing strategy selector — two-step add flow

**Test:** In a Draft quote, click Add Item, select a bundle item.
**Expected:** A pricing strategy panel appears below the picker showing "Component-based" and "Fixed price" toggle buttons. Selecting a strategy and clicking "Add to quote" adds the bundle to the line items. Clicking Cancel dismisses the panel. Non-bundle items add immediately without showing the panel.
**Why human:** Two-step UI flow with inline state requires visual confirmation of the toggle and confirm interaction.

### 7. Bundle tax rate correct after fix

**Test:** Add a bundle to a Draft quote where the bundle's component items have non-zero tax rates. Inspect the bundle parent row's Tax% column and the quote totals' tax amount.
**Expected:** The bundle parent row shows a non-zero tax rate (proportional to component taxes). Quote totals include tax from that bundle.
**Why human:** Math correction in `Money.percentageOf` needs live data verification to confirm the API returns non-zero `taxRate` on bundle parent rows.

---

## Gaps Summary

No gaps. All 21 observable truths verified (16 original + 5 gap closures from plans 03 and 04), all artifacts exist and are substantive, all key links are wired.

**Gap closure summary (plans 03 and 04):**

- **Bundle tax bug (QLIT-04):** `Money.percentageOf` in `money.value-object.ts` rewritten to use `(this.toMinorUnits() / base.toMinorUnits()) * 100` — eliminates Dinero integer division that returned 0 for all bundle tax rates.
- **Soft delete (QLIT-02, QLIT-04):** `QuoteLineItemStatus.DELETED` added to enum; `softDelete` and `softDeleteByParentLineItemId` replace hard delete in repository; `findAllByQuoteId` excludes `DELETED` items; `QuoteTotalsCalculator` skips `DELETED` items; `MongoDbWriter.updateMany` added to support batch soft-delete.
- **Picker width (QLIT-01, QUOT-03):** `SearchableItemPicker` popover increased from `w-80` (320px) to `w-96` (384px).
- **Bundle component alignment (QLIT-03):** CSS grid with `gridTemplateColumns` replaces flex layout in `QuoteLineItemsTable` component rows, achieving uniform column alignment.
- **Bundle pricing strategy selection (QLIT-02):** `priceStrategy` optional parameter threaded from `CreateQuoteLineItemRequest` through controller, updater service, and factory. `QuoteLineItemsCard` implements a two-step add flow for bundles — non-bundles still add immediately. `quoteApi.ts` `addLineItem` mutation updated to pass `priceStrategy` in request body when provided.

---

_Verified: 2026-03-14T21:00:00Z_
_Verifier: Claude (gsd-verifier)_
