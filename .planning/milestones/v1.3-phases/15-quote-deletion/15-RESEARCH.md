# Phase 15: Quote Deletion - Research

**Researched:** 2026-03-15
**Domain:** Quote soft-delete via status transition (NestJS API + React/RTK Query UI)
**Confidence:** HIGH

## Summary

Quote deletion is a straightforward feature that maps directly onto existing patterns in the codebase. The soft-delete-via-status-enum approach is already proven for line items (`QuoteLineItemStatus.DELETED`), and the transition machinery (`quote-transitions.ts`, `QuoteTransitionService`) handles status changes with validation. The UI already has AlertDialog confirmation patterns (in `QuoteActionStrip`), toast notifications (Sonner via `@/lib/toast`), and row-level dropdown menus (in `ItemsTable`).

The main work is: (1) add `DELETED` to `QuoteStatus` enum on both API and UI, (2) add `DRAFT -> DELETED` transition, (3) add a `deletedAt` timestamp field, (4) filter deleted quotes from list queries, (5) add delete action to detail page action strip, (6) add row-level dropdown with delete to quote list table/card components, and (7) add confirmation dialog with destructive styling.

**Primary recommendation:** Reuse the existing transition endpoint (`POST .../transition`) with `{ status: "deleted" }` rather than creating a new DELETE endpoint. This follows the established pattern and reuses all transition validation logic. The only API change needed beyond the enum/transition-map updates is filtering deleted quotes from the `findQuotesByBusinessId` query.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Delete option in the existing dropdown menu on the quote detail page
- Delete option in a row-level dropdown menu (three-dot menu) on the quote list
- Delete option only visible for Draft quotes -- hidden entirely for Sent/Accepted/Rejected/Expired statuses
- From detail page: navigate back to the quote list after successful deletion
- From quote list: optimistic remove -- row disappears immediately, API call runs in background
- Toast notification shown: "Quote deleted" (auto-dismissing)
- Destructive-styled alert dialog (red delete button)
- Shows quote details: "Delete Q-2026-001 for Dave Smith? This cannot be undone."
- Include quote number and customer name in the warning to prevent accidental deletion of the wrong quote

### Claude's Discretion
- Soft delete implementation details (DELETED status addition to QuoteStatus enum vs other approach)
- API endpoint design (new DELETE endpoint vs transition-based)
- Toast notification library/pattern choice
- Error handling if API delete fails after optimistic remove

### Deferred Ideas (OUT OF SCOPE)
None
</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| QMGT-01 | User can delete a quote (soft delete, only from Draft status) | Existing `QuoteStatus` enum + `ALLOWED_TRANSITIONS` map + `QuoteTransitionService` support adding DELETED status. Repository query needs status filter. UI has AlertDialog, toast, and DropdownMenu patterns ready. |
</phase_requirements>

## Standard Stack

### Core (Already in Project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Project standard |
| RTK Query | (via Redux Toolkit) | API mutations with cache invalidation | Project standard for all API calls |
| shadcn/ui AlertDialog | (Radix) | Confirmation dialog | Already used in QuoteActionStrip for transition confirmations |
| shadcn/ui DropdownMenu | (Radix) | Row-level action menus | Already used in ItemsTable, ItemsCardList for row actions |
| Sonner (via @/lib/toast) | - | Toast notifications | Project standard toast wrapper already in place |
| lucide-react | - | Icons (Trash2, MoreHorizontal) | Project standard icon library |

### No New Dependencies Required
This phase requires zero new packages. All UI components, patterns, and API infrastructure exist.

## Architecture Patterns

### API: Transition-Based Delete (Recommended)

**What:** Reuse the existing `POST /v1/business/:businessId/quote/:quoteId/transition` endpoint with `{ status: "deleted" }` body.

**Why over a new DELETE endpoint:**
- `QuoteTransitionService` already validates transitions, checks access control, and updates the repository
- `ALLOWED_TRANSITIONS` map already enforces which statuses can transition where
- Adding DRAFT -> DELETED is a one-line map entry
- No new controller method, request DTO, or service method needed
- The `transitionQuote` RTK Query mutation already exists on the frontend

