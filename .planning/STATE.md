---
gsd_state_version: 1.0
milestone: v1.8
milestone_name: Estimates
status: executing
stopped_at: Phase 43 context gathered
last_updated: "2026-04-11T16:54:49.021Z"
last_activity: 2026-04-11 -- Phase 41 planning complete
progress:
  total_phases: 7
  completed_phases: 0
  total_plans: 10
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-07)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** v1.8 Estimates -- roadmap restructured 2026-04-11, ready to plan new Phase 41

## Current Position

Phase: Phase 41 (Estimate Module CRUD (Backend)) -- not started
Plan: —
Status: Ready to execute
Last activity: 2026-04-11 -- Phase 41 planning complete

**Next step:** `/gsd-discuss-phase 41` to gather context for the new Phase 41 (Estimate Module CRUD (Backend)) which now absorbs the document-token rename and the new `estimate_line_items` module.

## Roadmap Summary

**Milestone:** v1.8 Estimates
**Phases:** 41-47 (7 phases, restructured 2026-04-11)
**Requirements:** 58/58 mapped (100% coverage)
**Granularity:** fine
**Full detail:** `.planning/milestones/v1.8-ROADMAP.md`

| Phase | Goal | Requirements | Status |
|-------|------|--------------|--------|
| 41. Estimate Module CRUD (Backend) | Trader can CRUD estimates with E-YYYY-NNN numbering and validated lifecycle; document-token unified; estimate_line_items module created | EST-01..09, CONT-01/02/05, RESP-08 | Not started |
| 42. Revisions | Invisible versioned revisions with partial unique index and history | REV-01..05 | Not started |
| 43. Estimate Frontend CRUD | Visual create/edit/list/detail with contingency slider and range display | CONT-03/04 | Not started |
| 44. Email & Send Flow | Send estimate with mandatory non-binding legal copy and audit HTML; new estimate-settings module | SND-01..07 | Not started |
| 45. Public Customer Page & Response Handling | Latest-revision resolution, 4-button response flow, structured decline | CUST-01..07, RESP-01..07 | Not started |
| 46. Follow-up Queue & Automation | BullMQ delayed 3/10/21d follow-ups with cancel-on-exit and AOF infra gate | FUP-01..08 | Not started |
| 47. Convert to Quote & Mark as Lost | Idempotent convert with mandatory review, back-link, markLost | CONV-01..06, LOST-01/02 | Not started |

**Critical path (estimates work end to end):** 41 -> 44 -> 45 -> 46 -> 47
**Parallel:** Phase 42 (revisions) and Phase 43 (frontend CRUD) can run in parallel after Phase 41 lands.

## Performance Metrics

**Velocity (cumulative):**

- Total plans completed: 69 (16 v1.0 + 6 v1.1 + 12 v1.2 + 18 v1.3 + 7 v1.4 + 7 v1.6-partial)
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
| v1.7 Onboarding & Landing | 6 | 13 | (see phase rows below) | — |
| v1.8 Estimates | 8 | TBD | — | — |
| Phase 35 P01 | 4min | 2 tasks | 4 files |
| Phase 35 P02 | 2min | 1 tasks | 2 files |
| Phase 37 P01 | 4min | 2 tasks | 13 files |
| Phase 37 P03 | 2min | 2 tasks | 4 files |
| Phase 38 P01 | 3min | 2 tasks | 5 files |
| Phase 38 P02 | 5min | 2 tasks | 19 files |
| Phase 37 P04 | 3min | 2 tasks | 5 files |

## Accumulated Context

### Decisions

Key decisions archived in PROJECT.md Key Decisions table.
v1.7 decisions archived with milestone completion.

### Roadmap Evolution

v1.8 roadmap created 2026-04-10:

- 8 phases (41-48) derived from 10 requirement categories
- Separate EstimateModule pattern chosen over polymorphic discriminator (see ARCHITECTURE.md §10)
- Foundations-first refactor (Phase 41) before any estimate code is written
- Original Phase 47 gated on production Redis AOF persistence (FUP-08 is a hard infra gate)
- Original Phase 45 gated on legal-review pass of non-binding email copy (SND-05 is non-removable)

v1.8 roadmap restructured 2026-04-11 (during /gsd-discuss-phase 41):

- Phase 41 (Foundations & Refactor) dissolved; renumbered to 7 phases (41-47)
- Separation-of-concerns chosen over DRY for entity-level modules: `quote_line_items` untouched; new `estimate_line_items` collection and module (absorbed into new Phase 41). `quote-settings` untouched; new `estimate-settings` module (absorbed into new Phase 44).
- Tokens remain unified: `quote-token` → `document-token` with `documentType` discriminator (absorbed into new Phase 41).
- Phase 46 still gated on Redis AOF persistence; Phase 44 still gated on UK-consumer-law legal review.
- See `.planning/notes/2026-04-11-v1.8-restructure-decisions.md` for full rationale.

### Pending Todos

None.

### Blockers/Concerns

- v1.5 Playwright testing (Phases 25-28) still in progress -- paused during v1.6/v1.7 work
- **Phase 45 legal-review gate**: Default estimate email template copy must pass UK-consumer-law review before Phase 45 can ship
- **Phase 47 infra gate**: Production Redis must have `appendonly yes` / `appendfsync everysec` before Phase 47 can ship (FUP-08)

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
| 260407-n0u | Map Resend email delivery errors to proper HTTP response codes | 2026-04-07 | 612bc03, 9747b79 | [260407-n0u-map-resend-email-delivery-errors-to-prop](./quick/260407-n0u-map-resend-email-delivery-errors-to-prop/) |
| 260407-orl | Set up Vitest test runner for trade-flow-ui | 2026-04-07 | 0d03ee1, ef96862 | [260407-orl-implement-the-capabilities-to-be-able-to](./quick/260407-orl-implement-the-capabilities-to-be-able-to/) |
| 260407-pwg | Document testing standards and tools in CLAUDE.md | 2026-04-07 | 586631e, 0447cbd | [260407-pwg-document-the-testing-standards-and-testi](./quick/260407-pwg-document-the-testing-standards-and-testi/) |
| 260407-rp3 | Ensure API and UI have passing unit tests, prettier, linting checks and Railway pre-build CI gate | 2026-04-07 | 3e7de43 | [260407-rp3-ensure-api-and-ui-have-passing-unit-test](./quick/260407-rp3-ensure-api-and-ui-have-passing-unit-test/) |
| 260407-s9g | Fix Railway build to run npm run ci instead of npm ci before deployment | 2026-04-07 | 9642daa, d3a8f6f | [260407-s9g-fix-railway-build-to-run-npm-run-ci-inst](./quick/260407-s9g-fix-railway-build-to-run-npm-run-ci-inst/) |
| 260410-tkt | Review onboarding defect fixes and refactor for better holistic solution | 2026-04-10 | 8715966 (api), 380345b (ui) | [260410-tkt-review-onboarding-defect-fixes-and-refac](./quick/260410-tkt-review-onboarding-defect-fixes-and-refac/) |
| 260411-a41 | Review recent commits against new comments code standard and refactor | 2026-04-11 | fa07323 (api), aa44616 (ui) | [260411-a41-review-recent-commits-against-new-commen](./quick/260411-a41-review-recent-commits-against-new-commen/) |

## Session Continuity

Last session: 2026-04-11T16:54:49.015Z
Stopped at: Phase 43 context gathered
Resume file: .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
