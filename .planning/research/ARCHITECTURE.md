# Architecture Research

**Domain:** Bundle completion and quote integration for trade business management
**Researched:** 2026-03-08
**Confidence:** HIGH (brownfield -- all patterns derived from existing codebase)

## System Overview

```
                         TRADE FLOW v1.2 -- NEW & MODIFIED COMPONENTS
                         =============================================

  UI (React/Vite)                                API (NestJS/MongoDB)
  ===============                                ====================

  ┌──────────────────────┐                       ┌──────────────────────────────┐
  │  QuoteDetailPage     │ ─── GET quote/:id ──> │  QuoteController             │
  │  (NEW page)          │                       │    + PATCH quote/:id     NEW │
  │  ┌────────────────┐  │                       │    + DELETE line-item    NEW │
  │  │ LineItemsList   │  │                       │    + PATCH status        NEW │
  │  │ ┌────────────┐ │  │                       ├──────────────────────────────┤
  │  │ │BundleExpand│ │  │                       │  QuoteUpdater                │
  │  │ │ (NEW comp) │ │  │                       │    + updateQuote()       NEW │
  │  │ └────────────┘ │  │                       │    + removeLineItem()    NEW │
  │  │ AddLineItem     │  │                       ├──────────────────────────────┤
  │  │ (NEW dialog)    │  │                       │  QuoteTransitionService  NEW │
  │  └────────────────┘  │                       │    (follows schedule pattern)│
  ├──────────────────────┤                       └──────────────────────────────┘
  │  BundleItemForm      │
  │  (MODIFY: enable     │                       ┌──────────────────────────────┐
  │   component editing) │ ─── PATCH item/:id ─> │  ItemController              │
  │  ┌────────────────┐  │                       │    (existing -- extend       │
  │  │ SearchableItem │  │                       │     UpdateItemRequest to     │
  │  │ Picker (NEW)   │  │                       │     accept components[])     │
  │  └────────────────┘  │                       ├──────────────────────────────┤
  ├──────────────────────┤                       │  ItemUpdaterService          │
  │  quoteApi.ts    NEW  │                       │    + validateComponents() NEW│
  │  (RTK Query)         │                       └──────────────────────────────┘
  └──────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | New vs Modified |
|-----------|----------------|-----------------|
| **QuoteDetailPage** | Full quote view with header, line items, totals, status actions | NEW page |
| **LineItemsList** | Renders line items with parent/child grouping for bundles | NEW component |
| **BundleExpandRow** | Collapsible row showing bundle component line items | NEW component |
| **AddLineItemDialog** | Item picker + quantity input to add line items to a quote | NEW component |
| **SearchableItemPicker** | Combobox/Command for filtering items by name/type | NEW shared component |
| **quoteApi.ts** | RTK Query endpoints for all quote CRUD operations | NEW API slice |
| **BundleItemForm** | Remove edit-mode read-only restriction on components | MODIFY |
| **BundleComponentsList** | Replace Select with SearchableItemPicker, enable in edit mode | MODIFY |
| **QuoteController** | Add PATCH quote, DELETE line-item, PATCH status endpoints | MODIFY |
| **QuoteUpdater** | Add updateQuote(), removeLineItem() methods | MODIFY |
| **QuoteTransitionService** | Status transition validation (mirrors ScheduleTransitionService) | NEW service |
| **UpdateItemRequest** | Accept `components[]` in bundleConfig for PATCH | MODIFY |
| **ItemUpdaterService** | Validate component references on bundle update | MODIFY |

## Architectural Patterns

### Pattern 1: Full Component Array Replacement for Bundle Editing

**What:** When editing bundle components via PATCH, the client sends the entire `bundleConfig.components[]` array. The API replaces the stored array wholesale -- no granular add/remove endpoints.

**When to use:** When the sub-resource (components) is small, tightly coupled to the parent, and always edited as a unit.

**Why this over granular endpoints:**
- Components are embedded in the Item document (not a separate collection) -- MongoDB naturally replaces embedded arrays atomically
- The existing `mergeExistingItemWithChanges` utility already handles partial PATCH merging -- extending it to merge components is consistent
- Component lists are small (typically 2-10 items) -- no performance concern with full replacement
- The UI already manages components as local form state with add/remove/update operations -- submitting the final array is the natural contract
- Granular endpoints (POST component, DELETE component, PATCH component) would triple the API surface for no user benefit, since the form always submits all components together

**Trade-offs:** If two users edit the same bundle simultaneously, last-write-wins. Acceptable for a solo-operator product.

**API contract:**

```typescript
// PATCH /v1/business/:businessId/item/:itemId
// Extends existing UpdateItemRequest
{
  "name": "Updated Bundle Name",           // optional, existing
  "bundleConfig": {                         // optional, existing object
    "priceStrategy": "component_based",     // existing
    "bundlePrice": null,                    // existing
    "components": [                         // NEW -- full replacement
      { "itemId": "abc123", "quantity": 2, "isOptional": false },
      { "itemId": "def456", "quantity": 1, "isOptional": true }
    ]
  }
}
```

**Implementation changes:**

```typescript
// update-item.request.ts -- extend UpdateBundleConfigRequest
export class UpdateBundleComponentRequest {
  @IsString() @IsNotEmpty() itemId: string;
  @IsNumber() @Min(0) quantity: number;
  @IsBoolean() isOptional: boolean;
}

