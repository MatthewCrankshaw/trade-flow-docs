---
phase: 35
slug: no-card-trial-api-endpoint
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 35 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x with ts-jest |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `npm test -- --testPathPattern=subscription` |
| **Full suite command** | `npm test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- --testPathPattern=subscription`
- **After every plan wave:** Run `npm test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 35-01-01 | 01 | 1 | TRIAL-01 | unit | `npm test -- --testPathPattern=trial-creator` | ❌ W0 | ⬜ pending |
| 35-01-02 | 01 | 1 | TRIAL-04 | unit | `npm test -- --testPathPattern=webhook-processor` | ✅ | ⬜ pending |
| 35-01-03 | 01 | 1 | TRIAL-01 | unit | `npm test -- --testPathPattern=subscription.controller` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/subscription/test/services/trial-creator.service.spec.ts` — stubs for TRIAL-01
- [ ] Test fixtures for Stripe mock responses (trial subscription object)

*Existing test infrastructure (jest, ts-jest, supertest) covers framework needs.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stripe Dashboard webhook registration | TRIAL-04 | Dashboard config, not code | Register `customer.subscription.created` in Stripe webhook settings |
| 30-day trial auto-cancel | TRIAL-01 | Requires waiting for trial expiry | Verify `trial_settings.end_behavior.missing_payment_method: 'cancel'` in Stripe subscription object |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
