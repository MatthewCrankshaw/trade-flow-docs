---
phase: 46-follow-up-queue-automation
verified: 2026-04-15T12:00:00Z
status: human_needed
score: 4/4 must-haves verified
overrides_applied: 0
re_verification:
  previous_status: gaps_found
  previous_score: 2/4
  gaps_closed:
    - "EstimateFollowupProcessor now uses EstimateFollowupEmailRenderer (not EstimateEmailRenderer) — followupStep passed to renderer, purpose-built template with 3 CTAs wired correctly"
    - "EstimateResponseHandler now injects ESTIMATE_FOLLOWUP_CANCELLER and calls cancelAllFollowups after every customer response transition (PROCEED, MESSAGE, DECLINE) with try/catch error isolation"
  gaps_remaining: []
  regressions: []
deferred:
  - truth: "Every exit transition (including conversion and mark-as-lost) cancels all pending follow-ups"
    addressed_in: "Phase 47"
    evidence: "Phase 47 SC#4: 'POST /v1/estimates/:id/mark-lost cancels all pending follow-ups'. Phase 47 SC#1: convert flow 'cancels pending follow-ups'. 46-CONTEXT.md: 'Conversion (Phase 47) → calls cancelAllFollowupsForChain', 'Mark as lost (Phase 47) → calls cancelAllFollowups'."
  - truth: "Comprehensive integration test walking all exit transitions with mocked queue asserting removal"
    addressed_in: "Phase 47"
    evidence: "SC#3 requires a full-chain integration test. Conversion and mark-as-lost transitions (the two missing exit paths) are implemented in Phase 47 — a complete multi-transition integration test is only feasible once all exit transitions are wired."
human_verification:
  - test: "Redis AOF Persistence Smoke Test"
    expected: "Start Docker Compose services (docker compose up -d). Send an estimate via POST /v1/estimates/:id/send. Verify worker logs show 'Scheduled 3 follow-ups + 1 expiry job'. Restart Redis: docker restart redis. Wait 90+ seconds. Check worker logs or BullMQ dashboard to confirm delayed jobs survived the restart."
    why_human: "Redis restart is infrastructure-level and cannot be safely automated in CI. The test requires a running Docker environment and waiting on wall-clock time. Documented as D-AOF-03 in 46-CONTEXT.md."
---

# Phase 46: Follow-up Queue & Automation Verification Report

**Phase Goal:** Sending an estimate automatically schedules 3/10/21-day follow-up emails that fire reliably across worker restarts, cancel cleanly on any exit transition, and auto-expire estimates 30 days after send.
**Verified:** 2026-04-15T12:00:00Z
**Status:** human_needed
**Re-verification:** Yes — after gap closure plans 46-06 and 46-07

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Sending an estimate calls `EstimateFollowupScheduler.scheduleFollowups()` enqueuing 3 delayed BullMQ jobs on ESTIMATE_FOLLOWUPS queue with UTC delays (72h/240h/504h), deterministic jobIds, and a 4th expiry job at 720h | VERIFIED | `estimate-followup-scheduler.service.ts` enqueues 4 jobs with FOLLOWUP_DELAYS_MS constants and `followupJobId()`/`expiryJobId()` builders. `estimate-email-sender.service.ts` calls `scheduleFollowups()` at step 9 gated on `sendType === "INITIAL"`. 6 passing unit tests confirm delays and jobId patterns. |
| 2 | `EstimateFollowupProcessor` re-reads estimate status before sending and no-ops for terminal statuses; each follow-up email uses the estimate-followup.html Maizzle template with Go Ahead/Message [FirstName]/Not right for me CTAs | VERIFIED | Line 15: `EstimateFollowupEmailRenderer` imported. Constructor at line 44: `estimateFollowupEmailRenderer` injected. Lines 107-114: `estimateFollowupEmailRenderer.render({ businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay })` called. `followupStep` cast to `"3d" | "10d" | "21d"` from STEP_TO_LABEL and passed to renderer. Tests assert correct renderer invoked with followupStep. |
| 3 | Customer response (PROCEED, MESSAGE, DECLINE) cancels all pending follow-up and expiry jobs via `cancelAllFollowups()` | VERIFIED | `estimate-response-handler.service.ts` lines 39-40: `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` with `IEstimateFollowupCanceller` type. Lines 63-70: `cancelAllFollowups(estimate.id, estimate.revisionNumber)` called after `publicTransition` with try/catch error isolation. 5 new test cases in spec file assert PROCEED/MESSAGE/DECLINE all cancel, ordering verified, error isolated. Revision exit path wired via `estimate-email-sender.service.ts` line 138. |
| 4 | Estimate auto-transitions to Expired 30 days after send; production Redis configured with `appendonly yes`/`appendfsync everysec` | VERIFIED (partial) | `handleExpiry()` calls `estimateTransitionService.publicTransition(estimateId, EXPIRED)` for worthy statuses, no-ops for terminal. `docker-compose.yaml` line 24: `--appendonly yes --appendfsync everysec`. `worker.ts`: `checkRedisAof()` logs FATAL error if not enabled. AOF restart smoke test requires human verification. |

