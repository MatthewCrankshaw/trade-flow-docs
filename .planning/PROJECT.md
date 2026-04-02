# Trade Flow

## What This Is

A business management application for sole tradespeople -- plumbers, electricians, builders, and other independent contractors -- that replaces scattered tools (paper notes, spreadsheets, generic invoicing apps, WhatsApp, calendar apps) with one streamlined system built around how trades actually work. Everything connects to the job: quotes, schedules, materials, labour, invoices, payments, and customer history. Tradespeople can now send quotes to customers via email, and customers can accept or reject quotes directly from a secure online link.

Two independent codebases: `trade-flow-api` (NestJS/MongoDB) and `trade-flow-ui` (React/Vite), each with their own git repo, managed and deployed independently. Feature branches are created in both repos for coordinated feature work.

## Core Value

A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

- ✓ User can sign up and authenticate via Firebase -- existing
- ✓ User can create and manage a business -- existing
- ✓ User can create and manage customers -- existing
- ✓ User can create and manage jobs (draft status, job types, descriptions) -- existing
- ✓ User can manage inventory items -- existing
- ✓ User can manage tax rates -- existing
- ✓ User can create quotes on jobs -- existing (basic display)
- ✓ Default job types and items generated per trade on business creation -- existing
- ✓ Onboarding flow guides new users through setup -- existing
- ✓ Email sending via Resend for verification and notifications -- existing (migrated from SendGrid in v1.3)
- ✓ User can create schedule entries on a job with date, start time, and duration -- v1.0
- ✓ User can view all schedule entries for a job in chronological order -- v1.0
- ✓ User can edit existing schedule entries (date, time, duration, visit type) -- v1.0
- ✓ User can cancel schedule entries (status change, entry preserved) -- v1.0
- ✓ User can add free-text notes to a schedule entry -- v1.0
- ✓ Schedule assignee defaults to logged-in user -- v1.0
- ✓ Duration defaults to 1 hour when not specified -- v1.0
- ✓ Schedule status lifecycle: Scheduled, Confirmed, Completed, Canceled, No-show -- v1.0
- ✓ Valid status transitions enforced by API with clear error messages -- v1.0
- ✓ User can select visit type when creating a schedule entry -- v1.0
- ✓ Default visit types generated per trade on business creation -- v1.0
- ✓ User can create custom visit types -- v1.0
- ✓ User can view and manage visit types for their business -- v1.0
- ✓ Schedule entries replace mock data in job detail page -- v1.0
- ✓ Job detail shows schedule count/status summary -- v1.0
- ✓ Empty state displayed when job has no schedule entries -- v1.0
- ✓ Items reference tax rates by ID instead of storing a numeric value -- v1.1
- ✓ Item create/edit forms show tax rate dropdown from business tax rates -- v1.1
- ✓ Item list/detail views unchanged (tax rate only visible in edit form) -- v1.1
- ✓ Item create/update validates referenced tax rate exists -- v1.1
- ✓ Default items during onboarding reference the correct default tax rate -- v1.1
- ✓ Quote line item factories resolve tax rate percentage from taxRateId -- v1.1
- ✓ User can create a bundle item without errors (unit defaults to "bundle") -- v1.2
- ✓ User can edit bundle components on an existing bundle (add/remove items, change quantities) -- v1.2
- ✓ User can search and filter items when selecting bundle components via a searchable dropdown -- v1.2
- ✓ User can view bundle components in a structured list showing item name, quantity, and unit -- v1.2
- ✓ User can create a new quote linked to a job and customer -- v1.2
- ✓ User can view a list of all quotes with real API data -- v1.2
- ✓ User can view quote detail with line items and calculated totals -- v1.2
- ✓ User can transition quote status (Draft → Sent → Accepted/Rejected) -- v1.2
- ✓ User can add standard items (material, labour, fee) to a quote -- v1.2
- ✓ User can add bundle items to a quote (creates parent + component line items) -- v1.2
- ✓ User can view bundle line items as a rolled-up line that expands to show individual components -- v1.2
- ✓ User can view quote totals (subtotal, tax, total) calculated from line items -- v1.2
- ✓ User can delete a quote (soft delete, only from Draft status) -- v1.3
- ✓ User can send a quote to a customer via email with a link to view the quote online -- v1.3
- ✓ User can configure a default quote email template in business settings with variable placeholders -- v1.3
- ✓ User can review and edit the pre-filled email message in a send dialog before sending -- v1.3
- ✓ User can re-send a quote already in Sent status -- v1.3
- ✓ User can see when a customer has viewed the quote online (viewed indicator with timestamp) -- v1.3
- ✓ Customer can view the full quote online via a secure link without logging in -- v1.3
- ✓ Customer can accept a quote with one click from the online view -- v1.3
- ✓ Customer can decline a quote with one click and provide an optional reason -- v1.3
- ✓ Quote status automatically transitions based on actions (Draft→Sent on send, Sent→Accepted/Rejected on customer response) -- v1.3
- ✓ Tradesperson receives email notification when a customer accepts or declines a quote -- v1.3
- ✓ BullMQ/ioredis queue infrastructure installed with Redis 7.4 in Docker Compose -- v1.4
- ✓ QueueModule with BullModule.forRootAsync, QueueProducer, and QUEUE_NAMES constants -- v1.4
- ✓ Worker entry point (src/worker.ts) via createApplicationContext with SIGTERM shutdown -- v1.4
- ✓ WorkerModule with EchoProcessor proving end-to-end queue flow -- v1.4
- ✓ Dual entry-point build (dist/main.js + dist/worker.js) via worker-cli.json -- v1.4
- ✓ worker:dev (nodemon hot reload), worker:prod, build:all npm scripts -- v1.4
- ✓ Worker as fourth Docker Compose service with multi-stage production Dockerfile -- v1.4
- ✓ User can start a 30-day free trial via Stripe Checkout (card required) -- v1.6
- ✓ Stripe webhook creates and syncs local subscription record (5 event types) -- v1.6
- ✓ User is charged £6/month after trial ends -- v1.6
- ✓ Paywall enforces read-only when trial expires, payment fails, or subscription canceled -- v1.6
- ✓ User can subscribe from /subscribe page with pricing card -- v1.6
- ✓ User can cancel subscription (cancels at period end, access continues) -- v1.6
- ✓ Support role users bypass subscription gating entirely -- v1.6
- ✓ Trial banner shows days remaining during trial period -- v1.6
- ✓ User can view subscription status in Settings > Billing -- v1.6
- ✓ User can manage billing via Stripe Billing Portal -- v1.6
- ✓ Luxon DateTime standardized across full stack (API + UI) -- v1.6

