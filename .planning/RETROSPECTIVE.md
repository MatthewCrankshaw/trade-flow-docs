# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 -- Scheduling

**Shipped:** 2026-03-07
**Phases:** 8 | **Plans:** 16 | **Execution time:** 1.5 hours

### What Was Built
- Visit type system: CRUD backend with default generation per trade, full management UI with color picker and status filtering
- Schedule module: data model, create/list/edit/cancel API with status state machine and structured filter parser
- Schedule UI: creation form with calendar, responsive list/detail views, edit mode, status transitions with confirmations
- Job detail integration: schedule-derived header hints and per-status badge breakdown replacing all mock data

### What Worked
- Consistent module pattern (visit-type -> schedule) made later phases faster -- avg plan time dropped from 10min (Phase 1) to 2min (Phase 8)
- Planning docs (CONTEXT.md, RESEARCH.md) gave each phase clear boundaries -- no scope creep during execution
- Two-repo architecture worked smoothly with coordinated feature work
- Smart defaults (1hr duration, auto-assignee) reduced required fields and simplified the creation form
- Deferring arrival window mode and conflict detection to v2 kept v1 focused and shippable

### What Was Inefficient
- ROADMAP.md plan checkboxes got out of sync with actual completion (some phases show unchecked plans that were completed) -- needs automated sync or skip manual tracking
- Some early phases had more verbose summaries than needed -- later phases were more concise

### Patterns Established
- Structured filter format `filter:<field>:<op>=value` as project-wide API standard
- STATUS_CONFIG record pattern for consistent badge rendering across views
- Job-scoped cache tags for RTK Query invalidation
- Luxon DateTime enforced in all DTOs for consistent date handling
- Split-file pattern for shadcn components to satisfy react-refresh lint rule

