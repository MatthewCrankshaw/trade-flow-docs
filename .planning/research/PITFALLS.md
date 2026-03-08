# Domain Pitfalls: Bundles & Quotes (v1.2)

**Domain:** Bundle editing, searchable item pickers, and quote management in a NestJS/React business management app
**Researched:** 2026-03-08

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

### Pitfall 1: Bundle Component Edits Not Propagating to Existing Quote Line Items

**What goes wrong:** A user edits a bundle's components (adds a component, removes one, changes quantities) through the item management UI. Existing quotes that use this bundle still have the OLD component line items. The user expects the quote to reflect the updated bundle, but it does not -- or worse, the quote totals are now inconsistent with the bundle definition.

**Why it happens:** The quote system snapshots bundle data at the time a bundle is added to a quote. The `QuoteBundleLineItemFactory` creates parent + child line items with `parentLineItemId` references, storing concrete `unitPrice`, `lineTotal`, `discountAmount`, and `taxRate` values. These are intentionally decoupled from the source item -- quotes are point-in-time snapshots. But developers (and users) may not realize this and expect edits to cascade.

**Consequences:** If you try to "sync" bundle edits to existing quotes, you break the snapshot model and introduce complex recalculation cascades. If you do not sync, users may be confused when their quote does not match the current bundle definition.

**Prevention:**
- Do NOT retroactively update existing quote line items when a bundle is edited. Quotes are snapshots. Document this decision in the UI with a clear message: "Changes to this bundle will not affect existing quotes."
- When displaying a bundle on a quote, show the snapshotted data, not the current item definition
- Consider adding a "refresh from current item" action on individual quote line items in a future milestone, but do NOT build it now -- it introduces recalculation complexity that is out of v1.2 scope

**Detection:** Manual testing: create a quote with a bundle, edit the bundle's components, verify the quote is unchanged.

**Phase relevance:** Bundle editing phase -- must make the deliberate decision NOT to sync, and communicate this in the UI.

---

### Pitfall 2: BundleItemForm Skips bundleConfig on Edit (Existing Bug)

**What goes wrong:** The current `BundleItemForm.tsx` (line 90-93) explicitly skips `bundleConfig` when `isEditMode` is true:
```typescript
if (!isEditMode) {
  itemData.bundleConfig = {
    priceStrategy: "component_based",
    bundlePrice: null,
    components: values.components,
  };
}
```
When enabling component editing, developers must change this guard. But naively including `bundleConfig` on every edit means the API `PATCH` endpoint must handle partial `bundleConfig` updates -- replacing the entire `bundleConfig` object versus merging individual fields.

**Why it happens:** The original code deliberately excluded `bundleConfig` from edits because component editing was not yet built. The `BundleComponentsList` component also sets `isReadOnly = mode === "edit"` (line 53), preventing any interaction. Removing these guards requires coordinated changes across form, component list, submit handler, and API merge logic.

**Consequences:** If you only remove the `isReadOnly` guard but forget the submit handler guard, the form renders editable components but submits without them. If you include `bundleConfig` but the API merge utility does not handle it, the entire config gets overwritten or lost.

**Prevention:**
- Remove both guards in the same commit: the submit handler's `if (!isEditMode)` check AND the `BundleComponentsList`'s `isReadOnly` logic
- The API's item update merge utility (`mergeExistingItemWithChanges`) must handle `bundleConfig` as a full replacement (not a deep merge) -- the UI should always send the complete component list when editing
- Validate that the updated `bundleConfig.components` array is non-empty (existing `BundleConfigValidator.ensureComponentsPresent()` handles this server-side)
- Add a Valibot schema validation on the form ensuring at least one component before submit

**Detection:** Edit a bundle, change a component quantity, submit -- verify the API receives the updated `bundleConfig` and the bundle's components reflect the change.

**Phase relevance:** Bundle component editing phase -- this is the first thing to fix.

---

### Pitfall 3: Double-Counting Bundle Components in Quote Totals

**What goes wrong:** The `QuoteTotalsCalculator` (line 14) skips line items with `parentLineItemId` to avoid double-counting -- it only sums parent (bundle-level) line items. But if the `parentLineItemId` field is not set correctly during bundle line item creation, component line items get counted BOTH as standalone items AND through their parent's total, inflating the quote total.

