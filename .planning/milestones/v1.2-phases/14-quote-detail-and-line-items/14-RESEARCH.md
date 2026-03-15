# Phase 14: Quote Detail and Line Items - Research

**Researched:** 2026-03-14
**Domain:** React UI -- quote line items table, bundle expansion, SearchableItemPicker integration, RTK Query mutations
**Confidence:** HIGH

## Summary

Phase 14 replaces the placeholder line items card in `QuoteDetailPage.tsx` (lines 141-153) with a fully functional line items table, integrates the existing `SearchableItemPicker` component for adding items, implements collapsible bundle rows matching the Phase 12 pattern, and wires up the existing `addLineItem` API endpoint via a new RTK Query mutation.

The API layer is already complete. The `POST /business/:businessId/quote/:quoteId/line-item` endpoint accepts `{ itemId, quantity }`, handles both standard and bundle items server-side (including component creation with `parentLineItemId` linking), recalculates totals via `QuoteTotalsCalculator`, and returns the full quote response with nested `lineItems[].components[]`. The controller's `mapLineItems` method already builds the parent/child hierarchy. All money values arrive as major-unit numbers (via `toMajorUnits()`).

The UI work is: (1) add `QuoteLineItem` types to match the API response shape, (2) add an `addLineItem` RTK Query mutation, (3) build line items table/card components with expandable bundle rows, (4) adapt `SearchableItemPicker` to include bundles for the quote context, and (5) replace the placeholder. The totals card already renders correctly -- it just needs the quote to have line items.

**Primary recommendation:** Build a `QuoteLineItemsCard` component containing the table/card list with expand/collapse state, an "Add Item" button that opens a modified `SearchableItemPicker` (including bundles), and handle the add mutation with tag invalidation to refresh totals.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Reuse `SearchableItemPicker` from Phase 11 -- same Popover+Command pattern with grouped search and type badges
- "Add Item" button in the Line Items card header opens the picker
- After selecting an item, it's added immediately with quantity 1 -- user adjusts quantity directly in the line item row
- Same item can be added as multiple separate line items (allow duplicate lines)
- Adding items restricted to Draft and Sent quotes only -- Accepted/Rejected are fully read-only
- Editing a Sent quote triggers the existing "modified since sent" indicator from Phase 13
- Collapsible rows consistent with Phase 12's expandable bundle pattern on items list
- Collapsed by default: shows bundle name, component count, quantity, and rolled-up total
- Expanded: reveals individual component breakdown with indentation
- Component prices always shown -- for component-based pricing, show actual component prices; for fixed-price bundles, show adjusted component prices that sum to the fixed total
- Fixed (non-optional) components are locked -- cannot be removed or modified per-quote
- Optional components (isOptional: true in bundle definition) can be toggled off per-quote
- Full detail columns: Item name, type badge, qty, unit, unit price, tax rate, line total, delete action
- Bundle rows show expand chevron instead of individual unit price/tax (those appear on component rows)
- Quantity is an inline number input -- click to edit, saves on blur or Enter
- Unit price is editable per line item -- API already tracks originalUnitPrice vs unitPrice with discountAmount
- Mobile: card per line item showing item name, type badge, qty x unit, and line total; bundles have expand chevron
- Empty state: CTA-focused -- "No line items yet. Add items to build your quote." with prominent Add Item button
- Totals card always visible, showing $0.00 when empty
- Totals update after API response (server-side calculation) -- not optimistic. Brief loading indicator while API processes
- Add Item button also in card header when items exist

