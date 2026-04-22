---
phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
plan: 02
subsystem: ui
tags: [react, rtk-query, job-detail, mock-removal, dinero]

requires:
  - phase: none
    provides: existing RTK Query hooks for customers, job-types, and quotes
provides:
  - Job detail page rendering real customer name, job type, and address from API
  - Commercial snapshot showing real aggregated quote totals
  - Clean job detail UI with no mock data or unbuilt feature stubs
affects: [01-03, 01-04]

tech-stack:
  added: []
  patterns:
    - RTK Query skip pattern for dependent queries (skip when parent data not loaded)
    - Quote aggregation via client-side filter and reduce on RTK Query cache

key-files:
  created: []
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/pages/JobDetailPage.tsx
    - trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx
    - trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx
    - trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx

key-decisions:
  - "Filter quotes client-side by jobId from the business-level RTK Query cache rather than adding a job-specific endpoint"
  - "Pass empty events array to JobTimeline as placeholder until Plan 03 wires real timeline data"

patterns-established:
  - "Dependent RTK Query calls: use skip option to prevent calls until parent data is available"
  - "Commercial aggregation: derive totals from quote data rather than storing separately"

requirements-completed: []

duration: 1min
completed: 2026-04-22
---

# Phase 01 Plan 02: Wire Real API Data and Remove Mocked Job Detail UI Summary

**Job detail page wired to real customer, job type, address, and quote data via RTK Query with all mock constants, unbuilt tabs, and stub UI removed**

## Performance

- **Duration:** 1 min (verification of pre-existing commits)
- **Started:** 2026-04-22T06:40:19Z
- **Completed:** 2026-04-22T06:41:26Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments
- Job interface updated with jobTypeId and address fields matching API response
- Real customer name, job type name, and address wired via useGetCustomerQuery and useGetJobTypeQuery with skip conditions
- Commercial snapshot shows real aggregated totalQuotedAmount from quote data
- Invoices tab, Notes tab, access notes section, and Create Invoice button completely removed
- All MOCK_ constants and generateMockTimeline file deleted
- npm run ci passes with zero errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Update Job type and remove mock data from JobDetailPage** - `225ce44` (feat) in trade-flow-ui
2. **Task 2: Remove unbuilt feature UI sections and update commercial summary** - `9ebba05` (feat) in trade-flow-ui

## Files Created/Modified
- `trade-flow-ui/src/types/api.types.ts` - Added jobTypeId and JobAddress type to Job interface
- `trade-flow-ui/src/types/index.ts` - Added JobAddress export
- `trade-flow-ui/src/pages/JobDetailPage.tsx` - Wired real API hooks, removed all mock constants, computed totalQuotedAmount
- `trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx` - Removed mock interfaces, access notes, simplified commercial to real total
- `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` - Removed Invoices and Notes tabs, mock data, inline sub-components
- `trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx` - Removed Create Invoice button and onCreateInvoice prop
- `trade-flow-ui/src/features/jobs/hooks/generate-mock-timeline.ts` - Deleted
- `trade-flow-ui/src/features/jobs/hooks/index.ts` - Removed mock timeline export

## Decisions Made
- Quotes filtered client-side from business-level RTK Query cache by jobId rather than adding a dedicated endpoint
- JobTimeline receives empty events array as placeholder until Plan 03 wires real activity data

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Job detail page now shows only real data from existing APIs
- Ready for Plan 03 (timeline/activity feed) and Plan 04 (remaining cleanup)
- Photos and Files tabs remain as empty-state placeholders (not in scope for this phase)

---
*Phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data*
*Completed: 2026-04-22*
