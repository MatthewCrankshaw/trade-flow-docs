---
phase: 08-job-detail-integration
plan: 01
subsystem: ui
tags: [react, date-fns, schedule, badges, job-detail]

# Dependency graph
requires:
  - phase: 07-schedule-edit-and-management-ui
    provides: Schedule CRUD UI, STATUS_CONFIG utility, schedule data model
provides:
  - deriveNextVisitHint function computing header hints from real schedule data
  - ScheduleSummaryCard with per-status color-coded badge breakdown
affects: []

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Status badge breakdown using STATUS_CONFIG record with reduce-based counting"
    - "Schedule-derived UI hints replacing hardcoded mock functions"

key-files:
  created: []
  modified:
    - trade-flow-ui/src/pages/JobDetailPage.tsx
    - trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx

key-decisions:
  - "deriveNextVisitHint handles all job statuses with schedule-aware logic for active, on-hold, and completed/closed states"
  - "Status badges use Partial<Record<ScheduleStatus, number>> reduce pattern showing only non-zero counts"

patterns-established:
  - "Schedule-derived hints: compute display strings from schedule array rather than hardcoding per job status"

requirements-completed: [INTG-01, INTG-02]

# Metrics
duration: 2min
completed: 2026-03-07
---

# Phase 8 Plan 1: Job Detail Integration Summary

**Schedule-derived next-visit header hints and per-status badge breakdown in ScheduleSummaryCard**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-07T20:59:01Z
- **Completed:** 2026-03-07T21:00:36Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- Replaced hardcoded getNextVisitHint with deriveNextVisitHint that computes header hints from real schedule data
- Enhanced ScheduleSummaryCard to show total visit count and color-coded per-status badges
- All job status scenarios covered: active (next upcoming), on-hold, completed/closed (last visit)

## Task Commits

Each task was committed atomically:

1. **Task 1: Replace getNextVisitHint with schedule-derived logic** - `86a216c` (feat)
2. **Task 2: Enhance ScheduleSummaryCard with per-status badge breakdown** - `550fc29` (feat)

## Files Created/Modified
- `trade-flow-ui/src/pages/JobDetailPage.tsx` - deriveNextVisitHint function replacing hardcoded getNextVisitHint
- `trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx` - ScheduleSummaryCard with STATUS_CONFIG badges

## Decisions Made
- deriveNextVisitHint handles all job statuses: on_hold returns fixed string, completed/closed shows last visit date, active shows next scheduled/confirmed visit
- Status badges computed via reduce into Partial<Record<ScheduleStatus, number>>, rendering only non-zero entries

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Job detail page now shows real schedule-derived data in both the sticky header and overview section
- All mock schedule data replaced; remaining MOCK_ prefixed data (customer, access notes, commercial) intentionally preserved per user decision

---
*Phase: 08-job-detail-integration*
*Completed: 2026-03-07*
