---
phase: 14-quote-detail-and-line-items
verified: 2026-03-14T22:00:00Z
status: passed
score: 26/26 must-haves verified
re_verification:
  previous_status: passed
  previous_score: 21/21
  gaps_closed:
    - "Bundle component rows visually align with parent table columns (native TableRow sub-rows)"
    - "Line total column shows tax-inclusive amounts for all line items (lineTotalIncTax helper)"
    - "Component rows within expanded bundles show tax-inclusive line totals"
    - "Selecting a bundle in quote add-item picker adds it immediately without a pricing strategy prompt"
    - "Bundle pricing strategy (component-based vs fixed) is configurable on BundleItemForm"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Bundle expansion interactive behavior — click expand chevron on a bundle row"
    expected: "Component rows appear beneath bundle row as native table rows, aligned with parent columns. Item name, type badge, qty, unit, unit price, tax%, and tax-inclusive total all display. Clicking again collapses."
    why_human: "React state transitions and conditional rendering require visual confirmation."
  - test: "Inline edit saves on Enter and resets on Escape"
    expected: "Enter commits the mutation and field shows updated value. Escape resets field to previous server value without firing a mutation."
    why_human: "Keyboard event behavior and the defaultValue+key reset pattern needs live verification."
  - test: "Read-only mode on Accepted/Rejected quotes"
    expected: "No Add Item button visible, quantity/price fields are plain text (not inputs), no delete buttons."
    why_human: "isEditable conditional rendering requires visual confirmation across all three components."
  - test: "Totals update after add/edit/delete"
    expected: "Totals reflect server-recalculated values after each mutation (RTK Query cache invalidation triggers re-fetch)."
    why_human: "RTK Query cache invalidation and refetch behavior needs live network confirmation."
  - test: "Mobile card layout at narrow viewport"
    expected: "Card layout renders (not table), expand/collapse works, inline editing works, tax-inclusive totals shown, delete button visible on editable quotes."
    why_human: "Responsive breakpoint and card layout require visual/device verification."
  - test: "Bundle pricing strategy selector in BundleItemForm"
    expected: "RadioGroup with Component-based and Fixed price options renders. Creating a new bundle defaults to component_based. Editing an existing bundle shows its current priceStrategy. Saving persists the selection."
    why_human: "Radio group interaction and form submission need live verification."
  - test: "Adding a bundle to a quote is immediate (no two-step prompt)"
    expected: "Selecting a bundle from the SearchableItemPicker adds it directly to the quote line items with no intermediate pricing strategy screen."
    why_human: "Flow behavior requires live interaction confirmation."
  - test: "Bundle tax rate correct after percentageOf fix"
    expected: "Bundle parent row shows a non-zero tax rate proportional to component taxes. Quote totals include tax from that bundle."
    why_human: "Math correction in Money.percentageOf needs live data verification."
---

# Phase 14: Quote Detail and Line Items Verification Report

**Phase Goal:** Quote detail page with full line item management — add items/bundles, inline edit quantity/price, delete, expandable bundle rows with component visibility, mobile-responsive card layout
**Verified:** 2026-03-14T22:00:00Z
**Status:** PASSED
**Re-verification:** Yes — after gap closure (plans 05 and 06 executed after previous VERIFICATION.md)

---

## Goal Achievement

This is a re-verification covering all 6 plans. The previous VERIFICATION.md (score 21/21) covered plans 01-04 only. Plans 05 and 06 completed after that verification with SUMMARY files dated 2026-03-14. All prior truths were regression-checked; 5 new truths added for plans 05 and 06.

