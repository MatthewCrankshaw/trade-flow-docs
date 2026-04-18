---
phase: 46-follow-up-queue-automation
plan: "02"
subsystem: trade-flow-api/estimate-followups
tags: [bullmq, queue, scheduler, canceller, follow-up, estimate]
dependency_graph:
  requires:
    - trade-flow-api/src/queue/queue.constant.ts (ESTIMATE_FOLLOWUPS queue name)
    - trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts (job payload types)
    - trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts (delays and jobId builders)
    - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts (IEstimateFollowupCanceller)
  provides:
    - trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts (EstimateFollowupScheduler)
    - trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts (BullMQEstimateFollowupCanceller)
    - trade-flow-api/src/estimate-followups/estimate-followups.module.ts (fully wired module with DI token rebinding)
  affects:
    - Plan 46-05 (EstimateEmailSender will call EstimateFollowupScheduler.scheduleFollowups)
    - Any consumer of ESTIMATE_FOLLOWUP_CANCELLER token (now resolves to BullMQEstimateFollowupCanceller)
tech_stack:
  added: []
  patterns:
    - Deterministic BullMQ jobIds for deduplication and targeted cancellation
    - Per-job error isolation in canceller (Promise.all with individual try/catch)
    - DI token rebinding via useExisting to swap Noop for BullMQ implementation
key_files:
  created:
    - trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts
    - trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts
  modified:
    - trade-flow-api/src/estimate-followups/estimate-followups.module.ts
    - trade-flow-api/src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts
    - trade-flow-api/src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts
decisions:
  - "Deterministic jobId pattern (estimate-followup:{id}:{rev}:{step}) enables BullMQ built-in dedup for idempotent scheduling"
  - "Per-job error isolation in canceller ensures partial Redis failures do not prevent removal of remaining jobs"
  - "DI token ESTIMATE_FOLLOWUP_CANCELLER rebound via useExisting so all existing injections resolve to BullMQ implementation"
metrics:
  duration: "5 minutes"
  completed: "2026-04-14"
  tasks_completed: 2
  files_changed: 5
---

# Phase 46 Plan 02: Scheduler & Canceller Services Summary

EstimateFollowupScheduler enqueues 4 delayed BullMQ jobs (3 follow-ups at 3/10/21 days + 1 expiry at 30 days) with deterministic jobIds; BullMQEstimateFollowupCanceller removes those 4 jobs with per-job error isolation; EstimateFollowupsModule wired with DI token rebinding from Noop to BullMQ.

## What Was Built

### EstimateFollowupScheduler
- `scheduleFollowups(estimateId, revisionNumber)` enqueues 4 jobs to the ESTIMATE_FOLLOWUPS queue
- 3 follow-up jobs named `"follow-up"` with delays of 72h, 240h, 504h (3, 10, 21 days)
- 1 expiry job named `"expiry"` with delay of 720h (30 days)
- Deterministic jobIds (`estimate-followup:{id}:{rev}:{step}` and `estimate-expiry:{id}:{rev}`) ensure BullMQ deduplicates repeat calls
- Typed payloads: `IFollowupJobData` for follow-ups, `IExpiryJobData` for expiry

### BullMQEstimateFollowupCanceller
- Implements `IEstimateFollowupCanceller` interface from Phase 42
- `cancelAllFollowups(estimateId, revisionNumber)` removes 4 jobs by deterministic jobId
- Per-job try/catch ensures partial failures (e.g., Redis connection issues) are logged but do not prevent other removals
- Returns without throwing even if jobs are already gone (queue.remove returns 0)

### Module Wiring
- `EstimateFollowupsModule` registers both services as providers
- `ESTIMATE_FOLLOWUP_CANCELLER` DI token rebound via `useExisting: BullMQEstimateFollowupCanceller`
- Exports `EstimateFollowupScheduler` and `ESTIMATE_FOLLOWUP_CANCELLER` for use by other modules

## Test Results

- **EstimateFollowupScheduler**: 6 tests passing (call count, delays, expiry delay, jobIds, follow-up payloads, expiry payload)
- **BullMQEstimateFollowupCanceller**: 4 tests passing (call count, jobIds, no-throw on missing, error isolation)

## Deviations from Plan

None - plan executed exactly as written.

## Verification

1. Scheduler tests: `npm run test -- --testPathPatterns=estimate-followup-scheduler` -- 6/6 passing
2. Canceller tests: `npm run test -- --testPathPatterns=bullmq-estimate-followup-canceller` -- 4/4 passing
3. TypeScript compilation: no new errors introduced (all errors are pre-existing in unrelated modules)

## Self-Check: PASSED

- [x] `trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts` exists
- [x] `trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts` exists
- [x] `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` updated with wiring
- [x] Commit `afaceba` exists (Task 1)
- [x] Commit `979a020` exists (Task 2)
