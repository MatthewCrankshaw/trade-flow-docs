# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2020-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2020-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2020-03-15)
- v1.3 Send Quotes -- Phases 15-19 (shipped 2026-03-21)
- v1.4 Monorepo & Worker Infrastructure -- Phases 20-23 (shipped 2026-03-22)
- v1.5 Automated E2E Playwright Testing -- Phases 24-28 (in progress)
- v1.6 Stripe Subscription Billing -- Phases 29-34 (shipped 2026-03-31)
- v1.7 Onboarding & Landing Page -- Phases 35-39 (in progress)

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

<details>
<summary>v1.5 Automated E2E Playwright Testing (Phases 24-28) -- IN PROGRESS</summary>

- [x] **Phase 24: Playwright Bootstrap & Auth** - Install and configure Playwright with global auth storageState (completed 2026-03-27)
- [ ] **Phase 25: API Seeding Infrastructure + Onboarding Tests** - Typed API client for test data seeding and onboarding flow tests
- [ ] **Phase 26: Core Job Flow Tests** - Customer, job, schedule, and quote creation tests
- [ ] **Phase 27: Quote Lifecycle Tests** - Email bypass, send flow, and customer accept/decline tests
- [ ] **Phase 28: Settings Tests + CI Integration** - Settings/inventory tests and GitHub Actions workflow

</details>

<details>
<summary>v1.6 Stripe Subscription Billing (Phases 29-34) -- SHIPPED 2026-03-31</summary>

- [x] Phase 29: Subscription Module Foundation (2/2 plans) -- completed 2026-03-29
- [x] Phase 30: Stripe Checkout and Webhooks (3/3 plans) -- completed 2026-03-29
- [x] Phase 31: Subscription API Endpoints and Tests (2/2 plans) -- completed 2026-03-29
- [x] Phase 32: Subscription Gate and Subscribe Pages (3/3 plans) -- completed 2026-03-29
- [x] Phase 33: Trial Banner and Billing Settings Tab (2/2 plans) -- completed 2026-03-29
- [x] Phase 34: Luxon DateTime Standardization (2/2 plans) -- completed 2026-03-30

Full details: `.planning/milestones/v1.6-ROADMAP.md`

</details>

### v1.7 Onboarding & Landing Page (In Progress)

**Milestone Goal:** Replace the dismissible onboarding flow with a mandatory, streamlined setup process -- from public landing page through profile, business, and no-card trial activation -- and enforce a hard paywall for invalid subscriptions.

- [x] **Phase 35: No-Card Trial API Endpoint** - Backend endpoint for no-card Stripe trial creation with webhook compatibility (completed 2026-04-02)
- [x] **Phase 36: Public Landing Page and Route Restructure** - Marketing landing page at root URL with three-tier route guard architecture (completed 2026-04-07)
- [x] **Phase 37: Onboarding Wizard Pages** - Mandatory profile and business setup with trial activation and old onboarding removal (completed 2026-04-07)
- [x] **Phase 38: Hard Paywall and Soft Paywall Removal** - Full-screen blocking paywall replacing soft write-action modal (completed 2026-04-02)
- [ ] **Phase 39: Welcome Dashboard and Final Cleanup** - Personalised welcome experience with getting-started checklist

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
- [x] 24-01-PLAN.md -- Install Playwright, configure three-project layout, implement programmatic Firebase auth setup, and deliver smoke tests

### Phase 25: API Seeding Infrastructure + Onboarding Tests
**Goal**: Tests can create and clean up isolated test data via the API, and the onboarding flow is verified end-to-end
**Depends on**: Phase 24
**Requirements**: FOUND-03, AUTH-01, AUTH-02, AUTH-03
**Success Criteria** (what must be TRUE):
  1. A typed API seeding helper can create a customer, job, and quote via HTTP and delete them after the test, with no leftover data between test runs
  2. A test verifies that a user with valid credentials can log in and land on the dashboard
  3. A test verifies that accessing a protected route without authentication redirects to the login page
  4. A test verifies the full onboarding wizard: business name entered, trade selected, wizard completed, and default job types/tax rates/items/visit types appear in the new business
**Plans**: 2 plans
Plans:
- [x] 25-01-PLAN.md -- Seeding infrastructure (api-client, db-cleanup, factories, fixtures), second user auth setup, auth redirect tests
- [ ] 25-02-PLAN.md -- Onboarding wizard E2E test with default entity verification and cleanup

