---
gsd_state_version: 1.0
milestone: v1.7
milestone_name: Onboarding & Landing Page
status: verifying
stopped_at: Completed 36-01-PLAN.md
last_updated: "2026-04-02T11:19:06Z"
last_activity: 2026-04-02 -- Phase 36-01 landing page components complete
progress:
  total_phases: 10
  completed_phases: 3
  total_plans: 14
  completed_plans: 5
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-31)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** Phase 35 — no-card-trial-api-endpoint

## Current Position

Phase: 36
Plan: Not started
Status: Phase complete — ready for verification
Last activity: 2026-04-02

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity (cumulative):**

- Total plans completed: 66 (16 v1.0 + 6 v1.1 + 12 v1.2 + 18 v1.3 + 7 v1.4 + 7 v1.6-partial)
- Total execution time: ~5 hours

**By Milestone:**

| Milestone | Phases | Plans | Execution Time | Avg/Plan |
|-----------|--------|-------|---------------|----------|
| v1.0 Scheduling | 8 | 16 | 1.5 hours | 6min |
| v1.1 Item Tax Rate Linkage | 2 | 6 | ~1 hour | 4min |
| v1.2 Bundles & Quotes | 4 | 12 | ~30 min | 3min |
| v1.3 Send Quotes | 5 | 18 | ~1 hour | 3min |
| v1.4 Worker Infrastructure | 4 | 7 | ~13min | 2min |
| v1.6 Stripe Billing | 6 | 14 | ~55min | 4min |
| Phase 35 P01 | 4min | 2 tasks | 4 files |
| Phase 35 P02 | 2min | 1 tasks | 2 files |

## Accumulated Context

### Decisions

Key decisions archived in PROJECT.md Key Decisions table.

- [Phase 35]: Used SUBSCRIPTION_ALREADY_ACTIVE error code for trial duplicate guard; Stripe v21+ item-level current_period_end pattern; Luxon DateTime.fromSeconds for Stripe timestamps
- [Phase 35]: customer.subscription.created handler reuses extractSubscriptionDates for consistent date mapping across all webhook handlers

### Roadmap Evolution

- Phases 35+36 can run in parallel (API vs UI repos)
- Phases 37+38 can run in parallel (onboarding vs paywall concerns)
- Phase 39 depends on Phase 37 (displayName feeds welcome greeting)

### Pending Todos

None.

### Blockers/Concerns

- v1.5 Playwright testing (Phases 25-28) still in progress -- paused during v1.6 work
- Existing user migration: mandatory onboarding guard must pass users who already have profile + business + subscription
- Stripe webhook compatibility: `customer.subscription.created` must handle both Checkout-created and API-created subscriptions
- `past_due` grace period duration is a product decision needed during Phase 38 planning

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 1 | Enforce Luxon DateTime usage in DTOs for dates and durations in trade-flow-api | 2026-03-01 | 67894c5 | [1-enforce-luxon-datetime-usage-in-dtos-for](./quick/1-enforce-luxon-datetime-usage-in-dtos-for/) |
| 2 | Merge schedule date and startTime into startDateTime | 2026-03-01 | 0b2933f | [2-merge-schedule-date-and-starttime-into-a](./quick/2-merge-schedule-date-and-starttime-into-a/) |
| 3 | Fix bundle fixed-price: add bundlePrice input field | 2026-03-15 | ef4ec77, fcbae3c | [3-fix-bundle-fixed-price-add-bundleprice-i](./quick/3-fix-bundle-fixed-price-add-bundleprice-i/) |
| 4 | Fix quote money display: formatDecimal -> formatAmount | 2026-03-15 | a34dbea, 2e4d839 | [4-fix-quote-money-display-convert-minor-un](./quick/4-fix-quote-money-display-convert-minor-un/) |
| 5 | Fix Dinero.js invalid amount: round non-integer values | 2026-03-15 | 30247f4 | [5-fix-dinero-js-invalid-amount-round-non-i](./quick/5-fix-dinero-js-invalid-amount-round-non-i/) |
| 6 | Show bundle unit price in quote line items | 2026-03-15 | ee013fe, 0def7cc | [6-show-bundle-unit-price-in-quote-line-ite](./quick/6-show-bundle-unit-price-in-quote-line-ite/) |
| 7 | Fix quote price input showing minor units | 2026-03-15 | a52b32d | [7-fix-quote-price-input-showing-minor-unit](./quick/7-fix-quote-price-input-showing-minor-unit/) |
| 8 | Simplify quote creation: select job first, auto-resolve customer | 2026-03-21 | e919d82, 0724df1 | [260321-thu-simplify-quote-creation-select-job-first](./quick/260321-thu-simplify-quote-creation-select-job-first/) |
| 9 | Auto-select newly created job in quote dialog | 2026-03-21 | 7689fdd, 1415055 | [260321-tua-auto-select-newly-created-job-in-quote-c](./quick/260321-tua-auto-select-newly-created-job-in-quote-c/) |
| 10 | Investigate and resolve npm package vulnerabilities | 2026-03-25 | 8db0d7a (api), 7fe865d (ui) | [260325-tsb-investigate-and-resolve-npm-package-vuln](./quick/260325-tsb-investigate-and-resolve-npm-package-vuln/) |

## Session Continuity

Last session: 2026-04-02T11:19:06Z
Stopped at: Completed 36-01-PLAN.md
Resume file: None
