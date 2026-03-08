# Phase 11: Bundle Bug Fix and Foundation - Context

**Gathered:** 2026-03-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Fix the bundle creation bug (unit field must default to "bundle") and replace the basic `<Select>` dropdown with a standalone, searchable item picker component using the existing Popover+Command (cmdk) pattern. The picker must work in both create and edit contexts (Phase 12 will enable edit mode).

</domain>

<decisions>
## Implementation Decisions

### Bug fix — unit field
- Bundle items must send `unit: "bundle"` on creation — the API requires a non-empty `unit` string
- The fix is in `BundleItemForm.tsx` submit handler (line 82-96) which currently omits the `unit` field

### Searchable item picker — component scope
- Build as a standalone shared component in `src/components/` (not inline in BundleComponentsList)
- Will be reused by both bundle forms (Phase 11/12) and quote line item addition (Phase 14)
- One item selected at a time — user clicks Add, picks one item, new row appears
- Auto-close dropdown after selection (consistent with CreateJobDialog pattern)

### Searchable item picker — display and grouping
- Each item in the dropdown shows: name, type badge (Material/Labour/Fee), and default price
- Items grouped by type using CommandGroup headers: Materials, Labour, Fees
- Search filters across all groups simultaneously (not per-group tabs)
- Items already in the bundle are hidden from the picker (not shown disabled)

### Duplicate handling
- Each item can only appear once in a bundle — one row per item
- Quantity is adjusted on the existing row, not by adding duplicate rows
- This is consistent with hiding already-added items from the picker

### Claude's Discretion
- Exact component API surface for SearchableItemPicker (props, callbacks)
- Price formatting in the dropdown (minor units → display)
- Empty state when no items match the search
- Whether to show item count per group header

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Follow the CreateJobDialog Popover+Command pattern for consistency.

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `CreateJobDialog.tsx`: Full Popover+Command searchable combobox with search state, filtering, selection, and "create new" inline flow — the pattern to follow
- `Command`, `CommandInput`, `CommandList`, `CommandGroup`, `CommandItem`, `CommandEmpty`, `CommandSeparator`: All shadcn/ui command components available
- `Popover`, `PopoverContent`, `PopoverTrigger`: Popover primitives available
- `cn()` utility for conditional classes
- `useCurrency` hook for price formatting

### Established Patterns
- `shouldFilter={false}` on Command with manual filtering via useMemo — consistent pattern in CreateJobDialog
- Items already loaded via `useGetItemsQuery` in parent and passed as `allItems` prop
- `availableItemsForBundle` filtering: excludes bundles and inactive items

### Integration Points
- `BundleComponentsList.tsx` line 120-136: Replace `<Select>` with new SearchableItemPicker
- `BundleItemForm.tsx` line 82-96: Add `unit: "bundle"` to itemData in submit handler
- New component location: `src/components/SearchableItemPicker.tsx` (or similar)

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 11-bundle-bug-fix-and-foundation*
*Context gathered: 2026-03-08*
