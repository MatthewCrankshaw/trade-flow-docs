# Technology Stack

**Project:** Trade Flow v1.2 - Bundles & Quotes
**Researched:** 2026-03-08

## Verdict: No New Dependencies Required

The existing stack already contains every library needed for this milestone. All three major UI capabilities (searchable combobox, expandable rows, bundle component management) are already installed and partially in use.

## Existing Stack (Already Installed - USE THESE)

### Searchable/Filterable Item Dropdown

| Technology | Version | Purpose | Status |
|------------|---------|---------|--------|
| `cmdk` | 1.1.1 | Command menu with fuzzy search, keyboard navigation | Already installed, wraps `Command` UI component |
| `radix-ui` (Popover) | 1.4.3 | Popover container for combobox pattern | Already installed, `Popover` UI component exists |

**Why no new library:** The project already has a working `Command + Popover` combobox pattern in `CreateJobDialog.tsx` (lines 289-473). This exact pattern -- Popover trigger button, Command with `shouldFilter={false}`, manual search/filter via `useMemo`, CommandInput + CommandList + CommandItem -- should be reused directly for the bundle item picker. The `CreateJobDialog` implementation includes search, "create new" inline options, and selection state management. The bundle item picker is simpler (no "create new" flow, just select existing items).

**Confidence:** HIGH (verified against `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx`)

**How to apply for bundle components:**
- Replace the current `<Select>` in `BundleComponentsList.tsx` (line 120-136) with the `Popover + Command` pattern
- Use `shouldFilter={false}` and filter manually with `useMemo` (matches existing pattern)
- Filter to `type !== "bundle"` and `status === "active"` (already done in `availableItemsForBundle`)
- Show item name, unit, and price in each CommandItem for better selection context

### Expandable Table Rows (Quote Bundle Lines)

| Technology | Version | Purpose | Status |
|------------|---------|---------|--------|
| `@radix-ui/react-collapsible` | 1.1.12 | Animate expand/collapse of content regions | Already installed, `Collapsible` UI component exists |
| shadcn `Table` components | n/a | Standard table primitives (Table, TableRow, TableCell, etc.) | Already installed |

**Why no new library:** The API already returns bundle line items with a nested `components` array (see `IQuoteLineItemResponse.components`). The UI needs a clickable row that expands to show child component rows. This is a `Collapsible` wrapping additional `TableRow` elements -- no data table library needed.

**Confidence:** HIGH (verified against `trade-flow-ui/src/components/ui/collapsible.tsx` and `trade-flow-api/src/quote/responses/quote.responses.ts`)

**Implementation pattern:**
```tsx
// Collapsible wrapping table rows for bundle expansion
<Collapsible>
  <TableRow>
    <TableCell>
      <CollapsibleTrigger>
        <ChevronRight className="transition-transform data-[state=open]:rotate-90" />
      </CollapsibleTrigger>
    </TableCell>
    <TableCell>{bundleItem.name}</TableCell>
    <TableCell>{formatCurrency(bundleItem.lineTotal)}</TableCell>
  </TableRow>
  <CollapsibleContent asChild>
    <>
      {bundleItem.components.map(component => (
        <TableRow key={component.id} className="bg-muted/30">
          <TableCell /> {/* indent spacer */}
          <TableCell className="pl-8">{component.name}</TableCell>
          <TableCell>{formatCurrency(component.lineTotal)}</TableCell>
        </TableRow>
      ))}
    </>
  </CollapsibleContent>
</Collapsible>
```

### Quote UI Integration

| Technology | Version | Purpose | Status |
|------------|---------|---------|--------|
| RTK Query | via `@reduxjs/toolkit` | API data fetching, caching, mutations | Already installed, `"Quote"` tag defined but no endpoints yet |
| `react-hook-form` | 7.71.1 | Form state management for quote creation | Already installed |
| `valibot` | 1.2.0 | Schema validation for quote forms | Already installed |
| `@hookform/resolvers` | installed | Bridges valibot schemas to react-hook-form | Already installed |

**Confidence:** HIGH (verified against `trade-flow-ui/src/services/api.ts` and `trade-flow-api/src/quote/controllers/quote.controller.ts`)

**What needs to be BUILT (not installed):**

1. **RTK Query endpoints** for quotes -- the `api.ts` has a `"Quote"` tag but no quote endpoints yet. Required:
   - `getQuotes(businessId)` -- GET `business/:businessId/quotes`
   - `getQuote(quoteId)` -- GET `quote/:quoteId`
   - `createQuote({ businessId, data })` -- POST `business/:businessId/quote`
   - `addQuoteLineItem({ businessId, quoteId, data })` -- POST `business/:businessId/quote/:quoteId/line-item`

