# Phase 1: Job Detail Cleanup — Remove mocked data and wire real data - Context

**Gathered:** 2026-04-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Remove all hardcoded/mocked data from the job detail page and replace with real API-sourced data. Build a job events system for the timeline. Clean up UI sections that reference unbuilt features (invoices, notes, access notes). After this phase, the job detail page shows only real data from existing APIs and a new job events timeline.

</domain>

<decisions>
## Implementation Decisions

### Scope & Data Wiring
- **D-01:** Wire real API data for all sections that have existing backends: customer details (via `useGetCustomerQuery`), job type name (via `useGetJobTypeQuery`), quotes, schedules, and estimates.
- **D-02:** Remove all 7 mock data blocks: `MOCK_CUSTOMER`, `MOCK_JOB_TYPE`, `MOCK_ACCESS_NOTES`, `MOCK_COMMERCIAL`, `generateMockTimeline()`, `MOCK_INVOICES`, `MOCK_NOTES`.
- **D-03:** Surface `jobTypeId` and `address` in the frontend Job type — both exist in the API response but aren't reflected in UI types. Wire job type name display and job address display.

### Timeline / Activity Feed
- **D-04:** Create a new `job_events` MongoDB collection to store activity events for jobs. This is a dedicated events collection, not an embedded array on the job entity.
- **D-05:** Build a dedicated backend timeline endpoint that queries `job_events` for a given job and returns a chronological event list.
- **D-06:** Event types to support: job status changes, schedule events (created/status changes), quote events (created/sent/accepted/rejected), estimate events (created/sent/responded/converted/lost).
- **D-07:** Event writing is inline in existing services — each service (quote-creator, schedule-updater, job-updater, etc.) writes a job event as part of its operation. No event listener/observer pattern.
- **D-08:** Forward-only event recording — no backfill migration for existing jobs. New events are recorded from this point forward only.

### Empty Feature Handling
- **D-09:** Remove the Invoices tab entirely from the job detail tabs. Do not show an empty state — remove it until the invoice feature is built.
- **D-10:** Remove the Notes tab entirely from the job detail tabs. Do not show an empty state — remove it until the notes feature is built.
- **D-11:** Remove the access notes section (parking/pets/keys/gate code) entirely. Defer to a future phase when site/property info is properly designed.
- **D-12:** Remove the "Create Invoice" placeholder action button. No disabled/coming-soon state — just remove it.
- **D-13:** Keep the tab structure with Schedules and Quotes tabs. Easy to add new tabs later when invoices/notes ship.

### Commercial Summary
- **D-14:** Replace the mock commercial summary with a quotes-only display showing total quoted amount aggregated from all non-deleted quotes for this job.
- **D-15:** Remove invoiced, paid, and outstanding fields from the commercial section. These return when the invoice feature ships.

### Claude's Discretion
- Component structure and file organization for the job events module (follow existing codebase patterns)
- Exact UI layout adjustments after removing sections (spacing, responsive behavior)
- Job event entity/DTO field design (event type enum values, metadata structure)
- Whether the timeline endpoint uses pagination or returns all events for a job

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Job Detail UI
- `trade-flow-ui/src/pages/JobDetailPage.tsx` — Main page with all mock data blocks (lines 31-54)
- `trade-flow-ui/src/features/jobs/hooks/generate-mock-timeline.ts` — Mock timeline generator to be replaced
- `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` — Tab component with MOCK_INVOICES and MOCK_NOTES
- `trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx` — Mock interfaces (MockCommercialData, MockAccessNotes)

### Job API
- `trade-flow-api/src/job/controllers/job.controller.ts` — Job HTTP endpoints
- `trade-flow-api/src/job/entities/job.entity.ts` — Job entity with address field
- `trade-flow-api/src/job/responses/job.response.ts` — Job API response shape
- `trade-flow-api/src/job/data-transfer-objects/job.dto.ts` — Job DTO

### Related APIs (for wiring)
- `trade-flow-ui/src/features/jobs/api/jobApi.ts` — RTK Query job endpoints
- `trade-flow-ui/src/features/customers/api/customerApi.ts` — Customer API (for customer detail fetch)
- `trade-flow-ui/src/features/job-types/api/jobTypeApi.ts` — Job type API (for job type name fetch)
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` — Quote API (for commercial summary)

### Codebase Patterns
- `.planning/codebase/CONVENTIONS.md` — Naming and architecture patterns
- `.planning/codebase/ARCHITECTURE.md` — Layer architecture reference

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `useGetCustomerQuery` — RTK Query hook ready to fetch customer by ID
- `useGetJobTypeQuery` — RTK Query hook ready to fetch job type by ID
- `useGetQuotesQuery` — RTK Query hook for fetching quotes (needs filtering by jobId)
- `useGetSchedulesByJobQuery` — Already used on job detail page for schedule data
- Existing NestJS module pattern (controller/service/repository/entity/DTO) for new job-event module

### Established Patterns
- RTK Query for all API data fetching with cache invalidation tags
- Feature-based module structure in both API and UI
- Inline service calls for cross-cutting concerns (e.g., quote-settings creation during business creation)
- Separation over DRY at entity boundaries — job_events is its own collection, not embedded

### Integration Points
- Job detail page (`JobDetailPage.tsx`) is the primary integration point for all data wiring
- Each existing service that creates/updates quotes, estimates, schedules, and jobs needs an inline job event write call
- Frontend Job type definition needs `jobTypeId` and `address` fields added
- New `jobEventApi.ts` RTK Query slice for timeline data

</code_context>

<specifics>
## Specific Ideas

- Timeline should show events from schedules, quotes, estimates, and job status changes — all four categories
- User wants the job detail page to be "honest" about what features exist — no placeholder/coming-soon stubs
- Total quoted amount is a single aggregated number, not a status breakdown

</specifics>

<deferred>
## Deferred Ideas

- **Access notes / site info** — Structured fields for parking, pets, keys, gate code. Needs proper design as part of a site/property feature.
- **Invoices tab** — Requires invoice module and API to be built first.
- **Notes tab** — Requires notes/comments module and API to be built first.
- **Timeline backfill migration** — Could retroactively create events from existing data timestamps for historical jobs.
- **Commercial summary: invoiced/paid/outstanding** — Returns when invoice feature ships.

</deferred>

---

*Phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data*
*Context gathered: 2026-04-21*
