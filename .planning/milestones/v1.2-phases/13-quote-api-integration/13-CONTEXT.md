# Phase 13: Quote API Integration - Context

**Gathered:** 2026-03-14
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire the quote UI to the real API — create quotes linked to jobs and customers, list quotes from real data (replacing mock data), navigate to quote detail, and manage quote status transitions (Draft → Sent → Accepted/Rejected). Every quote must be linked to a job.

</domain>

<decisions>
## Implementation Decisions

### Quote creation flow
- Quotes can be created from two places: the Quotes page ("New Quote" button) and the Job detail page
- When created from a job: customer and job are pre-filled, user just confirms/edits reference and adds optional notes
- When created from Quotes page: step 1 — select customer, step 2 — select existing job for that customer OR create a new job inline (same creation flow as existing CreateJobDialog), then quote is created for that job
- Quote reference number auto-generated as global sequential per business (e.g., Q-2026-001) — editable by user
- Quote date is required, defaults to today's date
- Every quote MUST have a jobId — no standalone quotes without a job

### Quote list display
- Remove the 4 stats cards — tabs only (All/Draft/Sent/Accepted/Rejected) filtering the list
- Each quote row shows: reference number, customer name, job name, status badge, date, total
- Clicking a quote row navigates to a full quote detail page (not dialog or inline expand)
- Quotes also appear as a tab/section on the Job detail page, filtered to that job's quotes
- Natural place to create a new quote for a job is from the job detail quotes section

### Status transitions
- Action strip at top of quote detail page with contextual buttons based on current status
- Draft shows "Send" button; Sent shows "Mark Accepted" / "Mark Rejected" buttons
- Confirmation dialogs only for irreversible transitions (Accept, Reject) — Send is immediate
- Transitioning to Sent records sentAt timestamp automatically — no valid-until date yet
- No backwards transitions — status only moves forward
- Quotes can be edited after being sent — they stay in Sent status but show a "modified since last sent" indicator in the UI
- Re-sending is allowed — updates sentAt and clears the modified indicator
- API needs a new status transition endpoint (PATCH or POST) — currently does not exist

### Data model — job linkage
- Add jobId to quote data model (API and UI) — stored alongside customerId
- Both jobId and customerId stored on the quote for query efficiency (consistent with schedule pattern)
- A job can have multiple quotes (e.g., option A basic, option B premium)
- Quote detail page shows job name as a clickable link to the job detail page

### Claude's Discretion
- Exact quote number generation logic (server-side sequential counter, format details)
- How the "modified since last sent" indicator is tracked (e.g., modifiedAfterSend boolean, or comparing sentAt vs updatedAt)
- Quote creation dialog layout and step flow UX
- API endpoint design for status transitions (PATCH /quotes/:id/status vs POST /quotes/:id/send etc.)
- How tab filtering works (client-side filter on fetched data vs separate API calls per status)

</decisions>

<specifics>
## Specific Ideas

- Inline job creation within the quote creation dialog should use the same pattern as the existing CreateJobDialog — not a simplified version
- Quote reference format: Q-YYYY-NNN (e.g., Q-2026-001) with year prefix and zero-padded sequential number
- The job detail page quotes section should mirror how schedules appear on job detail — consistent pattern

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `QuotesPage.tsx`: Full page with mock data, tabs, stats cards, search/filter UI — needs rewiring to real data and stats cards removed
- `QuotesDataView.tsx` / `QuotesTable.tsx` / `QuotesCardList.tsx`: DataView pattern (table+cards) already built for quotes
- `QuotesEmptyState`: Empty state component exists
- `CreateJobDialog.tsx`: Existing job creation dialog — pattern to follow for inline job creation within quote dialog
- `JobActionStrip.tsx`: Action strip pattern for contextual buttons — can follow for quote status actions
- `SearchableItemPicker.tsx`: Popover+Command pattern available if needed for customer/job selection
- `PrerequisiteAlert`: Already integrated in QuotesPage for missing business/customers
- `apiSlice`: RTK Query base with "Quote" tag type already registered
- `useCurrency` hook: Available for formatting quote totals

### Established Patterns
- RTK Query with tag invalidation for cache management (all features)
- DataView pattern: desktop table + mobile cards with `useMediaQuery` breakpoint
- Feature-based module structure: `features/quotes/` already exists with components
- Dialog-based creation: shadcn Dialog + React Hook Form + Valibot validation
- Controller → Service → Repository layering in API

### Integration Points
- API: Quote controller needs jobId in CreateQuoteRequest, new status transition endpoint, quote number generation
- API: Quote DTO needs jobId field added
- UI: `QuotesPage.tsx` — replace mock data with RTK Query hooks, remove stats cards
- UI: New `quoteApi.ts` RTK Query endpoints (create, list, get, transition)
- UI: `JobDetailPage.tsx` / `JobDetailTabs.tsx` — add quotes tab/section
- UI: New quote detail page + route
- UI: New create quote dialog

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 13-quote-api-integration*
*Context gathered: 2026-03-14*
