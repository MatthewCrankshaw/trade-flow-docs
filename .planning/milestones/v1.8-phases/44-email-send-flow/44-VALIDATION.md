---
phase: 44
slug: email-send-flow
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-12
---

# Phase 44 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API) / vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts` / `trade-flow-ui/vite.config.ts` |
| **Quick run command** | `npm run test -- --changedSince=HEAD~1` |
| **Full suite command** | `npm run ci` |
| **Estimated runtime** | ~30 seconds per repo |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --changedSince=HEAD~1`
- **After every plan wave:** Run `npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | SND-01 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-02 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-03 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-04 | — | N/A | N/A | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-05 | T-44-01 | Non-binding copy cannot be removed | unit | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-06 | — | Audit HTML persisted at send time | unit | `npm run test` | ❌ W0 | ⬜ pending |
| TBD | TBD | TBD | SND-07 | — | Re-send no new revision/token | unit | `npm run test` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/estimate-settings/test/` — test stubs for estimate-settings module
- [ ] `src/estimate/test/` — test stubs for send/re-send flows
- [ ] UI test stubs for SendEstimateDialog and Templates tab

*Existing test infrastructure covers framework setup — only test files need creation.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Email arrives in customer inbox | SND-03 | Requires live email delivery | Send test estimate, check inbox |
| Rich-text editor renders correctly | SND-02 | Visual verification | Open SendEstimateDialog, check formatting tools |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
