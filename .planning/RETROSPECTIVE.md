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

## Cross-Milestone Trends

### Process Evolution

| Milestone | Execution Time | Phases | Key Change |
|-----------|---------------|--------|------------|
| v1.0 | 1.5 hours | 8 | Established module-copy pattern, deferred v2 features cleanly |
| v1.1 | ~1 hour | 2 | Cross-module references, gap closure cycle, entity dropdown pattern |
| v1.2 | ~30 min | 4 | Reusable picker component, quote system, parallel execution, gap closure for UX polish |
| v1.3 | ~1 hour | 5 | Email delivery + customer response, token-based public access, email provider migration, gap closure for architecture |
| v1.4 | ~15 min | 4 | Infrastructure-only milestone, dual entry-point pattern, zero gap closure, lowest plan count per outcome |

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