### Phase 26: Core Job Flow Tests
**Goal**: The primary business object pipeline (customer -> job -> schedule -> quote) is tested end-to-end
**Depends on**: Phase 25
**Requirements**: JOB-01, JOB-02, JOB-03, JOB-04
**Success Criteria** (what must be TRUE):
  1. A test verifies that creating a customer via the UI results in the customer appearing in the customer list
  2. A test verifies that creating a job linked to a customer results in the job appearing in the job list
  3. A test verifies that adding a schedule entry to a job with a date, start time, and visit type displays the entry in the job detail page
  4. A connected end-to-end test runs the full sequence: create customer -> create job -> add schedule entry -> create quote, and each step passes without relying on pre-seeded data
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

### Phase 35: No-Card Trial API Endpoint
**Goal**: New users get a 30-day free trial without entering payment details, and the local subscription record is created reliably regardless of whether the subscription originated from Checkout or the direct Stripe API
**Depends on**: Nothing (first phase of v1.7, API-only work)
**Requirements**: TRIAL-01, TRIAL-04
**Success Criteria** (what must be TRUE):
  1. Calling POST /v1/subscription/trial creates a Stripe subscription with a 30-day trial period and no payment method required
  2. The local MongoDB subscription record is written synchronously in the endpoint response (belt-and-suspenders with webhook)
  3. When a trial ends without a payment method on file, the subscription auto-cancels via Stripe's `trial_settings.end_behavior.missing_payment_method: cancel`
  4. The `customer.subscription.created` webhook handler can create a fresh local record (not just update), ensuring compatibility with both Checkout-created and API-created subscriptions
**Plans**: 2 plans
Plans:
- [x] 35-01-PLAN.md -- SubscriptionTrialCreator service, findByUserId repo method, controller route, and unit tests
- [x] 35-02-PLAN.md -- customer.subscription.created webhook handler and checkout.session.completed upsert migration

### Phase 36: Public Landing Page and Route Restructure
**Goal**: Visitors can discover Trade Flow's value proposition and start a free trial from a public marketing page, and the app's route architecture supports three tiers of access (public, onboarding, authenticated+subscribed)
**Depends on**: Nothing (first phase of v1.7, UI-only work -- can run in parallel with Phase 35)
**Requirements**: LAND-01, LAND-02, LAND-03, LAND-04, LAND-05, LAND-06
**Success Criteria** (what must be TRUE):
  1. An unauthenticated visitor can view the landing page at the root URL with hero section, feature highlights, pricing card, and sign-up/login navigation
  2. The landing page loads without importing the app's Redux store, feature modules, or auth hooks (verified by bundle analysis)
  3. An authenticated user visiting the root URL is redirected to the dashboard without seeing the landing page
  4. The App.tsx route tree has three nested guard layers (ProtectedRoute, OnboardingGuard shell, HardPaywallGuard shell) ready for Phase 37 and 38 to implement
**Plans**: 2 plans
Plans:
- [ ] 36-01-PLAN.md -- Landing page components (header, hero, features, pricing, footer) with Firebase auth redirect
- [ ] 36-02-PLAN.md -- Route guard shells (OnboardingGuard, PaywallGuard), App.tsx route restructure, LoginPage mode param
**UI hint**: yes

### Phase 37: Onboarding Wizard Pages
**Goal**: Every new user completes a mandatory two-step setup (profile name, then business name + trade) before accessing the app, and existing users with completed profiles and businesses pass through automatically
**Depends on**: Phase 35 (trial endpoint), Phase 36 (route guard architecture)
**Requirements**: ONBD-01, ONBD-02, ONBD-03, ONBD-04, ONBD-05, ONBD-06, ONBD-07, TRIAL-02, TRIAL-03
**Success Criteria** (what must be TRUE):
  1. A new user who signs up is redirected to the profile setup page and cannot access any app page until they enter a display name
  2. After profile setup, the user is redirected to business setup where they enter a business name and select a trade; country defaults to UK and currency defaults to GBP without user input
  3. Completing business setup auto-creates default tax rates, job types, visit types, items, and quote email template, then starts the 30-day no-card trial
  4. The user can see trial days remaining in the app header and can add a payment method via Stripe Billing Portal at any time
  5. Existing users who already have a display name, business, and subscription bypass onboarding entirely; refreshing mid-wizard resumes at the correct step
**Plans**: 4 plans
**UI hint**: yes
Plans:
- [x] 37-01-PLAN.md -- OnboardingGuard redirect logic, wizard container, ProfileStep form, progress bar, trade constants
- [x] 37-02-PLAN.md -- BusinessStep with trade card grid, SetupLoadingScreen with sequential API calls, RTK Query mutations
- [x] 37-03-PLAN.md -- TrialBadge component with urgency variants and DashboardLayout header integration
- [x] 37-04-PLAN.md -- Gap closure: DefaultQuoteSettingsCreatorService for auto-created quote template, REQUIREMENTS.md tracking update

