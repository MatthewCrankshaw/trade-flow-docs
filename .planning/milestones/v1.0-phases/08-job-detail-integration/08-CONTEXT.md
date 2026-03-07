# Phase 8: Job Detail Integration - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace remaining mock/hardcoded schedule data on the job detail page with real API data. Enhance the schedule summary card to show a per-status breakdown with color-coded badges. Replace the hardcoded `getNextVisitHint()` with real schedule-derived data. Non-schedule mock data (customer, quotes, invoices, access notes, commercial) stays as-is until those features ship.

</domain>

<decisions>
## Implementation Decisions

### Schedule status summary format
- Stat grid layout with color-coded badges using existing STATUS_CONFIG colors
- Only statuses with count > 0 are shown as badges
- Total count shown as "{N} visits" above the badges
- All statuses included in the total count (scheduled, confirmed, completed, canceled, no-show)
- Next Visit section shows date/time only (no visit type) — "Thu 12 Mar, 2:00 PM"
- When no upcoming visits exist: display "No upcoming visits"

### Sticky header next visit hint
- Replace hardcoded `getNextVisitHint()` with logic that derives from real schedule data
- For active jobs: find the next upcoming schedule entry (scheduled or confirmed, future date, ascending sort) and display its date/time
- For active jobs with no upcoming visits: "Not scheduled yet"
- For completed/closed jobs with schedules: show last visit date (most recent past schedule)
- For completed/closed jobs with no schedules: "No visits recorded"
- On-hold jobs: "On hold" (job status takes precedence)

### Mock data cleanup scope
- Leave all non-schedule mock data as-is: MOCK_CUSTOMER, MOCK_JOB_TYPE, MOCK_ACCESS_NOTES, MOCK_COMMERCIAL (JobDetailPage.tsx)
- Leave MOCK_QUOTES, MOCK_INVOICES, MOCK_NOTES in JobDetailTabs.tsx
- No TODO comments needed — MOCK_ prefix is self-documenting
- Clean up when respective features (quotes, invoices, customer details) ship in future milestones

### Claude's Discretion
- Badge sizing and spacing within the summary card
- Exact date/time format for next visit hint in header (consistent with ScheduleSummaryCard)
- How to pass schedule data to the header hint (prop drilling vs computing in page)
- Whether to extract next-visit logic into a shared utility or keep inline

</decisions>

<specifics>
## Specific Ideas

- The ScheduleSummaryCard already exists with Next Visit + Completed/Total — enhance it in-place rather than rebuilding
- The badge breakdown should feel like a quick status glance, not a detailed report
- For completed/closed jobs, showing the last visit date answers "when did we last work on this?" — useful for job history context

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ScheduleSummaryCard` (JobOverviewSection.tsx:173): already computes next visit and completed count — extend with per-status badges
- `STATUS_CONFIG` (schedule-utils): color-coded badge config for all 5 statuses — reuse for summary badges
- `Badge` component (shadcn/ui): used throughout for status display
- `ScheduleEmptyState`: already handles zero-schedule case in overview section
- `useGetSchedulesByJobQuery` (RTK Query): already called in JobDetailPage, data passed down

### Established Patterns
- Status badge colors: Scheduled=blue, Confirmed=green, Completed=gray, Canceled=red, No-show=amber (Phase 6)
- Schedule data flows: JobDetailPage fetches → passes as props to JobOverviewSection and JobDetailTabs
- date-fns `format()` used in ScheduleSummaryCard for date formatting

### Integration Points
- `JobOverviewSection.tsx` → `ScheduleSummaryCard`: enhance with per-status badge breakdown
- `JobDetailPage.tsx:57-72` → `getNextVisitHint()`: replace with schedule-derived function, needs `schedules` param
- `JobDetailHeader`: receives `nextVisitHint` as string prop — no change to component interface needed

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 08-job-detail-integration*
*Context gathered: 2026-03-07*
