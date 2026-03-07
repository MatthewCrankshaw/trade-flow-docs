---
phase: 8
slug: job-detail-integration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-07
---

# Phase 8 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None (TypeScript + ESLint serve as automated validation) |
| **Config file** | trade-flow-ui/tsconfig.json, trade-flow-ui/eslint.config.js |
| **Quick run command** | `cd trade-flow-ui && npm run typecheck` |
| **Full suite command** | `cd trade-flow-ui && npm run lint && npm run typecheck` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npm run typecheck`
- **After every plan wave:** Run `cd trade-flow-ui && npm run lint && npm run typecheck`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 08-01-01 | 01 | 1 | INTG-02 | typecheck + manual | `cd trade-flow-ui && npm run typecheck` | N/A | pending |
| 08-01-02 | 01 | 1 | INTG-01 | typecheck + manual | `cd trade-flow-ui && npm run typecheck` | N/A | pending |
| 08-01-03 | 01 | 1 | INTG-01, INTG-02 | typecheck + lint | `cd trade-flow-ui && npm run lint && npm run typecheck` | N/A | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No test framework to install — TypeScript type checking and ESLint are the automated validation layer.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Per-status badge breakdown displays correctly | INTG-02 | Visual UI component — no test framework | Navigate to job detail, verify color-coded badges match STATUS_CONFIG colors, only non-zero statuses shown |
| Next visit hint shows correct data | INTG-01 | Requires real schedule state + job status combinations | Test with active job (upcoming schedule), completed job (last visit), on-hold job, job with no schedules |
| Schedule CRUD works end-to-end from job detail | INTG-01 | E2E user flow across multiple components | Create, edit, cancel schedule from job detail; verify summary updates |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