**Why it happens:** The `QuoteBundleLineItemFactory.buildComponentLineItem()` currently sets `parentLineItemId: null` (line 118) -- the parent ID is assigned later when the component is persisted. If the persistence step does not correctly set `parentLineItemId` on each component, the totals calculator includes them as top-level line items.

**Consequences:** Quote totals are 2x the actual amount for all bundle components. This is a financial correctness bug that directly affects customer-facing quotes.

**Prevention:**
- Trace the full pipeline: factory creates line items -> creator persists them -> ensure `parentLineItemId` is set on every component BEFORE `QuoteTotalsCalculator.calculateTotals()` runs
- The factory returns `IBundleLineItemGroupDto { parent, components }` -- the caller (quote creator) must set `parentLineItemId = parent.id` on each component before persisting
- Write a test that adds a 3-component bundle to a quote and asserts: (a) totals include the bundle parent only, (b) each component has `parentLineItemId` set, (c) the quote total equals `parent.lineTotal * (1 + taxRate/100)`

**Detection:** Unit test on `QuoteTotalsCalculator` with mixed standalone and bundle line items. Integration test on quote creation with bundles.

**Phase relevance:** Quote line item management phase -- validate this works end-to-end when wiring the quote UI to the API.

---

### Pitfall 4: RTK Query Cache Invalidation for Nested Quote Data

**What goes wrong:** A quote contains nested line items. When a line item is added/removed/updated via a mutation, the quote detail cache does not automatically refresh. The user adds an item to a quote, the mutation succeeds, but the quote detail view still shows the old line items (missing the new one) until a manual page refresh.

**Why it happens:** RTK Query's tag-based invalidation works at the entity level. If the "add line item" mutation only invalidates `QuoteLineItem` tags but not the `Quote` tag for the parent quote, the quote detail query (which includes line items in its response) stays stale. Conversely, if quote detail and line items are fetched separately, the invalidation strategy must cover both.

**Consequences:** Users think their changes were not saved. They may add the same item twice, or navigate away and lose trust in the system.

**Prevention:**
- Decide on ONE data fetching strategy for quote details:
  - **Option A (recommended):** The quote detail API returns the full quote with embedded line items (the backend already does this via `IQuoteDto.lineItems`). The "add line item" mutation invalidates `{ type: "Quote", id: quoteId }`, which triggers a refetch of the full quote detail
  - **Option B:** Fetch line items separately. Then BOTH `Quote` and `QuoteLineItem` tags must be invalidated on mutations. This adds complexity for no benefit
- Follow Option A: single quote detail endpoint returns everything, mutations invalidate the parent quote tag
- Tag structure: `providesTags: (result) => [{ type: "Quote", id: result.id }, "Quote"]`
- Mutation: `invalidatesTags: (result, error, arg) => [{ type: "Quote", id: arg.quoteId }]`

**Detection:** Add a line item to a quote, verify the quote detail view updates without manual refresh.

**Phase relevance:** Quote UI wiring phase -- define the cache strategy before building components.

---

## Moderate Pitfalls

### Pitfall 5: Searchable Dropdown Performance with Large Item Lists

**What goes wrong:** The current `BundleComponentsList` renders ALL available items in a `<Select>` dropdown. For a tradesperson with 50-200 items, this works. But as the list grows, the dropdown becomes unusable -- no search, no filtering, slow rendering. The `<SelectContent>` from shadcn/ui renders all options in the DOM at once.

**Why it happens:** The current implementation uses shadcn's basic `<Select>` component, which is built on Radix UI's Select primitive. It does not support filtering or search out of the box. Developers often reach for a `<Combobox>` pattern (Command + Popover) but implement it incorrectly -- e.g., opening a full dialog instead of an inline dropdown, or not handling keyboard navigation.

**Prevention:**
- Use shadcn/ui's `Combobox` pattern: `<Popover>` + `<Command>` (from cmdk library, already a shadcn dependency). This provides built-in search, keyboard navigation, and virtualized rendering
- Filter items client-side (the full item list is already in the RTK Query cache). Do NOT add a server-side search endpoint for this -- the item count per business is small enough for client-side filtering
- Filter OUT: (a) items with `type === "bundle"` (bundles cannot contain bundles), (b) items with `status !== "active"`, (c) items already selected as components in this bundle (prevent duplicates)
- Debounce the search input if implementing custom filtering (300ms is sufficient)

