---
phase: 31
slug: subscription-api-endpoints-and-tests
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 31 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x |
| **Config file** | `jest.config.js` (trade-flow-api) |
| **Quick run command** | `npm run test -- --testPathPattern=subscription` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern=subscription`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 31-01-01 | 01 | 1 | BILL-01 | unit | `npm run test -- --testPathPattern=subscription-retriever` | ❌ W0 | ⬜ pending |
| 31-01-02 | 01 | 1 | BILL-02 | unit | `npm run test -- --testPathPattern=subscription-updater` | ❌ W0 | ⬜ pending |
| 31-01-03 | 01 | 1 | BILL-03 | unit | `npm run test -- --testPathPattern=subscription-creator` | ❌ W0 | ⬜ pending |
| 31-02-01 | 02 | 2 | BILL-01,BILL-02,BILL-03 | integration | `npm run test -- --testPathPattern=subscription.controller` | ❌ W0 | ⬜ pending |
| 31-02-02 | 02 | 2 | TEST-01 | unit | `npm run test -- --testPathPattern=subscription-creator.service.spec` | ❌ W0 | ⬜ pending |
| 31-02-03 | 02 | 2 | TEST-02 | unit | `npm run test -- --testPathPattern=subscription-updater.service.spec` | ❌ W0 | ⬜ pending |
| 31-02-04 | 02 | 2 | TEST-03 | unit | `npm run test -- --testPathPattern=subscription-retriever.service.spec` | ❌ W0 | ⬜ pending |
| 31-02-05 | 02 | 2 | TEST-04 | unit | `npm run test -- --testPathPattern=subscription.repository.spec` | ❌ W0 | ⬜ pending |
| 31-02-06 | 02 | 2 | BILL-01,BILL-02,BILL-03 | unit | `npm run test -- --testPathPattern=subscription.guard` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/subscription/test/services/subscription-retriever.service.spec.ts` — stubs for TEST-03
- [ ] `src/subscription/test/services/subscription-creator.service.spec.ts` — stubs for TEST-01
- [ ] `src/subscription/test/services/subscription-updater.service.spec.ts` — stubs for TEST-02
- [ ] `src/subscription/test/repositories/subscription.repository.spec.ts` — stubs for TEST-04
- [ ] `src/subscription/test/controllers/subscription.controller.spec.ts` — stubs for BILL-01/02/03 + webhook signature
- [ ] `src/subscription/test/guards/subscription.guard.spec.ts` — stubs for guard behavior

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stripe Billing Portal redirects correctly | BILL-03 | Requires live Stripe test environment | Create portal session via API, open URL in browser, verify return_url lands on /settings?tab=billing |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
