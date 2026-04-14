---
phase: 46-follow-up-queue-automation
plan: 04
subsystem: estimate-followups
tags: [bullmq, processor, email, expiry, worker]
dependency_graph:
  requires: [46-01, 46-02]
  provides: [estimate-followup-processor, estimate-expiry-processor]
  affects: [worker-module, estimate-module]
tech_stack:
  added: []
  patterns: [combined-processor-dispatch-by-job-name, defence-in-depth-status-re-read]
key_files:
  created:
    - trade-flow-api/src/worker/processors/estimate-followup.processor.ts
  modified:
    - trade-flow-api/src/worker/worker.module.ts
    - trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts
    - trade-flow-api/src/worker/test/processors/estimate-expiry.processor.spec.ts
    - trade-flow-api/src/estimate/estimate.module.ts
decisions:
  - Combined processor pattern (single class dispatching by job.name) instead of two separate @Processor classes, since BullMQ only allows one processor per queue
  - Used repositories directly (not service-layer retrievers) for processor since no auth context available in background jobs
  - Used EstimateTransitionService.publicTransition for expiry (no auth user required)
  - Removed SITE_VISIT_REQUESTED from worthy statuses since the enum value does not exist in the codebase
metrics:
  duration: 7min
  completed: 2026-04-14
  tasks: 2
  files: 5
---

# Phase 46 Plan 04: Estimate Followup and Expiry Processors Summary

Combined BullMQ processor dispatching follow-up email jobs and 30-day expiry jobs by job.name, with defence-in-depth status re-read from MongoDB before any action (FUP-06).

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | EstimateFollowupProcessor with defence-in-depth status check | api@6401ea6 (test), api@fc4b32a (impl) | estimate-followup.processor.ts, estimate-followup.processor.spec.ts |
| 2 | EstimateExpiryProcessor and WorkerModule registration | api@bf17101 (test), api@d630c6c (impl) | estimate-followup.processor.ts, estimate-expiry.processor.spec.ts, worker.module.ts, estimate.module.ts |

## Implementation Details

### EstimateFollowupProcessor (follow-up email jobs)

- Extends `WorkerHost` with `@Processor(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)`
- Dispatches by `job.name`: `"follow-up"` -> handleFollowup, `"expiry"` -> handleExpiry
- Re-reads estimate from `EstimateRepository.findByIdOrFail` at execution time (FUP-06)
- Checks estimate status against worthy statuses: SENT, VIEWED, RESPONDED
- Reads document token via `DocumentTokenRepository.findActiveByDocumentId` using rootEstimateId
- Reads business name from `BusinessRepository.findByIdOrFail`
- Reads recipient email from `CustomerRepository.findByIdOrFail` (never from job payload per T-46-06)
- Renders email via `EstimateEmailRenderer.render`
- Sends via `EmailSenderService.sendEmail` (throws on failure for BullMQ retry)
- Creates audit row via `EstimateEmailSendCreator.create` with correct send type mapping
- Step 1 -> FOLLOWUP_3D, Step 2 -> FOLLOWUP_10D, Step 3 -> FOLLOWUP_21D

### EstimateExpiryProcessor (30-day expiry jobs)

- Handled as `job.name === "expiry"` branch in the same processor class
- Re-reads estimate status from MongoDB (defence-in-depth)
- Calls `EstimateTransitionService.publicTransition(estimateId, EstimateStatus.EXPIRED)` for worthy statuses
- Silently no-ops for terminal statuses (converted, declined, expired, lost, deleted)
- Catches and logs transition errors without re-throwing (job completes gracefully)

### WorkerModule Registration

- Imported `EstimateModule`, `EstimateFollowupsModule`, `EmailModule`, `DocumentTokenModule`, `BusinessModule`, `CustomerModule`
- Added `EstimateFollowupProcessor` to providers array
- `EstimateFollowupsModule` imported after `EstimateModule` for correct DI token rebinding

### EstimateModule Export Addition

- Exported `EstimateEmailSendCreator` to make it available to the processor via DI

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Removed SITE_VISIT_REQUESTED from worthy statuses**
- **Found during:** Task 1
- **Issue:** Plan referenced `EstimateStatus.SITE_VISIT_REQUESTED` but this value does not exist in the enum
- **Fix:** Removed from the worthy statuses array; kept SENT, VIEWED, RESPONDED
- **Files modified:** estimate-followup.processor.ts

**2. [Rule 2 - Missing functionality] Added module imports to WorkerModule**
- **Found during:** Task 2
- **Issue:** Processor depends on services from multiple modules not imported in WorkerModule
- **Fix:** Added imports for EstimateModule, EmailModule, DocumentTokenModule, BusinessModule, CustomerModule
- **Files modified:** worker.module.ts

**3. [Rule 2 - Missing functionality] Exported EstimateEmailSendCreator from EstimateModule**
- **Found during:** Task 2
- **Issue:** Processor needs EstimateEmailSendCreator for audit row creation but it was not exported
- **Fix:** Added to EstimateModule exports array
- **Files modified:** estimate.module.ts

**4. [Rule 3 - Blocking issue] Used repositories instead of service-layer retrievers**
- **Found during:** Task 1
- **Issue:** EstimateRetriever.findByIdOrFail and BusinessRetriever.findByIdOrFail require authUser parameter, but processor has no auth context
- **Fix:** Used EstimateRepository.findByIdOrFail, BusinessRepository.findByIdOrFail, CustomerRepository.findByIdOrFail directly (no policy check needed for trusted internal processor)
- **Files modified:** estimate-followup.processor.ts

## Verification Results

- Follow-up processor tests: 8/8 passed
- Expiry processor tests: 6/6 passed
- WorkerModule imports EstimateFollowupsModule: confirmed
- TypeScript compiles: clean (0 errors)

## Self-Check: PASSED
