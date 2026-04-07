---
phase: 37
slug: onboarding-wizard-pages
status: validated
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-07
---

# Phase 37 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework (API)** | Jest 30.x |
| **Config file (API)** | `trade-flow-api/package.json` jest section |
| **Quick run command (API)** | `cd trade-flow-api && npx jest --passWithNoTests` |
| **Full suite command (API)** | `cd trade-flow-api && npm run test` |
| **Framework (UI)** | None (Playwright E2E only) |
| **Config file (UI)** | `trade-flow-ui/e2e/` (Playwright) |
| **Quick run command (UI)** | N/A — no unit test framework |
| **Full suite command (UI)** | `cd trade-flow-ui && npx playwright test` |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npx jest --passWithNoTests`
- **After every plan wave:** Run `cd trade-flow-api && npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** ~5 seconds (API unit tests)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 37-01-01 | 01 | 1 | ONBD-05, ONBD-06, ONBD-07 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-01-02 | 01 | 1 | ONBD-01, ONBD-07 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-02-01 | 02 | 2 | ONBD-02 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-02-02 | 02 | 2 | ONBD-03 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-03-01 | 03 | 1 | TRIAL-02, TRIAL-03 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-03-02 | 03 | 1 | TRIAL-02 | unit (UI) | N/A | ❌ | ⚠️ escalated |
| 37-04-01 | 04 | 1 | ONBD-04 | unit (API) | `cd trade-flow-api && npx jest src/business/test/services/business-creator.service.spec.ts --no-coverage` | ✅ | ✅ green |
| 37-04-02 | 04 | 1 | ONBD-01-07, TRIAL-02-03 | docs | N/A | N/A | ✅ complete |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ escalated*

---

## Wave 0 Requirements

- [ ] Install Vitest + @testing-library/react in `trade-flow-ui` for component unit testing
- [ ] Configure `vite.config.ts` with `test` block for Vitest
- [ ] Install `msw` for RTK Query mock handlers

*Root cause: `trade-flow-ui` has no unit test framework. All 8 UI gaps require this infrastructure.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| OnboardingGuard redirects users without display name to /onboarding | ONBD-01 | No UI unit test framework | Log in as new user with no `name` field — confirm redirect to `/onboarding` |
| BusinessStep validates business name and trade selection | ONBD-02 | No UI unit test framework | On Step 2, try submitting with empty business name — confirm button is disabled |
| SetupLoadingScreen sends country:GB, currency:GBP | ONBD-03 | No UI unit test framework | Open DevTools Network tab, complete Step 2, inspect POST /v1/business body for `country: "GB"`, `currency: "GBP"` |
| ProfileStep validates and submits display name | ONBD-05 | No UI unit test framework | On Step 1, try submitting empty — confirm validation error. Enter name, submit — confirm PATCH /v1/user/me fires |
| useOnboardingStep resumes at business step when name exists but no business | ONBD-06 | No UI unit test framework | Create user with name but no business, navigate to /onboarding — confirm starts at Step 2 |
| OnboardingGuard passes through for complete users | ONBD-07 | No UI unit test framework | Log in as user with name + business — confirm no redirect, lands on /dashboard |
| TrialBadge renders desktop/mobile variants with urgency colors | TRIAL-02 | No UI unit test framework | Log in as trialing user — confirm badge shows in header with correct format and color tier |
| TrialBadge click opens Stripe Billing Portal | TRIAL-03 | No UI unit test framework | Click trial badge — confirm new tab opens to Stripe Billing Portal |

---

## Validation Audit 2026-04-07

| Metric | Count |
|--------|-------|
| Gaps found | 9 |
| Resolved | 1 |
| Escalated | 8 |

**Escalation root cause:** `trade-flow-ui` lacks a unit test framework (no Vitest/Jest configured). All 8 UI gaps blocked on Wave 0 infrastructure.

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify or Wave 0 dependencies
- [x] Sampling continuity: API tests run after each commit
- [ ] Wave 0 covers all MISSING references (blocked — Vitest not installed)
- [x] No watch-mode flags
- [x] Feedback latency < 5s (API tests)
- [ ] `nyquist_compliant: true` set in frontmatter (blocked — 8 escalated gaps)

**Approval:** partial 2026-04-07
