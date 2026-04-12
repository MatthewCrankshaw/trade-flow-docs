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
- v1.8 Estimates -- Phases 41-47 (in progress)

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

### v1.8 Estimates (In Progress)

**Milestone Goal:** Ship estimates as a parallel document type with price ranges, soft customer response flow, automated follow-ups, and seamless conversion to quotes -- handling the pre-site-visit "rough cost" conversation that currently lives in WhatsApp and missed calls.

- [ ] **Phase 41: Estimate Module CRUD (Backend)** - Full `src/estimate/` module mirroring `src/quote/` with counter, policy, CRUD services, status transitions, indexes, plus the `quote-token` → `document-token` rename (unified token module) and a new standalone `estimate_line_items` collection and module
- [ ] **Phase 42: Revisions** - parentEstimateId/rootEstimateId/revisionNumber/isCurrent with EstimateReviser and partial unique index
- [ ] **Phase 43: Estimate Frontend CRUD** - features/estimates, ContingencySlider, document-type toggle on create dialog, list/detail pages, range vs "from" display
- [ ] **Phase 44: Email & Send Flow** - Maizzle estimate templates with non-binding legal copy, EstimateEmailSender, send endpoint, SendEstimateDialog, plus a new standalone `estimate-settings` module and Business > Documents tab update
- [ ] **Phase 45: Public Customer Page & Response Handling** - PublicEstimateController with latest-revision resolution, 4-button response flow, structured decline reasons, view tracking
- [ ] **Phase 46: Follow-up Queue & Automation** - ESTIMATE_FOLLOWUPS BullMQ queue, scheduler, processor, deterministic jobIds, cancel on exit, auto-expiry, Redis AOF infra gate
- [ ] **Phase 47: Convert to Quote & Mark as Lost** - EstimateToQuoteConverter with mandatory review, idempotent convert endpoint, convertedToQuoteId back-link, markLost service

Full details: `.planning/milestones/v1.8-ROADMAP.md`

## Phase Details

### Phase 41: Estimate Module CRUD (Backend)
**Goal**: An authenticated trader can create, read, update, list, and soft-delete estimates with E-YYYY-NNN numbering and a validated status lifecycle via HTTP endpoints, using a dedicated `estimate_line_items` collection; and the `quote-token` module is unified into `document-token` (with `quote_tokens` renamed to `document_tokens`) so both document types share one secure customer-facing guard.
**Depends on**: Nothing (first phase of v1.8)
**Requirements**: EST-01, EST-02, EST-03, EST-04, EST-05, EST-06, EST-07, EST-08, EST-09, CONT-01, CONT-02, CONT-05, RESP-08
**Success Criteria** (what must be TRUE):
  1. `POST /v1/estimates` creates an estimate with an atomically generated `E-YYYY-NNN` number (per-business, per-year counter), writes line items to a new dedicated `estimate_line_items` collection via a new `EstimateLineItem*` module (repository, creator, policy, bundle/tax factories mirroring the quote line-item stack), and returns an API-computed `{ low, high }` price range using a stored contingency percentage.
  2. `GET /v1/estimates` returns a paginated list with tab filtering by status and `GET /v1/estimates/:id` returns full detail including line items, contingency, totals, customer info, status, and a response summary structure.
  3. `PATCH /v1/estimates/:id` edits a Draft estimate (scope, line items, contingency 0-30 in steps of 5, display mode, notes, customer, job) and `DELETE /v1/estimates/:id` soft-deletes from Draft only (`status: DELETED`, line-item history preserved).
  4. Status transition service enforces the lifecycle Draft -> Sent -> Viewed -> Responded -> (SiteVisitRequested / Converted / Declined / Expired / Lost) and rejects invalid transitions with a clear error code; required MongoDB indexes are in place.
  5. `quote-token` module is renamed to `document-token` end-to-end: entity field `quoteId` -> `documentId`, `documentType: "quote" | "estimate"` discriminator added, collection `quote_tokens` -> `document_tokens` via one-shot reversible migration, guard renamed to `DocumentSessionAuthGuard`, existing quote tokens continue to validate, and the public quote page (`/v1/public/quote/:token`) still resolves its token unchanged.
  6. `quote_line_items` collection is untouched -- no `parentType` field, no `estimateId` field, no migration, no backfill. The quote detail page renders line items identically to before this phase.