### Key Lessons
1. Following existing module patterns exactly (copy-adapt, don't reinvent) dramatically accelerates later phases
2. Separate transition service from updater service gives clean lifecycle management
3. ISO8601 single datetime field is cleaner than separate date + time fields
4. Cross-module validation in creator services catches errors early with specific error codes

### Cost Observations
- Model mix: primarily opus for planning and execution
- Total execution: 1.5 hours across 16 plans
- Notable: later phases completed 5x faster than early phases due to pattern reuse

---

## Milestone: v1.1 -- Item Tax Rate Linkage

**Shipped:** 2026-03-08
**Phases:** 2 | **Plans:** 6 | **Execution time:** ~1 hour

### What Was Built
- Item data layer migrated from numeric defaultTaxRate to taxRateId ObjectId reference (entity, DTO, repository, requests, response, mappers)
- Tax rate existence validation in both item creator and updater services with unit tests
- Onboarding flow reordered to create tax rates before items with ID threading
- Quote factories updated to resolve tax rate percentage from taxRateId via TaxRateRepository
- Tax rate dropdown on Material, Labour, and Fee item forms with "Name (rate%)" format and default pre-selection

### What Worked
- Phase 9 gap closure cycle worked well -- verifier caught missing updater validation and quote factory tests, gap plan addressed the critical one
- Cross-phase integration was clean -- Phase 10 UI consumed Phase 9 API contracts without surprises
- Client-side resolution pattern (RTK Query cache) avoided unnecessary server joins
- Following existing form patterns (FormSelect, useItemForm hook) made UI work straightforward
- Human verification checkpoint on item forms caught potential issues before completion

### What Was Inefficient
- Phase 9 had 4 plans for what was essentially 3 logical work items -- the gap closure plan (09-04) could have been avoided with better initial planning of the updater service validation
- SUMMARY.md frontmatter `requirements-completed` was inconsistent across plans -- some plans listed requirements, others didn't, making automated extraction unreliable
- ROADMAP.md plan checkboxes still not synced with completion (same issue as v1.0)

### Patterns Established
- ObjectId<->string conversion pattern for cross-entity references (taxRateId)
- Entity reference dropdown pattern: fetch via shared hook, filter active, map to {value, label}, render FormSelect
- Empty reference state: disabled control + inline warning with navigation link
- Tax rate validation pattern: both creator and updater validate existence before persisting

### Key Lessons
1. Cross-module validation (TaxRateRepository in ItemModule) is a clean pattern for referential integrity
2. Gap closure cycles add overhead -- investing in thorough initial plans (especially for update/delete operations) saves a planning+execution round
3. "Do not change" requirements (ITMUI-03) are valid constraints that can be verified by confirming zero references

### Cost Observations
- Model mix: primarily opus for execution, sonnet for verification
- Total execution: ~1 hour across 6 plans
- Notable: gap closure added ~30% overhead to Phase 9; without it, milestone would have been ~45 min

---

## Milestone: v1.2 -- Bundles & Quotes

**Shipped:** 2026-03-15
**Phases:** 4 | **Plans:** 12 | **Execution time:** ~30 min

### What Was Built
- Bundle creation bug fix (unit: "bundle") and reusable SearchableItemPicker (Popover+Command combobox)
- Bundle component editing: API update with full replacement semantics, validation, two-line display with type badges across edit form/table/cards
- Quote system: entity with jobId/quoteDate, auto-generated Q-YYYY-NNN numbers, status transitions (Draft/Sent/Accepted/Rejected), creation dialog, list with real data and tab filtering
- Quote detail page: line item CRUD (add/edit/delete), expandable bundle rows with native table sub-rows, inline qty/price editing, responsive mobile card layout
- Tax-inclusive line totals, Money.percentageOf math fix, soft-delete for line items
- Pricing strategy as item-level setting (BundleItemForm radio group) with immediate-add in quote flow

### What Worked
- Parallel wave execution for plans 14-05 and 14-06 cut execution time -- both completed simultaneously
- SearchableItemPicker designed for reuse in Phase 11 paid off immediately in Phase 14 (quote line items)
- Gap closure cycle (plans 14-03 through 14-06) effectively addressed UAT feedback without scope creep
- Denormalized fields (customerName, jobTitle) in quote API responses eliminated N+1 queries
- Recalculate-on-read totals (not persisted) avoided stale data entirely
- Soft delete pattern (DELETED status enum) kept line item history while excluding from totals/queries

### What Was Inefficient
- Phase 14 had 6 plans (4 were gap closure) -- initial plans 14-01 and 14-02 could have anticipated the soft-delete and tax-inclusive display needs, reducing gap closure rounds
- CSS grid approach for bundle components (14-04) was immediately replaced by native TableRow (14-05) -- should have used TableRow from the start
- Two-step bundle add flow added in 14-04 was removed in 14-06 -- design decision changed after UAT. Could have deferred pricing strategy UI to avoid rework
- ROADMAP.md plan checkboxes still desync (plans 14-05 and 14-06 unchecked despite completion) -- same issue as v1.0 and v1.1

### Patterns Established
- Inline edit: `defaultValue` with `key={id+value}` for server-driven input reset, blur/Enter to save, Escape to reset
- Tax-inclusive display: `lineTotalIncTax` helper centralises calculation; "Total (inc. tax)" column header for clarity
- Soft delete: DELETED status enum + filter in repository queries + guard in calculator services
- Dual-context dialog: same component with optional prefilled props switches between picker flow and pre-populated mode
- Atomic MongoDB counter: `findOneAndUpdate` with `$inc` for sequential number generation

### Key Lessons
1. Plan for soft-delete from the start -- adding it later required touching repository, service, calculator, and enum layers
2. Pricing/UX decisions that affect multiple components should be settled before implementation, not iterated through gap closure
3. Native table layout beats CSS grid for sub-row alignment -- don't fight the browser's column layout algorithm
4. Item-level vs quote-level settings distinction matters -- pricing strategy belongs on the item, not in the quote add flow
5. Parallel plan execution is effective when plans touch different files with no dependency

### Cost Observations
- Model mix: opus for execution, sonnet for verification and integration checking
- Total execution: ~30 min across 12 plans (plus gap closure planning/UAT overhead)
- Notable: average plan time down to 3 min (from 6 min in v1.0) -- pattern maturity and codebase familiarity

---

## Milestone: v1.3 -- Send Quotes

**Shipped:** 2026-03-21
**Phases:** 5 | **Plans:** 18 | **Execution time:** ~1 hour

### What Was Built
- Quote deletion: soft-delete for draft quotes with confirmation dialogs, optimistic UI removal, and token revocation cascade
- Token infrastructure: cryptographic 32-byte base64url tokens with 30-day expiry, public API endpoint with rate limiting and customer-safe response filtering
- Customer quote page: responsive public page with view tracking (firstViewedAt), error/expiry states, and tradesperson "Viewed" badge
- Quote email sending: Maizzle HTML email templates, Tiptap rich text editor in send dialog, configurable email templates via QuoteSettings module, Draft→Sent status transition
- Customer response: accept/decline endpoints with inline decline form, status banners, failure-tolerant notification emails to tradesperson
- Email provider migration: SendGrid replaced with Resend SDK for simpler integration
- 32+ unit tests across QuoteSessionAuthGuard, PublicQuoteRetriever, QuoteResponseHandler, NotificationEmailRenderer, and publicTransition

### What Worked
- Separate QuoteSettings module was the right call -- originally embedded in Business entity (Plan 01), gap closure (Plan 05/06) extracted it cleanly. Future features that add business settings won't bloat the Business entity
- QuoteSessionAuthGuard extracted in gap closure (Plan 17-03) was immediately reused by Phase 19 accept/decline endpoints -- validation, expiry check, first-view recording all shared
- Failure-tolerant notification emails (try/catch, don't block status transition) was a good defensive pattern -- email delivery shouldn't prevent customer actions
- Public RTK Query slice (separate from authenticated slice) prevented auth header leakage to public endpoints
- Resend migration (Plan 18-07) simplified error handling significantly -- { data, error } pattern vs try/catch around SendGrid
- 5 E2E flows all verified wired in milestone audit -- no cross-phase integration gaps

### What Was Inefficient
- Phase 18 had 7 plans (4 were gap closure) -- initial plans 18-01, 18-02, 18-03 didn't anticipate the QuoteSettings separation or SendGrid error handling needs
- Email template fields were added to Business entity in Plan 18-01 then extracted to QuoteSettings in Plan 18-05 -- could have used a separate module from the start
- DLVR-01 requirements accuracy issue (PDF attachment incorrectly claimed in-scope) was only caught by verifier -- better upfront requirements parsing would have avoided the gap closure plan
- Phase 17 initial verification found 3 architectural gaps (inline validation, repository leaking, missing tests) that required 2 gap closure plans -- stronger plan review could have caught the Controller→Service→Repository violation
- PDF generation (Phase 20) was planned, researched via discussion, then removed -- the scope reduction was correct but the discussion time was wasted

### Patterns Established
- Token-based public access pattern: QuoteTokenCreator → QuoteSessionAuthGuard → PublicQuoteRetriever chain for unauthenticated endpoints
- Maizzle + Tailwind email template pattern with dynamic import for ESM-only module in CommonJS NestJS
- Rich text editor pattern: Tiptap with StarterKit + Link extensions, HTML output for email body
- Separate settings module pattern: QuoteSettings module owns email template data independently from Business entity
- Public mutation rate limiting: stricter limits (10/min) on write endpoints vs read endpoints (60/min)
- Optimistic local state pattern: useState + onSuccess callback for immediate UI feedback after mutation (PublicQuoteResponseButtons)

### Key Lessons
1. Separate settings/config from core entities from the start -- embedding quoteEmail fields in Business entity caused a full extraction cycle
2. Gap closure plans are a sign of insufficient initial planning for cross-cutting concerns (error handling, architectural layering)
3. Token-based public access with a reusable guard is a clean pattern for customer-facing features without login
4. Email provider choices should be evaluated early -- Resend's simpler API saved significant error handling complexity vs SendGrid
5. Failure-tolerant side effects (notification emails) are essential for customer-facing flows -- never block the primary action
6. Verify requirements accuracy against CONTEXT.md during planning -- the DLVR-01/PDF-attachment mismatch was avoidable

### Cost Observations
- Model mix: opus for planning and execution, sonnet for verification and integration checking
- Total execution: ~1 hour across 18 plans (avg 3min/plan)
- Notable: gap closure plans accounted for ~40% of total plans (7/18) -- reducing this would significantly improve velocity
- Session count: ~6 sessions across 8 days

---

## Milestone: v1.4 -- Monorepo & Worker Infrastructure

**Shipped:** 2026-03-22
**Phases:** 4 | **Plans:** 7 | **Execution time:** ~15 min

### What Was Built
- Redis 7.4 in Docker Compose with `maxmemory-policy noeviction`, `@queue/*`/`@worker/*` path aliases across all compile targets
- `QueueModule` with `BullModule.forRootAsync`, `QueueProducer` service, and `QUEUE_NAMES` constants wired into `AppModule`
- `src/worker.ts` via `NestFactory.createApplicationContext(WorkerModule)` with SIGTERM shutdown and separate `worker-cli.json` producing `dist/worker.js`
- `EchoProcessor` consuming echo queue and `POST /v1/queue/test-echo` diagnostic endpoint proving end-to-end flow
- `worker:dev` (nodemon hot reload), `worker:prod`, `build:all` npm scripts and worker as fourth Docker Compose service with production Dockerfile stage

### What Worked
- Infrastructure-first milestone was clean -- each phase built on the previous with no circular dependencies or rework
- Dual entry-point pattern (preserving all existing path aliases) was exactly the right choice over NestJS CLI monorepo mode -- zero existing code changed
- Phases were small and focused (1-2 plans each) with zero gap closure needed -- clearest milestone yet with lowest plan count per outcome
- Key decisions resolved early (maxmemory-policy, createApplicationContext, deleteOutDir:false) prevented surprises during execution
- IORedis type mismatch (`as unknown as`) was a known BullMQ integration quirk resolved cleanly in Phase 21 without extra plans

### What Was Inefficient
- No milestone audit -- could have caught the 2 undelivered PROJECT.md active requirements (monorepo restructure, shared module pattern) before completing
- Echo processor and test-echo endpoint are infrastructure leftovers -- should probably be guarded/removed before real traffic arrives

### Patterns Established
- Dual entry-point NestJS pattern: `src/main.ts` + `src/worker.ts` with separate `nest-cli.json` configs producing independent `dist/` entries
- Queue name constants as single source of truth: `QUEUE_NAMES` in `src/queue/queue.constant.ts` shared by producer and consumer
- Worker module isolation: `WorkerModule` imports only the minimum (CoreModule, ConfigModule, LoggerModule, QueueModule) -- no HTTP modules
- Debug port separation: API on 9229, worker on 9230
- `deleteOutDir:false` in worker build config: required pattern when two CLI configs share the same `dist/` output directory

### Key Lessons
1. Pure infrastructure milestones benefit from tight phase scoping -- fewer plans per phase = fewer context switches and zero gap closure
2. The "what goes in WorkerModule" decision deserves explicit upfront design -- discovered CoreModule was required (MONGO_URL) only during Phase 22
3. Run milestone audit before completing even infrastructure milestones -- the 2 deferred requirements (restructure + shared modules) should have been explicitly scoped out or planned
4. NestJS dual-entry-point is production-viable for small monorepos -- NestJS CLI monorepo mode is not worth the webpack migration cost
5. IORedis type incompatibility with BullMQ bundled ioredis is a known issue -- `as unknown as ConnectionOptions` is the accepted workaround, not a code smell

### Cost Observations
- Model mix: opus for execution and planning
- Total execution: ~15 min across 7 plans (avg 2min/plan) -- fastest milestone velocity yet
- Notable: zero gap closure plans -- infrastructure phases with clear acceptance criteria are the easiest to execute correctly first time

---

## Milestone: v1.6 -- Stripe Subscription Billing

**Shipped:** 2026-03-31
**Phases:** 6 (29-34) | **Plans:** 14 | **Execution time:** ~55 min

### What Was Built
- SubscriptionModule with Stripe SDK v21 factory provider, MongoDB entity with userId-keyed unique indexes, rawBody enabled globally for webhook signature verification
- BullMQ STRIPE_WEBHOOKS queue with webhook controller enqueuing validated events, processor handling 5 Stripe event types with upsert idempotency
- Verify-session endpoint (local DB fast path + Stripe API fallback) and duplicate checkout guard
- Three subscription management endpoints (GET status, DELETE cancel-at-period-end, POST portal) with global SubscriptionGuard and support role bypass
- Unit tests for all subscription services, repository, webhook controller, and guard
- Frontend paywall: SubscriptionGatedLayout, PaywallModal for write actions, PricingCard, subscribe/success/cancel pages, RTK Query subscriptionApi with SubscriptionProvider
- TrialChip in app header with days-remaining display, Settings > Billing tab with SubscriptionStatusCard and Stripe Portal redirect
- Luxon DateTime standardized across full stack: shared toDateTime utility, ISubscriptionDto aligned to IBaseResourceDto, date-helpers module in UI replacing date-fns

### What Worked
- Stripe Checkout (hosted) was the right choice -- zero PCI scope, SCA/3DS handled automatically, no frontend Stripe packages needed
- BullMQ for webhook processing provided deduplication (jobId: event.id) and retry semantics out of the box -- v1.4 infrastructure investment paid off
- Soft paywall modal (not hard gate) for write actions was a balanced UX -- users can browse data but can't create/edit
- Verify-session endpoint as race condition safety net was essential -- webhook can arrive 1-30s after Checkout redirect
- SubscriptionGuard on API (server-side enforcement) prevents bypass via direct API calls
- Phase 34 (Luxon standardization) was a clean cross-cutting cleanup that benefited from fresh subscription module code

### What Was Inefficient
- v1.5 E2E Playwright testing was paused to start v1.6 -- interleaving milestones adds context switching overhead
- 3 requirements left unchecked (ACQ-04, GATE-02, GATE-03) that are likely implemented but not formally verified -- should have been checked during phase verification
- No milestone audit run before completion

### Patterns Established
- Stripe webhook processing: enqueue in controller (return 200 to Stripe immediately) -> BullMQ processor handles event types -> upsert by stripeSubscriptionId
- Subscription guard pattern: global NestJS guard with @SkipSubscriptionCheck decorator for exempt routes
- Soft paywall: useSubscription() hook + PaywallModal that opens on write-action handlers, blocks action without navigation
- SubscriptionContext/Provider separation (.ts/.tsx) to satisfy react-refresh ESLint rule
- URL query param tab routing (?tab=billing) for Settings page tabs -- preserves Stripe Portal return_url context
- toEntityFields helper for DateTime -> Date conversion at repository boundary

### Key Lessons
1. Stripe Checkout (hosted) is the fastest path to billing -- don't build Stripe Elements unless you need custom UI
2. Webhook processing should be async (BullMQ) not inline -- return 200 to Stripe immediately, process later
3. Verify-session endpoint is essential for any redirect-after-payment flow -- webhooks are not instant
4. Soft paywall (modal on write actions) is better UX than hard redirect for SaaS billing
5. Server-side subscription guard is mandatory even with frontend gate -- never trust client-only enforcement
6. Luxon standardization across full stack eliminates an entire class of date parsing bugs

### Cost Observations
- Model mix: opus for planning and execution
- Total execution: ~55 min across 14 plans (avg 4min/plan)
- Notable: zero gap closure plans for the billing phases (29-33) -- clear Stripe patterns and thorough research phase paid off
- Phase 34 (Luxon) was a cross-cutting cleanup that touched 30+ files but completed in 2 plans

---

## Milestone: v1.7 -- Onboarding & Landing Page

**Shipped:** 2026-04-07
**Phases:** 6 (35-40) | **Plans:** 13 | **Timeline:** 7 days (2026-04-01 -> 2026-04-07)

### What Was Built
- No-card 30-day free trial via Stripe API with belt-and-suspenders webhook reconciliation
- Public marketing landing page at root URL (hero, features, pricing, footer) with bundle isolation from app
- Three-tier route guard architecture (ProtectedRoute > OnboardingGuard > PaywallGuard)
- Mandatory two-step onboarding wizard (profile name + business/trade) with auto-created defaults (5 resource types)
- Hard paywall with three variant modes replacing soft paywall modal (7 files deleted, 10 pages cleaned)
- Personalised welcome dashboard with getting-started checklist and old onboarding system removal (15 files deleted)
- @SkipSubscriptionCheck decorator for onboarding endpoints (INT-01 gap closure)

### What Worked
- Milestone audit before completion caught the critical INT-01 gap (SubscriptionGuard blocking onboarding) -- Phase 40 fixed it surgically with a decorator
- Parallel phase execution (35+36 in different repos, 37+38 on different concerns) shortened the critical path
- Landing page bundle isolation was the right call -- lazy-loaded route doesn't import Redux/auth, keeping public page fast
- Three-tier guard architecture cleanly separates auth/onboarding/paywall concerns -- each guard has one responsibility
- Hard paywall was a significant simplification over soft paywall -- deleted 7 files and removed dispatch calls from 10 pages
- DefaultQuoteSettingsCreatorService completed the "5 default resource types on business creation" goal -- no gap left

### What Was Inefficient
- Phase 38 shows 1/2 plans complete in roadmap despite being marked complete -- plan tracking inconsistency persists
- Some phases completed out of order (38 before 37) due to parallel execution -- roadmap tracking doesn't handle non-linear well
- Audit was run late (after all phases except 40) -- running it earlier would have caught INT-01 sooner and avoided the emergency Phase 40

### Patterns Established
- @SkipSubscriptionCheck method-level decorator pattern for guard bypass (surgical, auditable)
- Three-tier route guard nesting (auth > onboarding > paywall) with each guard as a shell layout
- Landing page bundle isolation via lazy-loaded route avoiding app state imports
- Trade card grid selection pattern for onboarding with TRADE_OPTIONS constant
- SetupLoadingScreen pattern for sequential API calls with partial-completion retry
- TrialBadge with urgency color thresholds (yellow at <=10 days, red at <=3)

### Key Lessons
1. Run milestone audit early (after ~75% of phases) not at the end -- INT-01 was a critical integration gap that could have been caught earlier
2. Hard paywall is simpler than soft paywall -- one blocking route vs per-page gating logic in every feature
3. Mandatory onboarding eliminates the "incomplete account" problem entirely -- worth the upfront friction
4. Bundle-isolating public pages from the authenticated app pays off immediately in load performance
5. Decorator-based guard bypass (@SkipSubscriptionCheck) is cleaner than conditional logic inside the guard

### Cost Observations
- Model mix: opus for planning and execution
- Timeline: 7 days for 13 plans across 6 phases
- Notable: Phase 40 (gap closure) was 1 plan, ~3 min -- surgical fixes with decorators are fast when the pattern is established

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Execution Time | Phases | Key Change |
|-----------|---------------|--------|------------|
| v1.0 | 1.5 hours | 8 | Established module-copy pattern, deferred v2 features cleanly |
| v1.1 | ~1 hour | 2 | Cross-module references, gap closure cycle, entity dropdown pattern |
| v1.2 | ~30 min | 4 | Reusable picker component, quote system, parallel execution, gap closure for UX polish |
| v1.3 | ~1 hour | 5 | Email delivery + customer response, token-based public access, email provider migration, gap closure for architecture |
| v1.4 | ~15 min | 4 | Infrastructure-only milestone, dual entry-point pattern, zero gap closure, lowest plan count per outcome |
| v1.6 | ~55 min | 6 | Full SaaS billing, webhook processing via BullMQ, soft paywall, Luxon standardization, zero gap closure |
| v1.7 | ~7 days | 6 | User acquisition funnel, hard paywall replacing soft, milestone audit caught critical integration gap |

### Top Lessons (Verified Across Milestones)

1. Copy-adapt existing module patterns for new domain entities (visit-type -> schedule -> item-tax-rate -> quote -> quote-settings)
2. Defer non-essential modes/features to keep milestones shippable (confirmed: v1.3 deferred PDF generation)
3. Smart defaults reduce form complexity and improve UX for solo operators
4. Cross-module validation with specific error codes catches errors early (confirmed in v1.0, v1.1, v1.2, v1.3)
5. Plan update/delete operations explicitly from the start to avoid gap closure overhead (confirmed: v1.1 tax rate updater, v1.2 line item soft-delete, v1.3 QuoteSettings extraction)
6. Design reusable components early (SearchableItemPicker, QuoteSessionAuthGuard) -- pays off when the same component is needed across features/phases
7. Settle UX decisions (pricing strategy, display format) before implementation to avoid rework through gap closure
8. Separate config/settings from core entities from the start -- embedding then extracting causes a full rework cycle (confirmed: v1.3 QuoteSettings)
9. Gap closure plans should decrease over milestones -- 40% gap closure rate (v1.3) → 0% (v1.4): infrastructure phases with clear acceptance criteria execute cleanly
10. Infrastructure milestones (v1.4) unlock future product milestones -- tight scope + no gap closure = fastest shipping rate yet
11. Run milestone audit at ~75% completion, not after final phase -- catches integration gaps while there's still time to fix cheaply (confirmed: v1.7 INT-01)
12. Hard paywall is simpler than soft paywall for subscription enforcement -- one blocking route vs per-page gating in every feature (confirmed: v1.7 deleted 7 files + cleaned 10 pages)
