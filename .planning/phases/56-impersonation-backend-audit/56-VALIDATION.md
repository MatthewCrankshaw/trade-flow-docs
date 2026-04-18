---
phase: 56
slug: impersonation-backend-audit
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 56 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.2.0 |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 56-01-01 | 01 | 1 | IMP-01 | — | N/A | unit | `npm run test -- --testPathPattern impersonation` | ❌ W0 | ⬜ pending |
| 56-01-02 | 01 | 1 | IMP-02 | — | 403 on support-to-support impersonation | unit | `npm run test -- --testPathPattern impersonation` | ❌ W0 | ⬜ pending |
| 56-01-03 | 01 | 1 | IMP-06 | — | Expired JWT returns 401 | unit | `npm run test -- --testPathPattern impersonation` | ❌ W0 | ⬜ pending |
| 56-02-01 | 02 | 1 | IAUD-01 | — | Audit entry created on session start | unit | `npm run test -- --testPathPattern impersonation-audit` | ❌ W0 | ⬜ pending |
| 56-02-02 | 02 | 1 | IAUD-02 | — | Repository exposes only create/find | unit | `npm run test -- --testPathPattern impersonation-audit` | ❌ W0 | ⬜ pending |
| 56-02-03 | 02 | 1 | IAUD-03 | — | Reason field validated min 10 chars | unit | `npm run test -- --testPathPattern impersonation` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/impersonation/test/` — test directory structure
- [ ] `src/impersonation/test/mocks/` — shared mock factories for impersonation module

*Existing Jest infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

*All phase behaviors have automated verification.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