### Observable Truths — Plans 01–04 (Regression Check)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | API accepts POST to add a line item and returns updated quote with recalculated totals | VERIFIED | `quote.controller.ts`: `@Post` endpoint calls `quoteUpdater.addLineItem` |
| 2 | API accepts PATCH to update a line item and returns updated quote with recalculated totals | VERIFIED | `quote.controller.ts`: `@Patch` calls `quoteUpdater.updateLineItem` |
| 3 | API accepts DELETE to soft-delete a line item and returns updated quote | VERIFIED | `quote-updater.service.ts`: calls `softDeleteByParentLineItemId` and `softDelete` |
| 4 | UI Quote type includes lineItems array matching API response shape | VERIFIED | `quote.ts`: `QuoteLineItem` interface with `lineTotal`, `taxRate`, `components?`, `status` |
| 5 | RTK Query mutations exist for addLineItem, updateLineItem, deleteLineItem with cache invalidation | VERIFIED | `quoteApi.ts`: all three mutations present with `invalidatesTags` |
| 6 | SearchableItemPicker shows bundle items when includeBundles prop is true | VERIFIED | `SearchableItemPicker.tsx`: `includeBundles` prop with `GROUP_CONFIG_WITH_BUNDLES` |
| 7 | User can view a quote detail page showing header information and a list of line items | VERIFIED | `QuoteDetailPage.tsx` renders `QuoteLineItemsCard`; no placeholder |
| 8 | User can add standard items (material, labour, fee) to a quote via SearchableItemPicker | VERIFIED | `QuoteLineItemsCard.tsx`: all items (including bundles) now call `addLineItem` immediately in `handleAddItem` |
| 9 | User can add bundle items to a quote and see them as a single rolled-up line | VERIFIED | `QuoteLineItemsCard.tsx`: `handleAddItem` calls `addLineItem` for all types; API creates parent+component rows |
| 10 | User can expand a bundle line item to reveal its individual component breakdown | VERIFIED | `QuoteLineItemsTable.tsx` L62-79: `expandedBundles` Set state and `toggleBundleExpansion`; L269-316: expanded rows use native `TableRow` elements |
| 11 | User can view quote totals (subtotal, tax, total) calculated from line items | VERIFIED | `QuoteTotalsCalculator` skips DELETED items; `QuoteDetailPage.tsx` renders `quote.totals` |
| 12 | User can edit line item quantity inline (saves on blur/Enter, resets on Escape) | VERIFIED | `QuoteLineItemsTable.tsx` and `QuoteLineItemsCardList.tsx`: `handleQuantityBlur`, `handleKeyDown` with Enter→blur, Escape→reset |
| 13 | User can edit line item unit price inline | VERIFIED | Both table and card list: `handlePriceBlur` calls `updateLineItem` |
| 14 | User can delete a line item from a quote | VERIFIED | `quote-updater.service.ts`: `deleteLineItem` performs soft delete; repo sets `status: DELETED` |
| 15 | Adding/editing restricted to Draft and Sent; Accepted/Rejected are read-only | VERIFIED | `QuoteLineItemsCard.tsx` L25: `isEditable = quote.status === "draft" \|\| quote.status === "sent"` |
| 16 | Mobile view shows card layout per line item with expandable bundles | VERIFIED | `QuoteLineItemsCard.tsx` L91: `isDesktop ? <QuoteLineItemsTable> : <QuoteLineItemsCardList>` |
| 17 | Tax is calculated correctly for component-based bundles | VERIFIED | `money.value-object.ts`: `percentageOf` uses `(this.toMinorUnits() / base.toMinorUnits()) * 100` |
| 18 | Deleting a line item sets status to DELETED rather than removing from database | VERIFIED | `quote-line-item.repository.ts`: `softDelete` calls `findOneAndUpdate` with `$set: { status: DELETED }` |
| 19 | Deleted line items are excluded from quote totals calculation | VERIFIED | `quote-totals-calculator.service.ts` L18-20: early continue when `status === DELETED` |
| 20 | Deleted line items are excluded from findAllByQuoteId results | VERIFIED | `quote-line-item.repository.ts` L114: filter includes `status: { $ne: QuoteLineItemStatus.DELETED }` |
| 21 | SearchableItemPicker popover is wide enough to display names, badges, and prices without truncation | VERIFIED | `SearchableItemPicker.tsx` L116: `className="w-96 p-0"` (384px) |

