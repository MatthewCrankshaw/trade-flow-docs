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

### Active

<!-- Requirements for this milestone -- v1.6 Stripe Subscription Billing -->

- User is prompted to start a free trial (Stripe Checkout) after completing onboarding
- User can start a 30-day free trial by entering a card via Stripe Checkout
- User has full access to Trade Flow during the active trial period
- Stripe webhook creates and syncs local subscription record after checkout completes
- User is automatically charged £6/month after the 30-day trial ends
- User receives read-only access when trial expires, payment fails, or subscription is canceled
- User can subscribe (start trial) from a dedicated /subscribe page
- User can view their current subscription status in Settings > Billing
- User can cancel their subscription (cancels at period end, access continues until then)
- User can manage billing details (update card, view invoices) via Stripe Billing Portal
- Support role users bypass subscription gating entirely
- Trial banner shows days remaining during trial period

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
- **Codebase size:** ~21.9k LOC API (TypeScript) + ~23.9k LOC UI (TypeScript/TSX)
- **Worker infrastructure shipped (v1.4):** Dual entry-point NestJS monorepo with API + background worker connected via BullMQ/Redis. `npm run worker:dev` (hot reload) and `docker compose up` start all four services. Echo processor proves end-to-end queue flow; real processors (email, PDF) ship in future milestones
- **Monetization roadmap:** Future milestones will add Stripe subscription billing, async payment processing via BullMQ queues, and scheduled reminder emails -- v1.4 lays the infrastructure foundation

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

## Current Milestone: v1.6 Stripe Subscription Billing

**Goal:** Turn Trade Flow into a SaaS product — card-required 30-day free trial via Stripe Checkout (Stripe owns the trial), then £6/month recurring, with read-only enforcement when trial expires or payment fails, and a custom billing settings tab.

**Target features:**
- SubscriptionModule in trade-flow-api (MongoDB collection, userId-keyed)
- POST /v1/subscription/checkout → Stripe Checkout Session (trial_period_days: 30, card collected)
- Stripe webhook endpoint with raw body + signature verification; webhook creates/syncs local record
- Webhook event handling: subscription.created/updated/deleted, invoice.payment_failed/succeeded
- GET /v1/subscription, POST /v1/subscription/portal, DELETE /v1/subscription endpoints
- Unit tests for all services and repository
- SubscriptionGate component in trade-flow-ui (wraps business routes, redirects to /subscribe)
- Support role exemption bypasses SubscriptionGate
- /subscribe, /subscribe/success, /subscribe/cancel pages
- Persistent trial banner showing days remaining
- Settings > Billing tab with SubscriptionStatusCard (status, dates, cancel, manage billing CTA)
- Read-only mode when status is past_due, canceled, or trial expired

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
*Last updated: 2026-03-29 — Phase 32 complete: Subscription paywall gate, subscribe pages, and paywall triggers wired into all business page write-actions*
