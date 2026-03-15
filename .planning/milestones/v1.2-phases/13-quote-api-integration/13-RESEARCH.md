# Phase 13: Quote API Integration - Research

**Researched:** 2026-03-14
**Domain:** Full-stack quote CRUD with status transitions (NestJS API + React/RTK Query UI)
**Confidence:** HIGH

## Summary

Phase 13 wires the existing quote UI scaffolding to the real API. The API already has a quote module with create, list, and get-by-id endpoints, plus a full line item system. However, it is missing: (1) `jobId` on the quote data model, (2) a status transition endpoint, (3) quote number auto-generation, and (4) a repository `update` method. The UI has a QuotesPage with mock data, a DataView pattern (table + cards), and empty tab content for status filtering. It needs a new `quoteApi.ts` RTK Query file, a CreateQuoteDialog, a QuoteDetailPage with action strip, and integration into JobDetailTabs.

The existing schedule feature provides a near-perfect pattern for both the transition API (TransitionScheduleRequest + ScheduleTransitionService + ALLOWED_TRANSITIONS map) and the job detail tab integration (ScheduleList inside JobDetailTabs). The CreateJobDialog provides the pattern for multi-step creation with inline entity creation via Command/Popover pickers.

**Primary recommendation:** Follow the schedule transition pattern exactly for quote status transitions, and the job API pattern for RTK Query endpoints. Add `jobId` to the quote entity/DTO/request/response as the first task since everything else depends on it.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Quotes can be created from two places: the Quotes page ("New Quote" button) and the Job detail page
- When created from a job: customer and job are pre-filled, user just confirms/edits reference and adds optional notes
- When created from Quotes page: step 1 -- select customer, step 2 -- select existing job for that customer OR create a new job inline (same pattern as existing CreateJobDialog), then quote is created for that job
- Quote reference number auto-generated as global sequential per business (e.g., Q-2026-001) -- editable by user
- Quote date is required, defaults to today's date
- Every quote MUST have a jobId -- no standalone quotes without a job
- Remove the 4 stats cards -- tabs only (All/Draft/Sent/Accepted/Rejected) filtering the list
- Each quote row shows: reference number, customer name, job name, status badge, date, total
- Clicking a quote row navigates to a full quote detail page (not dialog or inline expand)
- Quotes also appear as a tab/section on the Job detail page, filtered to that job's quotes
- Natural place to create a new quote for a job is from the job detail quotes section
- Action strip at top of quote detail page with contextual buttons based on current status
- Draft shows "Send" button; Sent shows "Mark Accepted" / "Mark Rejected" buttons
- Confirmation dialogs only for irreversible transitions (Accept, Reject) -- Send is immediate
- Transitioning to Sent records sentAt timestamp automatically -- no valid-until date yet
- No backwards transitions -- status only moves forward
- Quotes can be edited after being sent -- they stay in Sent status but show a "modified since last sent" indicator
- Re-sending is allowed -- updates sentAt and clears the modified indicator
- API needs a new status transition endpoint -- currently does not exist
- Add jobId to quote data model (API and UI) -- stored alongside customerId
- Both jobId and customerId stored on the quote for query efficiency (consistent with schedule pattern)
- A job can have multiple quotes
- Quote detail page shows job name as a clickable link to the job detail page
- Inline job creation within the quote creation dialog should use the same pattern as the existing CreateJobDialog
- Quote reference format: Q-YYYY-NNN (e.g., Q-2026-001) with year prefix and zero-padded sequential number
- The job detail page quotes section should mirror how schedules appear on job detail -- consistent pattern