**Plans**: 8 plans
Plans:
- [x] 41-01-prechecks-and-foundation-PLAN.md — BLOCKING prod quote_tokens check, tsconfig path aliases, ErrorCodes enum extension
- [ ] 41-02-lift-bundle-helpers-PLAN.md — Move BundleConfigValidator/BundlePricingPlanner/BundleTaxRateCalculator from @quote/services to @item/services, generalise to ILineItemTaxInput
- [ ] 41-03-document-token-rename-PLAN.md — Pure code rename quote-token → document-token with documentType discriminator and PublicQuoteController type assertion
- [ ] 41-04-estimate-scaffold-PLAN.md — Estimate enums, entities, DTOs, requests, responses, mock generators, locked transition map + spec
- [ ] 41-05-estimate-repositories-and-stateless-services-PLAN.md — EstimateRepository (paginated), EstimateLineItemRepository, EstimateNumberGenerator, EstimateTotalsCalculator (range math), EstimateTransitionService, EstimatePolicy, EstimateLineItemPolicy
- [ ] 41-06-estimate-line-item-factories-PLAN.md — EstimateStandardLineItemFactory, EstimateBundleLineItemFactory, EstimateLineItemCreator, EstimateLineItemRetriever (mirror quote 1:1)
- [ ] 41-07-estimate-crud-services-PLAN.md — EstimateCreator, EstimateRetriever, EstimateUpdater (Draft-only, line-item CRUD), EstimateDeleter (soft-delete via transition)
- [ ] 41-08-controller-module-wiring-and-docs-PLAN.md — EstimateController (8 endpoints), EstimateModule wiring, AppModule registration, openapi.yaml update, ROADMAP success criterion #5 rewrite, final CI gate

### Phase 42: Revisions
**Goal**: A trader can invisibly revise a Sent estimate -- the new revision becomes current, the previous becomes non-current, history is queryable -- and the data model guarantees exactly one current revision per estimate chain.
**Depends on**: Phase 41
**Requirements**: REV-01, REV-02, REV-03, REV-04, REV-05
**Success Criteria** (what must be TRUE):
  1. `POST /v1/estimates/:id/revisions` creates a new revision of a Sent estimate under the same `E-YYYY-NNN` number, setting `parentEstimateId`, incrementing `revisionNumber`, and flipping `isCurrent` so only the new revision is current (enforced by a partial unique index on `(rootEstimateId, isCurrent: true)`).
  2. `GET /v1/estimates/:id/revisions` returns the full revision chain in `revisionNumber` order (oldest first) with send/view timestamps for each, suitable for the trader-only collapsed History section.
  3. Creating a revision atomically transitions the chain: the new Draft revision becomes current, the previous revision keeps its existing status with `isCurrent: false`, and the trader sees only the latest revision when loading the estimate detail page by root id. Cancellation of the previous revision's pending follow-ups happens at the moment the new revision is actually Sent (Phase 44 owns the call via the `IEstimateFollowupCanceller` binding from Phase 42).
  4. Attempting to concurrently create two revisions for the same chain results in exactly one success and one 409 Conflict (index-enforced), with no duplicate `isCurrent: true` rows.