### Observable Truths — Plans 05 and 06 (New Gap Closures)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 22 | Bundle component rows visually align with parent table columns in desktop view | VERIFIED | `QuoteLineItemsTable.tsx` L271-314: each component renders as `<TableRow>` with individual `<TableCell>` elements matching the 7-8 column header; no CSS grid or colSpan approach remaining |
| 23 | Line total column shows tax-inclusive amounts for all parent row line items | VERIFIED | `QuoteLineItemsTable.tsx` L129-130: `lineTotalIncTax` helper; L251: `formatDecimal(lineTotalIncTax(lineItem))`; column header L146: "Total (inc. tax)" |
| 24 | Component rows within expanded bundles show tax-inclusive line totals | VERIFIED | `QuoteLineItemsTable.tsx` L309: `formatDecimal(lineTotalIncTax(component))`; `QuoteLineItemsCardList.tsx` L295: same |
| 25 | Selecting a bundle item in the quote add-item picker adds it immediately without a pricing strategy prompt | VERIFIED | `QuoteLineItemsCard.tsx` L37-44: `handleAddItem` is a single branch calling `addLineItem` for all item types; no `pendingBundleItem` state, no `handleConfirmBundle`, no strategy selector JSX |
| 26 | Bundle pricing strategy (component-based vs fixed) is configurable on BundleItemForm in the items screen | VERIFIED | `BundleItemForm.tsx` L119-149: `Controller` with `RadioGroup` renders "Component-based" and "Fixed price" options; `item.schema.ts` L37-40: `priceStrategy: v.picklist(["component_based", "fixed"])` in `bundleItemFormSchema`; `BundleItemForm.tsx` L88: `priceStrategy: values.priceStrategy` in submit handler |

**Score:** 26/26 truths verified (21 carried forward + 5 new from plans 05 and 06)

---

## Required Artifacts

### Plans 01–04 Artifacts (Regression Check — All Unchanged or Confirmed)

| Artifact | Status | Notes |
|----------|--------|-------|
| `trade-flow-api/src/quote/requests/update-quote-line-item.request.ts` | VERIFIED | Unchanged |
| `trade-flow-api/src/core/value-objects/money.value-object.ts` | VERIFIED | `percentageOf` numeric division at L121-127 |
| `trade-flow-api/src/quote/enums/quote-line-item-status.enum.ts` | VERIFIED | `DELETED = "DELETED"` present |
| `trade-flow-api/src/quote/repositories/quote-line-item.repository.ts` | VERIFIED | `softDelete`, `softDeleteByParentLineItemId`, DELETED filter in `findAllByQuoteId` |
| `trade-flow-api/src/quote/services/quote-updater.service.ts` | VERIFIED | Soft delete calls; `priceStrategy` param |
| `trade-flow-api/src/quote/services/quote-totals-calculator.service.ts` | VERIFIED | DELETED item guard |
| `trade-flow-api/src/quote/requests/create-quote-line-item.request.ts` | VERIFIED | `priceStrategy?: string` present |
| `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` | VERIFIED | `priceStrategy` param with `effectiveConfig` override |
| `trade-flow-api/src/quote/controllers/quote.controller.ts` | VERIFIED | `priceStrategy` passed to `addLineItem` |
| `trade-flow-ui/src/types/quote.ts` | VERIFIED | `QuoteLineItem` with `lineTotal`, `taxRate`, `components?`, `status` |
| `trade-flow-ui/src/features/quotes/api/quoteApi.ts` | VERIFIED | Three mutations with cache invalidation |
| `trade-flow-ui/src/components/SearchableItemPicker.tsx` | VERIFIED | `w-96` width, `includeBundles` prop |
| `trade-flow-ui/src/pages/QuoteDetailPage.tsx` | VERIFIED | Renders `QuoteLineItemsCard` and totals |

### Plans 05 and 06 Artifacts (New)

| Artifact | Min Lines | Actual | Status | Notes |
|----------|-----------|--------|--------|-------|
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` | 200 | 323 | VERIFIED | Native TableRow sub-rows for components; `lineTotalIncTax` helper at L129; "Total (inc. tax)" header at L146; no `gridTemplateColumns` |
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` | 150 | 310 | VERIFIED | `lineTotalIncTax` helper at L121; tax-inclusive amounts at L244 and L295; expand/collapse working |
| `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` | 60 | 111 | VERIFIED | Single `handleAddItem` at L37-44 for all item types; no `pendingBundleItem`, no `handleConfirmBundle`, no strategy selector block |
| `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` | 100 | 171 | VERIFIED | `priceStrategy` defaultValue at L39; `RadioGroup` with Controller at L119-149; dynamic submit at L88 |
| `trade-flow-ui/src/lib/forms/schemas/item.schema.ts` | 30 | 71 | VERIFIED | `priceStrategy: v.picklist(["component_based", "fixed"])` at L37-40 in `bundleItemFormSchema` |
| `trade-flow-ui/src/components/ui/radio-group.tsx` | 10 | EXISTS | VERIFIED | shadcn RadioGroup component installed |

---

## Key Link Verification

