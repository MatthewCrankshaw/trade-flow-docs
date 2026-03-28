---
phase: 25-api-seeding-onboarding-tests
plan: 02
subsystem: testing
tags: [playwright, e2e, onboarding, business-creation, auth, defaults]

# Dependency graph
requires:
  - phase: 25-api-seeding-onboarding-tests
    plan: 01
    provides: ApiClient, DbCleanup, test fixture, onboarding-user.json storageState

provides:
  - AUTH-03 test: full onboarding wizard walkthrough with business creation and default entity verification
  - auth-03-onboarding-wizard.spec.ts covering two scenarios: fresh user creates business, returning user skips form

affects:
  - 26-core-job-flow

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Onboarding is not a dialog wizard — it navigates to /business where CreateBusinessForm lives"
    - "Radix Select combobox selector: getByRole('combobox', { name: /primary trade/i }).click() then getByRole('option') to pick value"
    - "Business confirmation dialog: getByRole('heading', { name: /your business is ready/i }) as completion signal"
    - "waitForDefaults retry helper (8 retries x 750ms) handles race condition between UI completion and API persistence"
    - "test.use({ storageState }) overrides project-level storageState for specific spec file"

key-files:
  created:
    - trade-flow-ui/e2e/tests/onboarding/onboarding-wizard.spec.ts
  modified: []

key-decisions:
  - "Onboarding flow is page-based (/business) not dialog-based — CreateBusinessForm renders on /business page for users with no business"
  - "BusinessConfirmationDialog ('Your business is ready!') is the success signal — no auto-redirect to /dashboard occurs"
  - "Plan template selectors replaced with actual selectors read from component source (getByLabel, getByRole combobox, heading)"
  - "Plan's toHaveURL(/dashboard/) assertion placed in second test (subsequent login) not first test — matches actual flow"
  - "waitForDefaults retries increased to 8x750ms (vs plan template 5x500ms) for more resilience in slow CI"

# Metrics
duration: 3min
completed: 2026-03-28
---

# Phase 25 Plan 02: Onboarding Wizard E2E Test Summary

**AUTH-03 E2E test: full onboarding wizard walkthrough verifying business creation, default entity seeding, and subsequent login skip behaviour via Playwright**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-03-28T09:16:21Z
- **Completed:** 2026-03-28T09:19:00Z
- **Tasks:** 1 (+ checkpoint)
- **Files modified:** 1 (created)

## Accomplishments

- Created `e2e/tests/onboarding/onboarding-wizard.spec.ts` with two test cases:
  1. `completes wizard and creates business with defaults` — navigates to /business, fills CreateBusinessForm (business name + primary trade), submits, waits for BusinessConfirmationDialog, then verifies via API: business exists with correct name, and all default entities created (job types, tax rates, items, visit types)
  2. `subsequent login skips onboarding wizard` — after business created, navigating to app lands on /dashboard without the "Create your business" card
- Used `test.use({ storageState: "e2e/.auth/onboarding-user.json" })` to load onboarding user context
- Implemented `waitForDefaults` retry helper (8 x 750ms) to guard against race condition between UI completion and default entity persistence
- Comprehensive `afterAll` cleanup via `DbCleanup.cleanOnboardingData(businessId, userId)`

## Task Commits

Committed to `trade-flow-ui` git repo:

1. **Task 1: Create onboarding wizard E2E test (AUTH-03)** - `56a6fd3` (feat)

## Files Created/Modified

- `e2e/tests/onboarding/onboarding-wizard.spec.ts` — AUTH-03: two tests (wizard walkthrough + subsequent login skip), 168 lines

## Decisions Made

- Onboarding is NOT a dialog wizard — `CreateBusinessForm` lives on `/business` page. A fresh user on `/dashboard` sees `NeedsBusinessCard` with "Create Business" button that navigates to `/business`
- `BusinessConfirmationDialog` ("Your business is ready!") is the success signal after form submission — no auto-redirect to dashboard
- Selectors derived from component source: `getByLabel(/business name/i)`, `getByRole("combobox", { name: /primary trade/i })`, `getByRole("option", { name: "Plumber" })`, `getByRole("button", { name: /create business/i })`
- `toHaveURL(/\/dashboard/)` assertion is in the "subsequent login" test (not the "completes wizard" test) because the actual flow does not redirect to /dashboard after business creation

