---
phase: 15
slug: quote-deletion
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-15
---

# Phase 15 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (API), Vitest (UI) |
| **Config file** | `trade-flow-api/package.json` (jest config) |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern=quote` |
| **Full suite command** | `cd trade-flow-api && npm run test` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run validate && cd ../trade-flow-ui && npm run lint && npm run typecheck`
- **After every plan wave:** Run `cd trade-flow-api && npm run test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 15-01-01 | 01 | 1 | QMGT-01 | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-transition` | ❌ W0 | ⬜ pending |
| 15-01-02 | 01 | 1 | QMGT-01 | unit | `cd trade-flow-api && npm run test -- --testPathPattern=quote-retriever` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts` — stubs for QMGT-01 (transition validation: DRAFT->DELETED allowed, non-DRAFT->DELETED rejected, deletedAt set)
- [ ] `trade-flow-api/src/quote/test/services/quote-retriever.service.spec.ts` — stubs for QMGT-01 (deleted quotes excluded from list)

*No test framework install needed — Jest already configured.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Destructive confirmation dialog with quote details | QMGT-01 | Visual/UX verification | Open draft quote detail → click Delete → verify dialog shows quote number + customer name, red button |
| Optimistic list row removal + toast | QMGT-01 | Visual/animation verification | Delete from quote list row → verify row disappears immediately, toast shows "Quote deleted" |
| Delete hidden for non-Draft quotes | QMGT-01 | UI state verification | Open Sent/Accepted/Rejected quote → verify no delete option in dropdown |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