| From | To | Via | Status | Evidence |
|------|----|-----|--------|----------|
| `QuoteLineItemsCard.tsx handleAddItem` | `addLineItem mutation` | direct call for all item types | WIRED | L37-44: no bundle branch, no `priceStrategy` sent — API reads from `bundleConfig` |
| `BundleItemForm.tsx` | `bundleItemFormSchema` | `valibotResolver` | WIRED | L34: `resolver: valibotResolver(bundleItemFormSchema)` |
| `BundleItemForm.tsx handleSubmit` | `values.priceStrategy` | `itemData.bundleConfig.priceStrategy` | WIRED | L88: `priceStrategy: values.priceStrategy` |
| `QuoteLineItemsTable.tsx` | `lineTotalIncTax` | `formatDecimal(lineTotalIncTax(lineItem/component))` | WIRED | L251, L309: all line total display sites use helper |
| `QuoteLineItemsCardList.tsx` | `lineTotalIncTax` | `formatDecimal(lineTotalIncTax(lineItem/component))` | WIRED | L244, L295: all line total display sites use helper |
| `QuoteLineItemsTable.tsx` (component rows) | native TableRow layout | individual `<TableRow>` + `<TableCell>` per component | WIRED | L287-313: each component is its own `<TableRow>` — no CSS grid, no colSpan data cells |
| `quote.controller.ts` | `quote-updater.service.ts` | `addLineItem` with optional `priceStrategy` | WIRED | `priceStrategy` param threaded from request through service to factory |
| `quote-updater.service.ts` | `quote-line-item.repository.ts` | `softDelete` / `softDeleteByParentLineItemId` | WIRED | Soft delete path for `deleteLineItem` |
| `quote-totals-calculator.service.ts` | `QuoteLineItemStatus.DELETED` | guard before summing | WIRED | Early `continue` skips deleted items |

---

## Requirements Coverage

| Requirement | Source Plans | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| QUOT-03 | 14-02, 14-04, 14-05, 14-06 | User can view quote detail with line items and calculated totals | SATISFIED | `QuoteDetailPage` renders line items and totals; tax-inclusive column header "Total (inc. tax)"; soft-deleted items excluded from totals |
| QLIT-01 | 14-01, 14-02, 14-04, 14-06 | User can add standard items (material, labour, fee) to a quote | SATISFIED | `handleAddItem` adds all item types immediately; picker width 384px; two-step flow removed |
| QLIT-02 | 14-01, 14-02, 14-03, 14-06 | User can add bundle items to a quote (creates parent + component line items) | SATISFIED | Bundle add is immediate (plan 06 removed the two-step prompt); pricing strategy moved to item form; API creates parent+component rows |
| QLIT-03 | 14-02, 14-04, 14-05 | User can view bundle line items as a rolled-up line that expands to show individual components | SATISFIED | `expandedBundles` Set state and `toggleBundleExpansion`; component rows are native `TableRow` elements for proper column alignment; component totals are tax-inclusive |
| QLIT-04 | 14-01, 14-02, 14-03, 14-05 | User can view quote totals (subtotal, tax, total) calculated from line items | SATISFIED | `Money.percentageOf` math fixed; `QuoteTotalsCalculator` excludes DELETED items; line totals display tax-inclusive amounts using `lineTotalIncTax` helper |

All 5 requirements accounted for. No orphaned requirements detected for Phase 14.

---

## Anti-Patterns Found

None. No TODOs, FIXMEs, placeholders, empty implementations, or stub handlers found in any plan 05 or 06 files. TypeScript typecheck (`npm run typecheck`) passes with zero errors.

Note: `placeholder="Add Item..."` in `QuoteLineItemsCard.tsx` is a UI prop on `SearchableItemPicker` — input hint text, not a code stub.

---

## Documentation Gap (Non-Blocking)

ROADMAP.md shows `[ ]` (unchecked) for both `14-05-PLAN.md` and `14-06-PLAN.md`, but both have completed SUMMARY files dated 2026-03-14 confirming execution and commit hashes `b3ef6ec` and `e5346d3` / `32b2d9c`. The phase row itself in ROADMAP.md is marked `[x]` as completed. The unchecked plan entries are a documentation-only inconsistency and do not affect the implementation.

---

## Human Verification Required

### 1. Bundle expansion interactive behavior