**Detection:** Load the item picker with 100+ items, verify search filters instantly, verify keyboard navigation works (arrow keys, enter to select, escape to close).

**Phase relevance:** Searchable item picker phase -- replace `<Select>` with `<Combobox>` pattern.

---

### Pitfall 6: Duplicate Components in Bundle Config

**What goes wrong:** A user adds the same item as a component twice in a bundle. The current UI does not prevent this -- the dropdown allows selecting the same `itemId` multiple times. The API may accept this, resulting in a bundle with two rows for "Copper Pipe" (e.g., quantity 5 and quantity 3) instead of one row with quantity 8.

**Why it happens:** There is no uniqueness check on `component.itemId` in either the form validation (`bundleItemFormSchema`) or the API validation (`BundleConfigValidator`). The `BundleComponentsList` uses `index` as the key (line 116), which hides the duplication in React rendering.

**Consequences:** Duplicate components create confusing bundle displays. On quotes, the same item appears twice as separate child line items. Pricing calculations may be correct numerically but the UX is confusing.

**Prevention:**
- Client-side: In the searchable dropdown, disable or hide items that are already selected as components. The `availableItemsForBundle` memo already filters by type and status -- add a filter to exclude `itemIds` that are already in the `components` array
- Client-side: Add Valibot validation that `components` has unique `itemId` values: `v.custom((components) => new Set(components.map(c => c.itemId)).size === components.length, "Duplicate components")`
- Server-side: Add a uniqueness check in `BundleConfigValidator` or the item update service. Throw `InvalidRequestError` with a clear message
- Use `component.itemId` as the React key instead of `index` when the item is selected

**Detection:** Try adding the same item twice in the bundle form. Verify it is prevented or visually warned.

**Phase relevance:** Bundle component editing phase -- add the constraint when enabling editing.

---

### Pitfall 7: Expandable Bundle Lines UX Confusion

**What goes wrong:** On the quote detail view, bundle line items show as a single rolled-up row (the parent) with an expand/collapse toggle to reveal components. Users do not understand what the expand icon means, or they think the components are separate billable items on top of the bundle price. The total appears to "not add up" visually.

**Why it happens:** The parent line item's `lineTotal` IS the bundle total (either fixed price or sum of components). The component line items are breakdowns, not additional charges. But without clear visual hierarchy, users (especially the tradesperson's customers who receive the quote) may misread the structure.

**Consequences:** Users distrust the quote totals. They may manually override prices to "fix" what they perceive as a calculation error. Customer-facing quotes look confusing.

**Prevention:**
- Visual hierarchy is critical:
  - Parent row: normal styling, shows bundle total, has a chevron/expand icon with "View components" tooltip
  - Child rows: indented (left padding or border), muted/secondary text color, smaller font or `text-muted-foreground`
  - Child rows should NOT show individual prices if the bundle uses `fixed` pricing strategy (the per-component prices are internal allocations, not meaningful to the customer)
  - For `component_based` pricing, child rows can show individual prices since they sum to the parent total
- Add a subtle label on the parent row: "Bundle" badge similar to the existing `ItemType` badge pattern
- On collapse, show the component count: "3 components" as secondary text on the parent row
- Do NOT show component totals in the quote summary/totals section -- only parent line items contribute to totals (the calculator already handles this)

**Detection:** Create a quote with a bundle (3 components) and a standalone item. Verify: (a) the total is bundle total + standalone total, (b) expanding the bundle shows components with clear visual nesting, (c) collapsing hides components cleanly.

**Phase relevance:** Quote detail UI phase -- design the component expansion before coding.

---

### Pitfall 8: Money Value Object Serialization Between API and UI

**What goes wrong:** The API uses a `Money` value object for all price fields (`unitPrice`, `lineTotal`, `discountAmount`, etc. on `IQuoteLineItemDto`). The UI receives these as serialized objects (likely `{ amount: number, currency: string }` or similar). If the UI treats these as raw numbers, arithmetic operations produce wrong results due to floating-point issues.

