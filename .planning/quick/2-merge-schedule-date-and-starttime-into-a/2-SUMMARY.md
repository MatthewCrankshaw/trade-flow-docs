---
phase: quick-2
plan: 01
subsystem: api
tags: [luxon, datetime, iso8601, mongodb, nestjs, class-validator]

# Dependency graph
requires:
  - phase: quick-1
    provides: Luxon DateTime enforcement in schedule DTO (date and startTime as DateTime)
provides:
  - Single startDateTime field across all schedule module layers (entity, DTO, request, response, repository, mappers)
  - ISO8601 UTC string storage in MongoDB entity
  - Luxon DateTime in DTO layer
  - @IsISO8601 validation on API request
affects: [schedule-ui, schedule-list-api, schedule-update-api]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Single ISO8601 datetime field for scheduling instead of separate date+time"
    - "DateTime.fromISO with { zone: 'utc' } for parsing ISO strings into Luxon DateTime"
    - "dto.startDateTime.toUTC().toISO()! for converting Luxon DateTime back to ISO string"

key-files:
  created: []
  modified:
    - trade-flow-api/src/schedule/entities/schedule.entity.ts
    - trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts
    - trade-flow-api/src/schedule/requests/create-schedule.request.ts
    - trade-flow-api/src/schedule/responses/schedule.response.ts
    - trade-flow-api/src/schedule/repositories/schedule.repository.ts
    - trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts
    - trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts
    - trade-flow-api/openapi.yaml
    - trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts
    - trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts
    - trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts
    - trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts

key-decisions:
  - "Used @IsISO8601({ strict: true }) for request validation instead of custom regex"
  - "DateTime.fromISO with { zone: 'utc' } ensures UTC parsing regardless of input timezone offset"

patterns-established:
  - "ISO8601 datetime pattern: entity stores string, DTO uses Luxon DateTime, request/response use string"

requirements-completed: [QUICK-2]

# Metrics
duration: 5min
completed: 2026-03-01
---

# Quick Task 2: Merge date+startTime into startDateTime Summary

**Merged separate date (YYYY-MM-DD) and startTime (HH:mm) fields into single startDateTime ISO8601 field across all schedule module layers with @IsISO8601 validation**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-01T13:18:24Z
- **Completed:** 2026-03-01T13:23:49Z
- **Tasks:** 2
- **Files modified:** 12

## Accomplishments
- Replaced separate date/startTime with single startDateTime across entity, DTO, request, response, repository, and both mappers
- Updated OpenAPI spec with date-time format for both CreateScheduleRequest and ScheduleResponse schemas
- All 35 schedule module tests pass with updated field references
- Zero lingering references to separate date or startTime fields in the schedule module

## Task Commits

Each task was committed atomically:

1. **Task 1: Replace date+startTime with startDateTime in all production code and OpenAPI spec** - `7b9cf24` (refactor)
2. **Task 2: Update all tests and mock generator for startDateTime** - `0b2933f` (test)

## Files Created/Modified
- `trade-flow-api/src/schedule/entities/schedule.entity.ts` - IScheduleEntity now has startDateTime: string
- `trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts` - IScheduleDto now has startDateTime: DateTime
- `trade-flow-api/src/schedule/requests/create-schedule.request.ts` - CreateScheduleRequest with @IsISO8601 validation
- `trade-flow-api/src/schedule/responses/schedule.response.ts` - IScheduleResponse now has startDateTime: string
- `trade-flow-api/src/schedule/repositories/schedule.repository.ts` - toDto/toEntity convert between ISO string and Luxon DateTime
- `trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts` - Converts request ISO string to DateTime
- `trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts` - Converts DateTime to UTC ISO string
- `trade-flow-api/openapi.yaml` - Both schedule schemas updated with startDateTime format: date-time
- `trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts` - Mock data uses startDateTime
- `trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts` - Entity/DTO conversion tests updated
- `trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts` - Request mapping tests updated
- `trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts` - Response mapping tests updated

## Decisions Made
- Used `@IsISO8601({ strict: true })` from class-validator for startDateTime validation, replacing the separate `@Matches` (YYYY-MM-DD) and `@IsMilitaryTime` (HH:mm) validators
- Used `DateTime.fromISO(value, { zone: "utc" })` to ensure parsed DateTime is always UTC regardless of input timezone offset
- Removed `IsMilitaryTime` and `Matches` imports (no longer needed), added `IsISO8601` import

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Schedule module now uses industry-standard ISO8601 datetime throughout
- Ready for timezone-aware scheduling features
- UI integration will need to send ISO8601 strings instead of separate date/time values

## Self-Check: PASSED

All 9 files verified present. Both commit hashes (7b9cf24, 0b2933f) confirmed in git log.

---
*Quick Task: 2-merge-schedule-date-and-starttime-into-a*
*Completed: 2026-03-01*
