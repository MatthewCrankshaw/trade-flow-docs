# Phase 11: Bundle Bug Fix and Foundation - Research

**Researched:** 2026-03-08
**Domain:** React UI -- bug fix + shared component creation (Popover+Command combobox)
**Confidence:** HIGH

## Summary

Phase 11 addresses two distinct concerns: (1) a straightforward bug where the `BundleItemForm` submit handler omits the required `unit` field, causing API validation errors on bundle creation, and (2) building a reusable `SearchableItemPicker` component that replaces the basic `<Select>` dropdown in `BundleComponentsList` with a searchable, grouped combobox.

The bug fix is a one-line addition. The picker component has an established, working pattern to follow -- `CreateJobDialog.tsx` already implements a `Popover`+`Command` (cmdk) searchable combobox with manual filtering, selection, and auto-close. All required shadcn/ui components (`Command`, `CommandInput`, `CommandList`, `CommandGroup`, `CommandItem`, `CommandEmpty`, `Popover`, `PopoverContent`, `PopoverTrigger`) are already installed and in use.

**Primary recommendation:** Follow the `CreateJobDialog.tsx` combobox pattern exactly. The bug fix is a single `unit: "bundle"` addition to `BundleItemForm.tsx` line 82-88.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Bundle items must send `unit: "bundle"` on creation -- the API requires a non-empty `unit` string
- The fix is in `BundleItemForm.tsx` submit handler (line 82-96) which currently omits the `unit` field
- Build as a standalone shared component in `src/components/` (not inline in BundleComponentsList)
- Will be reused by both bundle forms (Phase 11/12) and quote line item addition (Phase 14)
- One item selected at a time -- user clicks Add, picks one item, new row appears
- Auto-close dropdown after selection (consistent with CreateJobDialog pattern)
- Each item in the dropdown shows: name, type badge (Material/Labour/Fee), and default price
- Items grouped by type using CommandGroup headers: Materials, Labour, Fees
- Search filters across all groups simultaneously (not per-group tabs)
- Items already in the bundle are hidden from the picker (not shown disabled)
- Each item can only appear once in a bundle -- one row per item
- Quantity is adjusted on the existing row, not by adding duplicate rows
- This is consistent with hiding already-added items from the picker

### Claude's Discretion
- Exact component API surface for SearchableItemPicker (props, callbacks)
- Price formatting in the dropdown (minor units to display)
- Empty state when no items match the search
- Whether to show item count per group header

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| BNDL-01 | User can create a bundle item without errors (unit defaults to "bundle") | Bug traced: `BundleItemForm.tsx` submit handler omits `unit` field; `ItemFormDialog.tsx` passes `itemData.unit!` (undefined) to `CreateItemRequest` which requires `unit: string`. Fix: add `unit: "bundle"` to itemData object. |
| BNDL-03 | User can search and filter items when selecting bundle components via a searchable dropdown | Pattern established: `CreateJobDialog.tsx` Popover+Command combobox with `shouldFilter={false}`, manual `useMemo` filtering. All shadcn/ui primitives already installed. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| cmdk (via shadcn/ui Command) | Already installed | Searchable command palette / combobox | Used in CreateJobDialog; project standard for searchable dropdowns |
| Radix Popover (via shadcn/ui) | Already installed | Floating dropdown container | Pairs with Command for combobox pattern |
| React Hook Form + Valibot | Already installed | Form state and validation | Project standard for all forms |
| Tailwind CSS 4.x | Already installed | Styling | Project standard |
| lucide-react | Already installed | Icons (Check, ChevronsUpDown, Search) | Project standard icon library |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `useBusinessCurrency` hook | Existing | Format `defaultPrice` (minor units) for display | When showing price in picker dropdown |
| `cn()` utility | Existing | Conditional Tailwind classes | Standard for all conditional styling |
| Badge component | Existing shadcn/ui | Type badges (Material/Labour/Fee) | For item type indicators in picker |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Popover+Command | Radix Select with search | Not established in codebase; Command pattern already proven |
| Manual filtering with useMemo | cmdk built-in filtering | CreateJobDialog uses `shouldFilter={false}` with manual filtering for consistency |

