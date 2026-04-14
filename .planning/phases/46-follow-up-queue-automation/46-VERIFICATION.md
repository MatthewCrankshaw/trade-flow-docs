---
phase: 46-follow-up-queue-automation
verified: 2026-04-14T14:00:00Z
status: gaps_found
score: 2/4 must-haves verified
overrides_applied: 0
gaps:
  - truth: "EstimateFollowupProcessor sends follow-up emails using the purpose-built estimate-followup.html Maizzle template (3d/10d/21d) with Go Ahead, Message [FirstName], Not right for me CTAs"
    status: failed
    reason: "The processor imports EstimateEmailRenderer (the generic estimate email renderer) instead of EstimateFollowupEmailRenderer. The purpose-built estimate-followup.html template is never used in actual job execution. Customers receive the generic estimate email with a hardcoded 'Just following up on your estimate...' message, not the step-specific follow-up template. This is documented as CR-01 in the Phase 46 code review."
    artifacts:
      - path: "trade-flow-api/src/worker/processors/estimate-followup.processor.ts"
        issue: "Imports EstimateEmailRenderer at line 15 and calls this.estimateEmailRenderer.render() at line 107. EstimateFollowupEmailRenderer is neither imported nor injected. The followupStep variable computed at line 104 is only used in a log message, never passed to any renderer."
    missing:
      - "Replace EstimateEmailRenderer import with EstimateFollowupEmailRenderer from @email/services/estimate-followup-email-renderer.service"
      - "Inject EstimateFollowupEmailRenderer in processor constructor"
      - "Call estimateFollowupEmailRenderer.render({ businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay }) in handleFollowup"
      - "Pass followupStep variable to the renderer"

  - truth: "Every exit transition (customer response, revision, conversion, manual delete, mark-as-lost) cancels all pending follow-ups, verified by an integration test walking each transition"
    status: failed
    reason: "Customer response exit transition is NOT wired: EstimateResponseHandler (the Phase 45 service that handles customer PROCEED/MESSAGE/DECLINE responses) does not inject IEstimateFollowupCanceller and does not call cancelAllFollowups(). FUP-05 is mapped to Phase 46 in REQUIREMENTS.md traceability. Additionally, no 'integration test that walks each transition with a mocked queue and asserts removal' exists as required by SC#3. Conversion and mark-as-lost cancellation are correctly deferred to Phase 47 (documented in 46-CONTEXT.md)."
    artifacts:
      - path: "trade-flow-api/src/estimate/services/estimate-response-handler.service.ts"
        issue: "Constructor does not inject ESTIMATE_FOLLOWUP_CANCELLER. The handleResponse() method transitions estimate status (RESPONDED/DECLINED) without cancelling pending follow-up jobs. When a customer responds, the follow-up jobs remain in Redis and will fire anyway — only the FUP-06 processor guard provides partial mitigation."
    missing:
      - "Inject ESTIMATE_FOLLOWUP_CANCELLER into EstimateResponseHandler constructor"
      - "Call cancelAllFollowups(estimateId, revisionNumber) after successful response transition in handleResponse()"
      - "Add tests to estimate-response-handler.service.spec.ts asserting canceller is called on response"
      - "Create integration test walking all wired exit transitions (customer response + revision) with mocked queue, asserting all 4 job removals"

deferred:
  - truth: "Conversion exit transition cancels all pending follow-ups"
    addressed_in: "Phase 47"
    evidence: "46-CONTEXT.md explicitly documents: 'Conversion (Phase 47) → calls cancelAllFollowupsForChain'. Phase 47 goal and success criteria scope the convert-to-quote flow."
  - truth: "Mark-as-lost exit transition cancels all pending follow-ups"
    addressed_in: "Phase 47"
    evidence: "Phase 47 SC#4: 'POST /v1/estimates/:id/mark-lost transitions the estimate to Lost... cancels all pending follow-ups'. 46-CONTEXT.md: 'Mark as lost (Phase 47) → calls cancelAllFollowups'."

human_verification:
  - test: "Redis AOF smoke test"
    expected: "Schedule a 60-second delayed job on ESTIMATE_FOLLOWUPS queue, restart Redis with 'docker restart redis', wait 90+ seconds, confirm job fires in worker logs"
    why_human: "Redis restart is infrastructure-level, cannot be performed in automated CI. Documented as D-AOF-03 in 46-CONTEXT.md. Confirms that AOF persistence actually saves delayed jobs across restarts."
---

# Phase 46: Follow-up Queue & Automation Verification Report

