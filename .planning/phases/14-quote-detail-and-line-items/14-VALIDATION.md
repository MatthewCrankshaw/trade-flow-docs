---
phase: 14
slug: quote-detail-and-line-items
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-14
---

# Phase 14 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vite build + ESLint + TypeScript strict |
| **Config file** | `trade-flow-ui/vite.config.ts`, `trade-flow-ui/tsconfig.json` |
| **Quick run command** | `cd trade-flow-ui && npm run typecheck` |
| **Full suite command** | `cd trade-flow-ui && npm run lint && npm run typecheck` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npm run typecheck`
- **After every plan wave:** Run `cd trade-flow-ui && npm run lint && npm run typecheck` + `cd trade-flow-api && npm run validate`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 14-01-01 | 01 | 1 | QUOT-03 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 14-01-02 | 01 | 1 | QLIT-01 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 14-02-01 | 02 | 1 | QLIT-02 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 14-02-02 | 02 | 1 | QLIT-03 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |
| 14-03-01 | 03 | 2 | QLIT-04 | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.* No unit test framework is configured for the UI project (consistent with prior phases). Lint and typecheck provide automated validation.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Quote detail page shows header info and line items | QUOT-03 | Visual layout verification | Navigate to quote detail, verify header fields and line item table render |
| Bundle line item shows as single rolled-up row | QLIT-02 | Visual component behavior | Add a bundle item, verify it appears as one row with bundle indicator |
| Expand bundle reveals component breakdown | QLIT-03 | Interactive UI behavior | Click expand on bundle row, verify child components display |
| Quote totals calculate correctly with bundles | QLIT-04 | End-to-end calculation | Add mix of standard and bundle items, verify subtotal/tax/total match expected values |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