**Changes needed in API:**
1. `quote-status.enum.ts`: Add `DELETED = "deleted"`
2. `quote-transitions.ts`: Add `QuoteStatus.DELETED` to DRAFT's allowed transitions array
3. `QuoteTransitionService.transition()`: Add `deletedAt = DateTime.now()` branch for DELETED target
4. `quote.dto.ts` / `quote.entity.ts`: Add optional `deletedAt` field
5. `quote.repository.ts` `findQuotesByBusinessId()`: Add filter `{ status: { $ne: "deleted" } }` to exclude deleted quotes from list
6. `quote-retriever.service.ts`: No change needed if repository handles filtering

### UI: Delete from Detail Page

**What:** Add delete option to the existing `QuoteActionStrip` component, visible only when `quote.status === "draft"`.

**Pattern (from existing QuoteActionStrip):**
```typescript
// Add to QuoteActionStrip alongside existing transition buttons
// Uses same AlertDialog pattern already in the component
const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);

// Visible only for draft status
{quote.status === "draft" && (
  <Button
    variant="destructive"
    size="sm"
    className="gap-1.5"
    onClick={() => setShowDeleteConfirm(true)}
  >
    <Trash2 className="h-4 w-4" />
    Delete
  </Button>
)}

// Confirmation dialog with destructive action button
<AlertDialog open={showDeleteConfirm} onOpenChange={setShowDeleteConfirm}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Delete {quote.number}?</AlertDialogTitle>
      <AlertDialogDescription>
        Delete {quote.number} for {quote.customerName}? This cannot be undone.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction
        className={buttonVariants({ variant: "destructive" })}
        onClick={handleDelete}
      >
        Delete
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

**Post-delete:** `navigate("/quotes")` after successful mutation + `toast.success("Quote deleted")`.

### UI: Delete from Quote List (Row-Level Dropdown)

**What:** Add a three-dot DropdownMenu to each row in `QuotesTable` and `QuotesCardList`, with a "Delete" option visible only for draft quotes.

**Pattern (from ItemsTable DropdownMenu):**
```typescript
// New column in QuotesTable
{quote.status === "draft" && (
  <TableCell>
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon" onClick={(e) => e.stopPropagation()}>
          <MoreHorizontal className="h-4 w-4" />
          <span className="sr-only">Open menu</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem
          className="text-destructive"
          onClick={(e) => {
            e.stopPropagation();
            onDeleteQuote(quote);
          }}
        >
          <Trash2 className="mr-2 h-4 w-4" />
          Delete
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  </TableCell>
)}
```

**Optimistic remove from list:** Use RTK Query's `onQueryStarted` with `updateQueryData` to remove the quote from the cached list immediately, then roll back if the API call fails.

```typescript
deleteQuote: builder.mutation<void, { businessId: string; quoteId: string }>({
  query: ({ businessId, quoteId }) => ({
    url: `/v1/business/${businessId}/quote/${quoteId}/transition`,
    method: "POST",
    body: { status: "deleted" },
  }),
  async onQueryStarted({ businessId, quoteId }, { dispatch, queryFulfilled }) {
    const patchResult = dispatch(
      quoteApi.util.updateQueryData("getQuotes", businessId, (draft) => {
        return draft.filter((q) => q.id !== quoteId);
      })
    );
    try {
      await queryFulfilled;
    } catch {
      patchResult.undo();
    }
  },
  invalidatesTags: [{ type: "Quote", id: "LIST" }],
}),
```

### Anti-Patterns to Avoid
- **Physical deletion:** Never actually remove the document from MongoDB. Use status-based soft delete matching the line item pattern.
- **Separate DELETE endpoint:** Don't create a new `DELETE /v1/business/:businessId/quote/:quoteId` endpoint when the transition endpoint already handles status changes with validation.
- **Client-side status filtering:** Don't rely on the frontend to filter out deleted quotes. The API query must exclude them so all consumers get clean data.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Confirmation dialog | Custom modal | shadcn/ui AlertDialog (already in QuoteActionStrip) | Accessible, keyboard-navigable, battle-tested |
| Toast notifications | Custom notification system | `@/lib/toast` (Sonner wrapper already in project) | Consistent with all other features |
| Optimistic UI updates | Manual state management | RTK Query `onQueryStarted` + `updateQueryData` | Handles rollback automatically on failure |
| Dropdown menus | Custom popover | shadcn/ui DropdownMenu (already in ItemsTable) | Accessible, click-outside handling, keyboard nav |
| Status transition validation | Custom validation in controller | `ALLOWED_TRANSITIONS` map + `isValidTransition()` | Central, testable, already handles all edge cases |

## Common Pitfalls

### Pitfall 1: Deleted Quotes Appearing in List
**What goes wrong:** Adding DELETED status but forgetting to filter in the repository query.
**Why it happens:** `findQuotesByBusinessId` currently has no status filter -- it returns all quotes.
**How to avoid:** Add `status: { $ne: "deleted" }` to the MongoDB filter in `quote.repository.ts`.
**Warning signs:** Deleted quotes still visible in the list after deletion.

### Pitfall 2: Row Click Propagation Through Dropdown
**What goes wrong:** Clicking the dropdown menu or delete option also triggers row navigation (`navigate(/quotes/${quote.id})`).
**Why it happens:** The `TableRow` has an `onClick` handler, and click events bubble up from children.
**How to avoid:** Call `e.stopPropagation()` on the DropdownMenuTrigger button and DropdownMenuItem clicks (pattern already used in ItemsTable).
**Warning signs:** User clicks Delete but navigates to detail page instead.

### Pitfall 3: Optimistic Remove Without Rollback
**What goes wrong:** Quote disappears from list, API call fails, but quote stays invisible.
**Why it happens:** Optimistic update applied but no error handling to undo it.
**How to avoid:** Use RTK Query's `onQueryStarted` pattern with `patchResult.undo()` in the catch block. Also show `toast.error("Failed to delete quote")` on failure.
**Warning signs:** Quotes disappearing permanently on network errors.

### Pitfall 4: QuoteStatus Type Not Updated on Frontend
**What goes wrong:** TypeScript type `QuoteStatus` in `types/quote.ts` doesn't include "deleted", causing type errors or missed handling.
**Why it happens:** Frontend type is a union string literal, separate from the API enum.
**How to avoid:** Add `| "deleted"` to the `QuoteStatus` type. Also update `statusColors` map in QuoteDetailPage, QuotesTable, and QuotesCardList (though deleted quotes should never render, defensive coding).
**Warning signs:** TypeScript compilation errors after API returns "deleted" status.

### Pitfall 5: Stale Detail Page After List Delete
**What goes wrong:** User deletes a quote from the list optimistically, then navigates to a cached detail view of that deleted quote.
**Why it happens:** RTK Query may have the individual quote cached from a previous `getQuote` query.
**How to avoid:** Include `{ type: "Quote", id: quoteId }` in `invalidatesTags` for the delete mutation (already in the pattern above).
**Warning signs:** Navigating to a deleted quote's detail page shows stale data instead of 404.

## Code Examples

### Existing Transition Pattern (QuoteActionStrip.tsx)
```typescript
// Source: trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
const [transitionQuote, { isLoading }] = useTransitionQuoteMutation();

