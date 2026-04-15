---
phase: 49
slug: revision-frontend-ui
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-15
---

# Phase 49 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest 4.1.3 |
| **Config file** | `vitest.config.ts` |
| **Quick run command** | `npm run test` |
| **Full suite command** | `npm run ci` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test`
- **After every plan wave:** Run `npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 49-01-01 | 01 | 1 | REV-02 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |
| 49-01-02 | 01 | 1 | REV-04 | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| "Edit and resend" navigates to draft revision | REV-02 | Navigation side-effect requires browser context | Click "Edit and resend" on a Sent estimate, verify redirect to new draft |
| History section renders revision list | REV-04 | Visual layout verification | Open estimate with revisions, expand History, verify entries |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
