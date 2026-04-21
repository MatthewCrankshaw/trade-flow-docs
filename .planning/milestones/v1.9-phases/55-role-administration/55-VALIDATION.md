---
phase: 55
slug: role-administration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 55 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API), vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test` |
| **Full suite command** | `npm run ci` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test`
- **After every plan wave:** Run `npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 55-01-01 | 01 | 1 | RADM-01 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 55-01-02 | 01 | 1 | RADM-02 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 55-01-03 | 01 | 1 | RADM-03 | T-55-01 | Self-revocation rejected with ForbiddenError | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 55-01-04 | 01 | 1 | RADM-03 | T-55-02 | Last-admin revocation rejected with ForbiddenError | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 55-02-01 | 02 | 2 | RADM-04 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 55-02-02 | 02 | 2 | RADM-05 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] API test stubs for SupportRoleAssigner grant/revoke — `trade-flow-api/src/user/test/services/support-role-assigner.service.spec.ts`
- [ ] API test stubs for SupportRoleAdminController — `trade-flow-api/src/user/test/controllers/support-role-admin.controller.spec.ts`
- [ ] UI test stubs for RoleActions/GrantRoleDialog/RevokeRoleDialog — `trade-flow-ui/src/features/support/__tests__/`

*Existing test infrastructure covers framework setup.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Confirmation dialog appears before grant/revoke | RADM-05 | Visual interaction flow | 1. Navigate to user detail 2. Click Grant/Revoke 3. Verify dialog appears with user name and consequence text |
| Role badge updates immediately after grant/revoke | RADM-04 | Requires real API + cache invalidation | 1. Grant role 2. Verify badge appears without page refresh |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