const handleTransition = async (status: "sent" | "accepted" | "rejected") => {
  try {
    await transitionQuote({ businessId, quoteId: quote.id, status }).unwrap();
    toast.success(labels[status]);
  } catch {
    toast.error("Failed to update quote status");
  }
};
```

### Existing Row-Level DropdownMenu (ItemsTable.tsx)
```typescript
// Source: trade-flow-ui/src/features/items/components/ItemsTable.tsx
<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="ghost" size="icon" onClick={(e) => e.stopPropagation()}>
      <MoreHorizontal className="h-4 w-4" />
      <span className="sr-only">Open menu</span>
    </Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end">
    <DropdownMenuItem onClick={(e) => { e.stopPropagation(); onEditItem(item); }}>
      <Pencil className="mr-2 h-4 w-4" />
      Edit
    </DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

### Existing AlertDialog Confirmation (QuoteActionStrip.tsx)
```typescript
// Source: trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
<AlertDialog open={confirmAction !== null} onOpenChange={(open) => { if (!open) setConfirmAction(null); }}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Mark quote as accepted?</AlertDialogTitle>
      <AlertDialogDescription>Are you sure? This cannot be undone.</AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={handleConfirm}>Accept</AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

### Destructive Button Variant
```typescript
// Source: trade-flow-ui/src/components/ui/button.variants.ts
// The "destructive" variant is available: red background, white text
<AlertDialogAction className={buttonVariants({ variant: "destructive" })}>
  Delete