### Claude's Discretion
- Exact SearchableItemPicker integration for quote context (may need minor adaptations from bundle context)
- How optional component toggling works in the UI (checkbox, toggle, or remove button)
- Loading/saving indicator design for inline edits
- How the API handles optional component exclusion when adding a bundle to a quote
- Mobile card expand/collapse animation details
- Exact delete confirmation UX (or if immediate delete without confirmation given undo potential via API)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| QUOT-03 | User can view quote detail with line items and calculated totals | Existing `QuoteDetailPage.tsx` has header, action strip, totals card, and placeholder at lines 141-153. Replace placeholder with line items component. Totals card already renders `quote.totals` correctly. |
| QLIT-01 | User can add standard items (material, labour, fee) to a quote | API endpoint `POST /business/:businessId/quote/:quoteId/line-item` exists with `{ itemId, quantity }`. `QuoteStandardLineItemFactory` handles creation. Need RTK Query mutation + SearchableItemPicker integration. |
| QLIT-02 | User can add bundle items to a quote (creates parent + component line items) | `QuoteBundleLineItemFactory` handles bundle creation server-side, creates parent + component line items with `parentLineItemId` linking. Same API endpoint, item type detection is server-side. SearchableItemPicker needs modification to include bundles. |
| QLIT-03 | User can view bundle line items as a rolled-up line that expands to show individual components | API response already includes nested `components[]` on parent line items (via `mapLineItems` in controller). Phase 12's `ItemsTable`/`ItemsCardList` provide expandable row pattern with `expandedBundles` Set state. |
| QLIT-04 | User can view quote totals (subtotal, tax, total) calculated from line items | `QuoteTotalsCalculator` runs server-side after every addLineItem, skipping child items (`parentLineItemId` check). Totals returned in response as major-unit numbers. Existing totals card in `QuoteDetailPage.tsx` already renders these. |
</phase_requirements>

## Standard Stack

### Core (already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19 | UI framework | Project standard |
| Redux Toolkit / RTK Query | latest | State management & API | Project standard -- tag-based cache invalidation |
| shadcn/ui | latest | UI components (Table, Card, Badge, Popover, Command, Collapsible) | Project standard -- New York style |
| Tailwind CSS | 4.x | Styling | Project standard |
| lucide-react | latest | Icons (ChevronDown, ChevronRight, Plus, Trash2, Loader2) | Project standard |
| date-fns | latest | Date formatting | Project convention (not Luxon on UI) |

### Supporting (already available)
| Library | Purpose | When to Use |
|---------|---------|-------------|
| `@/components/SearchableItemPicker` | Item selection with search | Adding items to quote -- needs modification to include bundles |
| `@/hooks/useCurrency` | `formatDecimal` for major-unit amounts | All price/total display |
| `@/hooks/useMediaQuery` | Responsive breakpoint detection | Table vs card list switching |

### No New Dependencies
This phase requires zero new npm packages. Everything is already available.

## Architecture Patterns

### Component Structure
```
src/
├── features/quotes/
│   ├── components/
│   │   ├── QuoteLineItemsCard.tsx      # Main container: header + Add Item + table/cards
│   │   ├── QuoteLineItemsTable.tsx     # Desktop table with expandable bundle rows
│   │   ├── QuoteLineItemsCardList.tsx  # Mobile card list with expandable bundles
│   │   └── index.ts                    # Re-export new components
│   └── api/
│       └── quoteApi.ts                 # Add addLineItem mutation
├── components/
│   └── SearchableItemPicker.tsx        # Modify to support includeBundles prop
├── types/
│   └── quote.ts                        # Add QuoteLineItem type
└── pages/
    └── QuoteDetailPage.tsx             # Replace placeholder with QuoteLineItemsCard
```

### Pattern 1: RTK Query Mutation with Tag Invalidation
**What:** `addLineItem` mutation that invalidates the quote cache to refresh line items and totals.
**When to use:** Every add/edit/delete line item operation.
**Example:**
```typescript
// In quoteApi.ts
addLineItem: builder.mutation<
  Quote,
  { businessId: string; quoteId: string; itemId: string; quantity: number }
>({
  query: ({ businessId, quoteId, itemId, quantity }) => ({
    url: `/v1/business/${businessId}/quote/${quoteId}/line-item`,
    method: "POST",
    body: { itemId, quantity },
  }),
  transformResponse: (response: StandardResponse<Quote>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No quote data returned");
  },
  invalidatesTags: (_result, _error, { quoteId }) => [
    { type: "Quote", id: quoteId },
    { type: "Quote", id: "LIST" },
  ],
}),
```

