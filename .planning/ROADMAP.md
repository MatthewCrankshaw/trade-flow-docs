# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2020-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2020-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2020-03-15)
- v1.3 Send Quotes -- Phases 15-19 (shipped 2026-03-21)
- v1.4 Monorepo & Worker Infrastructure -- Phases 20-23 (shipped 2026-03-22)
- v1.5 Automated E2E Playwright Testing -- Phases 24-28 (in progress)

## Phases

<details>
<summary>v1.0 Scheduling (Phases 1-8) -- SHIPPED 2020-03-07</summary>

- [x] Phase 1: Visit Type Backend (2/2 plans) -- completed 2020-02-23
- [x] Phase 2: Visit Type Management UI (2/2 plans) -- completed 2020-02-28
- [x] Phase 3: Schedule Data Model and Create API (2/2 plans) -- completed 2020-03-01
- [x] Phase 4: Schedule Status and CRUD API (3/3 plans) -- completed 2020-03-01
- [x] Phase 5: Schedule Creation UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 6: Schedule List and Detail UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 7: Schedule Edit and Management UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 8: Job Detail Integration (1/1 plan) -- completed 2020-03-07

Full details: `.planning/milestones/v1.0-ROADMAP.md`

</details>

<details>
<summary>v1.1 Item Tax Rate Linkage (Phases 9-10) -- SHIPPED 2020-03-08</summary>

- [x] Phase 9: Item Tax Rate API (4/4 plans) -- completed 2020-03-08
- [x] Phase 10: Item Tax Rate UI (2/2 plans) -- completed 2020-03-08

Full details: `.planning/milestones/v1.1-ROADMAP.md`

</details>

<details>
<summary>v1.2 Bundles & Quotes (Phases 11-14) -- SHIPPED 2020-03-15</summary>

- [x] Phase 11: Bundle Bug Fix and Foundation (1/1 plan) -- completed 2020-03-08
- [x] Phase 12: Bundle Component Editing (2/2 plans) -- completed 2020-03-08
- [x] Phase 13: Quote API Integration (3/3 plans) -- completed 2020-03-14
- [x] Phase 14: Quote Detail and Line Items (6/6 plans) -- completed 2020-03-14

Full details: `.planning/milestones/v1.2-ROADMAP.md`

</details>

<details>
<summary>v1.3 Send Quotes (Phases 15-19) -- SHIPPED 2026-03-21</summary>

- [x] Phase 15: Quote Deletion (2/2 plans) -- completed 2026-03-15
- [x] Phase 16: Token Infrastructure and Public API (2/2 plans) -- completed 2026-03-15
- [x] Phase 17: Customer Quote Page (4/4 plans) -- completed 2026-03-20
- [x] Phase 18: Quote Email Sending (7/7 plans) -- completed 2026-03-21
- [x] Phase 19: Customer Response (3/3 plans) -- completed 2026-03-21

Full details: `.planning/milestones/v1.3-ROADMAP.md`

</details>

<details>
<summary>v1.4 Monorepo & Worker Infrastructure (Phases 20-23) -- SHIPPED 2026-03-22</summary>

- [x] Phase 20: Infrastructure Foundation (2/2 plans) -- completed 2026-03-22
- [x] Phase 21: Queue Module (1/1 plan) -- completed 2026-03-22
- [x] Phase 22: Worker Service Scaffold (2/2 plans) -- completed 2026-03-22
- [x] Phase 23: Developer Experience (2/2 plans) -- completed 2026-03-22

Full details: `.planning/milestones/v1.4-ROADMAP.md`

</details>

<details open>
<summary>v1.5 Automated E2E Playwright Testing (Phases 24-28) -- IN PROGRESS</summary>

- [ ] **Phase 24: Playwright Bootstrap & Auth** - Install and configure Playwright with global auth storageState
- [ ] **Phase 25: API Seeding Infrastructure + Onboarding Tests** - Typed API client for test data seeding and onboarding flow tests
- [ ] **Phase 26: Core Job Flow Tests** - Customer, job, schedule, and quote creation tests
- [ ] **Phase 27: Quote Lifecycle Tests** - Email bypass, send flow, and customer accept/decline tests
- [ ] **Phase 28: Settings Tests + CI Integration** - Settings/inventory tests and GitHub Actions workflow

</details>

## Phase Details

### Phase 24: Playwright Bootstrap & Auth
**Goal**: The test infrastructure exists and any authenticated test can run without per-test login overhead
**Depends on**: Nothing (first phase of v1.5)
**Requirements**: FOUND-01, FOUND-02
**Success Criteria** (what must be TRUE):
  1. Running `npx playwright test` from the trade-flow-ui root executes the test suite against the Vite dev server
  2. A global setup project authenticates once via the real Firebase login UI and writes storageState to disk
  3. Individual tests load the saved storageState and land on the dashboard without re-authenticating
  4. The `e2e/` directory structure (`fixtures/`, `helpers/`, `tests/`) exists and a first smoke test passes in Chromium
**Plans**: 1 plan
Plans:
- [ ] 24-01-PLAN.md — Install Playwright, configure three-project layout, implement programmatic Firebase auth setup, and deliver smoke tests

