---
phase: 46
slug: follow-up-queue-automation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-13
---

# Phase 46 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest 30.2.0 with ts-jest |
| **Config file** | `trade-flow-api/jest.config.js` |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-followup` |
| **Full suite command** | `cd trade-flow-api && npm run test` |
| **Estimated runtime** | ~30 seconds (unit tests only) |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern=estimate-followup`
- **After every plan wave:** Run `cd trade-flow-api && npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green (`npm run ci` passes in both repos)
- **Max feedback latency:** ~30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 46-W0-01 | 01 | 0 | FUP-01 | — | N/A | unit stub | `npm test -- --testPathPattern=estimate-followup-scheduler` | ❌ W0 | ⬜ pending |
| 46-W0-02 | 01 | 0 | FUP-05 | — | N/A | unit stub | `npm test -- --testPathPattern=bullmq-estimate-followup-canceller` | ❌ W0 | ⬜ pending |
| 46-W0-03 | 01 | 0 | FUP-06 | V4 | Processor re-reads status before sending | unit stub | `npm test -- --testPathPattern=estimate-followup.processor` | ❌ W0 | ⬜ pending |
| 46-W0-04 | 01 | 0 | FUP-07 | — | N/A | unit stub | `npm test -- --testPathPattern=estimate-expiry.processor` | ❌ W0 | ⬜ pending |
| 46-01-01 | 01 | 1 | FUP-01 | — | N/A | unit | `npm test -- --testPathPattern=estimate-followup-scheduler` | ❌ W0 | ⬜ pending |
| 46-01-02 | 01 | 1 | FUP-02 | — | N/A | unit | same | ❌ W0 | ⬜ pending |
| 46-01-03 | 01 | 1 | FUP-03 | — | Deterministic jobId deduplication | unit | same | ❌ W0 | ⬜ pending |
| 46-02-01 | 02 | 1 | FUP-05 | — | N/A | unit | `npm test -- --testPathPattern=bullmq-estimate-followup-canceller` | ❌ W0 | ⬜ pending |
| 46-03-01 | 03 | 2 | FUP-06 | V4 | Status re-read before send; businessId policy check | unit | `npm test -- --testPathPattern=estimate-followup.processor` | ❌ W0 | ⬜ pending |
| 46-03-02 | 03 | 2 | FUP-07 | — | N/A | unit | `npm test -- --testPathPattern=estimate-expiry.processor` | ❌ W0 | ⬜ pending |
| 46-04-01 | 04 | 2 | FUP-04 | — | N/A | unit | `npm test -- --testPathPattern=estimate-followup-email-renderer` | ❌ W0 | ⬜ pending |
| 46-05-01 | 05 | 3 | FUP-08 | — | AOF check refuses processor if disabled | unit | `npm test -- --testPathPattern=worker` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

All test files for Phase 46 are new (Wave 0 must create stubs):

- [ ] `src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts` — stubs for FUP-01, FUP-02, FUP-03
- [ ] `src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts` — stubs for FUP-05
- [ ] `src/worker/test/processors/estimate-followup.processor.spec.ts` — stubs for FUP-06
- [ ] `src/worker/test/processors/estimate-expiry.processor.spec.ts` — stubs for FUP-07

Mocking approach (per project convention): Pure unit tests with mocked Queue (`{ add: jest.fn(), remove: jest.fn() }`), mocked `EstimateRetriever`, and mocked `EmailSenderService`. No `mongodb-memory-server` — per Phase 42 D-HOOK-04 explicit instruction.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Redis AOF smoke test: schedule 60s delayed job, restart Redis, confirm job fires | FUP-08 (SC #4) | Redis restart is infrastructure-level, cannot be automated in CI | 1. POST to `/v1/queue/test-echo` with a 60s delay; 2. `docker restart redis`; 3. Wait 60s; 4. Assert job processed in worker logs (look for "test-echo processed") |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