### Pattern 2: Expandable Bundle Rows (from Phase 12)
**What:** `expandedBundles` Set<string> state with toggle function, rendering child rows when expanded.
**When to use:** Bundle line items in the table.
**Example:**
```typescript
// Same pattern as ItemsTable.tsx lines 77-97
const [expandedBundles, setExpandedBundles] = useState<Set<string>>(new Set());

const toggleBundleExpansion = (lineItemId: string) => {
  setExpandedBundles((prev) => {
    const next = new Set(prev);
    if (next.has(lineItemId)) next.delete(lineItemId);
    else next.add(lineItemId);
    return next;
  });
};
```

### Pattern 3: Item Name Resolution via Cross-Reference
**What:** API line items only contain `itemId`, not item names. Must cross-reference with items list.
**When to use:** Displaying line item names in the table.
**Critical detail:** The API response `IQuoteLineItemResponse` does NOT include item name -- only `itemId`, `type`, `unit`, `quantity`, prices. The UI must fetch the items list (via existing `useGetItemsQuery`) and build a lookup map, identical to Phase 12's `itemsById` pattern.
**Example:**
```typescript
const { data: items = [] } = useGetItemsQuery(businessId);
const itemsById = useMemo(() => {
  const map = new Map<string, Item>();
  items.forEach((item) => map.set(item.id, item));
  return map;
}, [items]);

// Then in render:
const itemName = itemsById.get(lineItem.itemId)?.name ?? "Unknown Item";
```

### Pattern 4: SearchableItemPicker Adaptation for Quotes
**What:** Current `SearchableItemPicker` filters OUT bundles (line 59: `item.type === "bundle"` returns false). For quotes, bundles must be included.
**How to adapt:** Add an `includeBundles?: boolean` prop (default false for backward compatibility). When true, add a "Bundles" group to `GROUP_CONFIG` and remove the bundle filter.
**Example:**
```typescript
interface SearchableItemPickerProps {
  items: Item[];
  excludeItemIds?: string[];
  onSelect: (item: Item) => void;
  placeholder?: string;
  disabled?: boolean;
  includeBundles?: boolean;  // NEW -- default false
}
```

### Pattern 5: Read-Only Guard Based on Quote Status
**What:** Line items are only editable on Draft and Sent quotes. Accepted/Rejected are read-only.
**When to use:** Disabling Add Item button, hiding delete buttons, disabling inline edits.
**Example:**
```typescript
const isEditable = quote.status === "draft" || quote.status === "sent";
```

### Anti-Patterns to Avoid
- **Optimistic updates for totals:** Decisions say totals update after API response (server-side calculation). Do NOT calculate totals client-side.
- **Storing line items separately from quote:** The API returns the full quote with line items. Use RTK Query cache invalidation, not separate line item state.
- **Duplicating item name on line item type:** The API deliberately does not include item name in the line item response. Use cross-reference pattern instead.
- **Editing component line items directly:** Out of scope per REQUIREMENTS.md. Components are read-only sub-rows.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Item search/selection | Custom dropdown with search | `SearchableItemPicker` (extend with bundle support) | Already tested, consistent UX |
| Expandable table rows | Custom accordion/disclosure | Phase 12 `expandedBundles` Set pattern + Fragment rows | Proven pattern, accessible |
| Price formatting | Custom number formatting | `useBusinessCurrency().formatDecimal()` | Handles currency, locale |
| Responsive table/cards | CSS-only approach | `useMediaQuery` hook + separate Table/CardList components | Project standard DataView pattern |
| Bundle price calculation | Client-side bundle math | Server-side `QuoteTotalsCalculator` + `BundlePricingPlanner` | Server is authoritative |

## Common Pitfalls

