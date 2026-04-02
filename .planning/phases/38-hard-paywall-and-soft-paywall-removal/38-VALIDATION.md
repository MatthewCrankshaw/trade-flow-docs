---
phase: 38
slug: hard-paywall-and-soft-paywall-removal
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 38 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | TypeScript compiler + ESLint (no test runner — frontend uses static analysis only) |
| **Config file** | `trade-flow-ui/tsconfig.app.json`, `trade-flow-ui/eslint.config.js` |
| **Quick run command** | `cd trade-flow-ui && npx tsc --noEmit` |
| **Full suite command** | `cd trade-flow-ui && npx tsc --noEmit && npx eslint src/` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-ui && npx tsc --noEmit`
- **After every plan wave:** Run `cd trade-flow-ui && npx tsc --noEmit && npx eslint src/`
- **Before `/gsd:verify-work`:** Full suite must be green + manual verification of all 3 paywall variants
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 38-01-01 | 01 | 1 | PAYWALL-01 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |
| 38-01-02 | 01 | 1 | PAYWALL-02 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |
| 38-01-03 | 01 | 1 | PAYWALL-05 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |
| 38-02-01 | 02 | 2 | PAYWALL-06 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |
| 38-02-02 | 02 | 2 | PAYWALL-03 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |
| 38-02-03 | 02 | 2 | PAYWALL-04 | type-check | `cd trade-flow-ui && npx tsc --noEmit` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. TypeScript compilation serves as the primary automated verification — broken imports and missing types from removal work will surface as compilation errors.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Invalid subscription sees full-screen blocking page | PAYWALL-01 | Visual/behavioral — requires browser rendering | Set subscription status to canceled, load app, verify PaywallPage renders instead of dashboard |
| Differentiated messaging per subscription state | PAYWALL-02 | Copy verification — 3 distinct headline/body variants | Test with trial expired, payment failed, and canceled states; verify correct headline and body for each |
| Stripe Billing Portal accessible from paywall | PAYWALL-03 | External integration — requires Stripe portal redirect | Click "Manage Billing" on paywall page, verify Stripe Billing Portal opens in new tab |
| "Your data is safe" reassurance visible | PAYWALL-04 | Copy verification — inline in body text | Verify body copy includes data safety reassurance on all 3 variants |
| Support role bypasses paywall entirely | PAYWALL-05 | Role-based behavior — requires support user login | Login as user with supportRoles, set subscription to canceled, verify normal app renders |

*If none: "All phase behaviors have automated verification."*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
