---
phase: 46-follow-up-queue-automation
plan: 05
subsystem: estimate-followups
tags: [wiring, integration, redis, aof, worker]
dependency_graph:
  requires: [46-02, 46-04]
  provides: [estimate-followup-scheduling-integration]
  affects: [estimate-email-sender, worker, app-module, docker-compose]
tech_stack:
  added: []
  patterns: [try-catch-isolation, aof-startup-check]
key_files:
  created: []
  modified:
    - trade-flow-api/src/estimate/services/estimate-email-sender.service.ts
    - trade-flow-api/src/estimate/test/services/estimate-email-sender.service.spec.ts
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/src/worker.ts
    - trade-flow-api/docker-compose.yaml
decisions:
  - "sendType INITIAL used as gate for step 9 scheduleFollowups — matches DRAFT status at entry for both first-send and revised-send paths"
  - "AOF check implemented as startup warning rather than processor gate — BullMQ @Processor decorator registers statically via NestJS DI and cannot be conditionally removed post-bootstrap"
  - "scheduleFollowups wrapped in isolated try/catch — scheduling failure does not block send response since email is already delivered"
metrics:
  duration: 6min
  completed: 2026-04-14
  tasks: 2
  files: 8
---

# Phase 46 Plan 05: Wire Follow-up Scheduling into Send Flow Summary

Wired EstimateFollowupScheduler into EstimateEmailSender as step 9, added Redis AOF startup verification to worker.ts, imported EstimateFollowupsModule in AppModule, and configured Docker Compose Redis with AOF persistence.

## Task 1: Amend EstimateEmailSender with step 9 scheduleFollowups and update AppModule

**Commit:** d7e3f31

- Injected `EstimateFollowupScheduler` into `EstimateEmailSender` constructor
- Added step 9 after the cancelFollowups block: calls `scheduleFollowups(estimateId, revisionNumber)` gated on `sendType === "INITIAL"` (matches DRAFT status at entry for both first-send and revised-send)
- Wrapped in isolated try/catch so scheduling failure is logged but does not propagate (email already delivered at step 3)
- Imported `EstimateFollowupsModule` in `AppModule` immediately after `EstimateModule` for DI token override (BullMQ canceller replaces noop)
- Added 4 test cases: first-send calls scheduler, revised-send calls scheduler, re-send skips scheduler, scheduling error swallowed
- All 12 tests pass

## Task 2: Redis AOF startup check in worker.ts and Docker Compose update

**Commit:** 12c8ecd

- Added `checkRedisAof()` function to `worker.ts` that creates a temporary IORedis client, runs `CONFIG GET appendonly`, and logs FATAL error if not `yes`
- The check runs at worker startup after NestJS context creation and before signal handlers
- Redis client is quit immediately after the check to avoid interfering with BullMQ connections
- Updated Docker Compose Redis service: added `--appendonly yes --appendfsync everysec` to command, added `redis-data` volume for persistence
- Full CI gate passes (`npm run ci` exits 0)
- All 41 Phase 46 tests pass together

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed pre-existing test failure in public-estimate-retriever.service.spec.ts**
- **Found during:** Task 2 (CI gate)
- **Issue:** WR-04 refactored `trackFirstView` to use `publicTransition` instead of `documentTokenRepository.updateFirstViewedAt`, but the test still asserted the old behavior
- **Fix:** Updated test to assert `publicTransition` is called for SENT status, removed unused `DocumentTokenRepository` mock and import
- **Files modified:** `src/estimate/test/services/public-estimate-retriever.service.spec.ts`
- **Commit:** 12c8ecd

**2. [Rule 3 - Blocking] Fixed pre-existing prettier formatting errors in Phase 46-04 processor files**
- **Found during:** Task 2 (CI gate)
- **Issue:** 4 prettier errors in `estimate-followup.processor.ts`, `estimate-expiry.processor.spec.ts`, `estimate-followup.processor.spec.ts`
- **Fix:** Ran `prettier --write` on affected files
- **Files modified:** 3 files in `src/worker/`
- **Commit:** 12c8ecd

**3. AOF check is a warning, not a processor gate**
- **Planned:** Conditionally refuse to register EstimateFollowupProcessor if AOF not enabled
- **Actual:** AOF check logs FATAL warning but processor still runs. BullMQ `@Processor` decorator registers statically via NestJS DI and cannot be conditionally removed post-bootstrap. The check serves as operational alerting.
- **Impact:** Minimal -- the processor has defence-in-depth status checks and will not silently drop jobs. AOF warning alerts operators to enable persistence.

## Self-Check: PASSED