### Claude's Discretion
- Exact quote number generation logic (server-side sequential counter, format details)
- How the "modified since last sent" indicator is tracked (e.g., modifiedAfterSend boolean, or comparing sentAt vs updatedAt)
- Quote creation dialog layout and step flow UX
- API endpoint design for status transitions (PATCH /quotes/:id/status vs POST /quotes/:id/send etc.)
- How tab filtering works (client-side filter on fetched data vs separate API calls per status)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| QUOT-01 | User can create a new quote linked to a job and customer | API: Add jobId to CreateQuoteRequest + entity + DTO + response. Add quoteDate field. Add quote number generation service. UI: New CreateQuoteDialog with customer/job pickers following CreateJobDialog pattern. |
| QUOT-02 | User can view a list of all quotes with real API data (replacing mock data) | UI: New quoteApi.ts RTK Query endpoints (getQuotes, getQuote). Replace mock data in QuotesPage. Update Quote type interface. Wire tab filtering (client-side). Update QuotesTable/QuotesCardList to show job name. |
| QUOT-04 | User can transition quote status (Draft -> Sent -> Accepted/Rejected) | API: New quote-transitions.ts map, TransitionQuoteRequest, QuoteTransitionService, POST transition endpoint. Add repository update method. UI: QuoteDetailPage with QuoteActionStrip, transitionQuote RTK Query mutation. Confirmation dialogs for Accept/Reject. |
</phase_requirements>

## Standard Stack

### Core (Already in Project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Already used -- Controller/Service/Repository layers |
| RTK Query | (Redux Toolkit) | API state management | Already used -- all features use injectEndpoints pattern |
| React Router DOM | 7.x | Client routing | Already used -- need new /quotes/:quoteId route |
| shadcn/ui | latest | UI components | Already used -- Dialog, Tabs, Badge, Command, Popover |
| class-validator | latest | API request validation | Already used -- IsString, IsEnum decorators |
| Luxon | latest | DateTime handling | Already used -- all dates in DTOs are Luxon DateTime |
| Valibot | latest | UI form validation | Established pattern for dialog forms |
| MongoDB (native driver) | latest | Database | Already used via MongoDbFetcher/MongoDbWriter |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| lucide-react | latest | Icons | FileText, Send, Check, X icons for quote actions |
| sonner (via toast) | latest | Toast notifications | Success/error feedback on create/transition |
| React Hook Form | latest | Form state | CreateQuoteDialog form management |

No new dependencies needed. Everything required is already installed.

## Architecture Patterns

### Recommended Project Structure

**API additions:**
```
trade-flow-api/src/quote/
├── controllers/
│   └── quote.controller.ts          # Add transition endpoint
├── entities/
│   └── quote.entity.ts              # Add jobId field
├── enums/
│   ├── quote-status.enum.ts         # Already exists
│   └── quote-transitions.ts         # NEW: transition map
├── requests/
│   ├── create-quote.request.ts      # Add jobId, quoteDate fields
│   └── transition-quote.request.ts  # NEW: status transition request
├── services/
│   ├── quote-creator.service.ts     # Add job validation, number generation
│   ├── quote-transition.service.ts  # NEW: transition logic
│   └── quote-number-generator.service.ts  # NEW: sequential numbering
├── repositories/
│   └── quote.repository.ts          # Add update() method
└── responses/
    └── quote.responses.ts           # Add jobId, quoteDate fields
```

**UI additions:**
```
trade-flow-ui/src/
├── features/quotes/
│   ├── api/
│   │   └── quoteApi.ts              # NEW: RTK Query endpoints
│   ├── components/
│   │   ├── CreateQuoteDialog.tsx     # NEW: multi-step creation
│   │   ├── QuoteActionStrip.tsx      # NEW: status transition buttons
│   │   ├── QuotesDataView.tsx        # UPDATE: real data props
│   │   ├── QuotesTable.tsx           # UPDATE: add job name column
│   │   └── QuotesCardList.tsx        # UPDATE: add job name
│   └── index.ts                     # UPDATE: export new components + API
├── pages/
│   ├── QuotesPage.tsx               # UPDATE: wire to real API
│   └── QuoteDetailPage.tsx          # NEW: full detail page
└── App.tsx                           # UPDATE: add /quotes/:quoteId route
```

### Pattern 1: Status Transition (Follow Schedule Pattern Exactly)

**What:** Transition map + service + POST endpoint + request validator
**When to use:** Any entity with a state machine lifecycle