**Installation:** No new packages needed. All dependencies already installed.

## Architecture Patterns

### Recommended Project Structure
```
src/
├── components/
│   └── SearchableItemPicker.tsx    # NEW: standalone shared component
├── features/items/
│   └── components/forms/
│       ├── BundleItemForm.tsx       # MODIFY: add unit: "bundle"
│       └── shared/
│           └── BundleComponentsList.tsx  # MODIFY: replace <Select> with SearchableItemPicker
└── lib/forms/schemas/
    └── item.schema.ts              # NO CHANGES needed
```

### Pattern 1: Popover+Command Combobox (from CreateJobDialog.tsx)
**What:** Searchable dropdown using Popover for positioning and Command (cmdk) for keyboard-navigable, filterable list
**When to use:** Any time a user needs to search and select from a list of options
**Example:**
```typescript
// Source: trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx
// Key elements of the pattern:

// 1. State management
const [popoverOpen, setPopoverOpen] = useState(false);
const [search, setSearch] = useState("");

// 2. Manual filtering with useMemo
const filteredItems = useMemo(() => {
  if (!search.trim()) return items;
  const s = search.toLowerCase();
  return items.filter((item) => item.name.toLowerCase().includes(s));
}, [items, search]);

// 3. Popover + Command with shouldFilter={false}
<Popover open={popoverOpen} onOpenChange={setPopoverOpen}>
  <PopoverTrigger asChild>
    <Button variant="outline" role="combobox" className="w-full justify-between">
      {/* trigger content */}
      <ChevronsUpDown className="ml-2 h-4 w-4 shrink-0 opacity-50" />
    </Button>
  </PopoverTrigger>
  <PopoverContent align="start">
    <Command shouldFilter={false}>
      <CommandInput
        placeholder="Search items..."
        value={search}
        onValueChange={setSearch}
      />
      <CommandList>
        <CommandEmpty>No items found.</CommandEmpty>
        <CommandGroup heading="Materials">
          {filteredMaterials.map((item) => (
            <CommandItem
              key={item.id}
              value={item.id}
              onSelect={() => {
                handleSelect(item);
                setPopoverOpen(false); // auto-close
                setSearch("");         // reset search
              }}
            >
              {item.name}
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

### Pattern 2: SearchableItemPicker Component API (Recommended)
**What:** Standalone picker component encapsulating the Popover+Command pattern
**When to use:** Bundle component selection (Phase 11/12) and quote line item addition (Phase 14)
**Example:**
```typescript
// Recommended component API surface
interface SearchableItemPickerProps {
  items: Item[];                    // All available items to show
  excludeItemIds?: string[];        // Items to hide (already selected)
  onSelect: (item: Item) => void;   // Callback when item is selected
  placeholder?: string;             // Trigger button text
  disabled?: boolean;
}

// Usage in BundleComponentsList:
<SearchableItemPicker
  items={availableItemsForBundle}
  excludeItemIds={components.map(c => c.itemId)}
  onSelect={handleItemSelected}
  placeholder="Add component..."
