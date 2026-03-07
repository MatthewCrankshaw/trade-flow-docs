---
phase: 04-schedule-status-and-crud-api
plan: 01
subsystem: api
tags: [nestjs, mongodb, state-machine, schedule, status-transitions, crud]

# Dependency graph
requires:
  - phase: 03-schedule-data-model-and-create-api
    provides: Schedule entity, DTO, repository (create/findByIdOrFail), creator service, module scaffolding
provides:
  - ALLOWED_TRANSITIONS state machine map with isValidTransition and getValidTransitions helpers
  - ScheduleRetrieverService with authorized findByIdOrFail, findByJobId, findByBusinessId
  - ScheduleUpdaterService with status-lock guard, visitType cross-module validation, confirmed->scheduled auto-reset
  - ScheduleTransitionService with state machine validation and descriptive error messages
  - ScheduleRepository update, findByJobId, findByBusinessId methods with native driver sort and pagination
  - UpdateScheduleRequest and TransitionScheduleRequest with class-validator decorators
  - mergeExistingScheduleWithChanges utility with undefined/null/value three-way handling
  - IScheduleFilterDto for status and date range filtering
  - 3 new error codes (SCHEDULE_3, SCHEDULE_4, SCHEDULE_5)
  - Updated indexes migration (jobId+startDateTime, businessId+startDateTime)
affects: [04-schedule-status-and-crud-api, 05-calendar-ui, 06-schedule-management-ui]

# Tech tracking
tech-stack:
  added: []
  patterns: [status-state-machine-map, status-lock-guard, confirmed-auto-reset, native-driver-sorted-queries]

key-files:
  created:
    - trade-flow-api/src/schedule/enum/schedule-transitions.ts
    - trade-flow-api/src/schedule/data-transfer-objects/schedule-filter.dto.ts
    - trade-flow-api/src/schedule/requests/update-schedule.request.ts
    - trade-flow-api/src/schedule/requests/transition-schedule.request.ts
    - trade-flow-api/src/schedule/controllers/mappers/merge-existing-schedule-with-changes.utility.ts
    - trade-flow-api/src/schedule/services/schedule-retriever.service.ts
    - trade-flow-api/src/schedule/services/schedule-updater.service.ts
    - trade-flow-api/src/schedule/services/schedule-transition.service.ts
    - trade-flow-api/src/migration/migrations/20260301100000-update-schedules-indexes.migration.ts
  modified:
    - trade-flow-api/src/schedule/repositories/schedule.repository.ts
    - trade-flow-api/src/schedule/schedule.module.ts
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
    - trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts

key-decisions:
  - "Separate ScheduleTransitionService from ScheduleUpdaterService for clean separation of lifecycle changes vs field updates"
  - "Repository findByJobId/findByBusinessId use connection.getDb() directly since MongoDbFetcher lacks sort support"
  - "Extracted common findByScope private method in repository to avoid duplicating filter/pagination logic"
  - "ScheduleRetrieverService exported from ScheduleModule for controller usage in Plan 02"

patterns-established:
  - "Status state machine as ReadonlyMap with isValidTransition/getValidTransitions pure function helpers"
  - "Status-lock guard: UPDATABLE_STATUSES array checked before allowing field updates"
  - "Auto-reset pattern: confirmed status resets to scheduled when time/duration changes"
  - "Native driver sorted queries via connection.getDb() with countDocuments for accurate pagination totals"

requirements-completed: [VSTAT-01, VSTAT-02, VSTAT-03]

# Metrics
duration: 6min
completed: 2026-03-01
---

# Phase 4 Plan 1: Schedule Status and CRUD API - Service Layer Summary

**Status state machine with 5 states and 5 transitions, plus retriever/updater/transition services with repository list+update methods and 29 new unit tests**

## Performance

- **Duration:** 6 min
- **Started:** 2026-03-01T15:24:49Z
- **Completed:** 2026-03-01T15:31:13Z
- **Tasks:** 2
- **Files modified:** 17

## Accomplishments

- State machine map correctly encodes all 5 allowed transitions and 3 terminal states with pure function helpers
- Three new services (retriever, updater, transition) following established JobUpdater/JobRetriever patterns
- Repository expanded with update, findByJobId, findByBusinessId methods using native driver for sorted, filtered, paginated queries
- 29 new unit tests covering valid/invalid transitions, status-lock guard, auto-reset behavior, and authorization checks (64 total schedule tests passing)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create state machine, error codes, filter DTO, requests, merge utility, and index migration** - `784806b` (feat)
2. **Task 2 RED: Add failing tests for retriever, updater, and transition services** - `f374097` (test)
3. **Task 2 GREEN: Implement retriever, updater, transition services with repository list+update methods** - `9275510` (feat)

_Note: Task 2 was TDD with RED/GREEN commits_

## Files Created/Modified

- `src/schedule/enum/schedule-transitions.ts` - ALLOWED_TRANSITIONS map, isValidTransition, getValidTransitions
- `src/schedule/data-transfer-objects/schedule-filter.dto.ts` - IScheduleFilterDto with status, from, to fields
- `src/schedule/requests/update-schedule.request.ts` - 5 optional fields with class-validator decorators
- `src/schedule/requests/transition-schedule.request.ts` - @IsEnum status field
- `src/schedule/controllers/mappers/merge-existing-schedule-with-changes.utility.ts` - Three-way merge with undefined/null/value handling
- `src/schedule/services/schedule-retriever.service.ts` - Authorized findByIdOrFail, findByJobId, findByBusinessId
- `src/schedule/services/schedule-updater.service.ts` - Status-lock guard, visitType validation, auto-reset
- `src/schedule/services/schedule-transition.service.ts` - State machine validation with descriptive errors
- `src/schedule/repositories/schedule.repository.ts` - Added update, findByJobId, findByBusinessId methods
- `src/schedule/schedule.module.ts` - Registered 3 new services, exported ScheduleRetrieverService
- `src/core/errors/error-codes.enum.ts` - Added SCHEDULE_3, SCHEDULE_4, SCHEDULE_5
- `src/core/errors/errors-map.constant.ts` - Added 3 error code entries
- `src/migration/migrations/20260301100000-update-schedules-indexes.migration.ts` - Drops stale index, creates jobId+startDateTime and businessId+startDateTime
- `src/schedule/test/mocks/schedule-mock-generator.ts` - Added createUpdateScheduleRequest, createTransitionScheduleRequest, createScheduleFilterDto
- `src/schedule/test/services/schedule-transition.service.spec.ts` - 11 tests for transition service
- `src/schedule/test/services/schedule-updater.service.spec.ts` - 12 tests for updater service
- `src/schedule/test/services/schedule-retriever.service.spec.ts` - 6 tests for retriever service (including filter passthrough)

## Decisions Made

- Separate ScheduleTransitionService from ScheduleUpdaterService for clean separation of lifecycle changes vs field updates
- Repository findByJobId/findByBusinessId use connection.getDb() directly since MongoDbFetcher lacks sort support (same approach as VisitTypeRepository.countActiveByBusinessId)
- Extracted common findByScope private method in repository to avoid duplicating filter/pagination/sort logic between findByJobId and findByBusinessId
- ScheduleRetrieverService exported from ScheduleModule for controller usage in Plan 02 and potential cross-module use

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- All service layer code ready for Plan 02 (controller endpoints: GET list, GET by ID, PATCH update, POST transition)
- ScheduleRetrieverService already exported for controller injection
- All request DTOs, merge utility, and filter DTO ready for controller integration
- 64 schedule tests passing, zero TypeScript errors

---
*Phase: 04-schedule-status-and-crud-api*
*Completed: 2026-03-01*
