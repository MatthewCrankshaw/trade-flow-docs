# Phase 15: Quote Deletion - Context

**Gathered:** 2026-03-15
**Status:** Ready for planning

<domain>
## Phase Boundary

Tradesperson can remove unwanted draft quotes from their system. Soft delete via DELETED status enum, restricted to Draft status only. Covers API endpoint, quote detail page delete action, and quote list row delete action.

</domain>

<decisions>
## Implementation Decisions

### Delete trigger placement
- Delete option in the existing dropdown menu on the quote detail page
- Delete option in a row-level dropdown menu (three-dot menu) on the quote list
- Delete option only visible for Draft quotes — hidden entirely for Sent/Accepted/Rejected/Expired statuses

### Post-delete behavior
- From detail page: navigate back to the quote list after successful deletion
- From quote list: optimistic remove — row disappears immediately, API call runs in background
- Toast notification shown: "Quote deleted" (auto-dismissing)

### Confirmation dialog
- Destructive-styled alert dialog (red delete button)
- Shows quote details: "Delete Q-2026-001 for Dave Smith? This cannot be undone."
- Include quote number and customer name in the warning to prevent accidental deletion of the wrong quote

### Claude's Discretion
- Soft delete implementation details (DELETED status addition to QuoteStatus enum vs other approach)
- API endpoint design (new DELETE endpoint vs transition-based)
- Toast notification library/pattern choice
- Error handling if API delete fails after optimistic remove

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Follow existing patterns from the codebase (line item soft delete via DELETED status, existing dropdown menus on detail/list pages).

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `QuoteLineItemStatus.DELETED`: Establishes soft-delete-via-status-enum pattern already used for line items
- `QuoteStatus` enum: Needs DELETED added (currently: DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED)
- `ALLOWED_TRANSITIONS` map in `quote-transitions.ts`: Add DRAFT -> DELETED transition
- Quote detail dropdown: Already has status transition actions, delete can be added to same menu
- Line item delete pattern in `QuoteLineItemsTable.tsx` / `QuoteLineItemsCardList.tsx`: Existing delete mutation + optimistic patterns

### Established Patterns
- Soft delete via status enum (not physical deletion) — from `QuoteLineItemStatus.DELETED`
- Controller → Service → Repository layering with `@quote/` path aliases
- RTK Query mutations with cache invalidation for list updates
- `useDeleteLineItemMutation` pattern: mutation hook with optimistic UI updates

### Integration Points
- `QuoteController`: Add delete endpoint alongside existing line item delete and transition endpoints
- `quote-transitions.ts`: Add DRAFT → DELETED to `ALLOWED_TRANSITIONS` map
- Quote list page: Add row-level dropdown menu (currently no per-row actions on quote list)
- Quote detail page: Add delete option to existing dropdown menu
- `quoteApi.ts`: Add deleteQuote mutation endpoint

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 15-quote-deletion*
*Context gathered: 2026-03-15*
