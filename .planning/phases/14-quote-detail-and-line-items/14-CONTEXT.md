# Phase 14: Quote Detail and Line Items - Context

**Gathered:** 2026-03-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can view full quote details with line items, add standard items and bundles to quotes via the SearchableItemPicker, see expandable bundle components, edit quantities and unit prices inline, and view calculated totals (subtotal, tax, total). Line items can be added/edited on Draft and Sent quotes; Accepted/Rejected quotes are read-only.

</domain>

<decisions>
## Implementation Decisions

### Add item interaction
- Reuse `SearchableItemPicker` from Phase 11 — same Popover+Command pattern with grouped search and type badges
- "Add Item" button in the Line Items card header opens the picker
- After selecting an item, it's added immediately with quantity 1 — user adjusts quantity directly in the line item row
- Same item can be added as multiple separate line items (allow duplicate lines) — common in trade quoting for same material used in different contexts
- Adding items restricted to Draft and Sent quotes only — Accepted/Rejected are fully read-only
- Editing a Sent quote triggers the existing "modified since sent" indicator from Phase 13

### Bundle line display
- Collapsible rows consistent with Phase 12's expandable bundle pattern on items list
- Collapsed by default: shows bundle name, component count, quantity, and rolled-up total
- Expanded: reveals individual component breakdown with indentation
- Component prices always shown — for component-based pricing, show actual component prices; for fixed-price bundles, show adjusted component prices that sum to the fixed total
- Fixed (non-optional) components are locked — cannot be removed or modified per-quote
- Optional components (isOptional: true in bundle definition) can be toggled off per-quote

### Line item table layout
- Full detail columns: Item name, type badge, qty, unit, unit price, tax rate, line total, delete action
- Bundle rows show expand chevron instead of individual unit price/tax (those appear on component rows)
- Quantity is an inline number input — click to edit, saves on blur or Enter (consistent with Phase 12)
- Unit price is editable per line item — API already tracks originalUnitPrice vs unitPrice with discountAmount
- Mobile: card per line item showing item name, type badge, qty x unit, and line total; bundles have expand chevron

### Empty and populated states
- Empty state: CTA-focused — "No line items yet. Add items to build your quote." with prominent Add Item button (replacing Phase 13's placeholder)
- Totals card always visible, showing $0.00 when empty — consistent layout, no shifting
- Totals update after API response (server-side calculation) — not optimistic. Brief loading indicator while API processes
- Add Item button also in card header when items exist (alongside the empty state CTA when no items)

### Claude's Discretion
- Exact SearchableItemPicker integration for quote context (may need minor adaptations from bundle context)
- How optional component toggling works in the UI (checkbox, toggle, or remove button)
- Loading/saving indicator design for inline edits
- How the API handles optional component exclusion when adding a bundle to a quote
- Mobile card expand/collapse animation details
- Exact delete confirmation UX (or if immediate delete without confirmation given undo potential via API)

</decisions>

<specifics>
## Specific Ideas

- Bundle component pricing must respect the bundle's priceStrategy: component-based shows actual prices, fixed-price shows proportionally adjusted prices that sum to the bundle's fixed total
- The collapsible bundle row pattern should match Phase 12's implementation on the items list — consistent expand/collapse UX across the app
- Optional components use the existing `isOptional` field on `BundleComponentDto` — this is already in the data model

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SearchableItemPicker` (`src/components/SearchableItemPicker.tsx`): Popover+Command picker with grouped search, type badges, exclusion list — built in Phase 11 for bundle components, reuse for quote line items
- `QuoteDetailPage.tsx`: Existing page with header, action strip, totals card, and line items placeholder ready to be replaced
- `QuoteActionStrip.tsx`: Status-aware action buttons already working from Phase 13
- `Collapsible` component (shadcn/ui): Available for expandable bundle rows, used in Phase 12's items list
- `useBusinessCurrency` hook: `formatDecimal` for major-unit amounts from API
- `ItemsTable.tsx` / `ItemsCardList.tsx`: Phase 12 expandable bundle row pattern to follow

### Established Patterns
- RTK Query with tag invalidation — `addLineItem` endpoint already exists at `POST /business/:businessId/quote/:quoteId/line-item`
- API returns full quote with line items after addLineItem — response includes `lineItems` with nested `components` for bundles
- `mapLineItems` in controller already handles parent/child line item hierarchy
- `QuoteLineItemEntity` has `parentLineItemId` for bundle component tracking
- DataView pattern for responsive table/cards with `useMediaQuery`

### Integration Points
- API: `addLineItem` endpoint exists — accepts `itemId` and `quantity`, returns updated quote
- API: May need new endpoints for: update line item quantity/price, delete line item, handle optional component exclusion
- UI: `QuoteDetailPage.tsx` line 141-153 — replace placeholder with line items table
- UI: `quoteApi.ts` — add RTK Query mutation for addLineItem (endpoint exists but UI hook may not)
- UI: Quote type in `types/quote.ts` — may need `lineItems` type added to match API response

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 14-quote-detail-and-line-items*
*Context gathered: 2026-03-14*
