---
phase: 30
slug: stripe-checkout-and-webhooks
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 30 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x |
| **Config file** | `jest.config.js` (trade-flow-api root) |
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
| 30-01-01 | 01 | 1 | WBHK-02 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-01-02 | 01 | 1 | WBHK-03 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-01-03 | 01 | 1 | WBHK-04 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-01-04 | 01 | 1 | WBHK-05 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-01-05 | 01 | 1 | WBHK-06 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-01-06 | 01 | 1 | WBHK-07 | unit | `npm run test -- --testPathPattern=stripe-webhook.processor` | ❌ W0 | ⬜ pending |
| 30-02-01 | 02 | 1 | ACQ-01 | unit | `npm run test -- --testPathPattern=subscription-creator` | ❌ W0 | ⬜ pending |
| 30-02-02 | 02 | 1 | ACQ-05 | unit | `npm run test -- --testPathPattern=subscription-retriever` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/subscription/test/processors/stripe-webhook.processor.spec.ts` — stubs for WBHK-02 through WBHK-07 (5 event handlers + idempotency)
- [ ] `src/subscription/test/services/subscription-creator.service.spec.ts` — stub for ACQ-01 duplicate checkout guard
- [ ] `src/subscription/test/services/subscription-retriever.service.spec.ts` — stub for ACQ-05 verify-session endpoint

*Existing Jest infrastructure covers all other phase requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stripe CLI webhook replay produces no duplicate DB records | WBHK-07 | Requires live Stripe CLI + running server | Run `stripe trigger checkout.session.completed` twice; confirm single subscription doc in MongoDB |
| `/subscribe/success` page confirms subscription after Checkout redirect | ACQ-03, ACQ-05 | Requires live Stripe Checkout flow | Complete Checkout in test mode; verify success page shows active/trialing status |
| BullMQ job marked failed when no local record exists for event | D-11 | Requires BullMQ dashboard or queue inspection | Send `customer.subscription.updated` before checkout.session.completed; verify job moves to failed queue |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
