---
phase: 53
slug: support-access-routing
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 53 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest 4.1.3 |
| **Config file** | `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 53-01-01 | 01 | 1 | SACC-03 | — | SupportGuard redirects non-support users | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 53-01-02 | 01 | 1 | SACC-01, SACC-02 | — | LoginPage role-aware redirect | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 53-02-01 | 02 | 1 | SACC-03 | — | Route tree support branch isolation | integration | `npm run test` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/features/auth/components/__tests__/SupportGuard.test.tsx` — stubs for SACC-03
- [ ] `src/pages/__tests__/LoginPage.test.tsx` — stubs for SACC-01, SACC-02

*Existing infrastructure covers test framework requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Support user sees support nav items only | SACC-03 | Visual/layout verification | Log in as support user, verify sidebar shows Dashboard/Users/Businesses only |
| Support user bypasses subscription gating | SACC-04 | Requires RBAC seeded data from Phase 51 | Log in as support user with no subscription, verify no paywall |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
