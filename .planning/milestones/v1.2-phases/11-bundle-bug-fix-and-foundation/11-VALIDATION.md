---
phase: 11
slug: bundle-bug-fix-and-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-08
---

# Phase 11 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest (via Vite) |
| **Config file** | `trade-flow-ui/vitest.config.ts` or Vite config |
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
| 11-01-01 | 01 | 1 | BNDL-01 | typecheck | `cd trade-flow-ui && npm run typecheck` | N/A | ⬜ pending |
| 11-02-01 | 02 | 1 | BNDL-03 | typecheck+lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No unit test framework is actively used in this project; verification is via typecheck + lint + manual testing.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Bundle creation succeeds without errors | BNDL-01 | Requires running app + API to verify end-to-end | 1. Open Items page 2. Click "New Item" 3. Select type "Bundle" 4. Fill name and components 5. Submit — should succeed without validation errors |
| SearchableItemPicker filters and groups items | BNDL-03 | Requires visual verification of dropdown UX | 1. Create/edit a bundle 2. Click "Add component" 3. Verify items grouped by type 4. Type search term — verify filtering 5. Select item — verify auto-close and row added |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
