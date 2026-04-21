---
phase: 52
slug: permission-guard-migration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 52 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.2.0 / ts-jest |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern {relevant-test}`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 52-01-01 | 01 | 1 | RBAC-08 | — | Guard rejects missing permission with 403 | unit | `npm run test -- --testPathPattern permission-guard` | ❌ W0 | ⬜ pending |
| 52-01-02 | 01 | 1 | RBAC-08 | — | Guard allows valid permission | unit | `npm run test -- --testPathPattern permission-guard` | ❌ W0 | ⬜ pending |
| 52-02-01 | 02 | 2 | RBAC-09 | — | hasPermission returns correct boolean | unit | `npm run test -- --testPathPattern has-permission` | ❌ W0 | ⬜ pending |
| 52-03-01 | 03 | 2 | RBAC-10 | — | SubscriptionGuard checks bypass_subscription permission | unit | `npm run test -- --testPathPattern subscription.guard` | ✅ | ⬜ pending |
| 52-03-02 | 03 | 2 | RBAC-10 | — | Policies use hasPermission instead of isSupportUser | unit | `npm run test -- --testPathPattern policy` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/auth/test/guards/permission-guard.spec.ts` — stubs for RBAC-08
- [ ] `src/user/test/utilities/has-permission.utility.spec.ts` — stubs for RBAC-09

*Existing test infrastructure covers policy and guard specs.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Solo business users see no role UI | RBAC-10 SC3 | Frontend-only check | Verify no role management components render for non-support users |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