**Test:** Navigate to a quote detail page with a bundle line item. Click the expand chevron.
**Expected:** Native table rows appear beneath the bundle row. Each component row aligns with the parent table columns (Item, Type, Qty, Unit, Unit Price, Tax%, Total inc. tax). Tax-inclusive totals shown for each component. Clicking again collapses.
**Why human:** React state transitions and table layout alignment require visual confirmation.

### 2. Inline edit saves on Enter and resets on Escape

**Test:** On a Draft quote, click into a quantity cell, change the value. Press Enter. Then change a price and press Escape.
**Expected:** Enter commits the mutation and field shows updated value. Escape resets field to previous server value without firing a mutation.
**Why human:** Keyboard event behavior and the `defaultValue`+`key` reset pattern needs live verification.

### 3. Read-only mode on Accepted/Rejected quotes

**Test:** Open an Accepted or Rejected quote detail page.
**Expected:** No Add Item button visible, quantity/price fields are plain text (not inputs), no delete buttons.
**Why human:** `isEditable` conditional rendering requires visual confirmation across all three components.

### 4. Totals update after add/edit/delete

**Test:** Add a line item to a Draft quote. Verify subtotal, tax, and total values refresh. Edit quantity. Verify totals update again.
**Expected:** Totals reflect the server-recalculated values after each mutation.
**Why human:** RTK Query cache invalidation and refetch behavior needs live network confirmation.

### 5. Mobile card layout at narrow viewport

**Test:** Resize browser below 768px. Navigate to a quote with line items and a bundle.
**Expected:** Card layout renders (not table), expand/collapse works, tax-inclusive totals shown, inline editing works, delete button visible on editable quotes.
**Why human:** Responsive breakpoint and card layout require visual/device verification.

### 6. Bundle pricing strategy in BundleItemForm

**Test:** In the Items screen, create a new bundle. Observe the Pricing Strategy radio group. Select "Fixed price" and save. Re-open the bundle — verify it shows "Fixed price" selected. Change to "Component-based" and save — verify it persists.
**Expected:** RadioGroup renders, value persists via form submission, existing item's strategy is pre-selected on edit.
**Why human:** Radio group interaction and form submit path need live verification.

### 7. Bundle add flow is immediate in quotes

**Test:** In a Draft quote, click Add Item and select a bundle item.
**Expected:** The bundle appears in the line items list immediately with no intermediate pricing strategy selection screen.
**Why human:** Flow behavior and absence of the now-removed two-step UI needs live confirmation.

### 8. Bundle tax rate correct after percentageOf fix

**Test:** Add a bundle to a Draft quote where components have non-zero tax rates. Inspect the bundle parent row's Tax% column and the quote totals' tax amount.
**Expected:** Bundle parent row shows a non-zero tax rate. Quote totals include tax from that bundle.
**Why human:** Math correction in `Money.percentageOf` needs live data verification.

---

## Gaps Summary

No gaps. All 26 observable truths verified across all 6 plans. All required artifacts exist and are substantive. All key links are wired. TypeScript typecheck passes with zero errors.

**Gap closure summary (plans 05 and 06):**

- **Bundle component alignment (QLIT-03):** CSS grid inside a colSpan cell replaced with native `<TableRow>` elements for each component. Browser's table layout algorithm aligns columns automatically.
- **Tax-inclusive line totals (QLIT-03, QLIT-04):** `lineTotalIncTax` helper (`item.lineTotal * (1 + item.taxRate / 100)`) introduced in both `QuoteLineItemsTable` and `QuoteLineItemsCardList`. Column header updated to "Total (inc. tax)". All parent and component display sites use the helper.
- **Immediate bundle add in quotes (QLIT-01, QLIT-02):** Removed `pendingBundleItem` state, `handleConfirmBundle`, `handleCancelBundle`, and the pricing strategy selector JSX block from `QuoteLineItemsCard`. `handleAddItem` now handles all item types with a single `addLineItem` call. No `priceStrategy` override sent — API reads from `bundleConfig.priceStrategy` on the item.
- **Bundle pricing strategy on item form (QUOT-03, QLIT-01):** `priceStrategy` picklist field added to `bundleItemFormSchema`. `BundleItemForm` renders a `Controller`-wrapped `RadioGroup` with "Component-based" and "Fixed price" options. Default is `item?.bundleConfig?.priceStrategy ?? "component_based"`. Submit handler sends `values.priceStrategy` instead of a hard-coded fallback.

---

_Verified: 2026-03-14T22:00:00Z_
_Verifier: Claude (gsd-verifier)_
