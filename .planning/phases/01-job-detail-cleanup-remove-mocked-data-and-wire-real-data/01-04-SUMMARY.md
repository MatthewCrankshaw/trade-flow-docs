---
phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
plan: 04
subsystem: frontend
tags: [job-event, rtk-query, timeline, cache-invalidation, react]
dependency_graph:
  requires: [job-event-module, get-job-event-endpoint, inline-event-writes-quote, inline-event-writes-schedule, inline-event-writes-estimate]
  provides: [job-event-timeline-ui, job-event-cache-invalidation]
  affects: [quoteApi.ts, scheduleApi.ts, estimateApi.ts, JobDetailPage.tsx]
tech_stack:
  added: []
  patterns: [rtk-query-cache-invalidation, type-based-event-rendering]
key_files:
  created:
    - trade-flow-ui/src/features/jobs/api/jobEventApi.ts
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
    - trade-flow-ui/src/services/api.ts
    - trade-flow-ui/src/features/jobs/api/index.ts
    - trade-flow-ui/src/features/jobs/components/JobTimeline.tsx
    - trade-flow-ui/src/features/jobs/components/index.ts
    - trade-flow-ui/src/pages/JobDetailPage.tsx
    - trade-flow-ui/src/features/quotes/api/quoteApi.ts
    - trade-flow-ui/src/features/schedules/api/scheduleApi.ts
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts
decisions:
  - "Used type-based lookup map for event display config (icon, color, label) instead of switch statement for cleaner extensibility"
  - "formatRelative from date-helpers for timeline timestamps since relative time is more useful than absolute dates in an activity feed"
  - "Removed obsolete TimelineEvent interface and barrel export since JobTimeline now consumes JobEvent directly from API types"
metrics:
  duration: 4m 30s
  completed: 2026-04-22T19:20:00Z
---

# Phase 01 Plan 04: Frontend Job Event Timeline Summary

RTK Query endpoint for job events, timeline UI wired to real data with type-based icons and descriptions, and cache invalidation across quote, schedule, and estimate mutations for auto-refresh.

## What Was Done

### Task 1: Create jobEventApi RTK Query slice and add JobEvent types (f02b2a2 in trade-flow-ui)

Created the frontend data layer for job events:

- **JobEventType** union type with all 12 event types matching the backend enum
- **JobEvent** interface matching the IJobEventResponse API contract
- **"JobEvent"** added to tagTypes in the central apiSlice configuration
- **jobEventApi.ts** RTK Query slice with `getJobEvents` endpoint using `providesTags` for cache management
- **Barrel exports** updated in both types/index.ts and jobs/api/index.ts

### Task 2: Wire timeline UI to real events and add cache invalidation (85b80a7 in trade-flow-ui)

Rewired the timeline component and added cache invalidation across all mutation endpoints:

- **JobTimeline** rewritten to accept `JobEvent[]` and `isLoading` props with type-based display config (icons, colors, human-readable labels for all 12 event types), loading skeleton, and empty state
- **JobDetailPage** wired `useGetJobEventsQuery` with skip logic and passes events/loading to timeline
- **quoteApi.ts** -- added `{ type: "JobEvent", id: "LIST" }` invalidation to `createQuote`, `transitionQuote`, and `sendQuote` mutations
- **scheduleApi.ts** -- added JobEvent invalidation to `createSchedule`, `updateSchedule`, and `transitionSchedule` mutations
- **estimateApi.ts** -- added JobEvent invalidation to `createEstimate`, `sendEstimate`, `convertEstimate`, and `markEstimateLost` mutations
- **Removed** obsolete `TimelineEvent` type export from components barrel

## Verification

- TypeScript compiles with zero errors (`npx tsc --noEmit` exits 0)
- Lint: 0 errors (1 pre-existing warning unrelated to this plan)
- Format: all files pass Prettier check
- Tests: 151 tests pass (19 suites); 3 pre-existing failures in CreateJobDialog.test.tsx from plan 02-02 (scrollIntoView mock issue in cmdk, not caused by this plan)

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all files are fully implemented with no placeholder data.

## Commits

| Task | Commit | Repository | Description |
|------|--------|------------|-------------|
| 1 | f02b2a2 | trade-flow-ui | jobEventApi RTK Query slice, JobEvent types, tagType registration |
| 2 | 85b80a7 | trade-flow-ui | Timeline UI wired to real events, cache invalidation on 10 mutations |

## Self-Check: PASSED
