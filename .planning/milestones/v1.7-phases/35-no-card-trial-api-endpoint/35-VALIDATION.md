---
phase: 35
slug: no-card-trial-api-endpoint
status: complete
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-02
audited: 2026-04-07
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
| 35-01-01 | 01 | 1 | TRIAL-01 | unit | `npx jest --testPathPatterns=subscription-trial-creator` | ✅ | ✅ green (6 tests) |
| 35-01-02 | 01 | 1 | TRIAL-04 | unit | `npx jest --testPathPatterns=stripe-webhook.processor` | ✅ | ✅ green (14 tests, 3 for created handler) |
| 35-01-03 | 01 | 1 | TRIAL-01 | unit | `npx jest --testPathPatterns=subscription-trial-creator` | ✅ | ✅ green (controller wiring via service tests) |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [x] `src/subscription/test/services/subscription-trial-creator.service.spec.ts` — 6 tests for TRIAL-01 (created during execution)
- [x] Test fixtures for Stripe mock responses (mock generators in spec files)

*All Wave 0 requirements satisfied during phase execution.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stripe Dashboard webhook registration | TRIAL-04 | Dashboard config, not code | Register `customer.subscription.created` in Stripe webhook settings |
| 30-day trial auto-cancel | TRIAL-01 | Requires waiting for trial expiry | Verify `trial_settings.end_behavior.missing_payment_method: 'cancel'` in Stripe subscription object |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 15s (tests run in ~2s)
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved (2026-04-07 audit)

---

## Validation Audit 2026-04-07

| Metric | Count |
|--------|-------|
| Gaps found | 0 |
| Resolved | 0 |
| Escalated | 0 |

All 3 task verification entries confirmed COVERED. Trial creator tests (6) and webhook processor tests (14) all pass green. No new tests needed.
