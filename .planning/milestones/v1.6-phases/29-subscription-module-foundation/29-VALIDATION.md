---
phase: 29
slug: subscription-module-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 29 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest 30.2.0 with ts-jest 29.3.1 |
| **Config file** | jest config in package.json (trade-flow-api) |
| **Quick run command** | `npm test -- --testPathPattern=subscription` |
| **Full suite command** | `npm test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Manual verification via curl/Stripe CLI (success criteria are runtime checks)
- **After every plan wave:** Run `npm test` (existing tests must not break)
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 60 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 29-01-01 | 01 | 1 | ACQ-02 | unit (Stripe SDK mocked) | `npm test -- --testPathPattern=subscription-creator` | ❌ W0 (deferred to Phase 31 per TEST-01) | ⬜ pending |
| 29-02-01 | 02 | 1 | WBHK-01 | unit (constructEvent mocked) | `npm test -- --testPathPattern=webhook` | ❌ W0 (deferred to Phase 31 per TEST-02) | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- Tests for ACQ-02 and WBHK-01 are explicitly deferred to Phase 31 (TEST-01 through TEST-04)
- Phase 29 success criteria are verified via runtime behavior (curl against running API), not automated tests
- Existing test suite must pass after all changes (`npm test`)

*Note: Wave 0 is "existing infrastructure covers Phase 29 runtime validation" — automated unit tests added in Phase 31.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `POST /v1/subscription/checkout` returns Stripe Checkout Session URL | ACQ-02 | Stripe SDK live call; tests deferred to Phase 31 | Run API locally, POST with valid auth header, confirm response contains `url` field pointing to checkout.stripe.com |
| `POST /v1/webhooks/stripe` with invalid signature returns 400 | WBHK-01 | Requires rawBody + constructEvent flow; tests deferred to Phase 31 | Use Stripe CLI or curl with invalid/missing `stripe-signature` header, confirm 400 response |
| `SubscriptionStatus` enum values defined and `subscriptions` collection indexes exist | WBHK-01, ACQ-02 | MongoDB index verification | Check MongoDB: `db.subscriptions.getIndexes()` shows unique indexes on `userId` and `stripeSubscriptionId` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
