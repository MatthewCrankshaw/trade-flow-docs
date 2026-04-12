---
phase: 45
slug: public-customer-page-response-handling
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-12
---

# Phase 45 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest 30.2.0 (API) / Vitest 4.1.3 (UI) |
| **Config file** | `trade-flow-api/jest.config.ts` / `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern={pattern}` |
| **Full suite command** | `npm run ci` (both repos) |
| **Estimated runtime** | ~30 seconds (API) / ~15 seconds (UI) |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern={changed-module}`
- **After every plan wave:** Run `npm run ci` in both repos
- **Before `/gsd-verify-work`:** Full suite must be green in both repos
- **Max feedback latency:** 45 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 45-01-01 | 01 | 1 | RESP-01..04 | — | N/A | docs | manual review | N/A | ⬜ pending |
| 45-02-01 | 02 | 1 | CUST-01 | T-45-01 | Token auth required for all public endpoints | unit | `npm run test -- --testPathPattern=public-estimate` | ❌ W0 | ⬜ pending |
| 45-02-02 | 02 | 1 | CUST-02 | — | N/A | unit | `npm run test -- --testPathPattern=public-estimate-retriever` | ❌ W0 | ⬜ pending |
| 45-02-03 | 02 | 1 | CUST-03 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-response-handler` | ❌ W0 | ⬜ pending |
| 45-03-01 | 03 | 2 | CUST-04, CUST-05 | — | N/A | component | `npm run test -- --testPathPattern=PublicEstimatePage` | ❌ W0 | ⬜ pending |
| 45-03-02 | 03 | 2 | RESP-05..07 | — | N/A | component | `npm run test -- --testPathPattern=EstimateResponseButtons` | ❌ W0 | ⬜ pending |
| 45-04-01 | 04 | 2 | CUST-06 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-notification` | ❌ W0 | ⬜ pending |
| 45-04-02 | 04 | 2 | CUST-07 | — | PECR compliance — zero non-essential cookies | manual | Browser DevTools audit | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/estimate/test/controllers/public-estimate.controller.spec.ts` — stubs for CUST-01, CUST-02
- [ ] `trade-flow-api/src/estimate/test/services/public-estimate-retriever.service.spec.ts` — stubs for CUST-02, CUST-03
- [ ] `trade-flow-api/src/estimate/test/services/estimate-response-handler.service.spec.ts` — stubs for RESP-01..04
- [ ] `trade-flow-ui/src/features/estimates/components/__tests__/PublicEstimatePage.test.tsx` — stubs for CUST-04, CUST-05
- [ ] `trade-flow-ui/src/features/estimates/components/__tests__/EstimateResponseButtons.test.tsx` — stubs for RESP-05..07

*Existing infrastructure covers framework installation — no Wave 0 framework setup needed.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| PECR compliance — zero non-essential cookies | CUST-07 | Requires browser DevTools cookie audit | Open `/estimate/:token` in incognito, check Application > Cookies — must be empty or session-only |
| Terminal state read-only view | RESP-06 | Visual verification of disabled buttons | Submit a response, revisit the token URL — buttons must be replaced with confirmation text |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 45s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