**API transition map:**
```typescript
// quote-transitions.ts (follow schedule-transitions.ts)
import { QuoteStatus } from "@quote/enums/quote-status.enum";

export const ALLOWED_TRANSITIONS: ReadonlyMap<QuoteStatus, readonly QuoteStatus[]> = new Map([
  [QuoteStatus.DRAFT, [QuoteStatus.SENT]],
  [QuoteStatus.SENT, [QuoteStatus.ACCEPTED, QuoteStatus.REJECTED, QuoteStatus.SENT]], // Re-send allowed
  [QuoteStatus.ACCEPTED, []], // Terminal
  [QuoteStatus.REJECTED, []], // Terminal
  [QuoteStatus.EXPIRED, []], // Terminal (not used yet but in enum)
]);
```

**API endpoint:**
```typescript
// POST /v1/business/:businessId/quote/:quoteId/transition
// Body: { status: "sent" | "accepted" | "rejected" }
```

**UI RTK Query mutation:**
```typescript
transitionQuote: builder.mutation<Quote, { businessId: string; quoteId: string; status: QuoteStatus }>({
  query: ({ businessId, quoteId, status }) => ({
    url: `/v1/business/${businessId}/quote/${quoteId}/transition`,
    method: "POST",
    body: { status },
  }),
  invalidatesTags: (_result, _error, { quoteId }) => [
    { type: "Quote", id: quoteId },
    { type: "Quote", id: "LIST" },
  ],
})
```

### Pattern 2: RTK Query API File (Follow jobApi.ts Pattern)

**What:** Feature-scoped API slice with injectEndpoints
**When to use:** Every feature module

```typescript
// quoteApi.ts
export const quoteApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getQuotes: builder.query<Quote[], string>({
      query: (businessId) => `/v1/business/${businessId}/quotes`,
      transformResponse: (response: StandardResponse<Quote>) => response.data,
      providesTags: (result) =>
        result
          ? [...result.map(({ id }) => ({ type: "Quote" as const, id })), { type: "Quote", id: "LIST" }]
          : [{ type: "Quote", id: "LIST" }],
    }),
    getQuote: builder.query<Quote, string>({
      query: (quoteId) => `/v1/quote/${quoteId}`,
      transformResponse: (response: StandardResponse<Quote>) => {
        if (response.data && response.data.length > 0) return response.data[0];
        throw new Error("No quote data returned");
      },
      providesTags: (_result, _error, id) => [{ type: "Quote", id }],
    }),
    createQuote: builder.mutation<Quote, { businessId: string; data: CreateQuoteRequest }>({
      query: ({ businessId, data }) => ({
        url: `/v1/business/${businessId}/quote`,
        method: "POST",
        body: data,
      }),
      invalidatesTags: [{ type: "Quote", id: "LIST" }],
    }),
    // transitionQuote shown above
  }),
});
```

### Pattern 3: CreateQuoteDialog (Follow CreateJobDialog Pattern)

**What:** Multi-step dialog with Command/Popover pickers for entity selection
**When to use:** Creation forms that reference other entities

Key design decisions:
- **From Quotes page:** Step 1 = select customer (Command picker), Step 2 = select/create job for that customer, Step 3 = confirm reference + notes
- **From Job detail:** Customer and job pre-filled, show reference + date + notes fields only
- Use same `shouldFilter={false}` + manual filtering pattern as CreateJobDialog
- Auto-generate quote number on form open, allow user to edit

### Pattern 4: Quote Detail Page (Follow JobDetailPage Pattern)

**What:** Full page with header info, action strip, and content
**When to use:** Resource detail views

```
/quotes/:quoteId -> QuoteDetailPage
  - Header: quote number, customer name (linked), job name (linked), status badge
  - QuoteActionStrip: contextual buttons based on status
  - Content area (placeholder for Phase 14 line items)
```

### Pattern 5: Job Detail Quotes Tab (Follow Schedule Tab Pattern)

**What:** Quotes section in JobDetailTabs replacing mock MOCK_QUOTES data
**When to use:** Showing related entities on a parent detail page

Currently JobDetailTabs has a hardcoded `MOCK_QUOTES` array and a `QuotesTable` sub-component. Replace with:
- Accept quotes data as prop (from RTK Query `getQuotesByJob` or filter from `getQuotes`)
- Use the same TabHeader + list/empty-state pattern as the schedule tab
- "Add" button opens CreateQuoteDialog with job/customer pre-filled