### Pitfall 1: Line Items Missing Item Names
**What goes wrong:** Rendering line items without names because `IQuoteLineItemResponse` only has `itemId`.
**Why it happens:** API deliberately denormalizes only `customerName`/`jobTitle` on quotes, not item names on line items.
**How to avoid:** Always fetch items list via `useGetItemsQuery(businessId)` and build `itemsById` lookup map. Show "Unknown Item" as fallback.
**Warning signs:** Blank name cells, or attempting to add `name` field to line item types that doesn't exist in API.

### Pitfall 2: Double-Counting Bundle Totals
**What goes wrong:** Summing all line items including components, resulting in doubled totals.
**Why it happens:** Bundle parent has `lineTotal` representing the bundle total, AND components each have their own `lineTotal`.
**How to avoid:** Server handles this correctly in `QuoteTotalsCalculator` (skips items with `parentLineItemId`). On the UI, trust the `quote.totals` from the response. Do NOT attempt client-side total calculation.
**Warning signs:** Totals that seem twice as large as expected when bundles are present.

### Pitfall 3: SearchableItemPicker Excludes Bundles
**What goes wrong:** User can't add bundles to quotes because the picker filters them out.
**Why it happens:** Line 59 of `SearchableItemPicker.tsx`: `if (item.type === "bundle" || item.status !== "active") return false;`
**How to avoid:** Add `includeBundles` prop. When true, don't filter out bundles. Add "Bundles" group config entry.
**Warning signs:** Bundle items not appearing in the picker when adding to quotes.

### Pitfall 4: Quote Type Missing lineItems Field
**What goes wrong:** TypeScript errors when accessing `quote.lineItems` because the current `Quote` interface in `types/quote.ts` doesn't include it.
**Why it happens:** Phase 13 didn't need to display line items, so the type was kept minimal.
**How to avoid:** Add `lineItems` array to the `Quote` interface matching the API response shape. The API already returns `lineItems` in every quote response.
**Warning signs:** TypeScript compile errors on `quote.lineItems`.

### Pitfall 5: Money Values Already in Major Units
**What goes wrong:** Displaying prices in cents/minor units (e.g., showing 1500 instead of 15.00).
**Why it happens:** Forgetting that the API controller calls `toMajorUnits()` on all Money values before sending.
**How to avoid:** Use `formatDecimal()` (not `formatAmount()`) for line item prices. `formatDecimal` is for major-unit numbers from API. `formatAmount` is for minor-unit numbers from entities.
**Warning signs:** Prices displayed 100x too large.

### Pitfall 6: Inline Edit Saving Without API Endpoint
**What goes wrong:** Building inline quantity/price editing UI with no API endpoint to save changes.
**Why it happens:** The decisions mention inline editing, but REQUIREMENTS.md `QEXT-03` (update/remove line items) is explicitly deferred to future release.
**How to avoid:** Phase 14 requirements are QLIT-01 through QLIT-04 (add and view). Check REQUIREMENTS.md -- QEXT-03 is a FUTURE requirement. Build the inline display but DO NOT implement save-on-blur for quantity/price changes. The "Add Item" flow adds with quantity 1 and default price -- that's the scope.
**Warning signs:** Building edit mutation endpoints that don't exist, or spending time on update/delete API work.

**CRITICAL CLARIFICATION:** Re-reading the CONTEXT.md decisions: "Quantity is an inline number input -- click to edit, saves on blur or Enter" and "Unit price is editable per line item." These are CONTEXT decisions that conflict with REQUIREMENTS.md where QEXT-03 (update/remove) is deferred. The planner must reconcile this -- the API has no update/delete endpoints. Options: (a) build API endpoints for update/delete as part of this phase despite QEXT-03 deferral, (b) show read-only values matching Phase 14 requirements strictly, (c) build the UI but disable saves until the API is ready. Recommendation: implement the inline editing UI AND the necessary API endpoints, since the CONTEXT decisions are more recent and specific than the broad QEXT-03 deferral.

## Code Examples