**Score:** 4/4 truths verified (Truths 1, 2, 3 fully verified; Truth 4 verified except for human AOF smoke test)

### Deferred Items

Items not yet met but explicitly addressed in later milestone phases.

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | Conversion exit transition cancels all pending follow-ups | Phase 47 | Phase 47 SC#1 scopes convert flow; 46-CONTEXT.md: "Conversion (Phase 47) → calls cancelAllFollowupsForChain" |
| 2 | Mark-as-lost exit transition cancels all pending follow-ups | Phase 47 | Phase 47 SC#4: "POST /v1/estimates/:id/mark-lost cancels all pending follow-ups"; 46-CONTEXT.md: "Mark as lost (Phase 47) → calls cancelAllFollowups" |
| 3 | Comprehensive integration test walking all exit transitions with mocked queue | Phase 47 | SC#3 full-chain integration test is only feasible once all exit transitions are wired (conversion/mark-as-lost are Phase 47 scope) |

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|---------|--------|---------|
| `trade-flow-api/src/queue/queue.constant.ts` | ESTIMATE_FOLLOWUPS queue name constant | VERIFIED | Contains `ESTIMATE_FOLLOWUPS: "estimate-followups"` |
| `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` | Module with BullModule.registerQueue and DI token rebinding | VERIFIED | Imports EstimateModule, registers queue, provides both services, rebinds ESTIMATE_FOLLOWUP_CANCELLER via `useExisting: BullMQEstimateFollowupCanceller` |
| `trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts` | IFollowupJobData and IExpiryJobData interfaces | VERIFIED | Both interfaces exported with correct fields |
| `trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts` | FOLLOWUP_DELAYS_MS and deterministic jobId builders | VERIFIED | FOLLOWUP_DELAYS_MS (72h/240h/504h/720h), followupJobId(), expiryJobId() all exported |
| `trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts` | EstimateFollowupScheduler with scheduleFollowups() | VERIFIED | Class exists, @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) wired, scheduleFollowups() enqueues 4 jobs |
| `trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts` | BullMQEstimateFollowupCanceller implementing IEstimateFollowupCanceller | VERIFIED | Implements interface, removes 4 deterministic jobIds with per-job error isolation |
| `trade-flow-api/src/email/templates/estimate-followup.html` | Maizzle template with 3 CTAs and disclaimer | VERIFIED | Contains Go Ahead, Not right for me, disclaimer, followupStep/viewUrl/estimateNumber/businessName/traderFirstName variables |
| `trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts` | EstimateFollowupEmailRenderer with render() method | VERIFIED | Class exported, render(EstimateFollowupEmailData) method exists, registered in EmailModule |
| `trade-flow-api/src/worker/processors/estimate-followup.processor.ts` | Processor using EstimateFollowupEmailRenderer with followupStep | VERIFIED | Line 15: `EstimateFollowupEmailRenderer` imported. Line 44: `estimateFollowupEmailRenderer` injected. Lines 104-114: `followupStep` cast to FollowupStep type and passed to `estimateFollowupEmailRenderer.render()`. No EstimateEmailRenderer reference remains. |
| `trade-flow-api/src/estimate/services/estimate-email-sender.service.ts` | Step 9 scheduleFollowups call on INITIAL send path | VERIFIED | Injects EstimateFollowupScheduler, calls scheduleFollowups in try/catch gated on `sendType === "INITIAL"` at line 142-144 |
| `trade-flow-api/src/estimate/services/estimate-response-handler.service.ts` | ESTIMATE_FOLLOWUP_CANCELLER injection and cancelAllFollowups call | VERIFIED | Lines 19-21: ESTIMATE_FOLLOWUP_CANCELLER and IEstimateFollowupCanceller imported. Lines 39-40: @Inject(ESTIMATE_FOLLOWUP_CANCELLER) constructor parameter. Lines 63-70: cancelAllFollowups() called with try/catch. |
| `trade-flow-api/src/worker.ts` | Redis AOF startup check | VERIFIED (partial) | checkRedisAof() runs CONFIG GET appendonly, logs FATAL if not yes. Does NOT refuse to register processor (known deviation — static NestJS DI limitation). |
| `trade-flow-api/docker-compose.yaml` | Redis AOF persistence configuration | VERIFIED | `command: redis-server --maxmemory-policy noeviction --appendonly yes --appendfsync everysec` at line 24 |
| `trade-flow-api/src/app.module.ts` | EstimateFollowupsModule imported after EstimateModule | VERIFIED | EstimateFollowupsModule imported at line 52, after EstimateModule |
| `trade-flow-api/src/worker/worker.module.ts` | EstimateFollowupsModule and EstimateFollowupProcessor registered | VERIFIED | Imports EstimateFollowupsModule, providers include EstimateFollowupProcessor |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|----|--------|---------|
| estimate-followup-scheduler.service.ts | queue/queue.constant.ts | @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) | WIRED | Confirmed in service constructor |
| estimate-followups.module.ts | estimate-followup-canceller.interface.ts | { provide: ESTIMATE_FOLLOWUP_CANCELLER, useExisting: BullMQEstimateFollowupCanceller } | WIRED | Line 14 of module confirms rebinding |
| estimate-followup.processor.ts | estimate-followup-email-renderer.service.ts | constructor injection + estimateFollowupEmailRenderer.render() | WIRED | Lines 15, 44, 107-114 — gap CR-01 CLOSED by 46-06 |
| estimate-email-sender.service.ts | estimate-followup-scheduler.service.ts | injected EstimateFollowupScheduler, called at step 9 | WIRED | Line 144: `this.estimateFollowupScheduler.scheduleFollowups()` |
| estimate-response-handler.service.ts | estimate-followup-canceller.interface.ts | @Inject(ESTIMATE_FOLLOWUP_CANCELLER) | WIRED | Lines 39-40, 63-70 — gap CLOSED by 46-07 |
| app.module.ts | estimate-followups.module.ts | imports array | WIRED | EstimateFollowupsModule at line 52 |
| worker.module.ts | EstimateFollowupProcessor | providers array | WIRED | EstimateFollowupProcessor in providers |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| estimate-followup.processor.ts / handleFollowup | estimate | EstimateRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | business | BusinessRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | customer | CustomerRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | html (rendered email) | EstimateFollowupEmailRenderer.render({ businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay }) | Yes — purpose-built template with real estimate data | FLOWING |
| estimate-followup.processor.ts / handleExpiry | estimate | EstimateRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-response-handler.service.ts / handleResponse | cancelAllFollowups call | IEstimateFollowupCanceller (resolved to BullMQEstimateFollowupCanceller) | Yes — BullMQ queue.remove() on real job IDs | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Queue constant resolves correctly | grep ESTIMATE_FOLLOWUPS queue.constant.ts | `ESTIMATE_FOLLOWUPS: "estimate-followups"` | PASS |
| Module DI token rebinding present | grep "useExisting.*BullMQ" estimate-followups.module.ts | Match found at line 14 | PASS |
| Processor uses correct renderer | grep "EstimateFollowupEmailRenderer\|EstimateEmailRenderer" estimate-followup.processor.ts | Only EstimateFollowupEmailRenderer found (import + constructor) | PASS |
| followupStep passed to renderer | grep "followupStep" estimate-followup.processor.ts | Line 104: computed, Line 109: passed to render call | PASS |
| AOF check in worker.ts | grep "appendonly" worker.ts | CONFIG GET appendonly found, FATAL log if not enabled | PASS |
| Docker Compose AOF | grep "appendonly" docker-compose.yaml | appendonly yes at line 24 | PASS |
| scheduleFollowups in email sender | grep "scheduleFollowups" estimate-email-sender.service.ts | Match at line 144 | PASS |
| Customer response handler has canceller | grep "cancelAllFollowups\|ESTIMATE_FOLLOWUP_CANCELLER" estimate-response-handler.service.ts | 3 matches (import, @Inject, call site) | PASS |
| Response handler spec has cancellation tests | grep "cancelAllFollowups" estimate-response-handler.service.spec.ts | 13 matches — mock + 5 test cases (PROCEED, MESSAGE, DECLINE, ordering, error isolation) | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|------------|------------|-------------|--------|---------|
| FUP-01 | 46-01, 46-02, 46-05 | Sending schedules 3-day, 10-day, 21-day BullMQ delayed jobs on ESTIMATE_FOLLOWUPS queue | SATISFIED | scheduleFollowups() verified wired into EstimateEmailSender step 9; queue constant exists; scheduler enqueues 4 jobs |
| FUP-02 | 46-01 | Follow-up delays relative in UTC (72h, 240h, 504h) | SATISFIED | FOLLOWUP_DELAYS_MS constants use absolute hour calculations ensuring UTC-relative delays |
| FUP-03 | 46-01, 46-02 | Deterministic jobId pattern `estimate-followup:{estimateId}:{revisionNumber}:{step}` | SATISFIED | followupJobId() and expiryJobId() functions produce correct patterns; tested in unit tests |
| FUP-04 | 46-03, 46-06 | Follow-up email includes estimate summary and three response CTAs | SATISFIED | Template (estimate-followup.html) EXISTS with correct CTAs. Processor now uses EstimateFollowupEmailRenderer (CR-01 CLOSED by 46-06). followupStep passed to renderer for step-specific copy. |
| FUP-05 | 46-02, 46-05, 46-07 | Any exit transition cancels all pending follow-ups | PARTIALLY SATISFIED | Customer response: WIRED (46-07). Revision cancel: WIRED (EstimateEmailSender step 8). Conversion/mark-as-lost: deferred to Phase 47. |
| FUP-06 | 46-04 | Processor defence-in-depth status check, silent no-op for non-worthy status | SATISFIED | FOLLOWUP_WORTHY_STATUSES array checked; processor returns early for non-worthy status |
| FUP-07 | 46-04 | Estimates auto-transition to Expired 30 days after Sent timestamp | SATISFIED | handleExpiry() calls publicTransition(estimateId, EXPIRED) for worthy statuses; unit tests pass |
| FUP-08 | 46-05 | Production Redis with appendonly yes / appendfsync everysec | PARTIALLY SATISFIED | Docker Compose configured; worker.ts AOF check warns on failure but does not refuse processor registration (known deviation — static NestJS DI limitation) |

