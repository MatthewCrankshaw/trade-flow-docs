# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2020-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2020-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2020-03-15)
- v1.3 Send Quotes -- Phases 15-19 (shipped 2026-03-21)
- v1.4 Monorepo & Worker Infrastructure -- Phases 20-23 (shipped 2026-03-22)
- v1.5 Automated E2E Playwright Testing -- Phases 24-28 (in progress)
- v1.6 Stripe Subscription Billing -- Phases 29-33 (planned)

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

- [x] **Phase 24: Playwright Bootstrap & Auth** - Install and configure Playwright with global auth storageState (completed 2026-03-27)
- [ ] **Phase 25: API Seeding Infrastructure + Onboarding Tests** - Typed API client for test data seeding and onboarding flow tests
- [ ] **Phase 26: Core Job Flow Tests** - Customer, job, schedule, and quote creation tests
- [ ] **Phase 27: Quote Lifecycle Tests** - Email bypass, send flow, and customer accept/decline tests
- [ ] **Phase 28: Settings Tests + CI Integration** - Settings/inventory tests and GitHub Actions workflow

</details>

<details>
<summary>v1.6 Stripe Subscription Billing (Phases 29-33) -- PLANNED</summary>

- [x] **Phase 29: Subscription Module Foundation** - SubscriptionModule skeleton, MongoDB entity/repo, SubscriptionStatus enum, rawBody: true in main.ts, Stripe provider, checkout + webhook endpoints
- [ ] **Phase 30: Stripe Checkout and Webhooks** - POST /v1/subscription/checkout and POST /v1/webhooks/stripe with all 5 event handlers and upsert idempotency
- [ ] **Phase 31: Subscription API Endpoints and Tests** - GET/DELETE /v1/subscription, POST /v1/subscription/portal, and unit tests for all services and repository
- [ ] **Phase 32: Subscription Gate and Subscribe Pages** - SubscriptionGate component, /subscribe page, /subscribe/success, /subscribe/cancel, App.tsx route wiring, RTK Query subscriptionApi
- [ ] **Phase 33: Trial Banner and Billing Settings Tab** - TrialBanner component, Settings > Billing tab with SubscriptionStatusCard

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
- [x] 24-01-PLAN.md — Install Playwright, configure three-project layout, implement programmatic Firebase auth setup, and deliver smoke tests

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
- [x] 25-01-PLAN.md — Seeding infrastructure (api-client, db-cleanup, factories, fixtures), second user auth setup, auth redirect tests
- [ ] 25-02-PLAN.md — Onboarding wizard E2E test with default entity verification and cleanup

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

### Phase 34: Use luxon DateTime in DTOs, extend IBaseResourceDto, move createdAt/updatedAt to entity only

**Goal:** [To be planned]
**Requirements**: TBD
**Depends on:** Phase 33
**Plans:** 0 plans

Plans:
- [ ] TBD (run /gsd:plan-phase 34 to break down)

---

## v1.6 Stripe Subscription Billing — Phase Details

### Phase 29: Subscription Module Foundation
**Goal**: The API is structurally ready to handle Stripe — raw body preserved for webhook signature verification, Stripe SDK wired, and the subscription data model exists in MongoDB
**Depends on**: Nothing (first phase of v1.6)
**Repo**: trade-flow-api
**Requirements**: ACQ-02, WBHK-01
**Success Criteria** (what must be TRUE):
  1. `POST /v1/subscription/checkout` returns a Stripe Checkout Session URL (Stripe SDK live, env vars loaded)
  2. `POST /v1/webhooks/stripe` with an invalid or missing signature returns 400, confirming raw body is preserved and signature verification is active
  3. `SubscriptionStatus` enum values (`trialing`, `active`, `past_due`, `canceled`, `incomplete`) are defined and the `subscriptions` MongoDB collection has unique indexes on `userId` and `stripeSubscriptionId`
**Plans**: 2 plans
Plans:
- [x] 29-01-PLAN.md — Infrastructure + module skeleton: rawBody, Stripe SDK, enum, entity, DTO, repo, provider, module registration
- [x] 29-02-PLAN.md — Endpoints: SubscriptionController (checkout) and WebhookController (signature verification)