### Anti-Patterns to Avoid
- **Don't fetch line items for list views:** The list endpoint already returns quotes with totals. Don't load full line item trees for the quotes list -- that's Phase 14.
- **Don't build a separate quotes-by-job endpoint initially:** Client-side filtering of the quotes list by jobId is simpler for MVP. Add a dedicated endpoint only if performance demands it.
- **Don't put transition logic in the controller:** Follow the schedule pattern -- controller calls service, service validates transition and calls repository.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Sequential numbering | Custom counter logic | MongoDB `findOneAndUpdate` with `$inc` on a counters collection OR `countDocuments` + 1 | Race conditions with concurrent creates; need atomic increment |
| Status validation | Ad-hoc if/else chains | ReadonlyMap transition table (schedule-transitions.ts pattern) | Centralized, testable, exhaustive |
| Form state management | Manual useState per field | React Hook Form + Valibot schema | Already established pattern; handles dirty tracking, validation |
| Date formatting | Manual string formatting | Luxon `DateTime.fromISO().toLocaleString()` | Already used throughout; handles timezones |
| Currency formatting | Manual number formatting | `useBusinessCurrency().formatAmount()` | Already available via `useCurrency` hook |

**Key insight:** The quote number generation is the only genuinely new problem. Use a dedicated MongoDB counter document (`{ _id: "quote_counter", businessId, lastNumber }`) with `findOneAndUpdate` + `$inc` for atomic sequential numbering. This avoids race conditions that `countDocuments + 1` would have.

## Common Pitfalls

### Pitfall 1: Hardcoded Quote Number "12345"
**What goes wrong:** The existing `mapToDto` in `quote.controller.ts` line 112 hardcodes `number: "12345"`. This will create all quotes with the same number.
**Why it happens:** Placeholder from initial scaffolding.
**How to avoid:** Implement QuoteNumberGenerator service that atomically generates the next number. Call it in QuoteCreator.create() before persisting.
**Warning signs:** Multiple quotes with number "12345" in the database.

### Pitfall 2: Missing Repository Update Method
**What goes wrong:** The QuoteRepository has `findByIdOrFail`, `findQuotesByBusinessId`, and `create` but NO `update` method. Status transitions will fail.
**Why it happens:** The repository was built for create-only initially.
**How to avoid:** Add an `update` method following the schedule repository pattern (findOneAndUpdate with $set + updateAuditFields).
**Warning signs:** "method not found" errors when trying to transition status.

### Pitfall 3: Quote Entity Missing jobId
**What goes wrong:** `IQuoteEntity` and `IQuoteDto` have no `jobId` field. Creating quotes linked to jobs is impossible without adding it.
**Why it happens:** Original data model didn't include job linkage.
**How to avoid:** Add `jobId: ObjectId` to entity, `jobId: string` to DTO, `jobId: string` to CreateQuoteRequest, and `jobId: string` to response. Map in repository toDto/toEntity and controller mapToDto/mapToResponse.
**Warning signs:** Compile errors when trying to pass jobId through the stack.

### Pitfall 4: Quote Type Mismatch Between UI Mock and API Response
**What goes wrong:** The current UI `Quote` type (in QuotesTable.tsx) has `{ customer: string, total: number, createdAt: string }` but the API response has `{ customerId: string, totals: { total: number }, sentAt?: Date }`. These don't match.
**Why it happens:** UI type was designed for mock data, not real API shape.
**How to avoid:** Redefine the Quote type in `@/types` to match the API response shape. The QuotesTable needs to resolve customer name and job name from their IDs (either via the API returning them denormalized, or via separate queries/lookups).
**Warning signs:** TypeScript errors when replacing mock data with API response data.

### Pitfall 5: Tab Filtering Without Loading State
**What goes wrong:** Client-side tab filtering (filtering the full quotes array by status) shows stale data or flashes empty state.
**Why it happens:** Filtering happens synchronously but the data fetch is async.
**How to avoid:** Use a single `getQuotes` query with `isLoading` state. Filter client-side with `useMemo`. Show skeleton/loading state while data is being fetched, not empty state.
**Warning signs:** Brief flash of "No quotes" before data appears.

