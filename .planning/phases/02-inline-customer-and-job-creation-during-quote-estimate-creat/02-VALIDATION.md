---
phase: 2
slug: inline-customer-and-job-creation-during-quote-estimate-creat
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-22
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API), vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test` (in respective repo) |
| **Full suite command** | `npm run ci` (in respective repo) |
| **Estimated runtime** | ~30 seconds per repo |

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
| TBD | TBD | TBD | TBD | — | N/A | unit | `npm run test` | ⬜ pending | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

*Note: Task IDs will be populated after PLAN.md files are created by the planner.*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. Both repos have established test frameworks with existing test patterns for controllers, services, and components.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Stacked dialog visual layering | D-07 | Visual z-index stacking requires browser | Open quote dialog → New Job → New Customer, verify each layer visually stacks |
| Mobile fullscreen dialog behavior | Claude's Discretion | Responsive layout requires viewport | Resize to mobile width, verify dialogs go fullscreen |
| Auto-select UX after inline creation | D-03, D-12 | Interaction timing and focus | Create inline entity, verify parent selector updates and shows new entity |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