### Phase 30: Stripe Checkout and Webhooks
**Goal**: A user can complete Stripe Checkout and the webhook handler creates and keeps the local subscription record in sync across all five Stripe event types
**Depends on**: Phase 29
**Repo**: trade-flow-api
**Requirements**: ACQ-01, ACQ-03, ACQ-05, WBHK-02, WBHK-03, WBHK-04, WBHK-05, WBHK-06, WBHK-07
**Success Criteria** (what must be TRUE):
  1. After completing Stripe Checkout, the `checkout.session.completed` webhook creates a subscription record in MongoDB with status `trialing`
  2. Sending `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, and `invoice.payment_failed` webhook events each update the local record to the correct status
  3. Replaying the same webhook event twice produces one record, not two (upsert by `stripeSubscriptionId` is idempotent)
  4. `GET /v1/subscription/verify-session?sessionId=` returns the subscription status so the success page can confirm without waiting for the webhook
**Plans**: 3 plans
Plans:
- [x] 30-01-PLAN.md — Queue infrastructure, entity/repo extensions, webhook controller enqueue
- [x] 30-02-PLAN.md — StripeWebhookProcessor with 5 event handlers in WorkerModule
- [x] 30-03-PLAN.md — Verify-session endpoint and duplicate checkout guard

### Phase 31: Subscription API Endpoints and Tests
**Goal**: All subscription management endpoints are available and the service layer has unit test coverage
**Depends on**: Phase 30
**Repo**: trade-flow-api
**Requirements**: BILL-01, BILL-02, BILL-03, TEST-01, TEST-02, TEST-03, TEST-04
**Success Criteria** (what must be TRUE):
  1. `GET /v1/subscription` returns status, `currentPeriodEnd`, `trialEnd`, and `cancelAtPeriodEnd` for the authenticated user
  2. `DELETE /v1/subscription` sets `cancel_at_period_end: true` on Stripe and updates the local record — access is not immediately removed
  3. `POST /v1/subscription/portal` returns a Stripe Billing Portal URL the frontend can redirect to
  4. Unit tests for `SubscriptionCreatorService`, `SubscriptionUpdaterService`, `SubscriptionRetrieverService`, and `SubscriptionRepository` all pass with the Stripe SDK mocked
**Plans**: 2 plans
Plans:
- [x] 31-01-PLAN.md — Subscription management endpoints (GET, DELETE, POST portal), SubscriptionGuard, @SkipSubscriptionCheck decorator
- [x] 31-02-PLAN.md — Unit tests for all services, repository, controller webhook spec, and guard spec

### Phase 32: Subscription Gate and Subscribe Pages
**Goal**: Unauthenticated-subscription users are redirected to /subscribe and cannot access business routes, while users completing checkout land on a confirming success page
**Depends on**: Phase 31
**Repo**: trade-flow-ui
**Requirements**: GATE-01, GATE-02, GATE-03, GATE-04, GATE-05, ACQ-04
**Success Criteria** (what must be TRUE):
  1. Navigating to `/customers` (or any business route) without an active subscription redirects to `/subscribe` — the subscribe page shows a pricing card with a "Start free trial" CTA
  2. Users with a support role can navigate to all business routes without being redirected, regardless of subscription status
  3. `/subscribe/success` shows a loading state while polling, then confirms the subscription is active once the webhook has been processed
  4. `/subscribe/cancel` offers a "Try again" option that returns the user to `/subscribe`
  5. The Settings page is reachable without an active subscription, and no flash of the subscribe page occurs for users who already have an active subscription (loading skeleton shown during fetch)
**Plans**: 3 plans
Plans:
- [ ] 32-01-PLAN.md — Subscription feature foundation: types, RTK Query subscriptionApi, paywallSlice, SubscriptionContext, hooks
- [ ] 32-02-PLAN.md — UI components (PaywallModal, PersistentCta, DashboardBanner, PricingCard, SubscriptionGatedLayout), subscribe pages, App.tsx route wiring
- [ ] 32-03-PLAN.md — Wire paywall triggers into all business page write-action handlers
**UI hint**: yes

### Phase 33: Trial Banner and Billing Settings Tab
**Goal**: Trialing users see how many days remain in their trial, and any user can view and manage their subscription from Settings > Billing
**Depends on**: Phase 32
**Repo**: trade-flow-ui
**Requirements**: TRIAL-01, TRIAL-02, BILL-04, BILL-05
**Success Criteria** (what must be TRUE):
  1. A persistent banner appears for trialing users showing the number of days remaining and a "Subscribe" CTA — the banner is absent when subscription status is `active`
  2. Settings > Billing tab shows the current subscription status and next billing date (or trial end date)
  3. When `cancelAtPeriodEnd` is true, the Billing tab shows "Cancels on [date]" rather than "Active"
  4. "Manage Billing" button on the Billing tab redirects the user to the Stripe Billing Portal
**Plans**: TBD
**UI hint**: yes

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
| 24. Playwright Bootstrap & Auth | v1.5 | 1/1 | Complete   | 2026-03-27 |
| 25. API Seeding Infrastructure + Onboarding Tests | v1.5 | 1/2 | In Progress|  |
| 26. Core Job Flow Tests | v1.5 | 0/? | Not started | - |
| 27. Quote Lifecycle Tests | v1.5 | 0/? | Not started | - |
| 28. Settings Tests + CI Integration | v1.5 | 0/? | Not started | - |
| 29. Subscription Module Foundation | v1.6 | 2/2 | Complete    | 2026-03-29 |
| 30. Stripe Checkout and Webhooks | v1.6 | 3/3 | Complete   | 2026-03-29 |
| 31. Subscription API Endpoints and Tests | v1.6 | 2/2 | Complete   | 2026-03-29 |
| 32. Subscription Gate and Subscribe Pages | v1.6 | 0/3 | Not started | - |
| 33. Trial Banner and Billing Settings Tab | v1.6 | 0/? | Not started | - |
