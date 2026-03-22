---
phase: 22
slug: worker-service-scaffold
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 22 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest 30.2.0 |
| **Config file** | `package.json` jest section |
| **Quick run command** | `npm test -- --testPathPattern="worker\|queue.*controller" --no-coverage` |
| **Full suite command** | `npm test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- --testPathPattern="worker\|queue" --no-coverage`
- **After every plan wave:** Run `npm test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 22-01-01 | 01 | 1 | WORK-01 | manual-only | `node dist/worker.js` + SIGTERM | N/A | ⬜ pending |
| 22-01-02 | 01 | 1 | WORK-02 | unit | `npm test -- --testPathPattern="worker.module" --no-coverage` | ❌ W0 | ⬜ pending |
| 22-01-03 | 01 | 1 | WORK-03 | unit | `npm test -- --testPathPattern="echo.processor" --no-coverage` | ❌ W0 | ⬜ pending |
| 22-01-04 | 01 | 1 | WORK-04 | manual-only | `nest build --config worker-cli.json && test -f dist/worker.js` | N/A | ⬜ pending |
| 22-02-01 | 02 | 1 | D-01 | unit | `npm test -- --testPathPattern="queue.controller" --no-coverage` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/worker/test/processors/echo.processor.spec.ts` — stubs for WORK-03
- [ ] `src/queue/test/controllers/queue.controller.spec.ts` — stubs for D-01
- [ ] No framework install needed — Jest already configured

*Existing infrastructure covers framework requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Worker boots via createApplicationContext | WORK-01 | Bootstrap script, not unit testable | `nest build --config worker-cli.json && node dist/worker.js` — verify logs show "Worker service started" with `"service":"worker"`, then send SIGTERM |
| worker-cli.json produces dist/worker.js | WORK-04 | Build config output verification | `nest build --config worker-cli.json && test -f dist/worker.js && echo "OK"` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