2. **Quote detail page** -- route, layout, line items display with expandable bundles

3. **Quote creation form** -- customer selector (reuse Combobox pattern from CreateJobDialog), title, notes

### Bundle Component Display & Editing

| Technology | Version | Purpose | Status |
|------------|---------|---------|--------|
| `lucide-react` | installed | Icons (ChevronRight, Package, Plus, Trash2, etc.) | Already installed |
| shadcn `Badge` | n/a | "optional" badges on components, status badges | Already installed |
| shadcn `Card` | n/a | Mobile card layout for quotes | Already installed |

**What needs to change in BundleComponentsList:**
- Current `isReadOnly = mode === "edit"` on line 53 blocks component editing. This guard must be removed to enable add/remove/update in edit mode.
- Replace `<Select>` (lines 120-136) with `Popover + Command` combobox for searchable item selection.
- Remove the "Component editing is not yet available" placeholder message (lines 164-166).

## Libraries Explicitly NOT Needed

| Library | Why Considered | Why Not |
|---------|---------------|---------|
| `@tanstack/react-table` | Expandable rows, sorting | Overkill for simple quote line items table (typically <20 rows, 5-7 columns). No other tables in the project use it. Collapsible + Table primitives achieve the same result with zero new deps. |
| `downshift` | Combobox/autocomplete | Already have cmdk + Popover pattern working in CreateJobDialog; switching adds no value |
| `react-select` | Searchable dropdown | Heavy (27KB min+gz), style-opinionated; cmdk is lighter and already integrated with shadcn theming |
| `react-pdf` / PDF libraries | Quote PDF generation | Out of scope for v1.2; quote acceptance/invoicing is next milestone |
| `@dnd-kit/core` | Drag-and-drop line item reordering | Over-engineering; line items don't need reordering in v1.2 |
| `decimal.js` / `dinero.js` | Currency math | API already handles Money value objects server-side; UI just displays pre-calculated values |

## Currency Formatting

The project already has a `useCurrency` hook. The quote UI should use this for formatting `unitPrice`, `lineTotal`, `subTotal`, `taxTotal`, and `total` values returned by the API (already in major units as numbers).

## Integration Points

### API Response Structure (Already Built)

The quote API returns line items with nested `components` for bundles:

```typescript
// IQuoteResponse.lineItems structure:
{
  id: string;
  itemId: string;
  quantity: number;
  unit: string;
  unitPrice: number;      // major units (e.g., 25.50)
  lineTotal: number;       // major units
  taxRate: number;          // percentage (e.g., 20)
  type: "material" | "labour" | "bundle";
  components?: IQuoteLineItemResponse[];  // only for bundles
}
```

The frontend does NOT need to compute totals or resolve bundle components -- the API handles all pricing and returns a nested structure ready for display.

### RTK Query Cache Strategy

Follow the existing pattern: queries tagged with `"Quote"`, mutations invalidate `"Quote"` tag. The `addLineItem` mutation should invalidate the specific quote's cache entry to refresh totals after a line item is added.

### Combobox Reuse Strategy

Extract the Popover + Command pattern from `CreateJobDialog` into a reusable `ComboboxSelect` component (or just follow the pattern inline). It will be used in at least two places:
1. Bundle component item picker (replacing `<Select>`)
2. Quote creation customer selector

Both use the same pattern: trigger button with chevron, Command with `shouldFilter={false}`, manual filtering via `useMemo`, CommandItem with check mark for selected state.

## Installation

```bash
# No new packages to install.
# All required dependencies are already in package.json.
```

## Sources

- Codebase: `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` -- Command+Popover combobox pattern (HIGH confidence)
- Codebase: `trade-flow-ui/src/components/ui/command.tsx` -- cmdk 1.1.1 wrapper (HIGH confidence)
- Codebase: `trade-flow-ui/src/components/ui/collapsible.tsx` -- Radix Collapsible component (HIGH confidence)
- Codebase: `trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx` -- current Select-based picker (HIGH confidence)
- Codebase: `trade-flow-api/src/quote/controllers/quote.controller.ts` -- existing API endpoints with nested line item mapping (HIGH confidence)
- Codebase: `trade-flow-api/src/quote/responses/quote.responses.ts` -- nested `components` array structure (HIGH confidence)
- Codebase: `trade-flow-ui/package.json` -- verified installed versions (HIGH confidence)

---
*Research completed: 2026-03-08*