### Anti-Patterns Found

No blockers or warnings found. Previous blockers (wrong renderer import, missing canceller injection) have been resolved.

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| estimate-followup.processor.ts | — | (previously) EstimateEmailRenderer — RESOLVED | CLEARED | EstimateFollowupEmailRenderer now correctly imported and injected |
| estimate-response-handler.service.ts | — | (previously) no canceller injection — RESOLVED | CLEARED | ESTIMATE_FOLLOWUP_CANCELLER now injected and called |

### Human Verification Required

#### 1. Redis AOF Persistence Smoke Test

**Test:** Start Docker Compose services (`docker compose up -d`). Send an estimate via `POST /v1/estimates/:id/send`. Verify worker logs show "Scheduled 3 follow-ups + 1 expiry job". Restart Redis: `docker restart redis`. Wait 90+ seconds for the delayed jobs to still exist. Check worker logs or BullMQ dashboard to confirm delayed jobs survived the restart.
**Expected:** Delayed jobs are still present in Redis after restart. If AOF is working, no jobs are lost.
**Why human:** Redis restart is infrastructure-level and cannot be safely automated in CI. The test requires a running Docker environment and waiting on wall-clock time. Documented as D-AOF-03 in 46-CONTEXT.md.

### Re-verification Summary

**Both gaps from the initial verification are closed:**

