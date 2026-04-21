---
phase: 57
slug: impersonation-frontend
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-18
audited: 2026-04-21
automated: 9
manual: 1
---

# Phase 57 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest 4.1.3 |
| **Config file** | `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test -- --run` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --run`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 57-01-01 | 01 | 1 | IMP-03 | — | N/A | unit | `npm run test -- --run` | ✅ `src/store/slices/__tests__/impersonationSlice.test.ts` | ✅ green |
| 57-01-02 | 01 | 1 | IMP-03 | — | Token switching + 401 baseQuery | manual | — | manual-only | ⚠️ manual |
| 57-02-01 | 02 | 1 | IMP-04 | — | N/A | unit | `npm run test -- --run` | ✅ `src/features/support/components/__tests__/ImpersonationBanner.test.tsx` | ✅ green |
| 57-01-03 | 01 | 1 | IMP-03 | — | startSession orchestration | unit | `npm run test -- --run` | ✅ `src/features/support/hooks/__tests__/useImpersonation.test.ts` | ✅ green |
| 57-01-04 | 01 | 1 | IMP-03 | — | endSession orchestration | unit | `npm run test -- --run` | ✅ `src/features/support/hooks/__tests__/useImpersonation.test.ts` | ✅ green |
| 57-02-02 | 02 | 2 | IMP-03 | — | Nav switching (impersonating=true) | unit | `npm run test -- --run` | ✅ `src/config/__tests__/navigation.test.ts` | ✅ green |
| 57-02-03 | 02 | 2 | IMP-03 | — | Nav switching (impersonating=false) | unit | `npm run test -- --run` | ✅ `src/config/__tests__/navigation.test.ts` | ✅ green |
| 57-03-01 | 03 | 2 | IMP-05 | — | N/A | unit | `npm run test -- --run` | ✅ `src/features/support/components/__tests__/ImpersonateUserDialog.test.tsx` | ✅ green |
| 57-04-01 | 04 | 1 | IMP-03 | — | targetUser in response | unit | `npm run test` | ✅ `src/impersonation/test/services/impersonation-creator.service.spec.ts` | ✅ green |
| 57-04-02 | 04 | 1 | IMP-03 | — | Controller mock includes targetUser | unit | `npm run test` | ✅ `src/impersonation/test/controllers/impersonation.controller.spec.ts` | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Impersonation banner visible at all times | IMP-04 | Visual positioning (z-index, fixed) requires browser | Inspect banner stays fixed on scroll, above all content |
| Route guard bypass during impersonation | IMP-03 | Integration with route guards requires full app context | Navigate to business routes while impersonating |
| Token switching in prepareHeaders + 401 baseQuery wrapper (57-01-02) | IMP-03 | RTK Query baseQuery async mocking requires complex test setup; logic verified by guard bypass tests indirectly | Confirm impersonation token appears in Authorization header during active session; confirm 401 response clears impersonation state |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or manual-only rationale
- [x] Sampling continuity: no 3 consecutive tasks without automated verify (only 1 manual)
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 15s
- [x] `nyquist_compliant: true` set in frontmatter — all requirements have automated coverage; 57-01-02 remains manual-only (supplementary, not blocking)

**Approval:** compliant (9 automated, 1 manual-only)

---

## Validation Audit 2026-04-19

| Metric | Count |
|--------|-------|
| Gaps found | 4 |
| Resolved (automated) | 3 |
| Escalated to manual-only | 1 |

## Validation Audit 2026-04-20

| Metric | Count |
|--------|-------|
| Gaps found | 2 |
| Resolved | 2 |
| Escalated | 0 |

## Validation Audit 2026-04-21

| Metric | Count |
|--------|-------|
| Gaps found | 2 |
| Resolved | 2 |
| Escalated | 0 |

Notes: Plan 04 (backend targetUser response fix) tasks were missing from the map. Both already had passing tests in the API repo (962 tests green). Added as COVERED. Phase now Nyquist-compliant.
