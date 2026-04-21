---
phase: 1
slug: job-detail-cleanup-remove-mocked-data-and-wire-real-data
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-21
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API) / vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts` / `trade-flow-ui/vite.config.ts` |
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
| TBD | TBD | TBD | TBD | — | N/A | unit | `npm run test` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Job detail page shows real data | D-01 | Visual verification | Navigate to job detail, confirm customer/job type display |
| Timeline displays events | D-05 | Visual verification | Create a quote, check timeline updates |
| Removed tabs not visible | D-09, D-10 | Visual verification | Confirm Invoices/Notes tabs are gone |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