**Why it happens:** The API's `Money` class handles precision internally (likely using minor units / cents). When serialized to JSON, the structure may not be obvious. Frontend developers may do `lineItem.unitPrice * lineItem.quantity` directly on the serialized values, bypassing the precision guarantees.

**Prevention:**
- Check exactly how `Money` serializes in the API response (read `money.value-object.ts` and the response mapper). Ensure the UI type definitions match the serialized shape
- On the UI, use a consistent formatting utility (`formatCurrency` from `useCurrency` hook) for display -- never do raw arithmetic on money values in the frontend
- The quote totals should be calculated server-side ONLY (via `QuoteTotalsCalculator`). The UI should display the pre-calculated `totals.subTotal`, `totals.taxTotal`, `totals.total` from the API response, NOT recalculate them client-side
- If any client-side price preview is needed (e.g., showing estimated total before saving), round to 2 decimal places and label it as "estimated"

**Detection:** Add a bundle with components that have fractional prices (e.g., $1.33 each, quantity 3). Verify the displayed total matches the API-calculated total exactly.

**Phase relevance:** Quote UI wiring phase -- establish the pattern for displaying money values early.

---

### Pitfall 9: Item Deletion or Deactivation After Adding to Quote

**What goes wrong:** A user creates a quote with a bundle. Later, they deactivate or delete one of the bundle's component items. The quote still references the item by `itemId`. When displaying the quote, the UI tries to look up the item name and gets a 404 or shows "Unknown Item."

**Why it happens:** Quote line items store `itemId` as a reference but the line item itself is a snapshot (it has its own `unitPrice`, `unit`, `type`). The problem is display-only: the line item data is self-contained for pricing, but the UI may fetch the item name from the items cache to display a friendly name instead of an ID.

**Prevention:**
- Store the item name on the quote line item at creation time (snapshot pattern). This may require adding a `name` or `description` field to `IQuoteLineItemDto`. If this is too much schema change for v1.2, handle it in the UI:
- UI fallback: If the item lookup from RTK Query cache fails, display the line item with generic text like "Item (removed)" rather than crashing or showing an ID
- Do NOT prevent item deactivation based on quote references -- that creates tight coupling and frustrates users

**Detection:** Create a quote with items, deactivate one of the items, view the quote. Verify the display degrades gracefully.

**Phase relevance:** Quote detail UI phase -- handle graceful degradation for deleted/deactivated items.

---

### Pitfall 10: Bundle Creation Bug (Unit Field Defaulting to "bundle")

**What goes wrong:** This is an existing known bug listed in the milestone scope. When creating a bundle item, the `unit` field defaults to "bundle" (from the `ItemType` enum value leaking into the unit field). This produces nonsensical display: "1 bundle" instead of "1 each" or "1 pkg."

**Why it happens:** Likely in the API's item creation path, the `unit` field is being set from the `type` field when not provided. For non-bundle items this works (e.g., "hour" for labour), but for bundles the type name leaks into the unit.

**Prevention:**
- Fix in the API: bundle items should default `unit` to "each" or "pkg" (or whatever the business convention is), not inherit from the type name
- Validate: the `CreateItemRequest` should NOT accept `type` as a valid value for `unit` when type is "bundle"
- Fix existing bad data: consider a migration or lazy fix (update on next edit) for any bundles already created with `unit: "bundle"`

**Detection:** Create a bundle item without specifying a unit. Verify it defaults to a sensible value like "each."

**Phase relevance:** First phase -- fix this bug before building on top of bundles.

---

## Minor Pitfalls

### Pitfall 11: Form State Reset When Switching Between Create and Edit Modes

**What goes wrong:** The `ItemFormDialog` may reuse the same form component for both create and edit. If the dialog opens for "create bundle," user fills in data, closes without saving, then opens "edit bundle" -- the form still has the unsaved create data instead of the bundle's actual data.

**Prevention:**
- Reset form state when the dialog opens by using `useForm` with `defaultValues` derived from the `item` prop
- Add a `key` prop to the form component that changes between create/edit (e.g., `key={item?.id ?? "create"}`) to force React to remount the form
- Alternatively, call `form.reset(defaultValues)` in a `useEffect` when the `item` prop changes

