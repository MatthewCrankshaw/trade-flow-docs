---
phase: 24-playwright-bootstrap-auth
plan: 01
subsystem: testing
tags: [playwright, e2e, firebase, vite, storagestate, indexeddb, chromium]

# Dependency graph
requires: []
provides:
  - Playwright 1.58.2 installed in trade-flow-ui as devDependency
  - playwright.config.ts with three-project layout (setup / chromium / chromium-unauth)
  - e2e/ directory structure (fixtures/, helpers/, tests/, .auth/)
  - auth.setup.ts: programmatic Firebase auth via signInWithEmailAndPassword in page.evaluate with storageState indexedDB: true
  - smoke-auth.spec.ts: authenticated smoke test verifying /dashboard landing
  - smoke-unauth.spec.ts: unauthenticated smoke test verifying /login form renders
  - .env.e2e.example: committed env template with all seven E2E_ variable names
affects:
  - 25-e2e-auth-onboarding
  - 26-e2e-job-flow
  - 27-e2e-quote-flow
  - 28-e2e-ci-integration

# Tech tracking
tech-stack:
  added:
    - "@playwright/test@1.58.2 — E2E test runner"
    - "@faker-js/faker@10.4.0 — test data generation (used in Phase 25+)"
    - "dotenv@17.3.1 — loads .env.e2e into playwright.config.ts"
  patterns:
    - "Playwright setup project pattern (testMatch: /.*\\.setup\\.ts/) over legacy globalSetup"
    - "storageState({ indexedDB: true }) required for Firebase v12 IndexedDB token persistence"
    - "signInWithEmailAndPassword inside page.evaluate — not signInWithCustomToken (different token types)"
    - "Separate chromium-unauth project with testMatch to prevent storageState injection on unauth tests"
    - "fileURLToPath(import.meta.url) for __dirname in ES module context (type=module project)"
    - "API health check in setup file before auth — aborts with descriptive error if API unreachable"

key-files:
  created:
    - "trade-flow-ui/playwright.config.ts — Playwright config with three projects, webServer, dotenv"
    - "trade-flow-ui/.env.e2e.example — Committed env template with all E2E_ variable names"
    - "trade-flow-ui/e2e/helpers/auth.setup.ts — Setup project: API health check, Firebase auth, storageState"
    - "trade-flow-ui/e2e/tests/smoke-auth.spec.ts — Authenticated smoke test: /dashboard URL assertion"
    - "trade-flow-ui/e2e/tests/smoke-unauth.spec.ts — Unauthenticated smoke test: login form ARIA assertions"
    - "trade-flow-ui/e2e/.auth/.gitignore — Tracks e2e/.auth/ dir, excludes user.json contents"
    - "trade-flow-ui/e2e/fixtures/.gitkeep — Placeholder for Phase 25 fixture files"
  modified:
    - "trade-flow-ui/package.json — Added @playwright/test, @faker-js/faker, dotenv as devDependencies"
    - "trade-flow-ui/.gitignore — Added Playwright entries: e2e/.auth/user.json, playwright-report/, test-results/, .env.e2e"

key-decisions:
  - "Three-project layout (setup/chromium/chromium-unauth) prevents storageState from leaking into unauth tests"
  - "signInWithEmailAndPassword in page.evaluate chosen over signInWithCustomToken — custom tokens require Admin SDK"
  - "chromium project testMatch: /^(?!.*unauth).*\\.spec\\.ts$/ excludes unauth specs from authenticated runner"
  - "fileURLToPath(import.meta.url) used for __dirname — project has type=module, no CommonJS __dirname available"
  - "storageState({ indexedDB: true }) mandatory — Firebase v12 stores all auth tokens in IndexedDB not localStorage"

patterns-established:
  - "Pattern: Playwright setup project with testMatch /.*\\.setup\\.ts/ runs before all other projects"
  - "Pattern: AUTH_FILE path via path.join(__dirname, '../.auth/user.json') relative to helper file location"
  - "Pattern: ESM __dirname shim: const __dirname = path.dirname(fileURLToPath(import.meta.url))"