export class UpdateBundleConfigRequest {
  @IsEnum(PriceStrategy) priceStrategy: PriceStrategy;
  @ValidateIf((o) => o.priceStrategy === PriceStrategy.FIXED)
  @IsNumber() @Min(0) bundlePrice: number | null;

  @IsOptional()  // NEW -- optional so priceStrategy-only updates still work
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => UpdateBundleComponentRequest)
  components?: UpdateBundleComponentRequest[];
}
```

```typescript
// merge-existing-item-with-changes.utility.ts -- extend mergeBundleConfig
const mergeBundleConfig = (existing, changes) => {
  if (changes) {
    // ... existing priceStrategy/bundlePrice merge ...
    // NEW: if components provided, replace entirely
    components: changes.components ?? existing.components,
  }
  return existing;
};
```

```typescript
// item-updater.service.ts -- add component validation
private async validateComponents(components: IBundleComponentDto[]): Promise<void> {
  // 1. All referenced itemIds exist and belong to same business
  // 2. No component is itself a bundle (no nested bundles)
  // 3. At least one component required
  // 4. No duplicate itemIds
}
```

### Pattern 2: Quote Detail Page with Expandable Bundle Lines

**What:** The quote detail page renders line items as a flat list where bundle parent lines show a chevron. Clicking expands to reveal child component lines indented beneath the parent. Components are always loaded (they come in the API response's `components[]` array on bundle line items) -- expansion is purely a UI toggle.

**When to use:** When hierarchical data is already returned by the API and the nesting is at most one level deep.

**Why this approach:**
- The API already structures the response with `components?: IQuoteLineItemResponse[]` on bundle line items (see `mapLineItems` in QuoteController) -- no additional API calls needed for expansion
- One level of nesting (parent bundle -> child components) maps cleanly to a collapsible row pattern
- Keeps the line item list scannable -- users see the rolled-up bundle price by default

**Component structure:**

```
QuoteDetailPage
├── QuoteHeader (title, number, status badge, customer, dates)
├── QuoteStatusActions (transition buttons based on current status)
├── LineItemsSection
│   ├── LineItemRow (standard item -- flat row)
│   ├── BundleLineItemRow (bundle parent -- row with expand chevron)
│   │   └── BundleComponentRows (child lines, shown when expanded)
│   └── AddLineItemButton (opens AddLineItemDialog)
├── QuoteTotals (subtotal, tax, total)
└── QuoteNotes (optional notes display)
```

**UI rendering logic:**

```typescript
// LineItemsList.tsx
function LineItemsList({ lineItems }: { lineItems: QuoteLineItem[] }) {
  return (
    <div>
      {lineItems.map((lineItem) =>
        lineItem.type === "bundle" ? (
          <BundleLineItemRow key={lineItem.id} lineItem={lineItem} />
        ) : (
          <StandardLineItemRow key={lineItem.id} lineItem={lineItem} />
        )
      )}
    </div>
  );
}