### Active

<!-- Current scope. Building toward these. -->

## Current Milestone: v1.7 Onboarding & Landing Page

**Goal:** Replace the dismissible onboarding flow with a mandatory, streamlined setup process — from public landing page through profile, business, and no-card trial activation — and enforce a hard paywall for invalid subscriptions.

**Target features:**
- Public landing page (single-page marketing at root path, no auth required)
- Mandatory profile setup (name required)
- Mandatory business setup (business name + primary trade, UK/GBP defaults, auto-creates all defaults)
- No-card-upfront free trial via Stripe (30-day trial, add card via Billing Portal later)
- Welcome dashboard with greeting and first-steps guidance
- Getting-started widget (create job, create quote) using existing onboarding widget pattern
- Hard paywall replacing soft write-action modal (blocking screen when subscription invalid)
- Remove existing dismissible onboarding flow

### Out of Scope

<!-- Explicit boundaries. -->

- Quote PDF generation -- deferred from v1.3, future milestone candidate
- Quote duplication -- future milestone candidate
- Quote-level discounts -- future milestone candidate
- Invoice generation and management -- next milestone candidate
- Payment tracking -- next milestone candidate
- Standalone calendar or "Today" screen -- future milestone (schedules show on job page only)
- Arrival window scheduling mode -- v2 scheduling (SMODE-01/02/03)
- Conflict detection/overlap warnings -- v2 scheduling (CONF-01/02/03)
- Customer notifications for schedule entries -- future
- Required items/materials on schedule entries -- future
- Team member assignment on schedules -- future (solo operator only for now)
- File/photo uploads on jobs -- separate feature
- Job notes (non-schedule) -- separate feature
- Data export/backup -- future
- Audit logging -- future
- Drag-and-drop calendar -- over-engineering for current use case
- Route optimization -- solo operator with local work
- Automated scheduling / AI -- tradespeople want control
- Recurring schedules / templates -- trades jobs are mostly one-off
- Customer-facing booking portal -- tradespeople get work via calls/referrals
- Google Calendar integration -- schedules are job-centric, not calendar-centric
- Customer account/login system -- token-based access is frictionless
- Electronic signature on quote acceptance -- legal complexity varies by jurisdiction
- Automated follow-up reminder emails -- adds scheduling infrastructure; premature
- Rich HTML quote template builder -- template builders are entire products
- Deposit collection on acceptance -- requires invoicing system (not yet built)
- Custom card input form (Stripe Elements) -- hosted Checkout handles PCI/SCA/3DS
- Custom dunning emails -- Stripe built-in dunning handles retries
- Multiple pricing tiers / annual billing -- single £6/month plan for now

## Context