**Plans**: 6 plans
Plans:
- [x] 42-01-phase-41-amendments-and-roadmap-rewrite-PLAN.md — Amend Phase 41 PLAN-05 (index topology) and PLAN-07 (root-write retrofit), rewrite Phase 42 SC #3 per D-HOOK-05 in both roadmap files
- [x] 42-02-conflict-error-and-followup-interface-PLAN.md — ConflictError class + error codes + createHttpError branch + IEstimateFollowupCanceller interface + NoopEstimateFollowupCanceller default binding
- [ ] 42-03-estimate-repository-revision-methods-PLAN.md — EstimateRepository.downgradeCurrent/insertRevision/restoreCurrent/findRevisionsByRootId/findCurrentInChainByRootId + EstimateLineItemRepository clone helpers + verify index declarations
- [ ] 42-04-estimate-reviser-service-PLAN.md — EstimateReviser service with two-write revise flow, compensating rollback, bundle parent/child line-item clone, D-HOOK-03 non-call assertion, EstimateRevisionMockGenerator
- [ ] 42-05-retriever-and-deleter-extensions-PLAN.md — EstimateRetriever (D-DET-01 non-current resolution, D-DET-02 list filter, findRevisionsByIdOrFail) + EstimateDeleter (D-REV-05/06 predecessor restoration)
- [ ] 42-06-controller-module-openapi-smoke-PLAN.md — POST + GET /v1/estimates/:id/revisions handlers, EstimateModule wiring, openapi.yaml update, manual smoke procedure for SC #4

### Phase 43: Estimate Frontend CRUD
**Goal**: A trader can visually create and edit estimates from the app with a document-type toggle, contingency slider, and range-or-"from" price display, running against the Phase 41 backend.
**Depends on**: Phase 41 (can run in parallel with Phase 42)
**Requirements**: CONT-03, CONT-04
**Success Criteria** (what must be TRUE):
  1. The shared Create Document dialog displays a Quote/Estimate toggle; selecting Estimate reveals the ContingencySlider (0-30% in 5% steps, default 10%), the display-mode toggle (range / "from £X"), the five trade-agnostic uncertainty chips (site inspection needed, hidden conditions, materials & supply, access & working space, scope unclear until investigation) plus a freeform notes field, and the submitted estimate appears in the list.
  2. The Estimates list page renders all estimates with status tab filtering; each row shows customer name, job title, `E-YYYY-NNN`, status badge, and API-returned price range formatted as `£X - £Y` or `From £X` per the estimate's display mode.
  3. The Estimate detail page shows line items, contingency percentage, the formatted price range/"from" display, status, customer info, a placeholder for response summary, and action buttons (Edit, Delete) that Draft estimates can use while non-Draft estimates show the appropriate disabled/locked states.
  4. The UI never multiplies base x contingency on the client; `formatRange(low, high, mode)` renders only what the API returns, verified by a golden-file test asserting API and UI agree to the penny.
**Plans**: 6 plans
Plans:
- [ ] 43-01-types-and-doc-updates-PLAN.md — src/types/estimate.ts, CONT-04 + SMART-04 edits, ROADMAP + v1.8-ROADMAP success criterion sync
- [ ] 43-02-slider-primitive-and-format-range-PLAN.md — @radix-ui/react-slider, shadcn Slider wrapper, formatRange helper, golden-file test + fixture
- [ ] 43-03-rtk-query-estimate-api-PLAN.md — "Estimate" tag, features/estimates/api/estimateApi.ts with 8 hooks targeting /v1/estimates routes
- [ ] 43-04-shared-create-dialog-and-estimate-form-PLAN.md — Extract CreateQuoteForm, build CreateDocumentDialog shell, ContingencySlider, UncertaintyChipGroup, UNCERTAINTY_CHIP_LABELS, CreateEstimateForm
- [ ] 43-05-estimate-list-components-and-page-PLAN.md — 9 mirrored components (table/cards/skeletons/line-items/action strip), EstimatesPage with 7 grouped tabs
- [ ] 43-06-estimate-detail-routing-and-migration-PLAN.md — EstimateDetailPage with inline Draft edits, /estimates routing, sidebar link, JobDetailPage + QuotesPage migration, delete CreateQuoteDialog.tsx, final CI gate
**UI hint**: yes

