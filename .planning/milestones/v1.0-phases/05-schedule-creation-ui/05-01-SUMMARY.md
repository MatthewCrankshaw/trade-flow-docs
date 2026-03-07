---
phase: 05-schedule-creation-ui
plan: 01
subsystem: ui
tags: [react, rtk-query, valibot, shadcn, react-day-picker, date-fns, calendar]

# Dependency graph
requires:
  - phase: 03-schedule-data-model-and-create-api
    provides: Schedule API endpoints (POST create, GET by job)
  - phase: 02-visit-type-management-ui
    provides: VisitType API and form dialog patterns
provides:
  - Schedule frontend types (ScheduleStatus, Schedule, CreateScheduleRequest)
  - RTK Query schedule API (getSchedulesByJob, createSchedule)
  - useScheduleActions hook for create mutation
  - Valibot schedule form schema (date, startTime, durationMinutes, visitTypeId, notes)
  - Calendar UI component (shadcn/react-day-picker)
  - "Schedule" RTK Query cache tag
affects: [05-02-schedule-form-dialog, 06-schedule-list-ui]

# Tech tracking
tech-stack:
  added: [react-day-picker, date-fns]
  patterns: [schedule feature module structure, job-scoped cache tags]

key-files:
  created:
    - trade-flow-ui/src/components/ui/calendar.tsx
    - trade-flow-ui/src/features/schedules/api/scheduleApi.ts
    - trade-flow-ui/src/features/schedules/hooks/useScheduleActions.ts
    - trade-flow-ui/src/lib/forms/schemas/schedule.schema.ts
    - trade-flow-ui/src/features/schedules/index.ts
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
    - trade-flow-ui/src/services/api.ts
    - trade-flow-ui/src/lib/forms/schemas/index.ts
    - trade-flow-ui/src/components/ui/button.tsx
    - trade-flow-ui/src/components/ui/button.variants.ts

key-decisions:
  - "Job-scoped cache tags (JOB-${jobId}) for schedule list invalidation instead of global LIST tag"
  - "Restored button.tsx split-file pattern after shadcn overwrite to satisfy react-refresh lint rule"

patterns-established:
  - "Schedule feature module: api/hooks/components with barrel exports, following visit-type pattern"
  - "Job-scoped RTK Query cache tags: { type: 'Schedule', id: 'JOB-${jobId}' }"

requirements-completed: [SCHED-01, VTYPE-01]

# Metrics
duration: 10min
completed: 2026-03-07
---

# Phase 5 Plan 1: Schedule Foundation Summary

**Schedule types, RTK Query API layer, Valibot form schema, Calendar component, and useScheduleActions hook for schedule creation UI**

## Performance

- **Duration:** 10 min
- **Started:** 2026-03-07T11:02:50Z
- **Completed:** 2026-03-07T11:13:31Z
- **Tasks:** 2
- **Files modified:** 18

## Accomplishments
- Installed shadcn Calendar component (react-day-picker + date-fns) for date picker UI
- Defined Schedule, ScheduleStatus, CreateScheduleRequest frontend types with barrel exports
- Created RTK Query scheduleApi with getSchedulesByJob query and createSchedule mutation using job-scoped cache tags
- Built Valibot schedule form schema with date, startTime, durationMinutes, visitTypeId, and notes fields
- Created useScheduleActions hook wrapping createSchedule mutation with error handling

## Task Commits

Each task was committed atomically:

1. **Task 1: Install Calendar component + add Schedule types and tagType** - `4cf833d` (feat)
2. **Task 2: Create schedule form schema, RTK Query API, and actions hook** - `c74d869` (feat)

## Files Created/Modified
- `src/components/ui/calendar.tsx` - shadcn Calendar component (react-day-picker)
- `src/types/api.types.ts` - Added ScheduleStatus, Schedule, CreateScheduleRequest types
- `src/types/index.ts` - Re-exported Schedule types from barrel
- `src/services/api.ts` - Added "Schedule" to tagTypes array
- `src/features/schedules/api/scheduleApi.ts` - RTK Query endpoints for schedule CRUD
- `src/features/schedules/api/index.ts` - API barrel export
- `src/features/schedules/hooks/useScheduleActions.ts` - Create mutation wrapper hook
- `src/features/schedules/hooks/index.ts` - Hooks barrel export
- `src/features/schedules/components/index.ts` - Components barrel (placeholder)
- `src/features/schedules/index.ts` - Feature barrel export
- `src/lib/forms/schemas/schedule.schema.ts` - Valibot schedule form validation schema
- `src/lib/forms/schemas/index.ts` - Added schedule schema to barrel
- `src/components/ui/button.tsx` - Restored split-file pattern after shadcn overwrite
- `src/components/ui/button.variants.ts` - Added xs and icon-xs size variants

## Decisions Made
- Used job-scoped cache tags (`JOB-${jobId}`) for schedule list invalidation rather than a global LIST tag, since schedules are always queried per-job
- Restored the button.tsx split-file pattern (buttonVariants in separate file) after shadcn calendar installation overwrote it, to satisfy the react-refresh/only-export-components lint rule

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed button.tsx lint error from shadcn calendar overwrite**
- **Found during:** Task 2 (lint verification)
- **Issue:** `npx shadcn@latest add calendar --overwrite` replaced button.tsx with inline buttonVariants, breaking the react-refresh/only-export-components lint rule
- **Fix:** Restored button.tsx to import buttonVariants from button.variants.ts; updated calendar.tsx to import buttonVariants from button.variants; added xs and icon-xs size variants to button.variants.ts
- **Files modified:** src/components/ui/button.tsx, src/components/ui/button.variants.ts, src/components/ui/calendar.tsx
- **Verification:** `npm run typecheck && npm run lint` passes clean
- **Committed in:** c74d869 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug fix)
**Impact on plan:** Necessary fix for lint compliance. No scope creep.

## Issues Encountered
None beyond the auto-fixed shadcn overwrite issue.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All foundation pieces ready for Plan 05-02: ScheduleFormDialog component, empty state, and JobDetailPage wiring
- Types, API, schema, and hooks all compile and lint clean
- Calendar component installed and working

---
*Phase: 05-schedule-creation-ui*
*Completed: 2026-03-07*
