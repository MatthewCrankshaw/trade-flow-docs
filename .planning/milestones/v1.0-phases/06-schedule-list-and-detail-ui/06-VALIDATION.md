---
phase: 06
slug: schedule-list-and-detail-ui
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-07
---

# Phase 06 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vite + TypeScript (typecheck + lint, no test runner) |
| **Config file** | tsconfig.json, eslint.config.js |
| **Quick run command** | `cd trade-flow-ui && npm run typecheck` |
| **Full suite command** | `cd trade-flow-ui && npm run typecheck && npm run lint` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npm run typecheck`
- **After every plan wave:** Run `cd trade-flow-ui && npm run typecheck && npm run lint`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 06-01-01 | 01 | 1 | SCHED-02 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 06-01-02 | 01 | 1 | SCHED-02, SCHED-05 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 06-01-03 | 01 | 1 | SCHED-05 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Add `useUpdateScheduleMutation` to `scheduleApi.ts` for notes PATCH endpoint
- [ ] Add update mutation to `useScheduleActions` hook

*Existing infrastructure covers remaining requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Schedule list shows real data in tab | SCHED-02 | UI interaction | Navigate to job with schedules, verify entries appear in Schedule tab |
| Upcoming/Past section headers | SCHED-02 | Visual | Verify section labels and correct ordering |
| Visit type color dots in list | SCHED-02 | Visual | Verify color dots match visit type colors |
| Status badge colors | SCHED-02 | Visual | Verify status-specific badge colors |
| Detail dialog opens on click | SCHED-02 | UI interaction | Click a schedule entry, verify dialog opens with details |
| Notes editable in dialog | SCHED-05 | UI interaction | Open detail dialog, edit notes, save, verify persistence |
| Notes indicator in list | SCHED-05 | Visual | Verify icon appears on entries with notes |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