### QuoteLineItem Type Definition
```typescript
// types/quote.ts -- add to existing file
export interface QuoteLineItem {
  id: string;
  itemId: string;
  quantity: number;
  unit: string;
  unitPrice: number;        // major units from API
  originalUnitPrice: number; // major units
  lineTotal: number;         // major units
  originalLineTotal: number; // major units
  discountAmount: number;    // major units
  taxRate: number;           // percentage (e.g., 20 for 20%)
  type: ItemType;
  status: string;
  components?: QuoteLineItem[];  // nested for bundles
}

// Update Quote interface to include lineItems
export interface Quote {
  // ... existing fields ...
  lineItems: QuoteLineItem[];
}
```

### Add Line Item RTK Query Mutation
```typescript
// quoteApi.ts -- add to existing endpoints
addLineItem: builder.mutation<
  Quote,
  { businessId: string; quoteId: string; itemId: string; quantity: number }
>({
  query: ({ businessId, quoteId, itemId, quantity }) => ({
    url: `/v1/business/${businessId}/quote/${quoteId}/line-item`,
    method: "POST",
    body: { itemId, quantity },
  }),
  transformResponse: (response: StandardResponse<Quote>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No quote data returned");
  },
  invalidatesTags: (_result, _error, { quoteId }) => [
    { type: "Quote", id: quoteId },
    { type: "Quote", id: "LIST" },
  ],
}),
```

### SearchableItemPicker Bundle Support
```typescript
// Modification to SearchableItemPicker.tsx
// Add to GROUP_CONFIG:
const GROUP_CONFIG_WITH_BUNDLES = [
  { type: "material", heading: "Materials" },
  { type: "labour", heading: "Labour" },
  { type: "fee", heading: "Fees" },
  { type: "bundle", heading: "Bundles" },
] as const;

// In filteredItems memo, conditionally include bundles:
const filteredItems = useMemo(() => {
  const excludeSet = new Set(excludeItemIds);
  return items.filter((item) => {
    if (!includeBundles && item.type === "bundle") return false;
    if (item.status !== "active") return false;
    if (excludeSet.has(item.id)) return false;
    if (search.trim()) {
      return item.name.toLowerCase().includes(search.toLowerCase());
    }
    return true;
  });
}, [items, excludeItemIds, search, includeBundles]);
```

