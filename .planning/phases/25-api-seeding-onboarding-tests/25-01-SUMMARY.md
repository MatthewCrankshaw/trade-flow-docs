---
phase: 25-api-seeding-onboarding-tests
plan: 01
subsystem: testing
tags: [playwright, mongodb, faker, firebase, e2e, seeding]

# Dependency graph
requires:
  - phase: 24-playwright-bootstrap-auth
    provides: Playwright config, auth setup pattern, smoke tests, storageState infrastructure
provides:
  - Typed HTTP seeding client (ApiClient) wrapping NestJS REST API endpoints
  - MongoDB cleanup helper (DbCleanup) for test data isolation by businessId
  - Faker-based factory functions for customer, job, and quote payloads
  - Extended Playwright test fixture exposing apiClient and dbCleanup
  - Second Firebase user auth setup writing onboarding-user.json storageState
  - AUTH-01 test: authenticated user lands on /dashboard
  - AUTH-02 test: unauthenticated access to /dashboard and /customers redirects to /login
affects:
  - 25-02-onboarding-tests
  - 26-core-job-flow
  - 27-quote-send-respond
  - 28-settings-inventory

# Tech tracking
tech-stack:
  added:
    - mongodb@7.0.0 (devDependency — matches trade-flow-api version for cleanup helper)
    - "@faker-js/faker (already installed) — used for entity payload generation"
  patterns:
    - ApiClient pattern: typed fetch wrapper with Bearer token auth, returns json.data[0] on POST and json.data on GET
    - DbCleanup pattern: MongoClient connected in fixture setup, closed in fixture teardown
    - extractTokenFromPage pattern: page.evaluate with firebase/auth getIdToken() to avoid token expiry from storageState
    - Dynamic businessId discovery: GET /v1/business in apiClient fixture (no hardcoded IDs)
    - D-03 auto-chaining: createJob auto-creates customer if customerId omitted
    - Auth setup pattern: two setup blocks in same auth.setup.ts file, second calls auth.signOut() before signing in

key-files:
  created:
    - trade-flow-ui/e2e/helpers/api-client.ts
    - trade-flow-ui/e2e/helpers/db-cleanup.ts
    - trade-flow-ui/e2e/helpers/seed-factories.ts
    - trade-flow-ui/e2e/fixtures/test-base.ts
    - trade-flow-ui/e2e/tests/auth/login.spec.ts
    - trade-flow-ui/e2e/tests/auth/redirect-unauth.spec.ts
  modified:
    - trade-flow-ui/e2e/helpers/auth.setup.ts
    - trade-flow-ui/.env.e2e.example
    - trade-flow-ui/.gitignore
    - trade-flow-ui/package.json

key-decisions:
  - "Token extraction via page.evaluate(getIdToken()) not from storageState JSON — avoids Firebase token expiry after 1h"
  - "Dynamic businessId discovery via GET /v1/business — no hardcoded IDs in fixtures"
  - "D-03 auto-chaining in createJob — callers don't need to pre-create customer for job-only tests"
  - "Two setup blocks in auth.setup.ts — auth.signOut() called before onboarding user login to prevent shared context bleed"
  - "visit_types endpoint as /v1/visit-type — consistent with REST API naming convention"

patterns-established:
  - "Seeding client pattern: ApiClient(baseUrl, token, businessId) constructed in test fixture with runtime businessId"
  - "Cleanup pattern: DbCleanup connected in fixture, used via cleanByBusinessId(businessId) in afterEach"
  - "Test project routing by filename: unauth.spec.ts -> chromium-unauth, all other .spec.ts -> chromium"
  - "Factory functions return plain objects matching API request body shape (not class instances)"

requirements-completed: [FOUND-03, AUTH-01, AUTH-02]

# Metrics
duration: 5min
completed: 2026-03-28
---

# Phase 25 Plan 01: API Seeding Infrastructure and Auth Tests Summary

**Typed API seeding client, MongoDB cleanup helper, Faker factories, and extended Playwright fixture with two Firebase user auth setups and AUTH-01/AUTH-02 test specs**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-03-28T09:07:49Z
- **Completed:** 2026-03-28T09:12:07Z
- **Tasks:** 3
- **Files modified:** 10 (6 created, 4 modified)

## Accomplishments

- Created typed ApiClient HTTP seeding client with auto-chaining createJob (D-03), typed create/get methods for all entity types, and Bearer token auth
- Created DbCleanup MongoDB helper for test data isolation — cleanByBusinessId across 9 collections, cleanOnboardingData for onboarding teardown
- Created Faker-based seed factories for customer, job, and quote payloads with timestamp uniqueness
- Created test-base.ts extended Playwright fixture exposing apiClient and dbCleanup with Firebase token extraction via page.evaluate
- Extended auth.setup.ts to authenticate both users (pre-onboarded + onboarding), writing user.json and onboarding-user.json storageState files
- Created AUTH-01 (login.spec.ts) and AUTH-02 (redirect-unauth.spec.ts) test specs in correct Playwright projects

## Task Commits

Each task was committed atomically to the `trade-flow-ui` git repo:

1. **Task 1: Install mongodb and create seeding infrastructure** - `fc846f7` (feat)
2. **Task 2: Update auth setup for two users and update env/config** - `a642720` (feat)
3. **Task 3: Create auth redirect test specs** - `b570a36` (feat)

**Plan metadata:** (docs commit, see below)

## Files Created/Modified