**Phase Goal:** Sending an estimate automatically schedules 3/10/21-day follow-up emails that fire reliably across worker restarts, cancel cleanly on any exit transition, and auto-expire estimates 30 days after send.
**Verified:** 2026-04-14T14:00:00Z
**Status:** gaps_found
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Sending an estimate calls `EstimateFollowupScheduler.scheduleFollowups()` enqueuing 3 delayed BullMQ jobs on ESTIMATE_FOLLOWUPS queue with UTC delays (72h/240h/504h), deterministic jobIds, and a 4th expiry job at 720h | VERIFIED | `estimate-followup-scheduler.service.ts` calls `queue.add("follow-up", ...)` 3 times + `queue.add("expiry", ...)` once with delays from FOLLOWUP_DELAYS_MS and jobIds from `followupJobId()`/`expiryJobId()`. EstimateEmailSender calls `scheduleFollowups()` at step 9 gated on `sendType === "INITIAL"`. 6 passing unit tests confirm delays and jobId patterns. |
| 2 | `EstimateFollowupProcessor` re-reads estimate status before sending and no-ops for terminal statuses; each follow-up email uses the estimate-followup.html Maizzle template with Go Ahead/Message [FirstName]/Not right for me CTAs | FAILED | Processor performs defence-in-depth status re-read (VERIFIED) and no-ops for terminal statuses (VERIFIED). However, `estimate-followup.processor.ts` line 15 imports `EstimateEmailRenderer` (the generic renderer) not `EstimateFollowupEmailRenderer`. The purpose-built `estimate-followup.html` template with 3 CTAs is never called. Customers receive a generic email with hardcoded copy. |
| 3 | Every exit transition cancels pending follow-ups via `cancelAllFollowups()`, verified by integration test walking each transition | FAILED | Revision cancel: WIRED (EstimateEmailSender step 8). Customer response: NOT WIRED (EstimateResponseHandler has no canceller). Conversion/mark-as-lost: deferred to Phase 47. No integration test walking multiple transitions with mocked queue exists. |
| 4 | Estimate auto-transitions to Expired 30 days after send; production Redis configured with `appendonly yes`/`appendfsync everysec` | PARTIALLY VERIFIED | ExpityProcessor: WIRED — `handleExpiry()` in processor calls `estimateTransitionService.publicTransition(estimateId, EXPIRED)` for worthy statuses, no-ops for terminal. Docker Compose Redis: `appendonly yes --appendfsync everysec` confirmed in `docker-compose.yaml` line 24. AOF startup check: runs at worker startup and logs FATAL error if not enabled — but does NOT refuse to register the processor (documented deviation in 46-05 SUMMARY). Redis restart smoke test is manual-only (D-AOF-03). |

**Score:** 2/4 truths fully verified (Truth 1 and partial of Truth 4)

### Deferred Items

