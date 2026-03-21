---
phase: quick
plan: 260321-tua
subsystem: ui
tags: [react, shadcn, callback, ux]

requires:
  - phase: 260321-thu
    provides: "Job-first quote creation with CreateJobDialog integration"
provides:
  - "Auto-selection of newly created job in quote dialog via onJobCreated callback"
affects: [quotes, jobs]

tech-stack:
  added: []
  patterns: ["Optional callback prop for parent notification of child dialog results"]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx
    - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx

key-decisions:
  - "Callback passes minimal {id, title} object rather than full Job type for loose coupling"

patterns-established:
  - "Child dialog callback pattern: onEntityCreated?.(minimal-data) for parent state sync"

requirements-completed: [auto-select-new-job]

duration: 1min
completed: 2026-03-21
---

# Quick Task 260321-tua: Auto-Select Newly Created Job in Quote Dialog Summary

**Optional onJobCreated callback on CreateJobDialog wired to auto-select new job in CreateQuoteDialog**

## Performance

- **Duration:** 1 min
- **Started:** 2026-03-21T21:30:58Z
- **Completed:** 2026-03-21T21:32:20Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- Added optional `onJobCreated` callback prop to CreateJobDialog (backward compatible)
- Wired callback in CreateQuoteDialog to auto-select job and auto-resolve customer info
- Eliminates manual re-selection after creating a new job from the quote dialog

## Task Commits

Each task was committed atomically:

1. **Task 1: Add onJobCreated callback to CreateJobDialog** - `7689fdd` (feat)
2. **Task 2: Wire callback in CreateQuoteDialog to auto-select new job** - `1415055` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` - Added optional onJobCreated prop, captures createJob result, invokes callback
- `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` - Added handleJobCreated handler, passes onJobCreated to CreateJobDialog

## Decisions Made
- Callback passes minimal `{id, title}` object rather than full Job type -- keeps coupling loose and matches what the quote form needs

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

Pre-existing lint errors in `QuoteEmailSettings.tsx` (refs accessed during render) -- unrelated to this task, not addressed.

## User Setup Required

None - no external service configuration required.

---
*Quick task: 260321-tua*
*Completed: 2026-03-21*