/>
```

### Pattern 3: Interaction Flow Change (Add Component)
**What:** Replace "Add" button + empty Select row with a single "Add" button that opens the picker directly
**When to use:** For the bundle component creation flow
**Detail:** Currently, clicking "Add" creates an empty row with a blank `<Select>`. The new flow should be: clicking "Add" opens the `SearchableItemPicker` popover. Selecting an item from the picker adds the row with the item pre-selected. This is cleaner and eliminates the "empty row" state.

### Anti-Patterns to Avoid
- **Using cmdk's built-in filtering:** The project standard is `shouldFilter={false}` with manual `useMemo` filtering. This gives full control over how items are filtered and grouped.
- **Inline combobox implementation:** Don't embed the Popover+Command logic directly in BundleComponentsList. Extract to `SearchableItemPicker` for reuse.
- **Showing disabled items:** User decision is to hide already-added items, not show them grayed out.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Searchable dropdown | Custom search input + list | shadcn/ui Command (cmdk) + Popover | Keyboard navigation, accessibility, focus management built in |
| Currency formatting | Manual division/formatting | `useBusinessCurrency().formatAmount(price)` | Handles minor units, locale, currency code |
| Type badges | Custom colored spans | shadcn/ui `<Badge>` component | Consistent styling, already used in codebase |
| Form validation | Manual validation logic | Valibot schema + React Hook Form | Project standard, already set up in `bundleItemFormSchema` |

**Key insight:** Every UI primitive needed for this phase already exists in the codebase. This phase is about composition, not creation of new primitives.

## Common Pitfalls

### Pitfall 1: Forgetting to Reset Search on Selection
**What goes wrong:** After selecting an item, the search term persists. Next time the picker opens, it shows filtered results instead of all items.
**Why it happens:** Easy to forget `setSearch("")` alongside `setPopoverOpen(false)`.
**How to avoid:** Always clear search state when closing the popover (both on select and on outside click). The `CreateJobDialog` pattern shows this.
**Warning signs:** Picker shows "no results" when opened after a previous search.

### Pitfall 2: Price Stored in Minor Units
**What goes wrong:** Displaying `defaultPrice` as-is shows values like "1500" instead of "15.00".
**Why it happens:** The API stores prices in minor units (pence/cents).
**How to avoid:** Use `useBusinessCurrency().formatAmount(item.defaultPrice)` which handles minor-to-major conversion and locale formatting.
**Warning signs:** Prices showing as hundreds/thousands when they should be single/double digits.

### Pitfall 3: The BundleItemForm unit Bug -- Understanding the Full Path
**What goes wrong:** `BundleItemForm` returns `itemData` without `unit` field. `ItemFormDialog.handleSubmit` does `unit: itemData.unit!` which becomes `undefined`. `CreateItemRequest` type requires `unit: string`. API rejects the request.
**Why it happens:** The bundle form schema (`bundleItemFormSchema`) intentionally omits `unit` (bundles always use "bundle"), but the submit handler forgets to add it.
**How to avoid:** Add `unit: "bundle"` to the `itemData` object in `BundleItemForm.handleSubmit` (line 82-88). The API's own default bundle items use `unit: "Bundle"` (see `default-business-items-creator.service.ts` line 362).
**Warning signs:** API error on bundle creation; `unit` field validation failure.

### Pitfall 4: Not Handling Empty Components Array on Initial Add
**What goes wrong:** If the picker integration changes the "Add" flow, the `components` validation (`length > 0`) could trigger prematurely.
**Why it happens:** The Valibot schema requires at least one component. If validation runs before the user has added their first component, it shows an error.
**How to avoid:** The current approach of checking on submit (not on every change) is correct. Ensure `shouldValidate: true` is only used when adding/removing, not on initial form mount.

## Code Examples

### Bug Fix: Add unit to BundleItemForm submit (BNDL-01)
```typescript
// Source: trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
// Current code (line 82-88):
const itemData: Partial<Item> = {
  type: "bundle",
  businessId,
  name: values.name.trim(),
  description: values.description?.trim() || undefined,
  status: values.status,
};

// Fixed code -- add unit field:
const itemData: Partial<Item> = {
  type: "bundle",
  businessId,
  name: values.name.trim(),
  description: values.description?.trim() || undefined,
  status: values.status,
  unit: "bundle",  // Required by API -- bundles always use "bundle" as unit
};
```

### GroupBy Pattern for Item Types
```typescript
// Group items by type for CommandGroup headers
const groupedItems = useMemo(() => {
  const groups: Record<string, Item[]> = {
    material: [],
    labour: [],
    fee: [],
  };

  for (const item of filteredItems) {
    if (groups[item.type]) {
      groups[item.type].push(item);
    }
  }

  return groups;
}, [filteredItems]);

