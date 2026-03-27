# Requirements: Trade Flow v1.5 — Automated E2E Playwright Testing

**Defined:** 2026-03-27
**Core Value:** A job is the centre of the business — Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.5 Requirements

### Foundation

- [ ] **FOUND-01**: Playwright is installed and configured in trade-flow-ui with `playwright.config.ts`, `webServer` config targeting the Vite dev server, Chromium-only, and an `e2e/` directory structure (`fixtures/`, `helpers/`, `tests/`)
- [ ] **FOUND-02**: A global auth setup project authenticates via the real Firebase login UI once and saves `storageState` for reuse across all tests (no per-test login)
- [ ] **FOUND-03**: A typed API seeding client (fetch-based helper) can create and delete test entities via the NestJS API, authenticated with a Firebase test user token
- [ ] **FOUND-04**: The NestJS API has a test-mode email bypass so `QuoteEmailSender` transitions a quote to SENT status without calling Resend when `E2E_TEST_MODE=true`
- [ ] **FOUND-05**: A GitHub Actions workflow checks out both repos, starts the Docker Compose stack (MongoDB, Redis), runs Playwright tests, and uploads traces/reports on failure

### Auth & Onboarding

- [ ] **AUTH-01**: Test verifies user can log in with valid credentials and land on the dashboard
- [ ] **AUTH-02**: Test verifies unauthenticated users are redirected to login when accessing protected routes
- [ ] **AUTH-03**: Test verifies the full onboarding wizard creates a business with default job types, tax rates, items, and visit types

### Core Job Flow

- [ ] **JOB-01**: Test verifies user can create a customer and view it in the customer list
- [ ] **JOB-02**: Test verifies user can create a job linked to a customer and view it in the job list
- [ ] **JOB-03**: Test verifies user can add a schedule entry to a job with date, time, and visit type
- [ ] **JOB-04**: Test verifies the connected journey: create customer → create job → add schedule → create quote (end-to-end flow)

### Quote Send & Respond

- [ ] **QTE-01**: Test verifies user can add line items to a quote and see correct calculated totals
- [ ] **QTE-02**: Test verifies user can send a quote and status transitions to SENT (via test-mode email bypass)
- [ ] **QTE-03**: Test verifies customer can view the public quote page via a seeded token in an unauthenticated browser context
- [ ] **QTE-04**: Test verifies customer can accept a quote and status transitions to ACCEPTED
- [ ] **QTE-05**: Test verifies customer can decline a quote with a reason and status transitions to REJECTED

### Settings & Inventory

- [ ] **SET-01**: Test verifies user can create, edit, and delete a tax rate
- [ ] **SET-02**: Test verifies user can create a material item linked to a tax rate and view it in the item list
- [ ] **SET-03**: Test verifies user can create and manage visit types
- [ ] **SET-04**: Test verifies user can save a quote email template with variable placeholders

## Future Requirements

Features identified but deferred from v1.5.

### Visual Regression

- **VIS-01**: Screenshot regression tests catch unintended visual changes to key pages
- **VIS-02**: Baseline screenshots captured per commit for comparison

### Extended Coverage

- **EXT-01**: Mobile viewport Playwright project tests responsive layouts
- **EXT-02**: Firefox and WebKit browser projects added to CI matrix
- **EXT-03**: Bundle item creation and quote line item expansion tested end-to-end

## Out of Scope

| Feature | Reason |
|---------|--------|
| Playwright Component Testing | Experimental, overlaps with Vitest for component tests; E2E should test integrated flows |
| Visual regression tests | Too early — UI still evolving; add after suite is stable for several weeks |
| Database-level assertions (direct MongoDB queries) | Couples tests to schema; assert on UI and API responses only |
| Performance/load testing | Playwright is not a load testing tool; use k6/Artillery separately |
| Exhaustive negative path testing | Unit test territory; E2E covers happy paths and 1-2 critical error paths |
| Email delivery verification via MailHog/Mailpit | Adds infrastructure complexity; test quote response page directly via seeded token |
| Parallel test execution (fullyParallel) | Requires per-test DB isolation not worth the complexity yet; run serially with workers:1 |
| Firebase Auth Emulator setup | Firebase test user with real auth is sufficient for local dev; emulator is CI-only future work |

## Traceability

Populated by roadmapper. Each requirement maps to exactly one phase.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | — | Pending |
| FOUND-02 | — | Pending |
| FOUND-03 | — | Pending |
| FOUND-04 | — | Pending |
| FOUND-05 | — | Pending |
| AUTH-01 | — | Pending |
| AUTH-02 | — | Pending |
| AUTH-03 | — | Pending |
| JOB-01 | — | Pending |
| JOB-02 | — | Pending |
| JOB-03 | — | Pending |
| JOB-04 | — | Pending |
| QTE-01 | — | Pending |
| QTE-02 | — | Pending |
| QTE-03 | — | Pending |
| QTE-04 | — | Pending |
| QTE-05 | — | Pending |
| SET-01 | — | Pending |
| SET-02 | — | Pending |
| SET-03 | — | Pending |
| SET-04 | — | Pending |

**Coverage:**
- v1.5 requirements: 21 total
- Mapped to phases: 0 (pending roadmap)
- Unmapped: 21 ⚠️

---
*Requirements defined: 2026-03-27*
*Last updated: 2026-03-27 after initial definition*
