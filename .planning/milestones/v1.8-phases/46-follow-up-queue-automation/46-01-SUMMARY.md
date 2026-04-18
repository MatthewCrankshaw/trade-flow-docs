---
phase: 46-follow-up-queue-automation
plan: 01
subsystem: infra
tags: [bullmq, nestjs, queue, estimate-followups]

requires:
  - phase: 41-estimate-module-crud
    provides: EstimateModule with ESTIMATE_FOLLOWUP_CANCELLER DI token and NoopEstimateFollowupCanceller
provides:
  - ESTIMATE_FOLLOWUPS queue constant in QUEUE_NAMES
  - EstimateFollowupsModule shell with BullModule.registerQueue
  - IFollowupJobData and IExpiryJobData job payload interfaces
  - FOLLOWUP_DELAYS_MS constants and deterministic jobId builder functions
  - @estimate-followups/* path aliases (tsconfig + jest)
  - Wave 0 test stubs for scheduler, canceller, and both processors
affects: [46-02, 46-03, 46-04, 46-05]

tech-stack:
  added: []
  patterns: [deterministic-jobid-builder-functions, followup-delay-constants]

key-files:
  created:
    - trade-flow-api/src/estimate-followups/estimate-followups.module.ts
    - trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts
    - trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts
    - trade-flow-api/src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts
    - trade-flow-api/src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts
    - trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts
    - trade-flow-api/src/worker/test/processors/estimate-expiry.processor.spec.ts
  modified:
    - trade-flow-api/src/queue/queue.constant.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json

key-decisions:
  - "Removed ESTIMATE_FOLLOWUP_CANCELLER import and commented-out providers from module shell to comply with no-commented-code convention; Plan 02 will add these"
  - "Test stubs use it.todo() only with no imports, keeping them minimal until Plan 02 implements the actual services"

patterns-established:
  - "Deterministic jobId pattern: estimate-followup:{id}:{rev}:{step} and estimate-expiry:{id}:{rev}"
  - "Delay constants as computed expressions for readability: 72 * 60 * 60 * 1000 instead of magic numbers"

requirements-completed: [FUP-01, FUP-02, FUP-03, FUP-05, FUP-06, FUP-07]

duration: 2min
completed: 2026-04-14
---

# Phase 46 Plan 01: Queue Infrastructure Foundation Summary

**ESTIMATE_FOLLOWUPS queue constant, EstimateFollowupsModule shell with BullMQ registration, job payload DTOs, deterministic jobId builders, and 4 Wave 0 test stubs**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-14T06:19:56Z
- **Completed:** 2026-04-14T06:22:07Z
- **Tasks:** 2
- **Files modified:** 10

## Accomplishments

- Extended QUEUE_NAMES with ESTIMATE_FOLLOWUPS constant shared by producer and consumer
- Created EstimateFollowupsModule shell with BullModule.registerQueue for the estimate-followups queue
- Defined IFollowupJobData and IExpiryJobData interfaces for type-safe job payloads
- Created FOLLOWUP_DELAYS_MS constants (3/10/21/30 day delays) and deterministic followupJobId/expiryJobId builder functions
- Added @estimate-followups/* path aliases to tsconfig.json and jest moduleNameMapper
- Created 4 Wave 0 test stub files with 12 it.todo() entries documenting expected behaviors

## Task Commits

Each task was committed atomically:

1. **Task 1: Queue constant, job DTOs, delay constants, and module shell** - `da860c1` (feat)
2. **Task 2: Wave 0 test stubs for all four test files** - `5ce9cf8` (test)

## Files Created/Modified

- `trade-flow-api/src/queue/queue.constant.ts` - Added ESTIMATE_FOLLOWUPS to QUEUE_NAMES
- `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` - Module shell with BullModule.registerQueue
- `trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts` - IFollowupJobData and IExpiryJobData interfaces
- `trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts` - Delay constants and jobId builder functions
- `trade-flow-api/tsconfig.json` - Added @estimate-followups/* path aliases
- `trade-flow-api/package.json` - Added jest moduleNameMapper entries for @estimate-followups/*
- `trade-flow-api/src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts` - 4 todo tests
- `trade-flow-api/src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts` - 3 todo tests
- `trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts` - 3 todo tests
- `trade-flow-api/src/worker/test/processors/estimate-expiry.processor.spec.ts` - 2 todo tests

## Decisions Made

- Removed ESTIMATE_FOLLOWUP_CANCELLER import and commented-out provider rebinding from module shell to comply with the no-commented-code convention; Plan 02 will add these providers when the actual services are implemented
- Test stubs use it.todo() only with no imports, keeping them minimal until Plan 02 implements the actual services

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Queue constant, module shell, DTOs, and delay constants ready for Plan 02 (EstimateFollowupScheduler and BullMQEstimateFollowupCanceller services)
- Test stubs ready to be expanded with real assertions in Plan 02
- Path aliases configured for all subsequent plans in this phase

## Self-Check: PASSED

- All 8 created files verified present on disk
- Commit da860c1 (Task 1) verified in git log
- Commit 5ce9cf8 (Task 2) verified in git log

---
*Phase: 46-follow-up-queue-automation*
*Completed: 2026-04-14*