### Expandable Bundle Row in Line Items Table
```typescript
// Pattern from ItemsTable.tsx adapted for quote line items
{lineItem.type === "bundle" && isExpanded && lineItem.components && (
  <TableRow className="bg-muted/30 hover:bg-muted/30">
    <TableCell colSpan={8} className="py-3">
      <div className="ml-6 space-y-2">
        <div className="mb-2 text-xs font-medium uppercase tracking-wide text-muted-foreground">
          Components ({lineItem.components.length})
        </div>
        {lineItem.components.map((component) => {
          const componentItem = itemsById.get(component.itemId);
          return (
            <div key={component.id} className="flex items-center justify-between text-sm">
              <div className="flex items-center gap-2">
                <span>{componentItem?.name ?? "Unknown"}</span>
                <Badge variant="secondary" className="text-xs">
                  {typeLabels[component.type]}
                </Badge>
              </div>
              <div className="flex items-center gap-4">
                <span>{component.quantity} x {component.unit}</span>
                <span>{currency.formatDecimal(component.unitPrice)}</span>
                <span>{component.taxRate}%</span>
                <span className="font-medium">{currency.formatDecimal(component.lineTotal)}</span>
              </div>
            </div>
          );
        })}
      </div>
    </TableCell>
  </TableRow>
)}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mock line items data | Real API with `addLineItem` endpoint | Phase 13 (backend) | Full CRUD pipeline ready |
| Placeholder "coming soon" | Functional line items card | Phase 14 (this phase) | Replaces lines 141-153 in QuoteDetailPage |
| Separate items picker per feature | Shared `SearchableItemPicker` | Phase 11 | Reuse with minor adaptation |

## Open Questions

1. **Delete Line Item API Endpoint**
   - What we know: No delete endpoint exists in `QuoteController`. REQUIREMENTS.md lists QEXT-03 (update/remove) as future.
   - What's unclear: The CONTEXT decisions mention a "delete action" column. Should a delete endpoint be built?
   - Recommendation: The planner should include a delete endpoint if the full line item table with delete column is built. The API pattern is straightforward (find line item, verify access, delete, recalculate totals). Without delete, users can only add -- never remove mistakes.

2. **Update Line Item API Endpoint (Quantity/Price)**
   - What we know: No update endpoint exists. CONTEXT says "saves on blur or Enter" for quantity/price.
   - What's unclear: Whether to build update endpoints as part of Phase 14 or defer.
   - Recommendation: Build update endpoint. The inline editing UX is a locked CONTEXT decision. Without it, adding with quantity 1 and no way to change is a poor UX.

3. **Optional Component Toggling**
   - What we know: `BundleComponent.isOptional` exists in the data model. CONTEXT mentions toggling optional components per-quote.
   - What's unclear: No API mechanism exists to exclude optional components when adding a bundle.
   - Recommendation: Defer optional component toggling to BEXT-01 (future requirement). For Phase 14, all components are included when a bundle is added. The UI should show the "optional" badge on components but not allow toggling yet.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vite build + ESLint + TypeScript strict |
| Config file | `trade-flow-ui/vite.config.ts`, `trade-flow-ui/tsconfig.json` |
| Quick run command | `cd trade-flow-ui && npm run typecheck` |
| Full suite command | `cd trade-flow-ui && npm run lint && npm run typecheck` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| QUOT-03 | Quote detail shows line items and totals | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A -- no unit test framework |
| QLIT-01 | Add standard items to quote | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |
| QLIT-02 | Add bundle items to quote | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |
| QLIT-03 | Expandable bundle line items | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |
| QLIT-04 | Quote totals from line items | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |

### Sampling Rate
- **Per task commit:** `cd trade-flow-ui && npm run typecheck && npm run lint`
- **Per wave merge:** `cd trade-flow-ui && npm run lint && npm run typecheck` + `cd trade-flow-api && npm run validate`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
None -- existing lint and typecheck infrastructure covers all phase requirements. No unit test framework is configured for the UI project (consistent with prior phases).

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/quote/controllers/quote.controller.ts` -- API endpoint shapes, `mapLineItems` hierarchy builder, `addLineItem` handler
- `trade-flow-api/src/quote/responses/quote.responses.ts` -- `IQuoteLineItemResponse` and `IQuoteResponse` interfaces (the contract)
- `trade-flow-api/src/quote/services/quote-updater.service.ts` -- `addLineItem` orchestration, standard vs bundle detection
- `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` -- Bundle line item creation with pricing plan
- `trade-flow-api/src/quote/services/quote-totals-calculator.service.ts` -- Server-side totals (skips children with parentLineItemId)
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` -- Current page with placeholder at lines 141-153
- `trade-flow-ui/src/components/SearchableItemPicker.tsx` -- Existing picker, filters out bundles at line 59
- `trade-flow-ui/src/features/items/components/ItemsTable.tsx` -- Phase 12 expandable bundle row pattern
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` -- Existing RTK Query endpoints (no addLineItem mutation yet)
- `trade-flow-ui/src/types/quote.ts` -- Current Quote type (missing lineItems field)

### Secondary (MEDIUM confidence)
- `.planning/REQUIREMENTS.md` -- QEXT-03 (update/remove line items) listed as future requirement, creates tension with CONTEXT decisions

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in project, no new dependencies
- Architecture: HIGH -- follows established Phase 12 patterns (expandable rows, itemsById map, DataView responsive)
- Pitfalls: HIGH -- verified through direct codebase examination (API response shapes, type definitions, filter logic)
- API integration: HIGH -- endpoint exists and tested in Phase 13, response shape verified from controller source
- Inline editing scope: MEDIUM -- CONTEXT decisions and REQUIREMENTS.md have tension on update/delete scope

**Research date:** 2026-03-14
**Valid until:** 2026-04-14 (stable -- all code is project-internal)
