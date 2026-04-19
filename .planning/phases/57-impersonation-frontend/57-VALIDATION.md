---
phase: 57
slug: impersonation-frontend
status: validated
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
audited: 2026-04-19
automated: 3
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
| 57-03-01 | 03 | 2 | IMP-05 | — | N/A | unit | `npm run test -- --run` | ✅ `src/features/support/components/__tests__/ImpersonateUserDialog.test.tsx` | ✅ green |

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
- [ ] `nyquist_compliant: true` set in frontmatter — blocked by 57-01-02 manual-only

**Approval:** partial (3 automated, 1 manual-only)

---

## Validation Audit 2026-04-19

| Metric | Count |
|--------|-------|
| Gaps found | 4 |
| Resolved (automated) | 3 |
| Escalated to manual-only | 1 |
