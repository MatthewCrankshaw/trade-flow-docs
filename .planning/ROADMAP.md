# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2020-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2020-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2020-03-15)
- v1.3 Send Quotes -- Phases 15-19 (shipped 2026-03-21)
- v1.4 Monorepo & Worker Infrastructure -- Phases 20-23 (shipped 2026-03-22)
- v1.5 Automated E2E Playwright Testing -- Phases 24-28 (in progress)
- v1.6 Stripe Subscription Billing -- Phases 29-34 (shipped 2026-03-31)
- v1.7 Onboarding & Landing Page -- Phases 35-40 (shipped 2026-04-07)

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

<details>
<summary>v1.7 Onboarding & Landing Page (Phases 35-40) -- SHIPPED 2026-04-07</summary>

- [x] Phase 35: No-Card Trial API Endpoint (2/2 plans) -- completed 2026-04-02
- [x] Phase 36: Public Landing Page and Route Restructure (2/2 plans) -- completed 2026-04-07
- [x] Phase 37: Onboarding Wizard Pages (4/4 plans) -- completed 2026-04-07
- [x] Phase 38: Hard Paywall and Soft Paywall Removal (2/2 plans) -- completed 2026-04-02
- [x] Phase 39: Welcome Dashboard and Final Cleanup (2/2 plans) -- completed 2026-04-07
- [x] Phase 40: SubscriptionGuard Onboarding Bypass (1/1 plan) -- completed 2026-04-07

Full details: `.planning/milestones/v1.7-ROADMAP.md`

</details>

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
| 35. No-Card Trial API Endpoint | v1.7 | 2/2 | Complete | 2026-04-02 |
| 36. Public Landing Page and Route Restructure | v1.7 | 2/2 | Complete | 2026-04-07 |
| 37. Onboarding Wizard Pages | v1.7 | 4/4 | Complete | 2026-04-07 |
| 38. Hard Paywall and Soft Paywall Removal | v1.7 | 2/2 | Complete | 2026-04-02 |
| 39. Welcome Dashboard and Final Cleanup | v1.7 | 2/2 | Complete | 2026-04-07 |
| 40. SubscriptionGuard Onboarding Bypass | v1.7 | 1/1 | Complete | 2026-04-07 |