## Deviations from Plan

**1. [Rule 1 - Bug] Onboarding flow is page-based, not dialog-based — selectors adjusted**
- **Found during:** Task 1 (reading component source before writing selectors)
- **Issue:** Plan template assumed a dialog-based wizard with "business name input" and "trade combobox" visible on the dialog. The actual implementation shows a `NeedsBusinessCard` on `/dashboard` with a "Create Business" button that navigates to `/business` where `CreateBusinessForm` renders
- **Fix:** Updated test to: (1) click "Create Business" button on dashboard, (2) wait for `/business` URL, (3) fill form fields using correct labels/selectors from `CreateBusinessForm.tsx`, (4) assert `BusinessConfirmationDialog` heading as completion signal
- **Files modified:** e2e/tests/onboarding/onboarding-wizard.spec.ts
- **Committed in:** 56a6fd3

**2. [Rule 1 - Bug] Fixed `test.use` calling `test` before binding**
- **Found during:** Task 1 verification (`npx playwright test --list` produced error)
- **Issue:** Initial draft used `import { test as base, expect } from "@playwright/test"` then called `test.use()` with no `test` binding
- **Fix:** Changed import to `import { test, expect } from "../../fixtures/test-base"` matching plan specification and removed the redundant `const test = base` re-declaration
- **Files modified:** e2e/tests/onboarding/onboarding-wizard.spec.ts
- **Committed in:** 56a6fd3 (same commit, fixed before commit)

**3. [Rule 2 - Missing Critical] toHaveURL(/\/dashboard/) kept in second test**
- **Found during:** Task 1 (tracing the actual redirect flow)
- **Issue:** Plan template placed dashboard redirect assertion inside the first ("completes wizard") test. The actual app does NOT redirect to `/dashboard` after business creation — it stays on `/business` and shows `BusinessConfirmationDialog`. The plan's success criteria only requires the file to CONTAIN `toHaveURL(/\/dashboard/)` — not specifically in the first test
- **Fix:** `toHaveURL(/\/dashboard/)` assertion is in the "subsequent login" test where it correctly applies
- **Files modified:** e2e/tests/onboarding/onboarding-wizard.spec.ts
- **Committed in:** 56a6fd3

## Known Stubs

None — the test uses real selectors derived from the component source.

## Awaiting Human Verification

Task 2 (checkpoint:human-verify) requires:
1. Both Firebase test users must exist in Firebase Console
2. `.env.e2e` must have all variables filled in (E2E_MONGO_URL, E2E_ONBOARDING_USER_EMAIL, E2E_ONBOARDING_USER_PASSWORD)
3. Docker + NestJS API + Vite dev server must be running
4. Run: `npx playwright test` from `trade-flow-ui/` root
5. Expected: 2 setup steps + 4 chromium tests + 3 chromium-unauth tests all pass

## Self-Check: PASSED

- FOUND: trade-flow-ui/e2e/tests/onboarding/onboarding-wizard.spec.ts
- CONFIRMED: 168 lines (>60 minimum)
- CONFIRMED: `test.use({ storageState: "e2e/.auth/onboarding-user.json" })` on line 7
- CONFIRMED: `test.describe("AUTH-03: Onboarding wizard"` on line 47
- CONFIRMED: test "completes wizard and creates business with defaults" on line 63
- CONFIRMED: test "subsequent login skips onboarding wizard" on line 145
- CONFIRMED: `await expect(page).toHaveURL(/\/dashboard/` on line 158
- CONFIRMED: `apiClient.getBusinesses()` call on line 110
- CONFIRMED: `getJobTypes(`, `getTaxRates(`, `getItems(`, `getVisitTypes(` all present
- CONFIRMED: `waitForDefaults` retry pattern present
- CONFIRMED: `cleanOnboardingData` in afterAll
- CONFIRMED: imports from `../../fixtures/test-base`
- CONFIRMED: playwright test --list shows both tests under chromium project

---
*Phase: 25-api-seeding-onboarding-tests*
*Completed: 2026-03-28*
