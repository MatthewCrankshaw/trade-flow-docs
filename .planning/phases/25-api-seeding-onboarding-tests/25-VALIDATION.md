---
phase: 25
slug: api-seeding-onboarding-tests
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-03-28
---

# Phase 25 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Playwright @playwright/test (installed in Phase 24) |
| **Config file** | trade-flow-ui/playwright.config.ts |
| **Quick run command** | `npx playwright test --project=chromium --grep @smoke` |
| **Full suite command** | `npx playwright test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npx playwright test --project=chromium --grep @smoke`
- **After every plan wave:** Run `npx playwright test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 25-01-01 | 01 | 1 | FOUND-03 | integration | `npx playwright test --list` (compilation check; infrastructure files, no dedicated spec) | ❌ W0 | ⬜ pending |
| 25-01-02 | 01 | 1 | FOUND-03 | integration | `npx playwright test --list` (auth setup compilation check) | ❌ W0 | ⬜ pending |
| 25-01-03 | 01 | 1 | AUTH-01, AUTH-02 | e2e | `npx playwright test e2e/tests/auth/login.spec.ts` and `npx playwright test e2e/tests/auth/redirect-unauth.spec.ts --project=chromium-unauth` | ❌ W0 | ⬜ pending |
| 25-02-01 | 02 | 2 | AUTH-03 | e2e | `npx playwright test e2e/tests/onboarding/onboarding-wizard.spec.ts --project=chromium` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `e2e/helpers/api-client.ts` — seeding client (FOUND-03)
- [ ] `e2e/helpers/db-cleanup.ts` — MongoDB cleanup helper (FOUND-03)
- [ ] `e2e/helpers/seed-factories.ts` — Faker factories (FOUND-03)
- [ ] `e2e/fixtures/test-base.ts` — extended Playwright fixture (FOUND-03)
- [ ] `e2e/tests/auth/login.spec.ts` — AUTH-01 (login test)
- [ ] `e2e/tests/auth/redirect-unauth.spec.ts` — AUTH-02 (redirect test)
- [ ] `e2e/tests/onboarding/onboarding-wizard.spec.ts` — AUTH-03 (onboarding wizard test)

*Existing Playwright infrastructure from Phase 24 covers framework setup.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Second Firebase test user exists | AUTH-03 | Firebase Console setup | Verify test user with no business exists in Firebase Console |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
