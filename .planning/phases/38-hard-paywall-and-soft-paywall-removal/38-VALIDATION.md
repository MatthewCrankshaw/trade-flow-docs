---
phase: 38
slug: hard-paywall-and-soft-paywall-removal
status: complete
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-02
validated: 2026-04-07
---

# Phase 38 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest 4.1.3 + @testing-library/react (unit/component tests) |
| **Config file** | `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `cd trade-flow-ui && npx vitest run` |
| **Full suite command** | `cd trade-flow-ui && npx vitest run` |
| **Estimated runtime** | ~2 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npx vitest run`
- **After every plan wave:** Run `cd trade-flow-ui && npx vitest run`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File | Status |
|---------|------|------|-------------|-----------|-------------------|------|--------|
| 38-01-01 | 01 | 1 | PAYWALL-01 | unit | `cd trade-flow-ui && npx vitest run src/features/auth/components/__tests__/PaywallGuard.test.tsx` | `src/features/auth/components/__tests__/PaywallGuard.test.tsx` | green |
| 38-01-02 | 01 | 1 | PAYWALL-02 | unit | `cd trade-flow-ui && npx vitest run src/pages/__tests__/PaywallPage.test.tsx` | `src/pages/__tests__/PaywallPage.test.tsx` | green |
| 38-01-03 | 01 | 1 | PAYWALL-05 | unit | `cd trade-flow-ui && npx vitest run src/features/auth/components/__tests__/PaywallGuard.test.tsx` | `src/features/auth/components/__tests__/PaywallGuard.test.tsx` | green |
| 38-02-01 | 02 | 2 | PAYWALL-06 | smoke | `cd trade-flow-ui && grep -rn "openPaywall\|PaywallModal\|PersistentCta\|DashboardBanner\|paywallSlice\|SubscriptionGatedLayout\|PricingCard\|SubscribePage" src/ --include="*.ts" --include="*.tsx"` | N/A (grep returns zero matches) | green |
| 38-02-02 | 02 | 2 | PAYWALL-03 | unit | `cd trade-flow-ui && npx vitest run src/pages/__tests__/PaywallPage.test.tsx` | `src/pages/__tests__/PaywallPage.test.tsx` | green |
| 38-02-03 | 02 | 2 | PAYWALL-04 | unit | `cd trade-flow-ui && npx vitest run src/pages/__tests__/PaywallPage.test.tsx` | `src/pages/__tests__/PaywallPage.test.tsx` | green |

*Status: green (all 6/6 requirements verified)*

---

## Nyquist Gap Resolution

All 6 requirements previously listed as manual-only have been covered by automated behavioral tests:

| Requirement | Gap Filled By |
|-------------|--------------|
| PAYWALL-01 | `PaywallGuard.test.tsx` — renders PaywallPage when isActive=false, renders Outlet when isActive=true |
| PAYWALL-02 | `PaywallPage.test.tsx` — verifies variant-specific headline/body for all 3 variants |
| PAYWALL-03 | `PaywallPage.test.tsx` — verifies Manage Billing button present on all 3 variants |
| PAYWALL-04 | `PaywallPage.test.tsx` — verifies data safety text in trial-expired and canceled body copy |
| PAYWALL-05 | `PaywallGuard.test.tsx` — renders Outlet when user.supportRoles is non-empty |
| PAYWALL-06 | grep command returns zero matches for all 8 soft paywall identifiers |

---

## Validation Sign-Off

- [x] All tasks have automated verify
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 15s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** nyquist-auditor — 2026-04-07
