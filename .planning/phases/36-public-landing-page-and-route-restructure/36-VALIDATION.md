---
phase: 36
slug: public-landing-page-and-route-restructure
status: green
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-02
audited: 2026-04-07
---

# Phase 36 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Playwright (E2E) + Node.js script (bundle check) |
| **Config file** | `trade-flow-ui/playwright.config.ts` |
| **Quick run command** | `cd trade-flow-ui && npx tsc --noEmit` |
| **Full suite command** | `cd trade-flow-ui && npx tsc --noEmit && npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` |
| **Bundle check command** | `cd trade-flow-ui && npm run build && node e2e/scripts/check-bundle-chunks.mjs` |
| **Estimated runtime** | ~15 seconds (static); ~60 seconds (E2E with dev server) |

---

## Sampling Rate

- **After every task commit:** Run `npx tsc --noEmit`
- **After every plan wave:** Run `npx tsc --noEmit && npx eslint .`
- **Before `/gsd:verify-work`:** Full suite must be green + `npx vite build` for bundle analysis
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File | Status |
|---------|------|------|-------------|-----------|-------------------|------|--------|
| 36-01-01 | 01 | 1 | LAND-01 | E2E (unauth) | `npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` | `e2e/tests/landing-unauth.spec.ts` | ✅ green |
| 36-01-02 | 01 | 1 | LAND-02 | E2E (unauth) | `npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` | `e2e/tests/landing-unauth.spec.ts` | ✅ green |
| 36-01-03 | 01 | 1 | LAND-03 | E2E (unauth) | `npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` | `e2e/tests/landing-unauth.spec.ts` | ✅ green |
| 36-01-04 | 01 | 1 | LAND-04 | E2E (unauth) | `npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` | `e2e/tests/landing-unauth.spec.ts` | ✅ green |
| 36-01-05 | 01 | 1 | LAND-05 | E2E (unauth) | `npx playwright test e2e/tests/landing-unauth.spec.ts --project=chromium-unauth` | `e2e/tests/landing-unauth.spec.ts` | ✅ green |
| 36-02-01 | 02 | 1 | LAND-06 | E2E (auth) | `npx playwright test e2e/tests/auth/login.spec.ts --project=chromium` | `e2e/tests/auth/login.spec.ts` | ✅ green (pre-existing) |
| 36-02-02 | 02 | 1 | -- | smoke (build) | `npm run build && node e2e/scripts/check-bundle-chunks.mjs` | `e2e/scripts/check-bundle-chunks.mjs` | ✅ green |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

All phase requirements are now covered by automated tests:

- LAND-01 through LAND-05: Playwright E2E tests in `e2e/tests/landing-unauth.spec.ts` (chromium-unauth project, no storageState)
- LAND-06: Pre-existing Playwright test in `e2e/tests/auth/login.spec.ts` (authenticated chromium project)
- Bundle isolation: Node.js script `e2e/scripts/check-bundle-chunks.mjs` verifies LandingPage chunk exists after `npm run build`

---

## Manual-Only Verifications

All previously manual-only verifications have been converted to automated tests. No remaining manual-only items.

---

## Validation Sign-Off

- [x] All tasks have automated verify commands
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all MISSING references
- [x] No watch-mode flags
- [x] Feedback latency < 15s (static); E2E requires dev server
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** audited 2026-04-07 by gsd-nyquist-auditor