### Pitfall 6: SentAt Timestamp Not Set on Transition
**What goes wrong:** Transitioning to "sent" status doesn't record when it was sent.
**Why it happens:** The schedule transition service just sets status. Quote transition needs additional timestamp logic.
**How to avoid:** In QuoteTransitionService, when target is SENT, also set `sentAt = DateTime.now()`. When target is ACCEPTED, set `acceptedAt`. When target is REJECTED, set `rejectedAt`.
**Warning signs:** sentAt is always null/undefined even after sending.

### Pitfall 7: Customer/Job Name Resolution in List View
**What goes wrong:** The API response only has customerId and jobId, but the list needs to show customer name and job name.
**Why it happens:** The quote entity only stores IDs for referential integrity.
**How to avoid:** Either (a) denormalize customer name and job name/title into the quote response by joining at the API layer (recommended -- add `customerName` and `jobTitle` to response), or (b) use existing RTK Query customer/job caches on the UI side. Option (a) is more reliable and avoids N+1 issues.
**Warning signs:** Quote list shows IDs instead of names, or requires multiple API calls per row.

## Code Examples

### Existing Quote Controller Create Endpoint (Reference)
```typescript
// Source: trade-flow-api/src/quote/controllers/quote.controller.ts
@Post("business/:businessId/quote")
public async create(
  @Req() request: { user: IUserDto; params: { businessId: string } },
  @Body() quote: CreateQuoteRequest,
): Promise<IResponse<IQuoteResponse>> {
  const businessId = request.params.businessId;
  const createdItem = await this.quoteCreator.create(request.user, this.mapToDto(quote, businessId));
  const response = this.mapToResponse(createdItem);
  return createResponse([response]);
}
```

### Schedule Transition Endpoint (Pattern to Follow)
```typescript
// Source: trade-flow-api/src/schedule/controllers/schedule.controller.ts
@Post("business/:businessId/job/:jobId/schedule/:scheduleId/transition")
public async transition(
  @Req() request: { user: IUserDto; params: { businessId: string; jobId: string; scheduleId: string } },
  @Body() requestBody: TransitionScheduleRequest,
): Promise<IResponse<IScheduleResponse>> {
  const existing = await this.scheduleRetriever.findByIdOrFail(request.user, request.params.scheduleId);
  const transitioned = await this.scheduleTransition.transition(request.user, existing, requestBody.status);
  const response = mapScheduleToResponse(transitioned);
  return createResponse([response]);
}
```

### RTK Query Transition Mutation (Pattern to Follow)
```typescript
// Source: trade-flow-ui/src/features/schedules/api/scheduleApi.ts
transitionSchedule: builder.mutation<Schedule, { businessId: string; jobId: string; scheduleId: string; status: ScheduleStatus }>({
  query: ({ businessId, jobId, scheduleId, status }) => ({
    url: `/v1/business/${businessId}/job/${jobId}/schedule/${scheduleId}/transition`,
    method: "POST",
    body: { status },
  }),
  invalidatesTags: (_result, _error, { jobId, scheduleId }) => [
    { type: "Schedule", id: scheduleId },
    { type: "Schedule", id: `JOB-${jobId}` },
  ],
})
```

### JobActionStrip (Pattern for QuoteActionStrip)
```typescript
// Source: trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx
// Contextual buttons based on status -- same pattern for quotes:
// Draft: "Send" button
// Sent: "Mark Accepted" / "Mark Rejected" buttons
// Accepted/Rejected: No action buttons (terminal states)
```

## State of the Art

| Old Approach (Current Code) | New Approach (Phase 13) | Impact |
|---------------------------|------------------------|--------|
| Hardcoded number "12345" | Sequential generator service | Unique, meaningful quote numbers |
| No jobId on quote | jobId stored on entity/DTO | Quotes linked to jobs as required |
| No update/transition API | POST /transition endpoint | Full status lifecycle support |
| Mock data in QuotesPage | RTK Query from real API | Real data throughout |
| No quote detail page | /quotes/:quoteId route | Full detail view with actions |
| Mock quotes in JobDetailTabs | Real quotes filtered by job | Job-scoped quote management |

## Discretion Recommendations