// BundleLineItemRow.tsx
function BundleLineItemRow({ lineItem }: { lineItem: QuoteLineItem }) {
  const [isExpanded, setIsExpanded] = useState(false);

  return (
    <>
      <div onClick={() => setIsExpanded(!isExpanded)} className="cursor-pointer">
        <ChevronRight className={cn("transition-transform", isExpanded && "rotate-90")} />
        {/* bundle name, qty, total */}
      </div>
      {isExpanded && lineItem.components?.map((component) => (
        <div key={component.id} className="pl-8 bg-muted/30">
          {/* component name, qty, unit price, total -- indented, muted */}
        </div>
      ))}
    </>
  );
}
```

### Pattern 3: RTK Query Cache Strategy for Quotes with Nested Line Items

**What:** Use quote-level cache tags with list invalidation. Do NOT attempt to cache individual line items separately -- treat the quote (with its embedded line items) as a single cache unit.

**Why this approach:**
- The API returns quotes with line items embedded (not as separate resources) -- the cache should mirror the API contract
- Line items only exist in the context of a quote -- there is no standalone line item endpoint
- Adding/removing a line item returns the full updated quote -- perfect for RTK Query's `transformResponse` to replace the cached quote
- Avoids the complexity of normalized line item caching which would require manual cache updates

**Implementation:**

```typescript
// quoteApi.ts
export const quoteApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getQuotes: builder.query<Quote[], string>({
      query: (businessId) => `/v1/business/${businessId}/quotes`,
      transformResponse: (response: StandardResponse<Quote>) => response.data,
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: "Quote" as const, id })),
              { type: "Quote", id: "LIST" },
            ]
          : [{ type: "Quote", id: "LIST" }],
    }),

    getQuote: builder.query<Quote, string>({
      query: (quoteId) => `/v1/quote/${quoteId}`,
      transformResponse: (response: StandardResponse<Quote>) => response.data[0],
      providesTags: (_result, _error, id) => [{ type: "Quote", id }],
    }),

    createQuote: builder.mutation<
      Quote,
      { businessId: string; data: CreateQuoteRequest }
    >({
      query: ({ businessId, data }) => ({
        url: `/v1/business/${businessId}/quote`,
        method: "POST",
        body: data,
      }),
      transformResponse: (response: StandardResponse<Quote>) => response.data[0],
      invalidatesTags: [{ type: "Quote", id: "LIST" }],
    }),

    addLineItem: builder.mutation<
      Quote,
      { businessId: string; quoteId: string; data: AddLineItemRequest }
    >({
      query: ({ businessId, quoteId, data }) => ({
        url: `/v1/business/${businessId}/quote/${quoteId}/line-item`,
        method: "POST",
        body: data,
      }),
      transformResponse: (response: StandardResponse<Quote>) => response.data[0],
      // Invalidate both the specific quote and the list (totals change)
      invalidatesTags: (_result, _error, { quoteId }) => [
        { type: "Quote", id: quoteId },
        { type: "Quote", id: "LIST" },
      ],
    }),

    removeLineItem: builder.mutation<
      Quote,
      { businessId: string; quoteId: string; lineItemId: string }
    >({
      query: ({ businessId, quoteId, lineItemId }) => ({
        url: `/v1/business/${businessId}/quote/${quoteId}/line-item/${lineItemId}`,
        method: "DELETE",
      }),
      transformResponse: (response: StandardResponse<Quote>) => response.data[0],
      invalidatesTags: (_result, _error, { quoteId }) => [
        { type: "Quote", id: quoteId },
        { type: "Quote", id: "LIST" },
      ],
    }),

    updateQuoteStatus: builder.mutation<
      Quote,
      { quoteId: string; status: QuoteStatus }
    >({
      query: ({ quoteId, status }) => ({
        url: `/v1/quote/${quoteId}/status`,
        method: "PATCH",
        body: { status },
      }),
      transformResponse: (response: StandardResponse<Quote>) => response.data[0],
      invalidatesTags: (_result, _error, { quoteId }) => [
        { type: "Quote", id: quoteId },
        { type: "Quote", id: "LIST" },
      ],
    }),
  }),
});
```

**Key cache decisions:**
- **Quote-level granularity:** Each quote (with all its line items) is one cache entry. Adding a line item invalidates that quote's tag, triggering a refetch that replaces the entire quote object including updated line items and recalculated totals.
- **List invalidation on mutations:** All mutations that change a quote also invalidate `{ type: "Quote", id: "LIST" }` because the quotes list page shows totals and status which may change.
- **No optimistic updates initially:** The API returns the full updated quote on every mutation. Let RTK Query refetch via tag invalidation. Optimistic updates can be added later if latency is a problem (it will not be at this scale).

### Pattern 4: Quote Status Transitions (Mirroring Schedule Pattern)

**What:** A dedicated `QuoteTransitionService` with a `ALLOWED_TRANSITIONS` map, following the exact pattern established by `ScheduleTransitionService` and `schedule-transitions.ts`.

**Why:** The schedule transition pattern is proven, tested, and understood. Quote status transitions have the same shape: a state machine with valid transitions and terminal states.

**Status transition map:**

```
                   ┌─────────┐
                   │  DRAFT  │
                   └────┬────┘
                        │
                   ┌────▼────┐
              ┌────│  SENT   │────┐
              │    └────┬────┘    │
              │         │         │
         ┌────▼────┐    │    ┌────▼─────┐
         │ACCEPTED │    │    │ REJECTED │
         └─────────┘    │    └──────────┘
                        │
                   ┌────▼────┐
                   │ EXPIRED │
                   └─────────┘
