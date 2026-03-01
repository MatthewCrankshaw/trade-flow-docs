# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-21)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** Phase 3: Schedule Data Model and Create API

## Current Position

Phase: 3 of 8 (Schedule Data Model and Create API)
Plan: 1 of 2 in current phase
Status: Plan 01 complete -- schedule module foundation built (data model, repository, policy, mappers, wiring)
Last activity: 2026-03-01 -- Plan 01 complete (schedule data model, repository, policy, module wiring, migration)

Progress: [########--] 80%

## Performance Metrics

**Velocity:**
- Total plans completed: 5
- Average duration: 9min
- Total execution time: 0.8 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 01-visit-type-backend | 2/2 | 20min | 10min |
| 02-visit-type-management-ui | 2/2 | 18min | 9min |
| 03-schedule-data-model-and-create-api | 1/2 | 3min | 3min |

**Recent Trend:**
- Last 5 plans: 12min, 8min, 13min, 5min, 3min
- Trend: accelerating

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Start Time + Duration model (not Start + End) -- matches tradesperson thinking
- Two scheduling modes deferred to v2 (arrival window mode)
- Conflict detection deferred to v2
- Visit types mirror existing JobType pattern
- Separate MongoDB collection for schedules (not embedded in Job)
- Used enum/ (singular) directory for visit-type to match JobType module pattern
- Repository update $set excludes name field to reinforce name immutability at persistence level
- countActiveByBusinessId uses MongoConnectionService.getDb() directly since MongoDbFetcher has no count method
- Used nullish coalescing (isDefault ?? false) in toDto for backward-compatible field addition without data migration
- Types barrel (types/index.ts) needs explicit re-export for @/types path alias to resolve correctly
- Server-side name uniqueness errors surfaced as inline form field errors for better UX
- Default icon "calendar" hardcoded for create requests (no icon selection UI per CONTEXT.md)
- Deactivate/activate available from both active and inactive filter views
- Followed visit-type module pattern exactly for schedule module structure
- All 5 ScheduleStatus values defined now as scaffolding for Phase 4 (only SCHEDULED used in Phase 3)
- DTO-to-response mapper uses relative imports matching the visit-type pattern convention

### Pending Todos

None yet.

### Blockers/Concerns

None yet.

## Session Continuity

Last session: 2026-03-01
Stopped at: Completed 03-01-PLAN.md
Resume file: .planning/phases/03-schedule-data-model-and-create-api/03-02-PLAN.md
