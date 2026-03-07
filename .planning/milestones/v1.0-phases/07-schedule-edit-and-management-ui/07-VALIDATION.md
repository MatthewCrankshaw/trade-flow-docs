---
phase: 7
slug: schedule-edit-and-management-ui
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-07
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — no frontend test framework configured |
| **Config file** | none — typecheck/lint/build is established project pattern |
| **Quick run command** | `cd trade-flow-ui && npm run typecheck && npm run lint` |
| **Full suite command** | `cd trade-flow-ui && npm run typecheck && npm run lint && npm run build` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npm run typecheck && npm run lint`
- **After every plan wave:** Run `cd trade-flow-ui && npm run typecheck && npm run lint && npm run build`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 1 | SCHED-03 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |
| 07-01-02 | 01 | 1 | SCHED-03 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |
| 07-01-03 | 01 | 1 | SCHED-03 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |
| 07-02-01 | 02 | 2 | SCHED-04 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |
| 07-02-02 | 02 | 2 | SCHED-04 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |
| 07-02-03 | 02 | 2 | SCHED-04 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | pending |

*Status: pending · green · red · flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No test framework setup needed — typecheck, lint, and build validation is the established project pattern across all prior phases.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Edit form pre-fills with existing schedule data | SCHED-03 | No test runner; visual verification | Open scheduled entry > three-dot menu > Edit > verify date/time/duration/visit-type pre-filled |
| Confirmed status reset warning on time change | SCHED-03 | Dirty field detection UX | Edit confirmed schedule > change time > verify inline warning appears |
| Cancel confirmation dialog flow | SCHED-04 | AlertDialog interaction | Click Cancel Visit > verify dialog text > click Go Back (nothing happens) > click Cancel Visit (status changes) |
| Canceled entry stays in list with muted styling | SCHED-04 | Visual rendering | After canceling > verify entry remains in list with canceled badge |
| Terminal status hides three-dot menu | SCHED-04 | UI state verification | Open completed/canceled/no-show entry > verify no menu, locked message shown |
| Notes editable on terminal statuses | SCHED-04 | Interactive verification | Open canceled entry > verify notes textarea is editable, Save Notes works |
| Non-terminal transition (Confirm) is immediate | SCHED-04 | Interaction pattern | Click Confirm on scheduled entry > verify no dialog, status updates immediately |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
