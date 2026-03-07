---
phase: 07-schedule-edit-and-management-ui
plan: 01
subsystem: ui
tags: [react, rtk-query, valibot, react-hook-form, schedule, edit-form]

requires:
  - phase: 06-schedule-list-and-detail-ui
    provides: ScheduleFormDialog (create mode), schedule-utils, scheduleApi, useScheduleActions
provides:
  - Edit mode in ScheduleFormDialog with pre-filled form values
  - scheduleEditSchema without notes field
  - transitionSchedule RTK Query mutation (useTransitionScheduleMutation)
  - Broadened updateSchedule mutation accepting full field set
  - VALID_TRANSITIONS, TERMINAL_STATUSES, isTerminalStatus in schedule-utils
  - updateSchedule and transitionStatus methods in useScheduleActions hook
  - Fixed ScheduleStatus type (no_show matching API enum)
affects: [07-02-schedule-management-actions]

tech-stack:
  added: []
  patterns:
    - "Union type form schema: scheduleEditSchema = v.omit(scheduleFormSchema, ['notes']) for mode-specific validation"
    - "Edit mode via optional prop: schedule?: Schedule triggers edit behavior in shared form dialog"

key-files:
  created: []
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/features/schedules/components/schedule-utils.ts
    - trade-flow-ui/src/features/schedules/api/scheduleApi.ts
    - trade-flow-ui/src/features/schedules/hooks/useScheduleActions.ts
    - trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx
    - trade-flow-ui/src/lib/forms/schemas/schedule.schema.ts

key-decisions:
  - "Conditional useController for notes field with eslint-disable to avoid hooks order issue in create vs edit modes"
  - "Key ScheduleFormContent by schedule.id to force form reset when switching between schedules"
  - "visitTypeId sent as null (not undefined) when empty in edit mode for explicit clearing"

patterns-established:
  - "Edit mode pattern: optional entity prop on form dialog (undefined=create, defined=edit)"
  - "Transition constants pattern: VALID_TRANSITIONS record + TERMINAL_STATUSES array + isTerminalStatus helper"

requirements-completed: [SCHED-03]

duration: 3min
completed: 2026-03-07
---

# Phase 7 Plan 01: Schedule Edit Form and API Layer Summary

**Edit mode for ScheduleFormDialog with pre-filled values, transition mutation endpoint, no_show type fix, and status transition constants**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-07T20:12:37Z
- **Completed:** 2026-03-07T20:16:02Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Fixed ScheduleStatus "no-show" to "no_show" across type, STATUS_CONFIG, and isMutedStatus to match API enum
- Added VALID_TRANSITIONS map, TERMINAL_STATUSES array, and isTerminalStatus helper for Plan 02 consumption
- Broadened updateSchedule mutation and added transitionSchedule mutation with full RTK Query cache invalidation
- Converted ScheduleFormDialog to support edit mode with pre-filled values, hidden notes, and confirmed status warning

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix no_show type mismatch, add transition constants, broaden API layer** - `a57b044` (feat)
2. **Task 2: Add edit mode to ScheduleFormDialog** - `7472dd2` (feat)

## Files Created/Modified
- `trade-flow-ui/src/types/api.types.ts` - Fixed ScheduleStatus no_show union member
- `trade-flow-ui/src/features/schedules/components/schedule-utils.ts` - Updated no_show references, added VALID_TRANSITIONS/TERMINAL_STATUSES/isTerminalStatus
- `trade-flow-ui/src/features/schedules/api/scheduleApi.ts` - Broadened updateSchedule data type, added transitionSchedule mutation
- `trade-flow-ui/src/features/schedules/hooks/useScheduleActions.ts` - Added updateSchedule, transitionStatus, isTransitioning
- `trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx` - Added edit mode with schedule prop, pre-filled values, hidden notes, confirmed warning
- `trade-flow-ui/src/lib/forms/schemas/schedule.schema.ts` - Added scheduleEditSchema and ScheduleEditFormValues

## Decisions Made
- Used conditional useController for notes field (eslint-disable for hooks rule) rather than always registering it, since edit schema omits notes entirely
- Keyed ScheduleFormContent by schedule.id at the component level (React key) rather than useForm key option which doesn't exist
- Send visitTypeId as null when empty string in edit mode for explicit clearing on the API side

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- `useForm` does not have a `key` option; used React component key on ScheduleFormContent instead to force remount
- Union type `ScheduleFormValues | ScheduleEditFormValues` required `as unknown as ScheduleFormValues` cast for notes access in create path

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- ScheduleFormDialog edit mode ready for Plan 02 to wire up from ScheduleDetailDialog
- transitionSchedule mutation ready for status dropdown consumption
- VALID_TRANSITIONS and isTerminalStatus ready for DropdownMenu action filtering

---
*Phase: 07-schedule-edit-and-management-ui*
*Completed: 2026-03-07*