- **Brownfield project:** Existing codebase with established patterns (see `.planning/codebase/`)
- **Backend pattern:** Strict Controller -> Service -> Repository layering (NestJS)
- **Frontend pattern:** Feature-based modules with Redux RTK Query for server state
- **Auth:** Firebase JWT (RS256) with server-side public key validation
- **API contract:** Standardized response format: `{ data: T[], pagination?, errors? }`
- **Email:** Resend SDK for transactional email (quote delivery, notifications) with Maizzle HTML templates
- **Public access:** Cryptographic token-based access for customer-facing quote pages (no login required)
- **Scheduling shipped (v1.0):** Visit types (CRUD + defaults per trade) and schedules (create/list/edit/cancel/status transitions) fully integrated into job detail page
- **Item tax rate linkage shipped (v1.1):** Items reference tax rates by ID; API validates references; UI shows tax rate dropdown on item forms; quote factories resolve rates
- **Bundles & Quotes shipped (v1.2):** Bundle creation/editing with SearchableItemPicker, quote system with creation dialog, list, detail view, line item management (add/edit/delete), expandable bundle rows, tax-inclusive totals, mobile-responsive card layout, status transitions (Draft/Sent/Accepted/Rejected)
- **Send Quotes shipped (v1.3):** Quote deletion, token infrastructure, customer-facing quote page with view tracking, email sending with configurable templates and Tiptap rich text editor, customer accept/decline with notification emails
- **Worker infrastructure shipped (v1.4):** Dual entry-point NestJS monorepo with API + background worker connected via BullMQ/Redis
- **E2E testing in progress (v1.5):** Playwright bootstrapped with global auth setup, API seeding infrastructure, partial test coverage
- **Stripe billing shipped (v1.6):** Full SaaS billing — Stripe Checkout with 30-day trial, webhook processing via BullMQ (5 event types), subscription management API (GET/DELETE/portal), frontend paywall with soft modal for write actions, trial chip, Settings > Billing tab
- **Luxon standardized (v1.6):** All DTOs use Luxon DateTime (no native Date), shared toDateTime utility, date-helpers module in UI
- **Codebase size:** ~22k LOC API (TypeScript) + ~24k LOC UI (TypeScript/TSX)

## Constraints

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns -- see `.planning/codebase/CONVENTIONS.md`
- **Two repos:** Changes span `trade-flow-api` and `trade-flow-ui` with coordinated feature branches
- **Solo operator:** Schedule assignee is always the logged-in user for now, but data model supports team assignment later
- **No standalone calendar:** Schedules only appear within a job's detail page (v1.0 scope)

## Key Decisions