### Phase 44: Email & Send Flow
**Goal**: A trader can review a pre-filled email, send an estimate via a secure public link, and re-send without creating a new revision -- with mandatory non-binding legal language baked into the default template stored in a new dedicated `estimate-settings` module that is entirely independent of `quote-settings`.
**Depends on**: Phase 41, Phase 43
**Requirements**: SND-01, SND-02, SND-03, SND-04, SND-05, SND-06, SND-07
**Success Criteria** (what must be TRUE):
  1. A new `estimate-settings` module is introduced (module, controller, service, repository, policy, DTOs, requests, responses, tests) with its own API surface exposing `estimateEmailTemplate`. `quote-settings` is untouched.
  2. The Business > Documents tab in trade-flow-ui is updated to fetch and save both template types in parallel via two independent APIs, and the existing quote template UI/UX still loads and saves unchanged.
  3. Trader clicks "Send Estimate" on a Draft estimate, reviews/edits the pre-filled subject and rich-text body in the SendEstimateDialog, and the estimate transitions to Sent with a secure `document-token` link delivered via Resend using a new Maizzle `estimate-sent` template.
  4. The default estimate email template contains mandatory non-binding legal copy ("This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit.") that cannot be removed by the user, and the subject line includes "Estimate" (not "Quote").
  5. The exact rendered HTML sent to the customer is persisted on the estimate at send time as an audit artefact.
  6. Re-sending a Sent (or revised) estimate delivers the email again without creating a new revision and without regenerating the token, and the estimate detail page reflects the updated `lastSentAt`.
**Plans**: 4 plans
Plans:
- [ ] 44-01-PLAN.md — estimate-settings backend module + BusinessCreator extension + tsconfig paths
- [ ] 44-02-PLAN.md — Maizzle estimate-sent.html template + EstimateEmailRenderer + audit collection + DocumentTokenRepository extensions + formatRange utility
- [ ] 44-03-PLAN.md — EstimateEmailSender service + SendEstimateRequest + transition map extensions + controller endpoint + OpenAPI + doc updates
- [ ] 44-04-PLAN.md — Frontend: Documents tab + EstimateEmailSettings + SendEstimateDialog + SendEstimateForm + EstimateActionStrip + EstimateDetailPage wiring
**UI hint**: yes
**Legal-review gate**: Default template copy and subject-line wording must pass a targeted UK-consumer-law copy review before this phase ships. Non-binding disclaimer is mandatory and non-removable (SND-05).

