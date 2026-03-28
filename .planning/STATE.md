---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
status: executing
stopped_at: Completed 25-api-seeding-onboarding-tests-01-PLAN.md
last_updated: "2026-03-28T09:13:43.386Z"
last_activity: 2026-03-28
progress:
  total_phases: 5
  completed_phases: 1
  total_plans: 3
  completed_plans: 2
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-27)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** Phase 25 — api-seeding-onboarding-tests

## Current Position

Phase: 25 (api-seeding-onboarding-tests) — EXECUTING
Plan: 2 of 2
Status: Ready to execute
Last activity: 2026-03-28

```
Progress: Phase 24 of 28
[░░░░░░░░░░░░░░░░░░░░░░░░░] 0% complete (0/5 phases)
```

## Performance Metrics

**Velocity (cumulative):**

- Total plans completed: 52 (16 v1.0 + 6 v1.1 + 12 v1.2 + 18 v1.3)
- Total execution time: ~4 hours

**By Milestone:**

| Milestone | Phases | Plans | Execution Time | Avg/Plan |
|-----------|--------|-------|---------------|----------|
| v1.0 Scheduling | 8 | 16 | 1.5 hours | 6min |
| v1.1 Item Tax Rate Linkage | 2 | 6 | ~1 hour | 4min |
| v1.2 Bundles & Quotes | 4 | 12 | ~30 min | 3min |
| v1.3 Send Quotes | 5 | 18 | ~1 hour | 3min |
| Phase 20-infrastructure-foundation P02 | 1min | 2 tasks | 2 files |
| Phase 20-infrastructure-foundation P01 | 2min | 2 tasks | 5 files |
| Phase 21-queue-module P01 | 4min | 2 tasks | 5 files |
| Phase 22 P01 | 2min | 2 tasks | 7 files |
| Phase 22 P02 | 2min | 1 tasks | 3 files |
| Phase 23 P01 | 1min | 1 tasks | 2 files |
| Phase 23-developer-experience P02 | 1min | 2 tasks | 2 files |
| Phase 24-playwright-bootstrap-auth P01 | 3 | 3 tasks | 9 files |
| Phase 25-api-seeding-onboarding-tests P01 | 5 | 3 tasks | 10 files |

## Accumulated Context

### Decisions

Key decisions archived in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- [v1.4]: Dual entry-point pattern over NestJS CLI monorepo mode -- preserves 20+ path aliases, avoids webpack
- [v1.4]: Redis `maxmemory-policy noeviction` required from day one -- BullMQ silently loses jobs otherwise
- [v1.4]: Worker uses `createApplicationContext()` -- no HTTP server needed
- [Phase 20-02]: No Redis volume -- ephemeral storage for transient BullMQ jobs
- [Phase 20-infrastructure-foundation]: BullMQ packages as production deps; path aliases only in tsconfig.json (build/check inherit via extends)
- [Phase 21-queue-module]: IORedis instance cast to ConnectionOptions via as unknown as to resolve type mismatch between top-level ioredis and bullmq bundled ioredis
- [Phase 22]: CoreModule included in WorkerModule -- MongoConnectionService requires MONGO_URL
- [Phase 22]: ThrottlerGuard overridden in unit test following PublicQuoteController test pattern
- [Phase 22]: deleteOutDir:false in worker-cli.json prevents clobbering dist/main.js during worker build
- [Phase 23]: Debug port 9230 for worker to avoid collision with API on 9229
- [Phase 23-02]: Worker reuses development Dockerfile stage -- no separate worker-specific development stage needed
- [v1.5 research]: Fixture pattern (test.extend) over Page Object Model -- composes better, Playwright-native lifecycle
- [v1.5 research]: Global auth setup project + storageState reuse -- prevents Firebase rate limiting from per-test login
- [v1.5 research]: API seeding via HTTP (not direct MongoDB) -- decouples tests from schema, auth via Firebase token
- [v1.5 research]: E2E_TEST_MODE bypass on QuoteEmailSender -- allows SENT status transition without real Resend call
- [v1.5 research]: Email delivery verification deferred -- test via seeded token URL on public quote page instead
- [Phase 24-playwright-bootstrap-auth]: Three-project Playwright layout (setup/chromium/chromium-unauth) with testMatch isolation prevents storageState leaking to unauthenticated tests
- [Phase 24-playwright-bootstrap-auth]: signInWithEmailAndPassword in page.evaluate chosen over signInWithCustomToken — custom tokens require Firebase Admin SDK and are a different token format
- [Phase 24-playwright-bootstrap-auth]: fileURLToPath(import.meta.url) for __dirname shim in ESM context — trade-flow-ui is type=module
- [Phase 25-api-seeding-onboarding-tests]: Token extraction via page.evaluate(getIdToken()) not from storageState JSON — avoids Firebase token expiry after 1h
- [Phase 25-api-seeding-onboarding-tests]: Dynamic businessId discovery via GET /v1/business in apiClient fixture — no hardcoded IDs
- [Phase 25-api-seeding-onboarding-tests]: D-03 auto-chaining in createJob: auto-creates customer when customerId omitted — simplifies job-centric test setup

### Pending Todos

None.

### Blockers/Concerns

- Firebase v12 IndexedDB persistence in Playwright storageState needs validation in Phase 24 -- fallback is signInWithCustomToken() in page evaluate
- Dedicated Firebase test user must be pre-created in Firebase Console before Phase 24 can run
- Two-repo CI checkout strategy (trade-flow-ui + trade-flow-api) needs validation in Phase 28

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

Last session: 2026-03-28T09:13:43.384Z
Stopped at: Completed 25-api-seeding-onboarding-tests-01-PLAN.md
Resume file: None
