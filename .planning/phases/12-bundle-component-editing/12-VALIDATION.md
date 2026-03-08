---
phase: 12
slug: bundle-component-editing
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-08
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (API) — no UI test framework |
| **Config file** | `trade-flow-api/jest.config.ts` |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern="item" --no-coverage` |
| **Full suite command** | `cd trade-flow-api && npm run test` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern="item" --no-coverage`
- **After every plan wave:** Run `cd trade-flow-api && npm run test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| TBD | 01 | 1 | BNDL-02 | unit | `cd trade-flow-api && npx jest --testPathPattern="merge-existing-item" --no-coverage` | ✅ (needs new cases) | ⬜ pending |
| TBD | 01 | 1 | BNDL-02 | unit | `cd trade-flow-api && npx jest --testPathPattern="update-item.request" --no-coverage` | ❌ W0 | ⬜ pending |
| TBD | 01 | 1 | BNDL-02 | unit | `cd trade-flow-api && npx jest --testPathPattern="item-updater" --no-coverage` | ✅ (needs new cases) | ⬜ pending |
| TBD | 02 | 1 | BNDL-04 | manual | Visual verification in browser | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts` — add test cases for component merging
- [ ] `trade-flow-api/src/item/test/services/item-updater.service.spec.ts` — add test cases for bundle component validation

*Existing infrastructure covers most requirements. Only new test cases needed in existing files.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Component display shows name, type badge, qty x unit | BNDL-04 | UI component, no test framework configured | Open items list, expand a bundle row, verify two-line format with type badges |
| Add/remove components in edit form | BNDL-02 | UI interaction, no test framework | Edit a bundle, add/remove components, verify save persists |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