requirements-completed: [FOUND-01, FOUND-02]

# Metrics
duration: 3min
completed: 2026-03-27
---

# Phase 24 Plan 01: Playwright Bootstrap & Auth Summary

**Playwright 1.58.2 bootstrapped in trade-flow-ui with three-project setup/chromium/chromium-unauth layout, programmatic Firebase auth via signInWithEmailAndPassword in page.evaluate with IndexedDB storageState capture, and two passing smoke tests.**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-27T11:47:05Z
- **Completed:** 2026-03-27T11:50:04Z
- **Tasks:** 3
- **Files modified:** 9

## Accomplishments

- Playwright 1.58.2 + @faker-js/faker + dotenv installed as devDependencies; Chromium binary downloaded
- playwright.config.ts with three-project layout ensuring storageState never leaks to unauthenticated tests
- auth.setup.ts with API health check, Firebase signInWithEmailAndPassword in page.evaluate, and storageState({ indexedDB: true }) for full Firebase v12 token capture
- Two smoke tests: authenticated (/dashboard URL assertion) and unauthenticated (/login ARIA selectors)
- `npx playwright test --list` discovers all 3 tests without TypeScript errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Install packages and scaffold e2e/ directory structure** - `142eb9e` (chore)
2. **Task 2: Create playwright.config.ts and .env.e2e.example** - `b047afa` (feat) — includes Rule 1 fix for ESM __dirname
3. **Task 3: Create auth setup file and smoke tests** - `cd03896` (feat) — includes Pitfall 5 fix for chromium testMatch

**Plan metadata:** (docs commit below)

## Files Created/Modified

- `trade-flow-ui/playwright.config.ts` — Three-project Playwright config with webServer, dotenv via import.meta.url, projects with correct testMatch
- `trade-flow-ui/.env.e2e.example` — Committed env template with E2E_BASE_URL, E2E_API_URL, E2E_FIREBASE_API_KEY, E2E_FIREBASE_AUTH_DOMAIN, E2E_FIREBASE_PROJECT_ID, E2E_TEST_USER_EMAIL, E2E_TEST_USER_PASSWORD
- `trade-flow-ui/e2e/helpers/auth.setup.ts` — Setup project with API health check, Firebase programmatic auth, storageState write
- `trade-flow-ui/e2e/tests/smoke-auth.spec.ts` — Authenticated smoke: page.goto("/dashboard"), toHaveURL(/\/dashboard/), getByRole("main").toBeVisible()
- `trade-flow-ui/e2e/tests/smoke-unauth.spec.ts` — Unauthenticated smoke: page.goto("/login"), ARIA textbox assertions for email + password
- `trade-flow-ui/e2e/.auth/.gitignore` — Tracks dir in git, excludes user.json (credentials)
- `trade-flow-ui/e2e/fixtures/.gitkeep` — Placeholder for Phase 25 fixture files
- `trade-flow-ui/package.json` — devDependencies: @playwright/test@^1.58.2, @faker-js/faker@^10.4.0, dotenv@^17.3.1
- `trade-flow-ui/.gitignore` — Appended: e2e/.auth/user.json, playwright-report/, test-results/, .env.e2e

## Decisions Made