### Phase 25: API Seeding Infrastructure + Onboarding Tests
**Goal**: Tests can create and clean up isolated test data via the API, and the onboarding flow is verified end-to-end
**Depends on**: Phase 24
**Requirements**: FOUND-03, AUTH-01, AUTH-02, AUTH-03
**Success Criteria** (what must be TRUE):
  1. A typed API seeding helper can create a customer, job, and quote via HTTP and delete them after the test, with no leftover data between test runs
  2. A test verifies that a user with valid credentials can log in and land on the dashboard
  3. A test verifies that accessing a protected route without authentication redirects to the login page
  4. A test verifies the full onboarding wizard: business name entered, trade selected, wizard completed, and default job types/tax rates/items/visit types appear in the new business
**Plans**: TBD
**UI hint**: yes

### Phase 26: Core Job Flow Tests
**Goal**: The primary business object pipeline (customer → job → schedule → quote) is tested end-to-end
**Depends on**: Phase 25
**Requirements**: JOB-01, JOB-02, JOB-03, JOB-04
**Success Criteria** (what must be TRUE):
  1. A test verifies that creating a customer via the UI results in the customer appearing in the customer list
  2. A test verifies that creating a job linked to a customer results in the job appearing in the job list
  3. A test verifies that adding a schedule entry to a job with a date, start time, and visit type displays the entry in the job detail page
  4. A connected end-to-end test runs the full sequence: create customer → create job → add schedule entry → create quote, and each step passes without relying on pre-seeded data
**Plans**: TBD
**UI hint**: yes

### Phase 27: Quote Lifecycle Tests
**Goal**: The complete quote send-and-respond flow is testable without real email delivery, and all customer response paths are verified
**Depends on**: Phase 26
**Requirements**: FOUND-04, QTE-01, QTE-02, QTE-03, QTE-04, QTE-05
**Success Criteria** (what must be TRUE):
  1. The NestJS API transitions a quote to SENT status when `E2E_TEST_MODE=true` is set, without making any Resend API call
  2. A test verifies that adding line items to a quote displays correct subtotal, tax, and total values
  3. A test verifies that sending a quote changes its status to SENT in the UI (using test-mode email bypass)
  4. A test verifies that an unauthenticated browser context can view the public quote page when navigated to via a seeded token URL
  5. A test verifies customer acceptance transitions the quote to ACCEPTED status; a separate test verifies decline with a reason transitions it to REJECTED
**Plans**: TBD
**UI hint**: yes

### Phase 28: Settings Tests + CI Integration
**Goal**: Settings and inventory management flows are verified, and the full test suite runs automatically on every push and pull request
**Depends on**: Phase 27
**Requirements**: SET-01, SET-02, SET-03, SET-04, FOUND-05
**Success Criteria** (what must be TRUE):
  1. A test verifies a user can create a tax rate, edit its name/percentage, and delete it
  2. A test verifies a user can create a material item linked to a tax rate and see it appear in the item list
  3. A test verifies a user can create and manage visit types (create new, verify it appears in the visit types list)
  4. A test verifies a user can save a quote email template with variable placeholders and the template persists after a page reload
  5. A GitHub Actions workflow runs on push and pull request: checks out both repos, starts the Docker Compose stack, executes the full Playwright suite, and uploads traces and reports as artifacts on failure
**Plans**: TBD

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Visit Type Backend | v1.0 | 2/2 | Complete | 2020-02-23 |
| 2. Visit Type Management UI | v1.0 | 2/2 | Complete | 2020-02-28 |
| 3. Schedule Data Model and Create API | v1.0 | 2/2 | Complete | 2020-03-01 |
| 4. Schedule Status and CRUD API | v1.0 | 3/3 | Complete | 2020-03-01 |
| 5. Schedule Creation UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 6. Schedule List and Detail UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 7. Schedule Edit and Management UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 8. Job Detail Integration | v1.0 | 1/1 | Complete | 2020-03-07 |
| 9. Item Tax Rate API | v1.1 | 4/4 | Complete | 2020-03-08 |
| 10. Item Tax Rate UI | v1.1 | 2/2 | Complete | 2020-03-08 |
| 11. Bundle Bug Fix and Foundation | v1.2 | 1/1 | Complete | 2020-03-08 |
| 12. Bundle Component Editing | v1.2 | 2/2 | Complete | 2020-03-08 |
| 13. Quote API Integration | v1.2 | 3/3 | Complete | 2020-03-14 |
| 14. Quote Detail and Line Items | v1.2 | 6/6 | Complete | 2020-03-14 |
| 15. Quote Deletion | v1.3 | 2/2 | Complete | 2026-03-15 |
| 16. Token Infrastructure and Public API | v1.3 | 2/2 | Complete | 2026-03-15 |
| 17. Customer Quote Page | v1.3 | 4/4 | Complete | 2026-03-20 |
| 18. Quote Email Sending | v1.3 | 7/7 | Complete | 2026-03-21 |
| 19. Customer Response | v1.3 | 3/3 | Complete | 2026-03-21 |
| 20. Infrastructure Foundation | v1.4 | 2/2 | Complete | 2026-03-22 |
| 21. Queue Module | v1.4 | 1/1 | Complete | 2026-03-22 |
| 22. Worker Service Scaffold | v1.4 | 2/2 | Complete | 2026-03-22 |
| 23. Developer Experience | v1.4 | 2/2 | Complete | 2026-03-22 |
| 24. Playwright Bootstrap & Auth | v1.5 | 0/1 | Not started | - |
| 25. API Seeding Infrastructure + Onboarding Tests | v1.5 | 0/? | Not started | - |
| 26. Core Job Flow Tests | v1.5 | 0/? | Not started | - |
| 27. Quote Lifecycle Tests | v1.5 | 0/? | Not started | - |
| 28. Settings Tests + CI Integration | v1.5 | 0/? | Not started | - |
