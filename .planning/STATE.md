# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-21)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** Phase 2: Visit Type Management UI

## Current Position

Phase: 2 of 8 (Visit Type Management UI)
Plan: 1 of 2 in current phase
Status: Plan 01 complete -- backend patches and frontend infrastructure
Last activity: 2026-02-28 -- Plan 01 complete (Backend isDefault + frontend RTK Query + Scheduling tab)

Progress: [######----] 60%

## Performance Metrics

**Velocity:**
- Total plans completed: 3
- Average duration: 11min
- Total execution time: 0.6 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 01-visit-type-backend | 2/2 | 20min | 10min |
| 02-visit-type-management-ui | 1/2 | 13min | 13min |

**Recent Trend:**
- Last 5 plans: 12min, 8min, 13min
- Trend: stable

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

### Pending Todos

None yet.

### Blockers/Concerns

None yet.

## Session Continuity

Last session: 2026-02-28
Stopped at: Completed 02-01-PLAN.md -- backend patches and frontend infrastructure
Resume file: .planning/phases/02-visit-type-management-ui/02-02-PLAN.md
