---
phase: 18
slug: quote-email-sending
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-21
---

# Phase 18 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 29.x (API), vitest (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `cd trade-flow-api && npx jest --testPathPattern quote-email --passWithNoTests` |
| **Full suite command** | `cd trade-flow-api && npx jest --passWithNoTests && cd ../trade-flow-ui && npx vitest run` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npx jest --testPathPattern quote-email --passWithNoTests`
- **After every plan wave:** Run full suite command
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 18-01-01 | 01 | 1 | DLVR-02 | unit | `npx jest --testPathPattern business` | ✅ | ⬜ pending |
| 18-01-02 | 01 | 1 | DLVR-02 | unit | `npx jest --testPathPattern business` | ✅ | ⬜ pending |
| 18-02-01 | 02 | 1 | DLVR-01, AUTO-01 | unit | `npx jest --testPathPattern quote-email` | ❌ W0 | ⬜ pending |
| 18-02-02 | 02 | 1 | DLVR-01 | unit | `npx jest --testPathPattern quote-email` | ❌ W0 | ⬜ pending |
| 18-03-01 | 03 | 2 | DLVR-03 | component | `npx vitest run --reporter=verbose` | ❌ W0 | ⬜ pending |
| 18-03-02 | 03 | 2 | DLVR-03, DLVR-04 | component | `npx vitest run --reporter=verbose` | ❌ W0 | ⬜ pending |
| 18-04-01 | 04 | 2 | DLVR-02 | component | `npx vitest run --reporter=verbose` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/quote-email/` — module directory for email sending service and tests
- [ ] `trade-flow-api/templates/` — Maizzle template directory with build config

*Existing infrastructure covers test frameworks and email sender service.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Email received in customer inbox | DLVR-01 | Requires real SendGrid delivery | Send quote in dev, check inbox |
| Rich text renders in email clients | DLVR-01 | Email client rendering varies | Send test email, view in Gmail/Outlook |
| Maizzle HTML renders correctly | DLVR-01 | Visual verification needed | Preview rendered template in browser |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