- `e2e/helpers/api-client.ts` - Typed HTTP seeding client wrapping NestJS REST endpoints (createCustomer, createJob with D-03 auto-chain, createQuote, getBusinesses, getJobTypes, getTaxRates, getItems, getVisitTypes)
- `e2e/helpers/db-cleanup.ts` - MongoDB cleanup helper: cleanByBusinessId (9 collections), cleanOnboardingData (business + users)
- `e2e/helpers/seed-factories.ts` - Faker factory functions: customerPayload, jobPayload, quotePayload with Date.now() uniqueness
- `e2e/fixtures/test-base.ts` - Extended test fixture with apiClient (Firebase token via page.evaluate) and dbCleanup fixtures
- `e2e/tests/auth/login.spec.ts` - AUTH-01: authenticated user lands on /dashboard with main + navigation visible
- `e2e/tests/auth/redirect-unauth.spec.ts` - AUTH-02: unauthenticated /dashboard and /customers both redirect to /login
- `e2e/helpers/auth.setup.ts` - Added second setup block for onboarding user with auth.signOut() + ONBOARDING_AUTH_FILE
- `.env.e2e.example` - Added E2E_MONGO_URL, E2E_ONBOARDING_USER_EMAIL, E2E_ONBOARDING_USER_PASSWORD
- `.gitignore` - Added e2e/.auth/onboarding-user.json to excluded files
- `package.json` - Added mongodb@7.0.0 as devDependency

## Decisions Made

- Firebase token extracted via `page.evaluate(() => getAuth().currentUser.getIdToken())` not from storageState JSON — storageState token can be expired after 1h causing silent auth failures
- businessId discovered dynamically at fixture startup via `GET /v1/business` — no hardcoded IDs
- `createJob` auto-chains customer creation when `customerId` omitted — simplifies test setup for job-centric tests
- Two setup blocks in same `auth.setup.ts` — second calls `auth.signOut()` to prevent same browser context bleed from first setup
- `visit_types` endpoint mapped to `/v1/visit-type` following NestJS REST convention

## Deviations from Plan

**1. [Rule 2 - Missing Critical] Added schedules and visit_types to CLEANUP_COLLECTIONS**
- **Found during:** Task 1 (db-cleanup.ts creation)
- **Issue:** Plan specified 7 cleanup collections but omitted schedules and visit_types — both are created during onboarding and test runs
- **Fix:** Added "schedules" and "visit_types" to CLEANUP_COLLECTIONS array
- **Files modified:** e2e/helpers/db-cleanup.ts
- **Verification:** All collections that belong to a business are now cleaned by cleanByBusinessId
- **Committed in:** fc846f7 (Task 1 commit)

**2. [Rule 1 - Bug] Fixed `_id` cast in cleanOnboardingData**
- **Found during:** Task 1 (db-cleanup.ts creation)
- **Issue:** Plan used `businessId as any` for MongoDB `_id` query — using `as unknown as object` is more type-safe
- **Fix:** Changed to `businessId as unknown as object` to avoid TypeScript `any` warning
- **Files modified:** e2e/helpers/db-cleanup.ts
- **Verification:** TypeScript compiles without warnings
- **Committed in:** fc846f7 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 missing critical, 1 bug fix)
**Impact on plan:** Both fixes improve correctness. No scope creep.

## Issues Encountered

None — all three tasks executed cleanly. `npx playwright test --list` shows all 7 tests discovered correctly across setup, chromium, and chromium-unauth projects.

## User Setup Required

Two external service actions are required before the onboarding test in Plan 02 can run:

1. **Firebase:** Create a second Firebase test user with no business — `E2E_ONBOARDING_USER_EMAIL` / `E2E_ONBOARDING_USER_PASSWORD`
2. **MongoDB connection string:** Add `E2E_MONGO_URL` to `.env.e2e` (defaults to `mongodb://localhost:27017`)

See `user_setup` section in `25-01-PLAN.md` for detailed steps.

## Next Phase Readiness

- All seeding infrastructure is ready for Phase 25 Plan 02 (onboarding tests) and Phase 26+ (core job flow tests)
- `apiClient` fixture available in any spec that imports `test` from `e2e/fixtures/test-base.ts`
- `dbCleanup` fixture available for test isolation and teardown
- Two storageState files will be written by `npx playwright test --project=setup` before other tests run
- Onboarding test (Plan 02) must use `test.use({ storageState: "e2e/.auth/onboarding-user.json" })` at file level

## Self-Check: PASSED

- FOUND: trade-flow-ui/e2e/helpers/api-client.ts
- FOUND: trade-flow-ui/e2e/helpers/db-cleanup.ts
- FOUND: trade-flow-ui/e2e/helpers/seed-factories.ts
- FOUND: trade-flow-ui/e2e/fixtures/test-base.ts
- FOUND: trade-flow-ui/e2e/tests/auth/login.spec.ts
- FOUND: trade-flow-ui/e2e/tests/auth/redirect-unauth.spec.ts
- CONFIRMED: e2e/fixtures/.gitkeep removed
- FOUND: fc846f7 (Task 1 commit in trade-flow-ui)
- FOUND: a642720 (Task 2 commit in trade-flow-ui)
- FOUND: b570a36 (Task 3 commit in trade-flow-ui)
- FOUND: 5c6f018 (docs commit in trade-flow-docs)

---
*Phase: 25-api-seeding-onboarding-tests*
*Completed: 2026-03-28*
