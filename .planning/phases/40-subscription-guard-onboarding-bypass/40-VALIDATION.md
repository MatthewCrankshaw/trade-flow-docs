---
phase: 40
slug: subscription-guard-onboarding-bypass
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-07
---

# Phase 40 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x with ts-jest transformer |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern=subscription-guard` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern=subscription-guard`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 40-01-01 | 01 | 1 | TRIAL-01, ONBD-01 | — | SkipSubscriptionCheck decorator on PATCH /v1/user/me | unit | `npm run test -- --testPathPattern=subscription-guard` | ❌ W0 | ⬜ pending |
| 40-01-02 | 01 | 1 | ONBD-02, ONBD-03 | — | SkipSubscriptionCheck decorator on POST /v1/user/me/business | unit | `npm run test -- --testPathPattern=subscription-guard` | ❌ W0 | ⬜ pending |
| 40-01-03 | 01 | 1 | ONBD-04 | — | Other write endpoints still blocked without subscription | unit | `npm run test -- --testPathPattern=subscription-guard` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Test stubs for subscription guard bypass scenarios — if not already covered by existing guard tests

*If none: "Existing infrastructure covers all phase requirements."*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Full onboarding flow end-to-end | ONBD-01 through ONBD-04 | Requires real Firebase auth + browser | Sign up new user, complete wizard steps, verify no 403 errors |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
