---
phase: 05
slug: schedule-creation-ui
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-07
---

# Phase 05 — Validation Strategy

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
| 05-01-01 | 01 | 1 | SCHED-01 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 05-01-02 | 01 | 1 | VTYPE-01 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 05-01-03 | 01 | 1 | INTG-03 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `npx shadcn@latest add calendar` — install react-day-picker + date-fns + Calendar component
- [ ] Add "Schedule" to tagTypes in `src/services/api.ts`
- [ ] Add Schedule types to `src/types/api.types.ts` and re-export from `src/types/index.ts`
- [ ] Add schedule form schema to `src/lib/forms/schemas/` and re-export from index

*These are infrastructure tasks that must be completed before feature work.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Form dialog opens from action strip | SCHED-01 | UI interaction | Click Schedule button on job detail, verify dialog appears |
| Visit type color dots in dropdown | VTYPE-01 | Visual | Open create form, verify color swatches next to each visit type |
| Empty state card display | INTG-03 | Visual | Navigate to job with no schedules, verify empty state card with CTA |
| Past date visual hint | SCHED-01 | Visual | Select a past date in calendar, verify muted styling |
| Toast on successful creation | SCHED-01 | UI interaction | Submit valid form, verify success toast appears |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
