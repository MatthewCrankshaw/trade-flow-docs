---
phase: 51
slug: rbac-data-model-seed
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 51 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 51-01-01 | 01 | 1 | RBAC-01 | — | N/A | unit | `npm run test -- --testPathPattern permission` | ❌ W0 | ⬜ pending |
| 51-01-02 | 01 | 1 | RBAC-02 | — | N/A | unit | `npm run test -- --testPathPattern permission` | ❌ W0 | ⬜ pending |
| 51-02-01 | 02 | 1 | RBAC-03 | — | N/A | unit | `npm run test -- --testPathPattern role` | ❌ W0 | ⬜ pending |
| 51-02-02 | 02 | 1 | RBAC-04 | — | N/A | unit | `npm run test -- --testPathPattern role` | ❌ W0 | ⬜ pending |
| 51-03-01 | 03 | 2 | RBAC-05 | — | N/A | unit | `npm run test -- --testPathPattern seed` | ❌ W0 | ⬜ pending |
| 51-03-02 | 03 | 2 | RBAC-06 | — | N/A | unit | `npm run test -- --testPathPattern seed` | ❌ W0 | ⬜ pending |
| 51-04-01 | 04 | 2 | RBAC-07 | — | N/A | unit | `npm run test -- --testPathPattern business-creator` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Test stubs for permission entity/repository/DTO
- [ ] Test stubs for role permission hydration
- [ ] Test stubs for seeder service
- [ ] Shared mock factories for permissions and roles

*Existing test infrastructure covers framework setup.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Super user env var lockout prevention | RBAC-06 | Requires app restart with env var | Set SUPER_USER_EMAIL, restart app, verify super user role assigned |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
