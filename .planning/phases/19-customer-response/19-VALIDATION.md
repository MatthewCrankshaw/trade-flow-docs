---
phase: 19
slug: customer-response
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-21
---

# Phase 19 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 29.x |
| **Config file** | trade-flow-api/jest.config.ts |
| **Quick run command** | `cd trade-flow-api && npx jest --testPathPattern="quote-response\|notification" --no-coverage` |
| **Full suite command** | `cd trade-flow-api && npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run quick run command
- **After every plan wave:** Run full suite command
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 19-01-01 | 01 | 1 | RESP-02, RESP-03, AUTO-02, AUTO-03 | unit | `npx jest quote-response` | ❌ W0 | ⬜ pending |
| 19-01-02 | 01 | 1 | RESP-04 | unit | `npx jest quote-response` | ❌ W0 | ⬜ pending |
| 19-02-01 | 02 | 1 | NOTF-01, NOTF-02 | unit | `npx jest notification-email` | ❌ W0 | ⬜ pending |
| 19-03-01 | 03 | 2 | RESP-02, RESP-03 | manual | Browser test | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/quote/test/services/quote-response.service.spec.ts` — stubs for RESP-02, RESP-03, AUTO-02, AUTO-03
- [ ] `trade-flow-api/src/email/test/services/notification-email-renderer.service.spec.ts` — stubs for NOTF-01, NOTF-02

*Existing test infrastructure covers framework needs.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Accept button click and inline confirmation | RESP-02 | Browser interaction | Open public quote URL, click Accept, verify banner appears |
| Decline with optional reason | RESP-03, RESP-04 | Browser interaction | Click Decline, enter reason, submit, verify banner |
| Expired quote shows expired message | RESP-02, RESP-03 | Requires expired quote state | Create quote with past validUntil, verify no buttons shown |
| Already-responded quote shows status | RESP-02 | Requires responded state | Revisit quote URL after responding, verify buttons replaced |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
