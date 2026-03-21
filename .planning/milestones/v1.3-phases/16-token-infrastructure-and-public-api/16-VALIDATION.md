---
phase: 16
slug: token-infrastructure-and-public-api
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-15
---

# Phase 16 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 29.x (API) |
| **Config file** | trade-flow-api/jest.config.ts |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern="quote-token"` |
| **Full suite command** | `cd trade-flow-api && npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern="quote-token"`
- **After every plan wave:** Run `cd trade-flow-api && npm run test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 16-01-01 | 01 | 1 | RESP-01 | unit | `npm run test -- --testPathPattern="quote-token"` | ❌ W0 | ⬜ pending |
| 16-01-02 | 01 | 1 | RESP-01 | unit | `npm run test -- --testPathPattern="quote-token"` | ❌ W0 | ⬜ pending |
| 16-02-01 | 02 | 1 | RESP-01 | unit | `npm run test -- --testPathPattern="public-quote"` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/quote-token/test/services/quote-token-generator.service.spec.ts` — stubs for token generation
- [ ] `src/quote-token/test/services/quote-token-validator.service.spec.ts` — stubs for token validation
- [ ] `src/quote/test/controllers/public-quote.controller.spec.ts` — stubs for public endpoint

*Existing jest infrastructure covers framework requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Rate limiting blocks excess requests | RESP-01 | Throttler behavior requires rapid sequential HTTP calls | Send 61 requests to `/v1/public/quote/:token` in 1 minute, verify 429 response |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