- Used `fileURLToPath(import.meta.url)` for __dirname shim — package.json has `"type": "module"` making CommonJS __dirname unavailable in playwright.config.ts
- Added `testMatch: /^(?!.*unauth).*\.spec\.ts$/` to chromium project — prevents smoke-unauth.spec.ts from running with storageState injected (Pitfall 5 avoidance)
- Used `signInWithEmailAndPassword` inside `page.evaluate` — not `signInWithCustomToken` which requires Firebase Admin SDK and a different token format (Pitfall 2 avoidance per RESEARCH.md)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed __dirname not defined in ES module scope**
- **Found during:** Task 2 (Create playwright.config.ts)
- **Issue:** `playwright.config.ts` used `__dirname` which is not available in ES module context. Project has `"type": "module"` in package.json, causing `ReferenceError: __dirname is not defined in ES module scope` when running `npx playwright test --list`
- **Fix:** Added `import { fileURLToPath } from "url"` and shim `const __dirname = path.dirname(fileURLToPath(import.meta.url))` to both playwright.config.ts and e2e/helpers/auth.setup.ts
- **Files modified:** trade-flow-ui/playwright.config.ts, trade-flow-ui/e2e/helpers/auth.setup.ts
- **Verification:** `npx playwright test --list` runs without errors
- **Committed in:** b047afa, cd03896 (Task 2 and Task 3 commits)

**2. [Rule 2 - Missing Critical] Added testMatch to chromium project to exclude unauth specs**
- **Found during:** Task 3 (Create auth setup file and smoke tests)
- **Issue:** After adding both smoke specs, `npx playwright test --list` showed smoke-unauth.spec.ts appearing in both chromium AND chromium-unauth projects. The chromium project would inject storageState into the unauthenticated test, causing it to redirect to /dashboard and fail (Pitfall 5 from RESEARCH.md)
- **Fix:** Added `testMatch: /^(?!.*unauth).*\.spec\.ts$/` to chromium project in playwright.config.ts
- **Files modified:** trade-flow-ui/playwright.config.ts
- **Verification:** `npx playwright test --list` shows smoke-unauth.spec.ts only in chromium-unauth; total 3 tests (not 4)
- **Committed in:** cd03896 (Task 3 commit)

---

**Total deviations:** 2 auto-fixed (1 bug - ESM __dirname, 1 missing critical - testMatch isolation)
**Impact on plan:** Both auto-fixes necessary for correctness. ESM fix was environment-specific (this project uses type=module). TestMatch fix directly prevents Pitfall 5 documented in the research. No scope creep.

## Issues Encountered

None beyond the auto-fixed deviations above.

## User Setup Required

**External services require manual configuration before running the full smoke suite.**

Before running `npx playwright test` with authentication:

1. Create a dedicated E2E test user in Firebase Console → Authentication → Users → Add user
   - Use a dedicated email (not your personal account), e.g. `e2e-test@your-project.com`

2. Copy `.env.e2e.example` to `.env.e2e` and fill in:
   - `E2E_FIREBASE_API_KEY` — Firebase Console → Project settings → General → Web API Key
   - `E2E_FIREBASE_AUTH_DOMAIN` — e.g. `your-project.firebaseapp.com`
   - `E2E_FIREBASE_PROJECT_ID` — Firebase Console → Project settings → General → Project ID
   - `E2E_TEST_USER_EMAIL` — the dedicated test user email you created
   - `E2E_TEST_USER_PASSWORD` — the test user password

3. Start the NestJS API and Docker: `docker-compose up -d && npm run start:dev` (in trade-flow-api)

4. Run the unauthenticated smoke (no credentials needed — only requires Vite dev server):
   ```bash
   npx playwright test e2e/tests/smoke-unauth.spec.ts --project=chromium-unauth
   ```

5. Run the full suite (requires .env.e2e + API running):
   ```bash
   npx playwright test
   ```

## Next Phase Readiness

- Playwright infrastructure complete — Phase 25 can add fixture files in e2e/fixtures/ and API seeding helpers
- storageState pattern established — all future authenticated tests inherit via chromium project dependency
- @faker-js/faker already installed for Phase 25 test data generation
- One concern: Firebase dynamic import inside page.evaluate (ESM via Vite) is LOW confidence per RESEARCH.md — needs runtime validation with real .env.e2e credentials. Fallback strategy documented in RESEARCH.md if needed.

---
*Phase: 24-playwright-bootstrap-auth*
*Completed: 2026-03-27*
