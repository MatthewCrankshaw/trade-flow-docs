---
phase: 23-developer-experience
plan: 01
subsystem: infra
tags: [nodemon, worker, hot-reload, npm-scripts]

# Dependency graph
requires:
  - phase: 22-worker-service-scaffold
    provides: worker entry point (src/worker.ts) and worker-cli.json build config
provides:
  - nodemon-worker.json for worker hot-reload development
  - worker:dev npm script for nodemon-based worker development
  - worker:prod npm script for compiled worker execution
  - build:all npm script for sequential API + worker builds
affects: [23-developer-experience, dockerfile-updates, deployment]

# Tech tracking
tech-stack:
  added: []
  patterns: [nodemon config per service entry point, debug port separation (API 9229, worker 9230)]

key-files:
  created: [trade-flow-api/nodemon-worker.json]
  modified: [trade-flow-api/package.json]

key-decisions:
  - "Debug port 9230 for worker to avoid collision with API on 9229"
  - "Watch full src/ directory since worker imports from shared modules (@core/*, @queue/*, @worker/*)"

patterns-established:
  - "Per-service nodemon config: nodemon.json for API, nodemon-worker.json for worker"
  - "build:all script for multi-target builds"

requirements-completed: [DEVX-01, DEVX-02]

# Metrics
duration: 1min
completed: 2026-03-22
---

# Phase 23 Plan 01: Worker Hot-Reload Config Summary

**Nodemon worker config with debug port 9230 and worker:dev/worker:prod/build:all npm scripts**

## Performance

- **Duration:** 1 min
- **Started:** 2026-03-22T16:21:56Z
- **Completed:** 2026-03-22T16:23:02Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments
- Created nodemon-worker.json mirroring API nodemon config but targeting src/worker.ts with debug port 9230
- Added worker:dev script for hot-reload development via nodemon
- Added worker:prod script for running compiled worker in production
- Added build:all convenience script for sequential API + worker builds

## Task Commits

Each task was committed atomically:

1. **Task 1: Create nodemon-worker.json and add worker npm scripts** - `eb60611` (feat) [trade-flow-api repo]

## Files Created/Modified
- `trade-flow-api/nodemon-worker.json` - Worker hot-reload config with debug port 9230 and src/worker.ts entry point
- `trade-flow-api/package.json` - Added worker:dev, worker:prod, and build:all scripts

## Decisions Made
- Debug port 9230 for worker avoids collision with API debug port 9229 -- allows simultaneous debugging
- Watch full src/ directory (not just src/worker/) since worker imports shared modules that may change

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Worker can now be started in dev mode with `npm run worker:dev` alongside the API
- Ready for plan 02 (additional developer experience improvements)

---
*Phase: 23-developer-experience*
*Completed: 2026-03-22*

## Self-Check: PASSED