### Phase 45: Public Customer Page & Response Handling
**Goal**: A customer can open the secure estimate link without logging in, always see the latest revision with non-binding language prominent, and respond via one of four structured buttons -- triggering notification and status transitions.
**Depends on**: Phase 41, Phase 44
**Requirements**: CUST-01, CUST-02, CUST-03, CUST-04, CUST-05, CUST-06, CUST-07, RESP-01, RESP-02, RESP-03, RESP-04, RESP-05, RESP-06, RESP-07
**Success Criteria** (what must be TRUE):
  1. `GET /v1/public/estimate/:token` via `DocumentSessionAuthGuard` returns the latest revision of the estimate chain (resolved via `rootEstimateId` + `revisionNumber` desc, even if an older revision was the one emailed), sets `firstViewedAt` once, and triggers the Sent -> Viewed transition.
  2. The customer page (`/estimate/:token`) displays scope, price as range or "from £X", contingency explanation, validity, uncertainty notes, trader business info, and the non-binding legal language prominently at the top -- with NO "Accept" button, NO signature mechanism, NO single fixed total, and zero non-essential cookies (PECR-clean).
  3. The four response buttons work end-to-end: "Book a site visit" opens a pre-populated availability message field capturing a structured site-visit-request; "Send me a quote" records proceed intent; "I have a question" opens a freeform inline message; "Not right now" captures a structured decline reason (Too expensive / Decided not to do the work / Going with another tradesperson / Just getting an idea of costs / Timing isn't right / Other + freeform).
  4. Any customer action persists the full response (type, reason, message, timestamp) on the estimate, transitions the estimate status to Responded / SiteVisitRequested / Declined as appropriate, sends a notification email to the trader with response type and message preview, and future visits to the token on a terminal-state estimate show a friendly read-only message with no active buttons.
**Plans**: TBD
**UI hint**: yes

### Phase 46: Follow-up Queue & Automation
**Goal**: Sending an estimate automatically schedules 3/10/21-day follow-up emails that fire reliably across worker restarts, cancel cleanly on any exit transition, and auto-expire estimates 30 days after send.
**Depends on**: Phase 44, Phase 45
**Requirements**: FUP-01, FUP-02, FUP-03, FUP-04, FUP-05, FUP-06, FUP-07, FUP-08
**Success Criteria** (what must be TRUE):
  1. Sending an estimate calls `EstimateFollowupScheduler.scheduleFollowups()` which enqueues three delayed BullMQ jobs on a new `ESTIMATE_FOLLOWUPS` queue with relative UTC delays (72h, 240h, 504h) and deterministic jobIds of shape `estimate-followup:{estimateId}:{revisionNumber}:{step}` -- a second call for the same estimate/revision is a silent no-op.
  2. `EstimateFollowupProcessor` performs a defence-in-depth status re-read before sending each follow-up and silently no-ops if the estimate is no longer in a follow-up-worthy status; each follow-up email uses its own Maizzle template (3d/10d/21d) and includes the estimate summary plus the same four response buttons as the initial send.
  3. Every exit transition (customer response, revision, conversion, manual delete, mark-as-lost) cancels all pending follow-ups for that estimate/revision via `cancelAllFollowups()`, verified by an integration test that walks each transition with a mocked queue and asserts removal.
  4. An estimate auto-transitions to Expired exactly 30 days after its Sent timestamp via a scheduled sweep or a delayed job, and production Redis is configured with `appendonly yes` / `appendfsync everysec` -- verified by a smoke test that schedules a 60-second delayed job, restarts Redis, and confirms the job still fires.
**Plans**: TBD
**Infra gate**: Production Redis must have AOF persistence enabled (`appendonly yes`, `appendfsync everysec`) before any follow-up ships. This is a hard gate, not a soft constraint -- without it, all scheduled follow-ups are silently lost on restart (FUP-08).

### Phase 47: Convert to Quote & Mark as Lost
**Goal**: A trader can idempotently convert a Sent/Responded estimate into a fully independent quote with mandatory review, and can manually mark an estimate as Lost with a structured reason -- both flows cancel any pending follow-ups and lock the source estimate.
**Depends on**: Phase 41, Phase 42, Phase 46
**Requirements**: CONV-01, CONV-02, CONV-03, CONV-04, CONV-05, CONV-06, LOST-01, LOST-02
**Success Criteria** (what must be TRUE):
  1. `POST /v1/estimates/:id/convert` (accepting an `Idempotency-Key` header) pulls from the latest revision, reads line items from `estimate_line_items`, copies them into `quote_line_items` with literal tax-rate percentages as a snapshot, drops contingency entirely, creates a new quote in Draft status, transitions the source estimate to Converted, sets `convertedToQuoteId` as a back-link, and locks the estimate from further revisions.
  2. The trader opens the convert flow from the estimate detail page, the new quote opens in edit mode for mandatory review before saving, and a double-click submission with the same `Idempotency-Key` within 24h returns the same quote id (exactly one quote created).
  3. The converted quote's detail page shows a "Converted from E-YYYY-NNN" back-link linking to the source estimate, and the source estimate is visibly locked (no Edit, no Revise) once Converted.
  4. `POST /v1/estimates/:id/mark-lost` transitions the estimate to Lost with a structured reason (same taxonomy as customer decline) or freeform text, cancels all pending follow-ups, and renders a locked detail page with the recorded reason visible to the trader.
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
| 41. Estimate Module CRUD (Backend) | v1.8 | 1/8 | In Progress|  |
| 42. Revisions | v1.8 | 2/6 | In Progress|  |
| 43. Estimate Frontend CRUD | v1.8 | 0/6 | Not started | - |
| 44. Email & Send Flow | v1.8 | 0/4 | Not started | - |
| 45. Public Customer Page & Response Handling | v1.8 | 0/? | Not started | - |
| 46. Follow-up Queue & Automation | v1.8 | 0/? | Not started | - |
| 47. Convert to Quote & Mark as Lost | v1.8 | 0/? | Not started | - |