<!-- Decisions that constrain future work. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Start Time + Duration (not Start + End) | Matches how tradespeople think -- "I'll be there at 9, it'll take 2 hours" | ✓ Good -- natural UX |
| Two scheduling modes deferred to v2 | Keep v1 scope tight; exact time covers majority of use cases | ✓ Good -- shipped faster |
| Warn on conflicts deferred to v2 | Tradespeople sometimes intentionally double-book; advisory-only adds complexity | ✓ Good -- no blocker |
| Schedules on job page only (no calendar) | Keep scope tight; "Today" screen is a future milestone | ✓ Good -- job-centric focus |
| Visit types generated per trade | Consistent with existing job types pattern; reduces setup friction | ✓ Good -- zero-config onboarding |
| Data model supports team assignment | Even though solo-only now, avoid costly migration later | ✓ Good -- future-proofed |
| Separate MongoDB collection for schedules | Not embedded in Job -- allows independent querying and indexing | ✓ Good -- clean separation |
| Structured filter format (filter:field:op=value) | Project-wide API filtering standard from Phase 4 | ✓ Good -- reusable pattern |
| Merged date+startTime into single startDateTime | ISO8601 field simplifies parsing and timezone handling | ✓ Good -- cleaner API |
| Luxon DateTime enforced in all DTOs | Consistent date handling across API | ✓ Good -- eliminated date bugs |
| taxRateId reference instead of numeric defaultTaxRate | Proper relational model; tax rate changes propagate automatically | ✓ Good -- clean data model |
| Client-side tax rate resolution via RTK Query cache | No server joins needed; UI resolves tax rate details from cache | ✓ Good -- simpler API |
| TaxRateRepository directly (not RetrieverService) for validation | Avoids unnecessary auth check; item creation itself is business-scoped | ✓ Good -- simpler validation |
| Tax rate only visible in edit form (not list/detail) | User decision: tax rate is secondary detail | ✓ Good -- clean UI |
| SearchableItemPicker in src/components/ for reuse | Used in both bundle forms and quote add-item flow | ✓ Good -- reused across features |
| Atomic quote_counters collection for Q-YYYY-NNN | Sequential numbering without race conditions | ✓ Good -- reliable |
| Denormalized customerName/jobTitle in quote response | Avoids N+1 queries on quote list | ✓ Good -- fast list rendering |
| Quote totals recalculated on every read (not persisted) | Single source of truth from line items; no stale totals | ✓ Good -- always correct |
| Soft delete via DELETED status enum | Preserves line item history; totals exclude deleted items | ✓ Good -- clean audit trail |
| Pricing strategy is item-level (not per-quote) | API reads bundleConfig.priceStrategy when no override sent | ✓ Good -- simpler quote flow |
| Native TableRow sub-rows for bundle components | Browser table layout aligns columns automatically | ✓ Good -- replaced CSS grid hack |
| lineTotalIncTax helper for display calculation | Tax-inclusive totals centralised; column header "Total (inc. tax)" | ✓ Good -- clear UX |
| Token-based public access (no customer login) | Frictionless for tradespeople's customers; one-time interactions per quote | ✓ Good -- zero friction |
| Separate QuoteSettings module (not Business entity) | Email template fields don't belong on Business; clean separation of concerns | ✓ Good -- cleaner architecture |
| Resend SDK instead of SendGrid | Simpler { data, error } pattern; better DX | ✓ Good -- cleaner integration |
| Maizzle + Tailwind for email templates | Aligns with frontend design system; proper email CSS inlining | ✓ Good -- consistent styling |
| Separate publicQuoteApi RTK Query slice | Unauthenticated endpoints must not send Firebase JWT headers | ✓ Good -- no auth leakage |
| QuoteSessionAuthGuard for public endpoints | Reusable guard for token validation, expiry, first-view tracking | ✓ Good -- DRY |
| Failure-tolerant notification emails | Email failure doesn't block status transition; try/catch with logging | ✓ Good -- resilient |
| PDF generation deferred from v1.3 | Reduced scope to ship core send/respond flow faster | — Pending review |
| Dual entry-point over NestJS CLI monorepo mode | CLI monorepo switches to webpack, breaking 20+ path aliases; massive migration for zero benefit | ✓ Good -- clean solution |
| Redis maxmemory-policy noeviction from day one | BullMQ silently loses jobs if Redis evicts keys under memory pressure | ✓ Good -- avoids silent data loss |
| Worker uses createApplicationContext (no HTTP server) | Workers process jobs, not HTTP requests; lighter startup and cleaner separation | ✓ Good -- right abstraction |
| CoreModule in WorkerModule | MongoConnectionService in CoreModule requires MONGO_URL; worker needs DB access | ✓ Good -- pragmatic |
| deleteOutDir:false in worker-cli.json | Prevents worker build from deleting dist/main.js; both entry points must coexist | ✓ Good -- critical for dual-build |
| Debug port 9230 for worker | Avoids port collision with API debugging on 9229 | ✓ Good -- clear convention |
| No Redis volume in Docker Compose | BullMQ jobs are transient in dev; ephemeral Redis is simpler and sufficient | ✓ Good -- right trade-off |
| Stripe Checkout (hosted) over Elements | Zero PCI scope, SCA/3DS handled automatically, no frontend Stripe packages | ✓ Good -- fastest path |
| Local MongoDB subscription record | Avoids Stripe API call on every gated request; webhook keeps in sync | ✓ Good -- fast reads |
| rawBody: true globally | Required for Stripe webhook signature verification; must be first task | ✓ Good -- critical |
| cancel_at_period_end (not immediate cancel) | User retains access until billing period end | ✓ Good -- fair UX |
| BullMQ for webhook processing | Async processing with deduplication (jobId: event.id) and retry semantics | ✓ Good -- resilient |
| Verify-session endpoint | Race condition safety net when webhook arrives after redirect (1-30s) | ✓ Good -- smooth UX |
| Soft paywall modal (not hard gate) for write actions | Non-disruptive; user can still browse data but can't create/edit | ✓ Good -- balanced |
| SubscriptionGuard on API (not just frontend) | Server-side enforcement prevents bypass via direct API calls | ✓ Good -- secure |
| Luxon DateTime in all DTOs (no native Date) | Consistent date handling across full stack; eliminated date parsing bugs | ✓ Good -- clean |

## Current State

**Shipped:** v1.6 Stripe Subscription Billing (2026-03-31)
**In progress:** v1.7 Onboarding & Landing Page

Trade Flow is now a monetized SaaS product with full billing infrastructure. The core product flow — from customer management through job tracking, quoting, and payment — is complete. Subscription billing via Stripe handles trial, payment, and access enforcement. Now rebuilding the onboarding experience and adding a public landing page.

Phase 35 complete (2026-04-02) — No-card trial API endpoint. POST /v1/subscription/trial creates a 30-day Stripe trial without payment details. Webhook handler for customer.subscription.created ensures belt-and-suspenders reliability.

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-02 after Phase 35 completion — no-card trial API endpoint*