Items not yet met but explicitly addressed in later milestone phases.

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | Conversion exit transition cancels pending follow-ups | Phase 47 | 46-CONTEXT.md: "Conversion (Phase 47) → calls cancelAllFollowupsForChain" |
| 2 | Mark-as-lost exit transition cancels pending follow-ups | Phase 47 | Phase 47 SC#4: "cancels all pending follow-ups"; 46-CONTEXT.md: "Mark as lost (Phase 47) → calls cancelAllFollowups" |

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|---------|--------|---------|
| `trade-flow-api/src/queue/queue.constant.ts` | ESTIMATE_FOLLOWUPS queue name constant | VERIFIED | Contains `ESTIMATE_FOLLOWUPS: "estimate-followups"` |
| `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` | Module with BullModule.registerQueue and DI token rebinding | VERIFIED | Imports EstimateModule, registers queue, provides both services, rebinds ESTIMATE_FOLLOWUP_CANCELLER via useExisting |
| `trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts` | IFollowupJobData and IExpiryJobData interfaces | VERIFIED | Both interfaces exported with correct fields |
| `trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts` | FOLLOWUP_DELAYS_MS and deterministic jobId builders | VERIFIED | FOLLOWUP_DELAYS_MS (72h/240h/504h/720h), followupJobId(), expiryJobId() all exported |
| `trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts` | EstimateFollowupScheduler with scheduleFollowups() | VERIFIED | Class exists, @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) wired, scheduleFollowups() enqueues 4 jobs |
| `trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts` | BullMQEstimateFollowupCanceller implementing IEstimateFollowupCanceller | VERIFIED | Implements interface, removes 4 deterministic jobIds with per-job error isolation |
| `trade-flow-api/src/email/templates/estimate-followup.html` | Maizzle template with 3 CTAs and disclaimer | VERIFIED | Contains Go Ahead, Not right for me, "This is an estimate" disclaimer, followupStep/viewUrl/estimateNumber/businessName/traderFirstName variables |
| `trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts` | EstimateFollowupEmailRenderer with render() method | VERIFIED | Class exported, render(EstimateFollowupEmailData) method exists, registered in EmailModule |
| `trade-flow-api/src/worker/processors/estimate-followup.processor.ts` | Combined processor handling follow-up and expiry jobs | STUB | Processor exists with correct @Processor decorator, defence-in-depth status re-read, expiry handling. CRITICAL: uses EstimateEmailRenderer instead of EstimateFollowupEmailRenderer. follow-up email rendering is broken. |
| `trade-flow-api/src/estimate/services/estimate-email-sender.service.ts` | Step 9 scheduleFollowups call on INITIAL send path | VERIFIED | Injects EstimateFollowupScheduler, calls scheduleFollowups in try/catch gated on sendType === "INITIAL" |
| `trade-flow-api/src/worker.ts` | Redis AOF startup check | VERIFIED (partial) | checkRedisAof() runs CONFIG GET appendonly, logs FATAL if not yes. Does NOT refuse to register processor (known deviation). |
| `trade-flow-api/docker-compose.yaml` | Redis AOF persistence configuration | VERIFIED | `command: redis-server --maxmemory-policy noeviction --appendonly yes --appendfsync everysec` |
| `trade-flow-api/src/app.module.ts` | EstimateFollowupsModule imported after EstimateModule | VERIFIED | EstimateFollowupsModule imported at line 52, after EstimateModule |
| `trade-flow-api/src/worker/worker.module.ts` | EstimateFollowupsModule and EstimateFollowupProcessor registered | VERIFIED | Imports EstimateFollowupsModule, providers include EstimateFollowupProcessor |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|----|--------|---------|
| estimate-followup-scheduler.service.ts | queue/queue.constant.ts | @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) | WIRED | Line 14 confirms injection |
| estimate-followups.module.ts | estimate-followup-canceller.interface.ts | { provide: ESTIMATE_FOLLOWUP_CANCELLER, useExisting: BullMQEstimateFollowupCanceller } | WIRED | Module correctly rebinds DI token |
| estimate-followup.processor.ts | estimate-followup-scheduler.service.ts | @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) via processor | WIRED | Processor registered on same queue |
| estimate-email-sender.service.ts | estimate-followup-scheduler.service.ts | injected EstimateFollowupScheduler, called as step 9 | WIRED | this.estimateFollowupScheduler.scheduleFollowups() called at line 144 |
| app.module.ts | estimate-followups.module.ts | imports array | WIRED | EstimateFollowupsModule at line 52, after EstimateModule |
| estimate-followup.processor.ts | estimate-followup-email-renderer.service.ts | processor constructor injection | NOT WIRED | CR-01 critical bug: processor imports EstimateEmailRenderer, not EstimateFollowupEmailRenderer |
| estimate-response-handler.service.ts | estimate-followup-canceller.interface.ts | ESTIMATE_FOLLOWUP_CANCELLER injection | NOT WIRED | EstimateResponseHandler has no canceller injection — customer response does not cancel follow-ups |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| estimate-followup.processor.ts / handleFollowup | estimate | EstimateRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | business | BusinessRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | customer | CustomerRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |
| estimate-followup.processor.ts / handleFollowup | html (rendered email) | EstimateEmailRenderer.render() | Connected but wrong renderer | HOLLOW — wrong renderer, EstimateFollowupEmailRenderer never called |
| estimate-followup.processor.ts / handleExpiry | estimate | EstimateRepository.findByIdOrFail | Yes — real MongoDB query | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Queue constant resolves correctly | `grep ESTIMATE_FOLLOWUPS trade-flow-api/src/queue/queue.constant.ts` | `ESTIMATE_FOLLOWUPS: "estimate-followups"` | PASS |
| Module DI token rebinding present | `grep "useExisting.*BullMQ" trade-flow-api/src/estimate-followups/estimate-followups.module.ts` | Match found | PASS |
| Processor uses wrong renderer | `grep "EstimateEmailRenderer\|EstimateFollowupEmailRenderer" trade-flow-api/src/worker/processors/estimate-followup.processor.ts` | Only EstimateEmailRenderer found | FAIL |
| AOF check in worker.ts | `grep "appendonly" trade-flow-api/src/worker.ts` | CONFIG GET appendonly found | PASS |
| Docker Compose AOF | `grep "appendonly" trade-flow-api/docker-compose.yaml` | appendonly yes found | PASS |
| scheduleFollowups in email sender | `grep "scheduleFollowups" trade-flow-api/src/estimate/services/estimate-email-sender.service.ts` | Match at line 144 | PASS |
| Customer response handler has no canceller | `grep "cancelAllFollowups\|ESTIMATE_FOLLOWUP_CANCELLER" trade-flow-api/src/estimate/services/estimate-response-handler.service.ts` | No matches | FAIL |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|------------|------------|-------------|--------|---------|
| FUP-01 | 46-01, 46-02, 46-05 | Sending schedules 3-day, 10-day, 21-day BullMQ delayed jobs on ESTIMATE_FOLLOWUPS queue | SATISFIED | scheduleFollowups() verified wired into EstimateEmailSender step 9; queue constant exists; scheduler enqueues 4 jobs |
| FUP-02 | 46-01 | Follow-up delays relative in UTC (72h, 240h, 504h) | SATISFIED | FOLLOWUP_DELAYS_MS constants use absolute hour calculations ensuring UTC-relative delays |
| FUP-03 | 46-01, 46-02 | Deterministic jobId pattern `estimate-followup:{estimateId}:{revisionNumber}:{step}` | SATISFIED | followupJobId() and expiryJobId() functions produce correct patterns; tested in unit tests |
| FUP-04 | 46-03 | Follow-up email includes estimate summary and three response CTAs | BLOCKED | Template (estimate-followup.html) EXISTS with correct CTAs. Processor does NOT use this template — it uses the generic EstimateEmailRenderer instead. CTAs never appear in actual sent emails. |
| FUP-05 | 46-02, 46-05 | Any exit transition cancels all pending follow-ups | BLOCKED | Revision cancel: WIRED. Customer response: NOT WIRED (EstimateResponseHandler). Conversion/mark-as-lost: deferred to Phase 47. |
| FUP-06 | 46-04 | Processor defence-in-depth status check, silent no-op for non-worthy status | SATISFIED | FOLLOWUP_WORTHY_STATUSES array checked; processor returns early for non-worthy status |
| FUP-07 | 46-04 | Estimates auto-transition to Expired 30 days after Sent | SATISFIED | handleExpiry() calls publicTransition(estimateId, EXPIRED) for worthy statuses; 6 unit tests pass |
| FUP-08 | 46-05 | Production Redis with appendonly yes / appendfsync everysec | PARTIALLY SATISFIED | Docker Compose configured; worker.ts AOF check warns on failure but does not refuse processor registration (known deviation — static NestJS DI limitation) |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| estimate-followup.processor.ts | 15 | Wrong renderer import (`EstimateEmailRenderer` instead of `EstimateFollowupEmailRenderer`) | BLOCKER | Follow-up emails use generic template; 3 CTAs (Go Ahead, Message [FirstName], Not right for me) and step-specific copy are never rendered |
| estimate-followup.processor.ts | 107-115 | Hardcoded generic message `"Just following up on your estimate..."` passed to wrong renderer | BLOCKER | Email content is wrong; followupStep variable computed but unused in render call |
| estimate-response-handler.service.ts | 0 | No ESTIMATE_FOLLOWUP_CANCELLER injection | BLOCKER | Customer response does not cancel follow-ups; stale follow-up jobs will fire after customer responds |