### Quote Number Generation (Claude's discretion)
**Recommendation:** Use a dedicated counters collection with atomic `findOneAndUpdate` + `$inc`. Store `{ businessId, year, lastNumber }` as the counter key. Generate format `Q-{YYYY}-{NNN}` where NNN is zero-padded to 3 digits. This handles concurrent creates safely.

### Modified-Since-Sent Indicator (Claude's discretion)
**Recommendation:** Compare `updatedAt` vs `sentAt` timestamps. If `updatedAt > sentAt`, show the indicator. No extra boolean field needed -- the audit fields already track `updatedAt`. This is simpler and automatically correct. The API response already has both fields available.

### Status Transition API Design (Claude's discretion)
**Recommendation:** Use `POST /v1/business/:businessId/quote/:quoteId/transition` with body `{ status }`. This matches the schedule pattern exactly. Consistent with the existing API conventions. The transition service handles timestamp setting (sentAt, acceptedAt, rejectedAt) internally.

### Tab Filtering (Claude's discretion)
**Recommendation:** Client-side filtering. Fetch all quotes once with `getQuotes(businessId)`, then filter in the component with `useMemo` based on selected tab. The expected data volume (< 100 quotes per business typically) makes this simpler and more responsive than separate API calls per status.

### Quote Creation Dialog Layout (Claude's discretion)
**Recommendation:** Single dialog, not multi-step wizard. When opened from Quotes page: show customer picker, then job picker (filtered by selected customer with inline create option), then reference/date/notes. When opened from Job detail: hide customer/job pickers (pre-filled), show only reference/date/notes. Use a `prefilledJobId` + `prefilledCustomerId` prop pattern to control which fields are visible.

## Open Questions

1. **Customer and Job Name Denormalization**
   - What we know: API response only has customerId/jobId. List view needs names.
   - What's unclear: Whether to denormalize in the API response or resolve on the UI side.
   - Recommendation: Denormalize in the API -- add `customerName` and `jobTitle` to the quote response by looking them up in the retriever service. This avoids N+1 issues on the UI and is consistent with how a list endpoint should work. Alternative: use existing RTK Query caches for customers/jobs since those are already loaded.

2. **Quote Date Field**
   - What we know: CONTEXT.md says "Quote date is required, defaults to today's date." The entity has no explicit `quoteDate` field.
   - What's unclear: Whether to use `createdAt` (audit field) or add a separate `quoteDate` field.
   - Recommendation: Add a dedicated `quoteDate: DateTime` field to the entity/DTO. The `createdAt` audit field should not be user-editable, but quote date should be (user might backdate a quote). This is a new field on the entity.

## Sources

### Primary (HIGH confidence)
- trade-flow-api/src/quote/ -- Full quote module source code (entities, DTOs, controllers, services, repositories)
- trade-flow-api/src/schedule/ -- Schedule transition pattern (transition map, service, controller, request)
- trade-flow-ui/src/features/quotes/ -- Existing quote UI components (QuotesTable, QuotesDataView, QuotesCardList)
- trade-flow-ui/src/pages/QuotesPage.tsx -- Current page with mock data
- trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx -- Multi-step creation dialog pattern
- trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx -- Action strip pattern
- trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx -- Job detail tab integration with mock quotes
- trade-flow-ui/src/features/schedules/api/scheduleApi.ts -- RTK Query transition mutation pattern
- trade-flow-ui/src/features/jobs/api/jobApi.ts -- RTK Query CRUD pattern
- trade-flow-ui/src/services/api.ts -- Base API slice with "Quote" tag already registered
- trade-flow-ui/src/App.tsx -- Router configuration

### Secondary (MEDIUM confidence)
- CONTEXT.md decisions -- User-locked implementation decisions
- REQUIREMENTS.md -- QUOT-01, QUOT-02, QUOT-04 requirement definitions

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All libraries already in project, patterns well-established
- Architecture: HIGH -- Direct patterns exist in schedule and job features to follow
- Pitfalls: HIGH -- Identified from direct code inspection of current gaps (missing jobId, hardcoded number, no update method, type mismatches)

**Research date:** 2026-03-14
**Valid until:** 2026-04-14 (stable -- all patterns from existing codebase)