```

**Implementation:**

```typescript
// quote-transitions.ts (NEW file -- mirrors schedule-transitions.ts)
export const ALLOWED_TRANSITIONS: ReadonlyMap<QuoteStatus, readonly QuoteStatus[]> = new Map([
  [QuoteStatus.DRAFT, [QuoteStatus.SENT]],
  [QuoteStatus.SENT, [QuoteStatus.ACCEPTED, QuoteStatus.REJECTED, QuoteStatus.EXPIRED]],
  [QuoteStatus.ACCEPTED, []],   // terminal
  [QuoteStatus.REJECTED, []],   // terminal
  [QuoteStatus.EXPIRED, []],    // terminal
]);

export const isValidTransition = (from: QuoteStatus, to: QuoteStatus): boolean => {
  const allowed = ALLOWED_TRANSITIONS.get(from);
  return allowed !== undefined && allowed.includes(to);
};

export const getValidTransitions = (from: QuoteStatus): readonly QuoteStatus[] => {
  return ALLOWED_TRANSITIONS.get(from) ?? [];
};
```

```typescript
// quote-transition.service.ts (NEW -- mirrors schedule-transition.service.ts)
@Injectable()
export class QuoteTransitionService {
  constructor(
    private readonly quoteRepository: QuoteRepository,
    private readonly quotePolicy: QuotePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
  ) {}

  public async transition(
    authUser: IUserDto,
    existing: IQuoteDto,
    targetStatus: QuoteStatus,
  ): Promise<IQuoteDto> {
    const accessController = this.accessControllerFactory.create(this.quotePolicy);
    accessController.canUpdate(authUser, existing);

    if (!isValidTransition(existing.status, targetStatus)) {
      const validNext = getValidTransitions(existing.status);
      throw new InvalidRequestError(
        ErrorCodes.QUOTE_INVALID_TRANSITION,
        `Cannot transition from '${existing.status}' to '${targetStatus}'. Valid: ${validNext.join(", ") || "none"}`,
      );
    }

    const updated: IQuoteDto = {
      ...existing,
      status: targetStatus,
      ...this.getTimestampUpdates(targetStatus),
    };
    return this.quoteRepository.update(updated);
  }

  private getTimestampUpdates(targetStatus: QuoteStatus): Partial<IQuoteDto> {
    const now = DateTime.now();
    switch (targetStatus) {
      case QuoteStatus.SENT: return { sentAt: now };
      case QuoteStatus.ACCEPTED: return { acceptedAt: now };
      case QuoteStatus.REJECTED: return { rejectedAt: now };
      default: return {};
    }
  }
}
```

**API endpoint:**

```typescript
// QuoteController -- add status transition endpoint
@UseGuards(JwtAuthGuard)
@Patch("quote/:quoteId/status")
public async updateStatus(
  @Req() request: { user: IUserDto; params: { quoteId: string } },
  @Body() body: { status: QuoteStatus },
): Promise<IResponse<IQuoteResponse>> {
  // Fetch, transition, return
}
```

**UI integration:**

```typescript
// QuoteStatusActions.tsx -- renders available transition buttons
function QuoteStatusActions({ quote }: { quote: Quote }) {
  const [updateStatus] = useUpdateQuoteStatusMutation();

  // Derive available transitions from current status
  const transitions: Record<string, QuoteStatus[]> = {
    draft: ["sent"],
    sent: ["accepted", "rejected"],
  };

  const available = transitions[quote.status] ?? [];

  return (
    <div className="flex gap-2">
      {available.includes("sent") && (
        <Button onClick={() => updateStatus({ quoteId: quote.id, status: "sent" })}>
          Mark as Sent
        </Button>
      )}
      {available.includes("accepted") && (
        <Button onClick={() => updateStatus({ quoteId: quote.id, status: "accepted" })}>
          Accept
        </Button>
      )}
      {available.includes("rejected") && (
        <Button
          variant="destructive"
          onClick={() => updateStatus({ quoteId: quote.id, status: "rejected" })}
        >
          Reject
        </Button>
      )}
    </div>
  );
}
```

## Data Flow

### Adding a Line Item to a Quote (Existing + New UI)

```
User clicks "Add Line Item" on QuoteDetailPage
    |