// Render grouped items
const typeLabels: Record<string, string> = {
  material: "Materials",
  labour: "Labour",
  fee: "Fees",
};
```

### Excluding Already-Selected Items
```typescript
// Filter out items already in the bundle
const availableItems = useMemo(() => {
  const selectedIds = new Set(components.map(c => c.itemId));
  return availableItemsForBundle.filter(item => !selectedIds.has(item.id));
}, [availableItemsForBundle, components]);
```

### Price Display in Picker
```typescript
// Using useBusinessCurrency for price formatting
const { formatAmount } = useBusinessCurrency();

// In CommandItem:
<span className="text-muted-foreground text-sm">
  {item.defaultPrice !== null ? formatAmount(item.defaultPrice) : "No price"}
</span>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `<Select>` dropdown | Popover+Command combobox | Already in use (CreateJobDialog) | Searchable, keyboard-navigable, groupable |
| Inline component logic | Shared standalone components in `src/components/` | Project convention | Reusable across features |

**Deprecated/outdated:**
- The current `<Select>` in BundleComponentsList is the approach being replaced. It has no search capability and doesn't scale beyond a handful of items.

## Open Questions

1. **Should the "Add" button open the picker directly, or should the picker replace the Select in each row?**
   - What we know: Context says "One item selected at a time -- user clicks Add, picks one item, new row appears." This implies the Add button triggers the picker.
   - What's unclear: Does the Add button become the picker trigger, or does clicking Add open a separate popover?
   - Recommendation: Make the Add button the PopoverTrigger. When an item is selected, a new component row is added with that item pre-filled. This eliminates the "empty itemId" intermediate state.

2. **Case sensitivity for unit value: "bundle" vs "Bundle"**
   - What we know: API default items use `"Bundle"` (capital B). The `CreateItemRequest` type just says `unit: string`.
   - What's unclear: Whether the API treats these differently.
   - Recommendation: Use lowercase `"bundle"` to match the `type: "bundle"` convention. The API accepts any string.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest (via Vite) |
| Config file | `trade-flow-ui/vitest.config.ts` (if exists) or Vite config |
| Quick run command | `cd trade-flow-ui && npm run typecheck && npm run lint` |
| Full suite command | `cd trade-flow-ui && npm run typecheck && npm run lint && npm run build` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BNDL-01 | Bundle creation sends unit: "bundle" | manual | Verify via `npm run typecheck` (type safety) + manual test in UI | N/A |
| BNDL-03 | SearchableItemPicker filters and groups items | manual | `npm run typecheck && npm run lint` (compile + lint verification) | N/A |

### Sampling Rate
- **Per task commit:** `cd trade-flow-ui && npm run typecheck && npm run lint`
- **Per wave merge:** `cd trade-flow-ui && npm run build`
- **Phase gate:** Full build green before verify

### Wave 0 Gaps
None -- existing lint and typecheck infrastructure covers compile-time verification. No unit test framework is actively used in this project; verification is via typecheck + lint + manual testing.

## Sources

### Primary (HIGH confidence)
- `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` -- full working Popover+Command combobox pattern
- `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` -- bug location confirmed (unit field missing)
- `trade-flow-ui/src/features/items/components/ItemFormDialog.tsx` -- bug propagation path (`itemData.unit!` passes undefined)
- `trade-flow-ui/src/types/api.types.ts` -- `CreateItemRequest` confirms `unit: string` is required
- `trade-flow-api/src/business/services/default-business-items-creator.service.ts` -- API uses `unit: "Bundle"` for default bundles
- `trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx` -- current Select implementation to replace
- `trade-flow-ui/src/lib/forms/schemas/item.schema.ts` -- bundle schema (no unit field, by design)
- `trade-flow-ui/src/hooks/useCurrency.ts` -- `useBusinessCurrency().formatAmount()` for price display

### Secondary (MEDIUM confidence)
- shadcn/ui Command component docs -- standard cmdk wrapper API

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and in active use in the codebase
- Architecture: HIGH -- exact pattern exists in CreateJobDialog; bug path fully traced through code
- Pitfalls: HIGH -- all pitfalls derived from direct code inspection, not conjecture

**Research date:** 2026-03-08
**Valid until:** 2026-04-08 (stable -- no dependency changes expected)