</AlertDialogAction>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Physical deletion | Soft delete via status enum | Established in v1.2 for line items | Consistent pattern across all quote-related entities |
| Separate DELETE endpoints | Transition-based status changes | Established in v1.2 | Single endpoint handles all status changes with validation |

## Open Questions

1. **Should the dropdown column always render (empty for non-draft) or conditionally appear?**
   - What we know: ItemsTable always shows the dropdown column for all rows. The quote list currently has no actions column.
   - What's unclear: Whether an empty actions column for non-draft quotes looks awkward.
   - Recommendation: Always render the column but only show the dropdown trigger for draft quotes. This avoids layout shifts and is consistent with ItemsTable pattern.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (API), none detected (UI) |
| Config file | `trade-flow-api/jest` config in package.json |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern=quote` |
| Full suite command | `cd trade-flow-api && npm run test` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| QMGT-01a | DRAFT -> DELETED transition allowed | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-transition` | No - Wave 0 |
| QMGT-01b | Non-DRAFT -> DELETED transition rejected | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-transition` | No - Wave 0 |
| QMGT-01c | Deleted quotes excluded from list query | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-retriever` | No - Wave 0 |
| QMGT-01d | deletedAt timestamp set on delete | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-transition` | No - Wave 0 |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npm run validate && cd ../trade-flow-ui && npm run lint && npm run typecheck`
- **Per wave merge:** Full validation on both projects
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] No existing test files for quote module (zero `.spec.ts` files found in `trade-flow-api/src/quote/`)
- [ ] Tests for transition validation logic (`isValidTransition` with DELETED status)
- [ ] Tests for repository query filtering deleted quotes

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/quote/enums/quote-status.enum.ts` - Current QuoteStatus enum (DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED)
- `trade-flow-api/src/quote/enums/quote-transitions.ts` - ALLOWED_TRANSITIONS map and validation helpers
- `trade-flow-api/src/quote/services/quote-transition.service.ts` - Transition service with access control
- `trade-flow-api/src/quote/controllers/quote.controller.ts` - All quote endpoints including transition
- `trade-flow-api/src/quote/repositories/quote.repository.ts` - findQuotesByBusinessId with no status filter
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` - RTK Query endpoints including transitionQuote
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` - Existing AlertDialog + transition pattern
- `trade-flow-ui/src/features/quotes/components/QuotesTable.tsx` - Quote list table (no row actions yet)
- `trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx` - Quote list cards (no row actions yet)
- `trade-flow-ui/src/features/items/components/ItemsTable.tsx` - Reference DropdownMenu row-action pattern
- `trade-flow-ui/src/lib/toast.tsx` - Sonner toast wrapper with success/error methods
- `trade-flow-ui/src/types/quote.ts` - Frontend QuoteStatus type definition

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All libraries already in use, zero new dependencies
- Architecture: HIGH - All patterns (transition, AlertDialog, DropdownMenu, toast, optimistic updates) are established in codebase
- Pitfalls: HIGH - Identified from direct code inspection of current query/component implementations

**Research date:** 2026-03-15
**Valid until:** 2026-04-15 (stable -- no external dependencies or fast-moving libraries)