**Phase relevance:** Bundle editing phase -- verify form reset behavior when enabling edit mode.

---

### Pitfall 12: Optimistic Updates vs. Server Recalculation for Quote Totals

**What goes wrong:** After adding a line item to a quote, the UI could optimistically update the displayed totals before the server response arrives. But the server-calculated totals (using `Money` precision, bundle pricing allocation, blended tax rates) may differ from the client's naive arithmetic. The totals "flash" -- showing one value, then correcting to another.

**Prevention:**
- Do NOT implement optimistic updates for quote totals. The calculation logic (bundle pricing plans, proportional discount allocation, blended tax rates) is too complex to replicate client-side
- Show a loading state on the totals section while the mutation is in flight. A simple spinner or skeleton on the totals row is sufficient
- The response from the "add line item" mutation (or the refetched quote detail) provides the authoritative totals -- display those

**Phase relevance:** Quote UI wiring phase -- avoid the temptation to optimize perceived performance with optimistic updates on calculated fields.

---

### Pitfall 13: Quantity Validation Edge Cases on Bundle Components

**What goes wrong:** The quantity input on bundle components accepts `0`, negative values, or non-numeric strings. `parseFloat(e.target.value) || 1` (current code, line 149) handles empty/NaN by falling back to 1, but does not prevent 0 or negative quantities. The API's `BundleConfigValidator` checks for empty components but does not validate individual component quantities.

**Prevention:**
- Client-side: Valibot schema should enforce `quantity > 0` on each component: `v.number([v.minValue(0.01, "Quantity must be greater than zero")])`
- Server-side: Add a quantity check in `BundleConfigValidator` or in the item creator/updater service -- each component must have `quantity > 0`
- The quantity input should use `min="0.01"` (already present) but also clamp on blur, not just on change, to catch paste/autofill edge cases

**Phase relevance:** Bundle component editing phase -- tighten validation when enabling editing.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Bundle creation bug fix | Unit defaults to "bundle" (Pitfall 10) | Fix default unit value for bundle type in API |
| Bundle component editing | Submit handler skips bundleConfig on edit (Pitfall 2) | Remove both UI guards (submit handler + read-only flag) in same commit |
| Bundle component editing | Duplicate components allowed (Pitfall 6) | Add uniqueness validation client-side and server-side |
| Bundle component editing | Quantity edge cases (Pitfall 13) | Enforce quantity > 0 in Valibot schema and API validator |
| Searchable item picker | Performance with basic Select (Pitfall 5) | Use Combobox pattern (Popover + Command) from shadcn/ui |
| Quote line item management | Double-counting bundle components (Pitfall 3) | Verify parentLineItemId is set before totals calculation |
| Quote UI wiring | Cache invalidation for nested data (Pitfall 4) | Single quote detail endpoint; mutations invalidate parent Quote tag |
| Quote UI wiring | Money serialization mismatch (Pitfall 8) | Display server-calculated totals only; use formatCurrency for display |
| Quote UI wiring | Optimistic update temptation (Pitfall 12) | Do not optimistically update totals; show loading state instead |
| Quote detail view | Expandable bundles confuse users (Pitfall 7) | Clear visual hierarchy: indented, muted child rows; "Bundle" badge on parent |
| Quote detail view | Deleted items break display (Pitfall 9) | Graceful fallback for missing item references |
| All editing phases | Bundle edits do not affect existing quotes (Pitfall 1) | Intentional design: quotes are snapshots; communicate in UI |
| All form phases | Form state not resetting (Pitfall 11) | Use key prop or form.reset() on dialog open |

## Summary: Top 3 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Submit handler silently drops bundleConfig on edit (Pitfall 2) | HIGH | Bundle component editing appears to work but saves nothing | Remove the `if (!isEditMode)` guard; update API merge utility |
| Cache not refreshing after quote line item changes (Pitfall 4) | HIGH | Users think changes were not saved; may duplicate work | Define tag invalidation strategy upfront; mutations invalidate parent Quote tag |
| Bundle component double-counting in totals (Pitfall 3) | MEDIUM | Quote totals 2x actual amount; financial correctness issue | Verify parentLineItemId pipeline end-to-end; write explicit tests |

---
*Research completed: 2026-03-08*
