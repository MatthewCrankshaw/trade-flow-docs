# Phase 12: Bundle Component Editing - Context

**Gathered:** 2026-03-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Enable full component management (add/remove/change quantity) on existing bundles via the edit form, and provide a clear structured read-only display of bundle components on the items list page. The API must be extended to support updating bundle components (currently only priceStrategy/bundlePrice can be updated).

</domain>

<decisions>
## Implementation Decisions

### Component display format (edit form)
- Two-line format per component: item name + colour-coded type badge on first line, quantity x unit on second line
- Type badges use same colour coding as items list page (Material/Labour/Fee each get their own colour)
- Header shows component count: "Components (3)" — no cost/price shown
- No cost shown per row or in header — cost is a quote concern, not bundle definition

### Component display format (read-only view)
- Bundle rows on items list page (table and card) are expandable — collapsed by default
- Expanded view uses same two-line format: name + type badge, qty x unit
- On mobile card view, expandable section shows "Components (3)" that reveals the list
- Consistent with Phase 14's expandable bundle lines on quotes

### Edit interactions
- No confirmation on component removal — nothing saved until form submitted, easily reversible by cancelling
- Quantity editing via direct input field (same as create form) — no stepper buttons
- Block save with validation error when zero components: "At least one component is required" — consistent with API validation (1-100 components)
- No change tracking or visual diff — standard form behavior, consistent with other edit forms

### Empty state
- Simple text "No components added yet" with SearchableItemPicker add button below

### Claude's Discretion
- Exact chevron/expand icon styling for collapsible rows
- Indentation and spacing for nested component rows in table
- How the card view expandable section animates
- API update endpoint changes needed to support component array updates

</decisions>

<specifics>
## Specific Ideas

- Expanded components in the items list should feel like the expandable bundle lines planned for Phase 14 quotes — establishing the pattern here
- Type badges should match existing item type badge styling for visual consistency across the app

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SearchableItemPicker` (`src/components/SearchableItemPicker.tsx`): Already built in Phase 11 — handles item search, grouping by type, exclusion of already-added items
- `BundleComponentsList` (`src/features/items/components/forms/shared/BundleComponentsList.tsx`): Existing component list for create form — needs enhancement for two-line display format
- `BundleItemForm` (`src/features/items/components/forms/BundleItemForm.tsx`): Bundle form with component handlers (add/update/remove) — needs to work in edit context
- `Collapsible` component (`src/components/ui/collapsible.tsx`): shadcn/ui collapsible available for expandable rows
- `useCurrency` hook: Available for any price formatting if needed

### Established Patterns
- `ItemsDataView` / `ItemsTable` / `ItemsCardList`: DataView pattern switches between table (desktop) and cards (mobile) — expandable rows need to work in both
- React Hook Form + Valibot schema validation for forms
- RTK Query for data fetching with cache invalidation
- Type badges already exist in `SearchableItemPicker` — can extract/reuse the badge styling

### Integration Points
- `ItemsTable.tsx` / `ItemsCardList.tsx`: Add expandable component display for bundle rows
- `BundleItemForm.tsx`: Ensure edit mode populates existing components and sends full component array on update
- `itemApi.ts`: Update mutation needs to send bundleConfig with components
- API `UpdateItemRequest`: Must be extended to accept components array in bundleConfig
- API `ItemUpdaterService`: Must validate component changes (same rules as create: active, non-bundle, 1-100 count)
- API `mergeExistingItemWithChanges`: Must handle component array merging

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 12-bundle-component-editing*
*Context gathered: 2026-03-08*