**Gap 1 — Wrong email renderer (CLOSED by 46-06):** `EstimateFollowupProcessor` now imports `EstimateFollowupEmailRenderer` (line 15) and injects it as `estimateFollowupEmailRenderer` (line 44). The `handleFollowup()` method calls `estimateFollowupEmailRenderer.render({ businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay })` at lines 107-114. The `followupStep` variable (cast to `"3d" | "10d" | "21d"` from STEP_TO_LABEL) is passed to the renderer — not just logged. No reference to `EstimateEmailRenderer` remains. 11/11 processor tests pass.

**Gap 2 — Customer response does not cancel follow-ups (CLOSED by 46-07):** `EstimateResponseHandler` now injects `ESTIMATE_FOLLOWUP_CANCELLER` with `IEstimateFollowupCanceller` type (lines 39-40). The `handleResponse()` method calls `cancelAllFollowups(estimate.id, estimate.revisionNumber)` after `publicTransition` succeeds (lines 63-70), wrapped in try/catch so queue failures never break the customer response flow. 5 new test cases verify PROCEED, MESSAGE, and DECLINE cancellation, ordering, and error isolation. 19/19 response handler tests pass.

Conversion and mark-as-lost cancellation remain deferred to Phase 47 per the roadmap design. The one remaining human verification item (Redis AOF smoke test) is infrastructure-level and cannot be automated.

---

_Verified: 2026-04-15T12:00:00Z_
_Verifier: Claude (gsd-verifier)_