AddLineItemDialog opens
    |
SearchableItemPicker shows items (from cached getItems query)
    |
User selects item + sets quantity, clicks Add
    |
addLineItem mutation fires
    POST /v1/business/:bid/quote/:qid/line-item { itemId, quantity }
    |
QuoteUpdater.addLineItem()
    |-- fetches item
    |-- if BUNDLE: QuoteBundleLineItemFactory creates parent + child line items
    |-- if STANDARD: QuoteStandardLineItemFactory creates single line item
    |-- QuoteLineItemCreator persists to MongoDB
    |-- QuoteTotalsCalculator recalculates
    |-- returns full IQuoteDto with all line items + totals
    |
Response returns full quote
    |
RTK Query invalidates { Quote, id: quoteId } + { Quote, id: "LIST" }
    |
QuoteDetailPage re-renders with new line item visible
```

### Editing Bundle Components (New Flow)

```
User opens BundleItemForm in edit mode
    |
BundleComponentsList now editable (removing isReadOnly restriction)
    |
SearchableItemPicker replaces Select for item selection
    |
User adds/removes/reorders components locally (form state)
    |
User clicks "Update"
    |
PATCH /v1/business/:bid/item/:itemId
    { bundleConfig: { priceStrategy, bundlePrice, components: [...full array] } }
    |
mergeExistingItemWithChanges() replaces components array
    |
ItemUpdaterService.update()
    |-- validateComponents(): all itemIds exist, none are bundles, no duplicates
    |-- itemRepository.update()
    |
RTK Query invalidates { Item, id: itemId } + { Item, id: "LIST" }
```

### Quote Status Transition

```
User clicks "Mark as Sent" on QuoteDetailPage
    |
updateQuoteStatus mutation fires
    PATCH /v1/quote/:quoteId/status { status: "sent" }
    |
QuoteTransitionService.transition()
    |-- validates transition (draft -> sent is valid)
    |-- sets sentAt timestamp
    |-- persists via quoteRepository.update()
    |
RTK Query invalidates Quote tags
    |
