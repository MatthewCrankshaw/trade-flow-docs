# Milestones

## v1.4 Monorepo & Worker Infrastructure (Shipped: 2026-03-22)

**Phases completed:** 4 phases, 7 plans
**Timeline:** 1 day (2026-03-22)
**Codebase:** ~21.9k LOC API (TypeScript)

**Key accomplishments:**

- BullMQ/ioredis npm dependencies installed and `@queue/*`/`@worker/*` TypeScript path aliases registered across all compile targets (tsc, Jest, NestJS build)
- Redis 7.4 Alpine container added to Docker Compose with `maxmemory-policy noeviction` and `REDIS_URL` environment variable via ConfigService
- `QueueModule` with `BullModule.forRootAsync()`, `QueueProducer` service, and `QUEUE_NAMES` constants wired into `AppModule` as shared queue infrastructure
- `src/worker.ts` entry point via `NestFactory.createApplicationContext(WorkerModule)` with graceful SIGTERM/SIGINT shutdown and separate `worker-cli.json` build producing `dist/worker.js`
- `EchoProcessor` consuming echo queue via `@nestjs/bullmq WorkerHost` and `POST /v1/queue/test-echo` diagnostic endpoint proving end-to-end queue flow
- `nodemon-worker.json` with `worker:dev`, `worker:prod`, and `build:all` npm scripts for independent worker development and production startup
- Worker added as fourth Docker Compose service with multi-stage Dockerfile producing a production worker image running `node dist/worker.js`

---

## v1.3 Send Quotes (Shipped: 2026-03-21)

**Phases completed:** 5 phases, 18 plans, 35 tasks

**Key accomplishments:**

- Soft-delete support for quotes via DELETED status enum, DRAFT->DELETED transition, deletedAt timestamp, and list query exclusion filter
- Delete UI for draft quotes with detail page button, list row dropdown, confirmation dialog, optimistic removal, and toast notifications
- Cryptographically secure QuoteToken module with 32-byte base64url token generation, 30-day expiry, lookup/validation, and bulk revocation per quote
- Public GET /v1/public/quote/:token endpoint with @nestjs/throttler rate limiting, customer-safe response filtering, and automatic token revocation on quote deletion
- firstViewedAt tracking on quote tokens with enriched error responses and viewedAt on authenticated quote API
- Public quote page with responsive line items, error states, loading skeleton, disabled PDF button, and tradesperson viewed badge using separate unauthenticated RTK Query API slice
- Extracted QuoteSessionAuthGuard and PublicQuoteRetriever to restore strict Controller->Service->Repository layering for public quote endpoint
- 13 unit tests covering QuoteSessionAuthGuard token validation/expiry/first-view, PublicQuoteRetriever response mapping/filtering/bundles, and slimmed controller delegation
- QuoteEmailSender orchestration service with Maizzle HTML email template, SendGrid delivery, status transition, and full business entity extension for email template fields
- SendQuoteDialog with Tiptap rich text editor, template variable resolution, and RTK Query sendQuote mutation wired into QuoteActionStrip
- QuoteEmailSettings component with subject/message template fields, variable badges, and save-to-business via updateBusiness mutation, integrated as Quote Email tab in Settings page
- Hardened email sending with SendGrid error extraction (502/503) and split DLVR-01 into email-link vs PDF-attachment requirements
- Separate QuoteSettings module with GET/PATCH API, quote_settings MongoDB collection, and Business entity cleaned of email template fields
- QuoteSettings RTK Query endpoints with Business page Quote Email tab, replacing Settings page location and business-entity email fields
- Swapped email provider from SendGrid to Resend SDK with { data, error } return pattern, zero SendGrid references remaining
- Accept/decline POST endpoints on public controller with QuoteResponseHandler orchestrating status transitions, decline reason persistence, and failure-tolerant tradesperson notification emails
- Accept/decline button UI with inline decline form, expired notice, optimistic status banners, and RTK Query mutation wired to public quote endpoints
- 19 unit tests for QuoteResponseHandler (accept/decline/expiry/email isolation), NotificationEmailRenderer (template/XSS), and publicTransition (transitions/idempotency/no-auth)

---

## v1.2 Bundles & Quotes (Shipped: 2026-03-15)

**Phases completed:** 4 phases, 12 plans, 25 tasks
**Timeline:** 7 days (2026-03-08 to 2026-03-14)
**Execution time:** ~30 min (avg 3min/plan)
**Codebase:** ~19k LOC API + ~22.3k LOC UI (TypeScript)
**Git range:** feat(11-01) to feat(14-06) — 23 feature commits across 2 repos

**Key accomplishments:**

- Fixed bundle creation bug and built reusable SearchableItemPicker component (Popover+Command combobox)
- Full bundle component editing — add/remove/update quantities on existing bundles with two-line display and type badges
- Quote system wired to API — creation dialog (dual-context), list with real data and tab filtering, status transitions (Draft/Sent/Accepted/Rejected)
- Quote detail page with line item CRUD — add items/bundles, inline edit quantity/price, soft-delete, expandable bundle rows with native table sub-rows
- Bundle pricing strategy moved to item-level setting (BundleItemForm radio group) with immediate-add in quote flow
- Tax-inclusive line totals, correct bundle tax calculation (Money.percentageOf fix), responsive mobile card layout

---

## v1.1 Item Tax Rate Linkage (Shipped: 2026-03-08)

**Phases completed:** 2 phases, 6 plans
**Timeline:** 1 day (2026-03-08)
**Execution time:** ~1 hour (avg 4min/plan)
**Codebase:** ~18.2k LOC API + ~20.3k LOC UI (TypeScript)

**Key accomplishments:**

- Item entity migrated from numeric defaultTaxRate to taxRateId ObjectId reference across entity, DTO, repository, requests, response, and mappers
- Tax rate existence validation on both item create and update services with unit tests
- Onboarding flow reordered — tax rates created before items, default tax rate ID threaded to item creator
- Quote factories resolve tax rate percentage from taxRateId via TaxRateRepository
- Item forms (Material, Labour, Fee) show tax rate dropdown with "Name (rate%)" format and default pre-selection
- OpenAPI spec fully migrated from defaultTaxRate to taxRateId

---

## v1.0 Scheduling (Shipped: 2026-03-07)

**Phases completed:** 8 phases, 16 plans, 0 tasks

**Timeline:** 15 days (2026-02-21 to 2026-03-07)
**Execution time:** 1.5 hours (avg 6min/plan)
**Codebase:** ~17.4k LOC API + ~20.3k LOC UI (TypeScript)

**Key accomplishments:**

- Visit type system with CRUD backend, default generation per trade, and full management UI
- Schedule data model and API with cross-module validation, smart defaults, and structured filter parser
- Status state machine (Scheduled/Confirmed/Completed/Canceled/No-show) with validated transitions
- Schedule creation UI with calendar picker, time/duration selects, visit type dropdown, and empty state
- Responsive schedule list (table/card), detail dialog with editable notes, and status badges
- Edit mode, dropdown menu with status transitions, confirmation dialogs, and locked-state messaging
- Job detail integration replacing mock data with schedule-derived hints and per-status badge breakdown

---
