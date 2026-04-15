---
phase: 48
slug: di-token-fix-cleanup
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-15
---

# Phase 48 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API), vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern={pattern}` |
| **Full suite command** | `npm run ci` (both repos) |
| **Estimated runtime** | ~30 seconds per repo |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern={relevant}` in affected repo
- **After every plan wave:** Run `npm run ci` in both repos
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 48-01-01 | 01 | 1 | FUP-05 | — | DI resolves real BullMQ canceller | unit | `npm run test -- --testPathPattern=estimate-reviser` | ✅ | ⬜ pending |
| 48-01-02 | 01 | 1 | — | — | Noop registered once only | unit | `npm run test -- --testPathPattern=estimate` | ✅ | ⬜ pending |
| 48-01-03 | 01 | 1 | — | — | ConvertEstimateRequest removed | typecheck | `npm run typecheck` | ✅ | ⬜ pending |
| 48-02-01 | 02 | 1 | — | — | site_visit_requested removed from types | typecheck | `npm run typecheck` | ✅ | ⬜ pending |
| 48-02-02 | 02 | 1 | — | — | No dead branches in components | unit | `npm run test` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

*All phase behaviors have automated verification.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