QuoteDetailPage re-renders with new status badge + different action buttons
```

## Integration Points

### New API Endpoints Required

| Endpoint | Method | Purpose | Returns |
|----------|--------|---------|---------|
| `/v1/quote/:quoteId/status` | PATCH | Status transition | Full quote |
| `/v1/business/:bid/quote/:qid/line-item/:lineItemId` | DELETE | Remove line item | Full quote |
| `/v1/business/:bid/quote/:qid` | PATCH | Update quote metadata (title, notes, validUntil) | Full quote |

### Modified API Endpoints

| Endpoint | Change | Impact |
|----------|--------|--------|
| `PATCH /v1/business/:bid/item/:itemId` | Accept `bundleConfig.components[]` in request body | Backward compatible -- components is optional |

### Cross-Module Dependencies

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Quote -> Item | QuoteUpdater fetches items via ItemRetrieverService | Existing dependency, no change |
| Quote -> TaxRate | Bundle factory resolves tax rates | Existing dependency, no change |
| UI Items -> UI Quotes | SearchableItemPicker shared by BundleItemForm and AddLineItemDialog | NEW shared component in `src/components/` or `src/features/items/components/` |
| BundleItemForm -> itemApi | Edit mode now sends PATCH with components | MODIFY -- form submit handler changes |
| QuoteDetailPage -> quoteApi | All new page, consumes new RTK Query endpoints | NEW |
| QuoteDetailPage -> itemApi | Uses `useGetItemsQuery` to populate SearchableItemPicker in AddLineItemDialog | Existing API, new consumer |

## Anti-Patterns

### Anti-Pattern 1: Separate Cache Entries for Line Items

**What people do:** Create a separate RTK Query tag type for line items (e.g., `QuoteLineItem`) and try to update individual line items in the cache.
**Why it's wrong:** Line items are not independent resources -- they exist only within a quote. The API does not expose standalone line item endpoints. Separate caching creates sync issues between the quote cache and line item cache (e.g., totals not updating when a line item changes).
**Do this instead:** Cache at the quote level. Every mutation returns the full quote. Let tag invalidation handle refetching.

### Anti-Pattern 2: Granular Component Editing Endpoints

**What people do:** Create `POST bundle/:id/component`, `DELETE bundle/:id/component/:componentId`, `PATCH bundle/:id/component/:componentId` endpoints.
**Why it's wrong:** Components are embedded in the Item document. Granular endpoints suggest they are independent resources, create unnecessary API surface, and make the UI responsible for orchestrating multiple calls.
**Do this instead:** Full array replacement via the existing PATCH item endpoint. The UI manages component state locally and submits the final result.

### Anti-Pattern 3: Client-Side Transition Validation Only

**What people do:** Only check valid transitions in the UI (disabling buttons) without server validation.
**Why it's wrong:** Any HTTP client can call the API directly. Status transitions are a business rule that must be enforced server-side.
**Do this instead:** Validate transitions on both sides. The UI uses the transition map to show/hide action buttons (good UX). The API uses the same map to enforce the rules (security).

### Anti-Pattern 4: Optimistic Updates for Quote Mutations

**What people do:** Use RTK Query's `onQueryStarted` with `updateQueryData` to optimistically update the quote cache before the server responds.
**Why it's wrong for this product:** Quote mutations involve server-side calculations (totals, tax, pricing). Optimistic updates would show stale/incorrect totals. The solo-operator scale means latency is negligible.
**Do this instead:** Let mutations invalidate tags and trigger refetches. The response is fast enough that the UX difference is imperceptible.

## Suggested Build Order

Build order is driven by dependencies -- each step produces something the next step needs.

| Order | What | Depends On | Rationale |
|-------|------|------------|-----------|
| 1 | Fix bundle creation bug (unit field) | Nothing | Unblocks everything else, standalone fix |
| 2 | Extend UpdateItemRequest with components[] + validation | Step 1 | API contract needed before UI can send components |
| 3 | SearchableItemPicker component | Nothing | Shared component needed by steps 4 and 8 |
| 4 | BundleComponentsList edit mode + BundleItemForm changes | Steps 2, 3 | UI can now edit bundle components |
| 5 | Improved bundle display (components section with qty x unit) | Step 1 | Standalone UI improvement, no API change |
| 6 | quoteApi.ts RTK Query endpoints | Nothing (API already exists for create/list/get/add-line-item) | Wire UI to existing API |
| 7 | QuoteDetailPage (read-only: header, line items, totals, bundle expand) | Step 6 | Core quote detail experience |
| 8 | AddLineItemDialog on QuoteDetailPage | Steps 3, 6, 7 | Add items to quotes from the detail page |
| 9 | Quote status transitions (API: QuoteTransitionService + endpoint) | Nothing | Independent API work |
| 10 | QuoteStatusActions on QuoteDetailPage | Steps 7, 9 | Status buttons on detail page |
| 11 | Remove line item (API endpoint + UI button) | Steps 6, 7 | Polish -- less critical than adding |

**Parallelization opportunities:**
- Steps 1, 3, 6, 9 can all start in parallel (no dependencies on each other)
- Steps 2 and 5 can run in parallel after step 1
- Steps 7 and 4 can run in parallel after their respective dependencies

## Sources

- Existing codebase analysis (HIGH confidence -- all patterns derived from existing code):
  - `trade-flow-api/src/schedule/services/schedule-transition.service.ts` -- transition pattern
  - `trade-flow-api/src/schedule/enum/schedule-transitions.ts` -- transition map pattern
  - `trade-flow-api/src/quote/controllers/quote.controller.ts` -- existing response structure with nested line items
  - `trade-flow-api/src/quote/services/quote-updater.service.ts` -- existing addLineItem flow
  - `trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts` -- PATCH merge pattern
  - `trade-flow-api/src/item/requests/update-item.request.ts` -- existing bundle config update structure
  - `trade-flow-ui/src/features/items/api/itemApi.ts` -- RTK Query tag/cache pattern
  - `trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx` -- current read-only edit mode
  - `trade-flow-ui/src/services/api.ts` -- base API slice with "Quote" tag already registered

---
*Architecture research for: v1.2 Bundle Completion & Quote Integration*
*Researched: 2026-03-08*
