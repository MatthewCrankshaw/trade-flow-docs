---
phase: 33
slug: trial-banner-and-billing-settings-tab
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 33 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | ESLint + TypeScript (frontend has no test runner) |
| **Config file** | `eslint.config.js`, `tsconfig.app.json` |
| **Quick run command** | `npm run lint && npm run typecheck` |
| **Full suite command** | `npm run lint && npm run typecheck && npm run build` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run lint && npm run typecheck`
- **After every plan wave:** Run `npm run lint && npm run typecheck && npm run build`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 33-01-01 | 01 | 1 | TRIAL-01, TRIAL-02 | typecheck | `npm run lint && npm run typecheck` | ✅ | ⬜ pending |
| 33-01-02 | 01 | 1 | TRIAL-01, TRIAL-02 | typecheck | `npm run lint && npm run typecheck` | ✅ | ⬜ pending |
| 33-02-01 | 02 | 2 | BILL-04, BILL-05 | typecheck | `npm run lint && npm run typecheck` | ✅ | ⬜ pending |
| 33-02-02 | 02 | 2 | BILL-04, BILL-05 | typecheck | `npm run lint && npm run typecheck` | ✅ | ⬜ pending |
| 33-02-03 | 02 | 2 | BILL-04, BILL-05 | typecheck | `npm run lint && npm run typecheck` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. All shadcn UI components (Tabs, Badge, Card, Button, Skeleton) are already installed. No new framework or test file stubs needed.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| TrialChip shows days remaining for trialing users | TRIAL-01 | No test runner — visual verification only | Seed a trialing user, load app, verify amber chip in header shows "X days left in trial" |
| TrialChip hidden when status is active | TRIAL-02 | No test runner — visual verification only | Use active subscription user, verify chip is absent from header |
| Billing tab shows subscription status and dates | BILL-04 | No test runner — visual verification only | Navigate to `/settings?tab=billing`, verify status card renders with correct status and dates |
| cancelAtPeriodEnd shows "Cancels on [date]" | BILL-05 | No test runner — visual verification only | Use a subscription with cancelAtPeriodEnd=true, verify billing tab shows "Cancels on [date]" |
| Manage Billing redirects to Stripe Portal | BILL-04 | External service redirect | Click "Manage Billing", verify redirect to Stripe Portal URL |
| Portal return lands on Billing tab | BILL-04 | URL routing — browser navigation | Return from Stripe Portal, verify `?tab=billing` pre-selects Billing tab |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