### Human Verification Required

#### 1. Redis AOF Persistence Smoke Test

**Test:** Start Docker Compose services (`docker compose up -d`). Send an estimate via `POST /v1/estimates/:id/send`. Verify worker logs show "Scheduled 3 follow-ups + 1 expiry job". Restart Redis: `docker restart redis`. Wait 90+ seconds for the delayed jobs to still exist. Check worker logs or BullMQ dashboard to confirm delayed jobs survived the restart.
**Expected:** Delayed jobs are still present in Redis after restart. If AOF is working, no jobs are lost.
**Why human:** Redis restart is infrastructure-level and cannot be safely automated in CI. The test requires a running Docker environment and waiting on wall-clock time.

### Gaps Summary

**Two actionable gaps block goal achievement:**

**Gap 1 — Wrong email renderer (blocks FUP-04 and SC#2):** The `EstimateFollowupProcessor.handleFollowup()` method uses `EstimateEmailRenderer` instead of `EstimateFollowupEmailRenderer`. The purpose-built `estimate-followup.html` template with step-specific copy variations (3d/10d/21d) and the three response CTAs ("Go Ahead", "Message [FirstName]", "Not right for me") is completely bypassed. Customers receive a generic estimate email with hardcoded copy. This is a critical bug documented as CR-01 in the Phase 46 code review but not yet fixed.

**Gap 2 — Customer response does not cancel follow-ups (blocks FUP-05 and SC#3):** When a customer submits a PROCEED, MESSAGE, or DECLINE response via the public estimate page, `EstimateResponseHandler` transitions the estimate status but does not call `cancelAllFollowups()`. The 4 pending BullMQ jobs remain in Redis. The FUP-06 processor guard provides partial mitigation (no-op for non-worthy statuses after DECLINED), but RESPONDED is still in FOLLOWUP_WORTHY_STATUSES — meaning the 3d/10d/21d follow-up jobs will still fire after a customer responds with PROCEED or MESSAGE. The SC#3 integration test requirement (walking each exit transition with a mocked queue) is also unmet.

Conversion and mark-as-lost cancellation are correctly deferred to Phase 47 per the roadmap design.

---

_Verified: 2026-04-14T14:00:00Z_
_Verifier: Claude (gsd-verifier)_
