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
| **Framework** | Jest (configured in trade-flow-api) |
| **Config file** | `trade-flow-api/jest.config.ts` or `package.json` jest config |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern="quote-response\|notification-email" --no-coverage` |
| **Full suite command** | `cd trade-flow-api && npm run test && cd ../trade-flow-ui && npm run typecheck && npm run lint` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern="quote-response|notification-email" --no-coverage`
- **After every plan wave:** Run `cd trade-flow-api && npm run test && cd ../trade-flow-ui && npm run typecheck && npm run lint`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 19-01-01 | 01 | 1 | RESP-02 | unit | `npm run test -- --testPathPattern="quote-response-handler" --no-coverage` | ❌ W0 | ⬜ pending |
| 19-01-02 | 01 | 1 | RESP-03 | unit | `npm run test -- --testPathPattern="quote-response-handler" --no-coverage` | ❌ W0 | ⬜ pending |
| 19-01-03 | 01 | 1 | RESP-04 | unit | `npm run test -- --testPathPattern="quote-response-handler" --no-coverage` | ❌ W0 | ⬜ pending |
| 19-01-04 | 01 | 1 | AUTO-02 | unit | `npm run test -- --testPathPattern="quote-transition" --no-coverage` | ✅ existing | ⬜ pending |
| 19-01-05 | 01 | 1 | AUTO-03 | unit | `npm run test -- --testPathPattern="quote-transition" --no-coverage` | ✅ existing | ⬜ pending |
| 19-02-01 | 02 | 1 | NOTF-01 | unit | `npm run test -- --testPathPattern="quote-response-handler\|notification-email" --no-coverage` | ❌ W0 | ⬜ pending |
| 19-02-02 | 02 | 1 | NOTF-02 | unit | `npm run test -- --testPathPattern="quote-response-handler\|notification-email" --no-coverage` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/quote-token/test/services/quote-response-handler.service.spec.ts` — stubs for RESP-02, RESP-03, RESP-04, NOTF-01, NOTF-02
- [ ] `trade-flow-api/src/email/test/services/notification-email-renderer.service.spec.ts` — covers notification template rendering
- [ ] `trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts` — may need update for new `publicTransition` method (existing file likely exists)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Accept/Decline buttons render correctly at bottom of quote card | RESP-02, RESP-03 | Visual layout + mobile touch target verification | Open public quote URL, verify buttons below totals section, test on mobile viewport |
| Expired quote shows banner instead of buttons | D-08 | Visual state rendering | Set `validUntil` to past date, open public URL, verify no response buttons shown |
| Inline status banner appears after response | D-03 | Visual transition state | Accept/decline a quote, verify banner replaces buttons, quote details remain visible |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