### Phase 38: Hard Paywall and Soft Paywall Removal
**Goal**: Users with invalid subscriptions see a full-screen blocking page with contextual messaging and a path to resubscribe, replacing the soft modal that only blocked write actions
**Depends on**: Phase 36 (route guard architecture -- can run in parallel with Phase 37)
**Requirements**: PAYWALL-01, PAYWALL-02, PAYWALL-03, PAYWALL-04, PAYWALL-05, PAYWALL-06
**Success Criteria** (what must be TRUE):
  1. A user with an expired trial, failed payment, or canceled subscription sees a full-screen paywall page instead of the app
  2. The paywall displays differentiated messaging based on subscription state (trial expired vs payment failed vs canceled) and includes "your data is safe" reassurance
  3. The user can access the Stripe Billing Portal directly from the paywall screen to add a payment method or resubscribe
  4. Support role users bypass the hard paywall entirely and see the normal app
  5. The soft paywall modal, PersistentCta, DashboardBanner, SubscriptionGatedLayout, and all per-page `openPaywall()` dispatch calls are removed
**Plans**: 2 plans
Plans:
- [x] 38-01-PLAN.md -- PaywallPage component (3 variants), PaywallGuard blocking logic, useSubscription simplification, route tree adjustments
- [ ] 38-02-PLAN.md -- Delete soft paywall components (7 files), clean Redux store, remove all openPaywall dispatch calls from feature pages
**UI hint**: yes

### Phase 39: Welcome Dashboard and Final Cleanup
**Goal**: New users land on a personalised welcome experience that guides them toward their first job and quote, and all remnants of the old onboarding system are removed
**Depends on**: Phase 37 (display name for greeting, business existence for welcome variant)
**Requirements**: DASH-01, DASH-02, DASH-03
**Success Criteria** (what must be TRUE):
  1. A new user who just completed onboarding sees a personalised welcome message using their display name on the dashboard
  2. The dashboard shows a getting-started widget with "Create your first job" and "Send your first quote" checklist items that link to the relevant pages
  3. Checklist items show completion state (checked when the user has created a job or sent a quote) and the widget hides once all items are complete
**Plans**: 2 plans
Plans:
- [ ] 39-01-PLAN.md -- Welcome widget components (WelcomeSection, GettingStartedChecklist, ChecklistItem), useGettingStarted hook, DashboardPage integration
- [ ] 39-02-PLAN.md -- Old onboarding system removal (delete files, clean imports from App.tsx and store)
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order. Note: Phases 35+36 can run in parallel (different repos). Phases 37+38 can run in parallel (different concerns). Phase 39 depends on 37.

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
| 24. Playwright Bootstrap & Auth | v1.5 | 1/1 | Complete | 2026-03-27 |
| 25. API Seeding + Onboarding Tests | v1.5 | 1/2 | In Progress | |
| 26. Core Job Flow Tests | v1.5 | 0/? | Not started | - |
| 27. Quote Lifecycle Tests | v1.5 | 0/? | Not started | - |
| 28. Settings Tests + CI Integration | v1.5 | 0/? | Not started | - |
| 29. Subscription Module Foundation | v1.6 | 2/2 | Complete | 2026-03-29 |
| 30. Stripe Checkout and Webhooks | v1.6 | 3/3 | Complete | 2026-03-29 |
| 31. Subscription API Endpoints and Tests | v1.6 | 2/2 | Complete | 2026-03-29 |
| 32. Subscription Gate and Subscribe Pages | v1.6 | 3/3 | Complete | 2026-03-29 |
| 33. Trial Banner and Billing Settings Tab | v1.6 | 2/2 | Complete | 2026-03-29 |
| 34. Luxon DateTime Standardization | v1.6 | 2/2 | Complete | 2026-03-30 |
| 35. No-Card Trial API Endpoint | v1.7 | 2/2 | Complete    | 2026-04-02 |
| 36. Public Landing Page and Route Restructure | v1.7 | 2/2 | Complete    | 2026-04-07 |
| 37. Onboarding Wizard Pages | v1.7 | 4/4 | Complete    | 2026-04-07 |
| 38. Hard Paywall and Soft Paywall Removal | v1.7 | 1/2 | Complete    | 2026-04-02 |
| 39. Welcome Dashboard and Final Cleanup | v1.7 | 0/2 | Not started | - |
