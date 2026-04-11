# Phase 41: Estimate Module CRUD (Backend) - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning

<domain>
## Phase Boundary

A full backend estimate module (`src/estimate/`) mirroring the existing `src/quote/` module, exposing authenticated CRUD endpoints for estimates with `E-YYYY-NNN` numbering, a validated status lifecycle, a dedicated `estimate_counters` atomic counter, and a dedicated `estimatelineitems` collection with its own full `EstimateLineItem*` stack.

Ships in the same phase: the `quote-token` module is replaced wholesale by a new `document-token` module that accepts `documentType: "quote" | "estimate"`. The existing `/v1/public/quote/:token` endpoint continues to resolve via the new module after a pure code rename. Because the application has no production data, this is a code-level rename only — no data migration runs in Phase 41.

Phase 41 delivers the state machine and transition service but NOT the HTTP endpoints that trigger non-CRUD transitions. Send flow (→ Sent), response flow (→ Viewed / Responded / SiteVisitRequested / Declined), follow-ups (→ Expired), convert (→ Converted), and mark-lost (→ Lost) all belong to Phases 44, 45, 46, and 47 respectively. The estimate entity reserves nullable fields for those concerns now so no schema changes land mid-milestone.

Revisions (REV-*) are strictly Phase 42 scope, but Phase 41 reserves the revision-related entity fields (`parentEstimateId`, `rootEstimateId`, `revisionNumber`, `isCurrent`) and writes sensible root-revision defaults on create so Phase 42 only adds the new revise endpoint and the partial unique index.

</domain>

<decisions>
## Implementation Decisions

### Token Rename (D-TKN)

- **D-TKN-01:** Pure code rename. Delete `src/quote-token/` entirely, create `src/document-token/` from scratch. Collection constant in `DocumentTokenRepository` is `"document_tokens"`. No migration file, no rollback path, no legacy-data handling. Application has no production data to preserve.
- **D-TKN-02:** `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard` renamed in place. Request field `req.quoteToken` → `req.documentToken` renamed in place. No back-compat alias, no deprecated re-export.
- **D-TKN-03:** `PublicQuoteController` reads `req.documentToken` and asserts `documentType === "quote"` at the entry of every handler. If a caller ever hits `/v1/public/quote/:token` with a token whose `documentType === "estimate"` the controller responds with a 404 (resource not found by type).
- **D-TKN-04:** `IDocumentTokenEntity` and `IDocumentTokenDto` replace the quote-token types. Field shape: `{ token, documentType, documentId, expiresAt, revokedAt?, sentAt, recipientEmail, firstViewedAt? }`. Field rename `quoteId → documentId`.
- **D-TKN-05:** Unit tests on `DocumentTokenRepository` and `DocumentSessionAuthGuard` only. No legacy-shape integration test. No forward-facing type-dispatch test. Existing `PublicQuoteController` tests cover the end-to-end wire-up.
- **D-TKN-06:** The rule for migrations: if Phase 41 ever does need a migration, it lands as an `IMigration` class in `src/migration/migrations/` and runs via authenticated `POST /v1/migrations/run`. Migrations MUST NOT run on app startup. (Stated for the record and to guide the planner if circumstances change.)
- **D-TKN-07:** Roadmap success criterion #5 is rewritten. New wording replaces "existing quote tokens continue to validate… one-shot reversible migration" with: *"The `quote-token` module is fully replaced by the new `document-token` module at the code level. The existing `/v1/public/quote/:token` endpoint continues to resolve tokens with `documentType === "quote"` via the new guard. No data migration is required because no production data exists."* Planner must update this criterion in ROADMAP.md and STATE.md as part of this phase.

### Contingency, Totals, and Price Range Shape (D-CONT)

- **D-CONT-01:** Range formula is **base is the floor, contingency is pure upside**: `low = subtotal`, `high = subtotal × (1 + contingencyPercent / 100)`. The base (line-item subtotal) is the tradesperson's honest price if everything goes to plan; the contingency covers surprises that **add** cost. Contingency never reduces the floor. Rejected alternatives: symmetric `(1 - c%)` to `(1 + c%)`, and `c%` interpreted as half-band.
- **D-CONT-02:** Tax is computed per line item first (reusing existing per-line-item `taxRate` logic already in `QuoteTotalsCalculator`). The base subtotal and base tax total are then each scaled by `(1 + c%)` to derive the high end. This preserves mixed tax rates within a single estimate via proportional scaling. For v1.8 each estimate carries an implicit effective rate derived from its own line items. Mixed-rate breakout at the API level is a future tax-module concern.
- **D-CONT-03:** All tax-related field names are **country-agnostic**. No field is named `vat`, `VAT`, `exclVat`, `inclVat`, `vatRate`, or `vatRegistered`. The terminology is `tax`, `exclTax`, `inclTax`, `taxRate`. This matches existing Trade Flow convention — `IQuoteLineItemDto.taxRate` and `IQuoteTotalsDto.taxTotal` are already generic.
- **D-CONT-04:** The `IEstimatePriceRangeDto` does not carry a `taxRegistered` boolean or an effective `taxRate` field. Businesses that do not collect tax simply have `taxRate: 0` on all line items, which naturally produces `tax: Money.zero()`, `exclTax === inclTax`. Caller can derive the effective rate from `tax / exclTax` if needed.
- **D-CONT-05:** `IEstimatePriceRangeDto` is **not persisted** on the entity. It is derived by a new `EstimateTotalsCalculator` service on every service-layer read path (creator, retriever, updater), exactly mirroring how `QuoteTotalsCalculator.calculateTotals()` attaches totals to `IQuoteDto` today. The entity stores only `contingencyPercent` (validated 0-30 in steps of 5, default 10) and `displayMode` (default "range").
- **D-CONT-06:** `IEstimateDto` carries both a conventional `totals: IEstimateTotalsDto` (shape `{ subTotal, taxTotal, total }`, identical to `IQuoteTotalsDto`) AND a `priceRange: IEstimatePriceRangeDto` field. Totals represents the pre-contingency base (for audit, email templates, convert-to-quote pre-fill). Price range represents the customer-facing contingency-adjusted range. They are additive, not alternatives.
- **D-CONT-07:** Both display modes (`range`, `from`) are retained as defined in CONT-02 and CONT-03. Rationale captured in <specifics>. The API always returns both `low` and `high` regardless of `displayMode`. The UI owns the labelling switch ("£X – £Y" in range mode; "From £X" in from mode). The API never branches on display mode; the UI never does contingency math. This preserves CONT-05 and makes revisions that flip modes free of recomputation.

### EstimateLineItem Mirror Fidelity (D-LI)

- **D-LI-01:** Three services currently living under `@quote/services/` are quote-agnostic by structure even though they are filed under the quote module:
  - `BundleConfigValidator` (already uses only `@item/` types)
  - `BundlePricingPlanner` (already uses only `@item/` and `@core/` types)
  - `BundleTaxRateCalculator` (uses a trivial `IQuoteLineItemDto[]` coupling that exercises only `{ unitPrice, quantity, taxRate }`)
  These three move to `@item/services/` as a pure refactor. `BundleTaxRateCalculator` is generalised to accept a new structural interface `ILineItemTaxInput` (or similar) in `@item/interfaces/` containing only `{ unitPrice: Money; quantity: number; taxRate: number }`. Quote module imports update to `@item/services/...`. Estimate module imports the same three shared helpers. This refactor is part of Phase 41.
- **D-LI-02:** All entity-scoped classes in the quote line-item stack are **duplicated 1:1** into the estimate module with `quote → estimate` / `Quote → Estimate` renames in files, classes, types, collection constants, and tests. The duplicated set is:
  - `QuoteLineItemRepository` → `EstimateLineItemRepository`
  - `QuoteBundleLineItemFactory` → `EstimateBundleLineItemFactory`
  - `QuoteStandardLineItemFactory` → `EstimateStandardLineItemFactory`
  - `QuoteTotalsCalculator` → `EstimateTotalsCalculator` (with the added `priceRange` computation from D-CONT-05)
  - Any `QuoteLineItemCreator` / `QuoteLineItemRetriever` service (if present)
  - `IQuoteLineItemDto` → `IEstimateLineItemDto`
  - `IQuoteLineItemEntity` → `IEstimateLineItemEntity`
  - `QuoteLineItemStatus` → `EstimateLineItemStatus`
  - `QuoteLineItemPolicy` → `EstimateLineItemPolicy` (if present)
  - All corresponding test files mirror 1:1
- **D-LI-03:** File-for-file mirror is preferred over estimate-specific reshaping. Side-by-side code review must remain possible. The two stacks are expected to drift over time but should start identical modulo the renames.
- **D-LI-04:** Bug fixes in the bundle helpers happen once (in `@item/services/`). Bug fixes in the duplicated entity-scoped classes touch two places forever. This is the accepted cost of D-01 from the 2026-04-11 restructure.

### Estimate Entity Shape (D-ENT)

- **D-ENT-01:** `IEstimateEntity` field inventory is the full shape in <specifics> below. Notable points:
  - `estimateDate: Date` is user-editable and defaults to now on create, mirroring `quoteDate` on the quote entity.
  - Fields reserved for later phases (Phase 42: revisions; Phase 43: uncertainty UI; Phase 44: send; Phase 45: response; Phase 46: expiry; Phase 47: convert/lost) are all nullable or have safe root-revision defaults. Phase 41 writes `parentEstimateId: null`, `rootEstimateId: null`, `revisionNumber: 1`, `isCurrent: true` on create.
- **D-ENT-02:** MongoDB collection naming:
  - Estimates: `estimates`
  - Estimate line items: `estimatelineitems` (no underscore — mirrors existing `quotelineitems` convention, overrides the `estimate_line_items` wording in `.planning/research/ARCHITECTURE.md` and ROADMAP.md prose)
  - Estimate counters: `estimate_counters` (underscore — mirrors existing `quote_counters` convention)
  - Document tokens: `document_tokens` (underscore — renamed from `quote_tokens`)
- **D-ENT-03:** Required Phase 41 indexes (Phase 42 owns the revision-specific indexes):
  ```
  db.estimates.createIndex({ businessId: 1, createdAt: -1 });
  db.estimates.createIndex({ jobId: 1, createdAt: -1 });
  db.estimates.createIndex(
    { businessId: 1, number: 1 },
    { unique: true, partialFilterExpression: { deletedAt: null } }
  );
  ```
  These are added via the MongoDB driver in the repository constructor or at app bootstrap (planner to choose based on existing quote-module precedent — check how `QuoteRepository` handles index creation).
- **D-ENT-04:** `EstimateNumberGenerator` service is a literal mirror of `QuoteNumberGenerator` in `trade-flow-api/src/quote/services/quote-number-generator.service.ts`. Same `findOneAndUpdate` + `$inc` + `upsert: true` pattern on the `estimate_counters` collection keyed by `{ businessId, year }`. Format is `E-YYYY-NNN` with 3-digit zero-padded sequence.

### Status Lifecycle and Transitions (D-TXN)

- **D-TXN-01:** `EstimateStatus` enum values are lowercase strings: `draft`, `sent`, `viewed`, `responded`, `site_visit_requested`, `converted`, `declined`, `expired`, `lost`, `deleted`. Matches the existing `QuoteStatus` lowercase convention.
- **D-TXN-02:** `ALLOWED_TRANSITIONS` map in `src/estimate/enums/estimate-transitions.ts` mirrors the structure of `src/quote/enums/quote-transitions.ts`. Full map is in <specifics>. `isValidTransition` and `getValidTransitions` helper functions are mirrored as well.
- **D-TXN-03:** The transition map includes `SENT → SENT` as a valid no-op. This enables Phase 44's re-send flow to call the transition service without conditional branching. (The quote module already uses this pattern for re-send.)
- **D-TXN-04:** Phase 41 ships an `EstimateTransitionService` (mirrors `QuoteTransitionService`) that validates transitions against the map and rejects invalid ones with a specific error code (new entry in `ErrorCodes` enum: `ESTIMATE_INVALID_TRANSITION` or similar — planner chooses consistent naming). This service is used INTERNALLY by later phases:
  - Phase 44 `EstimateEmailSender` calls it on send (`Draft → Sent`).
  - Phase 45 `EstimateResponseHandler` calls it on customer response (`Sent/Viewed → Responded/Declined/SiteVisitRequested`, `Sent → Viewed`).
  - Phase 46 `EstimateFollowupProcessor` calls it on expiry (`Sent/Viewed/Responded → Expired`).
  - Phase 47 converter calls it on convert (`Sent/Viewed/Responded/SiteVisitRequested → Converted`) and mark-lost calls it (`Sent/Viewed/Responded → Lost`).
- **D-TXN-05:** **Phase 41 exposes no HTTP endpoint that triggers a non-CRUD transition.** The only public endpoints are `POST /v1/estimates`, `GET /v1/estimates`, `GET /v1/estimates/:id`, `PATCH /v1/estimates/:id`, `DELETE /v1/estimates/:id`. Draft is the only writable status. The transition service exists but is never reachable from a Phase 41 HTTP route.
- **D-TXN-06:** Transition map is unit-tested in `src/estimate/enums/estimate-transitions.spec.ts` mirroring the existing quote-transitions test. Every allowed transition has a positive assertion; every disallowed transition has a negative assertion; at least one round-trip `A → B → A` test confirms terminal states reject further transitions.

### Draft-Only Enforcement (D-DRAFT)

- **D-DRAFT-01:** `EstimateUpdater` and `EstimateDeleter` service methods re-fetch the current status of the estimate and throw `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_EDITABLE, "Estimate can only be edited in Draft status")` (or equivalent naming) if `status !== EstimateStatus.DRAFT`. This mirrors the pattern in `QuoteUpdater` / `QuoteDeleter` if it exists; planner to verify the exact error code convention against `trade-flow-api/src/core/errors/error-codes.enum.ts`.
- **D-DRAFT-02:** `EstimatePolicy.canWrite` handles ownership only (business scoping). It does NOT handle status-based gating. The rule is: **policy checks who; service checks whether.**
- **D-DRAFT-03:** The DELETE semantics for soft delete: `EstimateDeleter` sets `status = EstimateStatus.DELETED` and `deletedAt = DateTime.now()` on the estimate document. Line items in `estimatelineitems` are **not** touched (history preserved per EST-05). The estimate disappears from list queries because the unique index on `{ businessId, number }` is partial on `deletedAt: null` and the list query filters on `deletedAt: null`.

### Response Summary Reservation (D-RESP)

- **D-RESP-01:** `IEstimateResponseSummaryDto` is fully defined in Phase 41 with shape `{ type, respondedAt, message?, declineReason?, siteVisitAvailability? }`. Always `null` on `IEstimateDto.responseSummary` in Phase 41 because no customer response flow exists yet.
- **D-RESP-02:** The entity reserves the following nullable fields now for Phase 45 to populate:
  - `lastResponseType?: EstimateResponseType`
  - `lastResponseAt?: Date`
  - `lastResponseMessage?: string`
  - `declineReason?: EstimateDeclineReason`
  - `siteVisitAvailability?: string`
  Phase 45 writes these inline on the estimate entity (not in a separate collection). The estimate retriever constructs the `responseSummary` DTO from these embedded fields on read.
- **D-RESP-03:** `EstimateResponseType` and `EstimateDeclineReason` enums are defined in Phase 41 even though no code sets them yet. Shapes are in <specifics>. This locks the vocabulary that Phase 45's public controller will write against.

### Uncertainty Field Reservation (D-UNC)

- **D-UNC-01:** `uncertaintyReasons?: string[]` and `uncertaintyNotes?: string` are reserved on `IEstimateEntity` / `IEstimateDto` in Phase 41. `CreateEstimateRequest` and `UpdateEstimateRequest` accept these fields as optional arrays and strings (with class-validator decorators). Phase 41 never populates them itself but accepts them through the CRUD surface so Phase 43's slider/chip UI wires into fields that already exist. Accepts the scope blur deliberately.
- **D-UNC-02:** No enum for uncertainty reason values in Phase 41. The four chip values (`site_inspection`, `pipework`, `materials`, `access`) are defined when Phase 43 builds the UI; Phase 41 stores whatever strings come through the request. Validation is limited to "must be a string array of strings". This avoids blocking Phase 43 on a design decision that belongs to the UI layer.

### Claude's Discretion (planner may override with rationale)

- **Validator for `contingencyPercent`:** class-validator decorator enforcing `Min(0)`, `Max(30)`, and `IsIn([0, 5, 10, 15, 20, 25, 30])`. Exact decorator stack and error wording are the planner's call.
- **Default `displayMode`:** `"range"` on create if not specified. Entity-level default via the create service, not via MongoDB schema default.
- **Default `contingencyPercent`:** `10` on create if not specified.
- **Default `estimateDate`:** `DateTime.now()` on create if not specified (UTC).
- **Default revision fields on create:** `parentEstimateId: null`, `rootEstimateId: null`, `revisionNumber: 1`, `isCurrent: true`.
- **Request DTO classes:** `CreateEstimateRequest`, `UpdateEstimateRequest`, `ListEstimatesRequest` (query params for pagination, status filter tab). File-for-file mirror of the quote request class structure.
- **Response DTO classes:** `EstimateResponse`, `EstimateListResponse` (paginated). Response mapping done in private controller methods; responses are independent of DTOs per existing convention.
- **Endpoint list:**
  - `POST /v1/estimates` (authenticated, `JwtAuthGuard`)
  - `GET /v1/estimates` (paginated, status filter via query param)
  - `GET /v1/estimates/:id`
  - `PATCH /v1/estimates/:id`
  - `DELETE /v1/estimates/:id`
- **Pagination pattern:** mirrors existing quote list endpoint.
- **Line-item write patterns on PATCH:** mirrors `QuoteUpdater` (add/update/delete line items through the same single update request). Check `trade-flow-api/src/quote/services/quote-updater.service.ts` for the exact contract.

### Folded Todos

None — `.planning/STATE.md` shows "Pending Todos: None." No todos were folded into this phase.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Milestone roadmap and restructure decisions
- `.planning/milestones/v1.8-ROADMAP.md` — Phase 41 goal, requirements, success criteria, dependency graph
- `.planning/notes/2026-04-11-v1.8-restructure-decisions.md` — D-01 (line-item separation), D-02 (token unification), D-03 (settings separation), D-04 (milestone restructure). Authoritative over research docs where they conflict.

### Requirements
- `.planning/REQUIREMENTS.md` — EST-01..09, CONT-01/02/05, RESP-08 (and the CONT-03/04 fields that Phase 41 reserves)

### Research
- `.planning/research/ARCHITECTURE.md` §2 (Data Model — note: §2's shared-collection recommendation is SUPERSEDED by D-01 in the restructure notes; read for entity field inventory only), §3 (Revisions — informs reserved fields), §4 (DocumentToken generalisation — still applies to the code rename)
- `.planning/research/PITFALLS.md` §Pitfall 7 (type discrimination), §Pitfall 8 (counter race), §Pitfall 9 (view tracking double-count — applies when Phase 45 wires `firstViewedAt`)

### Existing code to mirror (in trade-flow-api)
- `src/quote/quote.module.ts` — module wiring pattern
- `src/quote/controllers/quote.controller.ts` — controller shape, guards, response mapping
- `src/quote/services/quote-creator.service.ts`, `quote-retriever.service.ts`, `quote-updater.service.ts` — service layer pattern including `calculateTotals` attachment points
- `src/quote/services/quote-number-generator.service.ts` — counter pattern; literal mirror for `EstimateNumberGenerator`
- `src/quote/services/quote-totals-calculator.service.ts` — totals derivation pattern; mirrored and extended with `priceRange` computation
- `src/quote/services/quote-transition.service.ts` (and its `.spec.ts`) — transition service and its tests
- `src/quote/enums/quote-transitions.ts` (and its `.spec.ts`) — transition map and test pattern
- `src/quote/enums/quote-status.enum.ts`, `quote-line-item-status.enum.ts` — enum value conventions
- `src/quote/repositories/quote.repository.ts`, `quote-line-item.repository.ts` — repository pattern, collection constants, DTO↔Entity mapping
- `src/quote/entities/quote.entity.ts`, `quote-line-item.entity.ts` — entity field conventions
- `src/quote/data-transfer-objects/quote.dto.ts`, `quote-line-item.dto.ts`, `quote-totals.dto.ts` — DTO conventions, including JSDoc style
- `src/quote/policies/quote.policy.ts`, `quote-line-item.policy.ts` — policy layer pattern
- `src/quote/services/bundle-config-validator.service.ts` — to be moved to `@item/services/`
- `src/quote/services/bundle-pricing-planner.service.ts` — to be moved to `@item/services/`
- `src/quote/services/bundle-tax-rate-calculator.service.ts` — to be moved to `@item/services/` and generalised
- `src/quote/services/quote-bundle-line-item-factory.service.ts`, `quote-standard-line-item-factory.service.ts` — to be mirrored as Estimate*Factory
- `src/quote-token/` (entire tree) — to be deleted
- `src/quote-token/controllers/public-quote.controller.ts` — stays (will live under a new location) — the controller is the survivor of the quote-token deletion; refactored to use `DocumentSessionAuthGuard`. Its canonical new location: planner decides whether it stays under `src/quote-token/` (nonsensical after the rename), moves into a new `src/public-quote/` module, or folds into `src/quote/controllers/`. Recommended: move to `src/quote/controllers/public-quote.controller.ts` under the quote module since it's quote-specific by response shape.

### API contract
- `trade-flow-api/openapi.yaml` — canonical OpenAPI spec; update alongside the new endpoints
- `trade-flow-api/CLAUDE.md` — NestJS conventions, layering rules, error handling patterns

### Migration infrastructure (for reference only — no migration in Phase 41)
- `trade-flow-api/src/migration/interfaces/migration.interface.ts` — `IMigration` interface
- `trade-flow-api/src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts` — example migration shape
- `trade-flow-api/src/migration/controllers/migration.controller.ts` — `POST /v1/migrations/run`, `POST /v1/migrations/rollback`, `GET /v1/migrations/status` endpoints

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- **`QuoteTotalsCalculator` pattern** (`trade-flow-api/src/quote/services/quote-totals-calculator.service.ts`): iterates line items, computes `subTotal` and `taxTotal` from per-line-item `unitPrice × quantity` and `taxRate`, attaches `totals` to the DTO on every read path. `EstimateTotalsCalculator` is a near-literal copy with an additional pass that multiplies both the subtotal and tax total by `(1 + contingencyPercent/100)` to populate `IEstimatePriceRangeDto.high`. `low.exclTax` equals `totals.subTotal`; `low.tax` equals `totals.taxTotal`; `high` is the scaled mirror.
- **`QuoteNumberGenerator` pattern** (`quote-number-generator.service.ts`): 30-line service using `findOneAndUpdate` with `$inc` and `upsert: true` on `quote_counters` keyed by `{ businessId, year }`. `EstimateNumberGenerator` is a literal mirror targeting `estimate_counters` and emitting `E-YYYY-NNN`.
- **`QuoteTransitionService` + `ALLOWED_TRANSITIONS` map**: already implements the exact state-machine shape Phase 41 needs. Mirror literally with the Estimate lifecycle values.
- **`BundleConfigValidator`, `BundlePricingPlanner`, `BundleTaxRateCalculator`**: quote-agnostic by structure even though they currently live under `@quote/services/`. Lifted to `@item/services/` as part of Phase 41 (see D-LI-01) so both quote and estimate modules share them without duplicating bundle math.
- **`AccessControllerFactory`, `AuthorizedCreatorFactory`**: existing core services. `EstimateCreator` and `EstimateRetriever` use them exactly as `QuoteCreator` / `QuoteRetriever` do.
- **`IMigration` infrastructure**: Phase 41 does not ship a migration, but if a later phase needs one, the infrastructure is already in place and must be driven via `POST /v1/migrations/run` rather than app startup.

### Established Patterns

- **Controller → Service → Repository layering** is strict. Controllers never touch entities, services never touch the DB, repositories never know about DTOs flowing through layers other than their own. The estimate module follows this without exception.
- **Entities keep native `Date`; DTOs use Luxon `DateTime`.** Conversion lives in the repository `toDto` / `toEntity` methods using `toDateTime` / `toOptionalDateTime` from `@core/utilities/to-date-time.utility`.
- **Error handling:** services throw domain errors (`InvalidRequestError`, `ResourceNotFoundError`, `ForbiddenError`); controllers wrap every handler in try/catch and `createHttpError(error)` converts to HTTP.
- **Tests live in `src/estimate/test/`** organised as `controllers/`, `services/`, `repositories/`, `policies/`, `mocks/`. Mock generators (`EstimateMockGenerator`, `EstimateLineItemMockGenerator`) produce DTOs with optional `overrides`. Existing `QuoteMockGenerator` is the template.
- **No `any`, no `as`, no `@ts-ignore`.** Country-agnostic tax field naming matches this codebase's existing conventions (`taxRate`, `taxTotal`) — do not introduce `vat*` fields.
- **Response format** is always `createResponse([item])` — single items wrapped in an array.

### Integration Points

- **`EstimateModule` imports `CoreModule`, `AuthModule`, `ItemModule`, `BusinessModule`, `CustomerModule`, `JobModule`, `TaxRateModule`** — same dependency footprint as `QuoteModule`.
- **`AppModule` registers `EstimateModule`** alongside `QuoteModule`.
- **`DocumentTokenModule` exports `DocumentTokenRepository` and `DocumentSessionAuthGuard`** — consumed by the renamed `PublicQuoteController` in Phase 41 and by the future Phase 45 public estimate controller.
- **`Business` / `Job` / `Customer` read paths**: `EstimateCreator` validates that `businessId`, `customerId`, and `jobId` resolve to accessible resources via their respective retriever services (mirror `QuoteCreator`).
- **`openapi.yaml`** must be updated with the new estimate endpoints and the renamed document-token internals. The public quote endpoint URL path is unchanged.

### Known Pitfalls That Apply

- **Pitfall 7 (type discrimination)**: `EstimateStatus` and `QuoteStatus` share some string values (`draft`, `sent`, `deleted`) but are separate enums. Never cast between them. Services accept one or the other as typed input, never a union. Routes are fully separate. Separate collections, separate repositories, separate policies.
- **Pitfall 8 (counter race)**: `findOneAndUpdate` + `$inc` + `upsert: true` is the ONLY correct pattern. Never read-then-write.
- **Pitfall 9 (view tracking)**: `firstViewedAt` is set once via a conditional update filter. Phase 45 owns the write, but the entity field is reserved in Phase 41 and MUST NOT be overwritten on subsequent reads. The planner should note this constraint on the field's definition.
- **Combined-PR risk**: the token rename ships alongside new estimate code. Code review must eyeball both in the same PR. Keep commits atomic — one commit for the token rename, separate commits for estimate module files, one final commit that wires imports.

</code_context>

<specifics>
## Specific Ideas

### Rationale for two display modes (`range` and `from`)

Two distinct epistemologies of uncertainty drive the two modes:

**Range mode** (`£X – £Y`) applies when the **scope is known but the exact cost has a confidence interval**. Example: a kitchen tap replacement. The tradesperson knows the steps and can price labour + materials with confidence; the contingency (5-10%) covers things like a seized isolator valve or a corroded supply tail. Range communicates: *"I know what I'm doing, here's the confidence interval."*

**"From £X" mode** applies when the **scope itself is unknown** — diagnostic and investigation work. Example: "my shower has low pressure" or "the oven trips the breaker." The tradesperson can commit to a minimum (callout + first hour of diagnosis) but genuinely cannot bound the ceiling until they've investigated. The real answer might be a £20 cartridge or a £2000 repipe. "From" communicates: *"I don't know the scope, here's the minimum commitment."*

A single-mode model breaks both cases:
- **Range-only** forces a dishonest high end for diagnostic work (300% contingency looks ridiculous).
- **From-only** loses the customer's budget ceiling for well-defined jobs.

The API still returns both `low` and `high` in both modes — both values are meaningful and revisions can flip the mode without recomputation. Only the UI labelling differs.

### Concrete entity and DTO definitions (locked during discussion)

**`src/document-token/entities/document-token.entity.ts`**
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface IDocumentTokenEntity extends IBaseEntity {
  token: string;
  documentType: "quote" | "estimate";
  documentId: ObjectId;
  expiresAt: Date;
  revokedAt?: Date;
  sentAt: Date;
  recipientEmail: string;
  firstViewedAt?: Date;
}
```

A matching `IDocumentTokenDto` replaces `IQuoteTokenDto`. `DocumentTokenRepository.COLLECTION = "document_tokens"`. `DocumentSessionAuthGuard` attaches `request.documentToken: IDocumentTokenDto`.

**`src/estimate/entities/estimate.entity.ts`**
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";
import { ObjectId } from "mongodb";

export interface IEstimateEntity extends IBaseEntity {
  // Ownership & routing
  businessId: ObjectId;
  customerId: ObjectId;
  jobId: ObjectId;

  // Identity
  number: string;                 // E-YYYY-NNN
  title: string;
  estimateDate: Date;
  notes?: string;

  // Lifecycle
  status: EstimateStatus;
  sentAt?: Date;                  // Phase 44 writes
  firstViewedAt?: Date;           // Phase 45 writes
  respondedAt?: Date;             // Phase 45 writes
  convertedAt?: Date;             // Phase 47 writes
  convertedToQuoteId?: ObjectId;  // Phase 47 writes
  declinedAt?: Date;              // Phase 45 writes
  lostAt?: Date;                  // Phase 47 writes
  expiresAt?: Date;               // Phase 46 writes (30 days after sentAt)
  deletedAt?: Date;

  // Contingency & display (CONT-01, CONT-02, CONT-03)
  contingencyPercent: number;     // 0, 5, 10, 15, 20, 25, 30 (default 10)
  displayMode: EstimateDisplayMode;

  // Uncertainty reservation (CONT-04 — Phase 43 UI)
  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;

  // Customer response reservation (RESP-08 — Phase 45 writes)
  lastResponseType?: EstimateResponseType;
  lastResponseAt?: Date;
  lastResponseMessage?: string;
  declineReason?: EstimateDeclineReason;
  siteVisitAvailability?: string;

  // Revision reservation (REV-01..05 — Phase 42 writes)
  parentEstimateId?: ObjectId | null;
  rootEstimateId?: ObjectId | null;
  revisionNumber: number;         // 1 on root
  isCurrent: boolean;             // true on root

  // Line-item collection reference
  lineItems?: ObjectId[];
}
```

**`src/estimate/data-transfer-objects/estimate.dto.ts`**
```typescript
import { DateTime } from "luxon";
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { DtoCollection } from "@core/collections/dto.collection";
import { IEstimateLineItemDto } from "@estimate/data-transfer-objects/estimate-line-item.dto";
import { IEstimateTotalsDto } from "@estimate/data-transfer-objects/estimate-totals.dto";
import { IEstimatePriceRangeDto } from "@estimate/data-transfer-objects/estimate-price-range.dto";
import { IEstimateResponseSummaryDto } from "@estimate/data-transfer-objects/estimate-response-summary.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export interface IEstimateDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  customerId: string;
  jobId: string;

  number: string;
  title: string;
  estimateDate: DateTime;
  notes?: string;

  status: EstimateStatus;
  sentAt?: DateTime;
  firstViewedAt?: DateTime;
  respondedAt?: DateTime;
  convertedAt?: DateTime;
  convertedToQuoteId?: string;
  declinedAt?: DateTime;
  lostAt?: DateTime;
  expiresAt?: DateTime;
  deletedAt?: DateTime;
  updatedAt?: DateTime;

  contingencyPercent: number;
  displayMode: EstimateDisplayMode;

  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;

  // Revision fields (read-only in Phase 41; Phase 42 adds the write path)
  parentEstimateId?: string | null;
  rootEstimateId?: string | null;
  revisionNumber: number;
  isCurrent: boolean;

  lineItems: DtoCollection<IEstimateLineItemDto>;
  totals: IEstimateTotalsDto;
  priceRange: IEstimatePriceRangeDto;

  // Response summary (always null in Phase 41; Phase 45 populates)
  responseSummary: IEstimateResponseSummaryDto | null;
}
```

**`src/estimate/data-transfer-objects/estimate-totals.dto.ts`**
```typescript
import { Money } from "@core/value-objects/money.value-object";

export interface IEstimateTotalsDto {
  subTotal: Money;
  taxTotal: Money;
  total: Money;  // subTotal + taxTotal (base, pre-contingency)
}
```

**`src/estimate/data-transfer-objects/estimate-price-range.dto.ts`**
```typescript
import { Money } from "@core/value-objects/money.value-object";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export interface IEstimatePriceRangeBoundDto {
  exclTax: Money;
  tax: Money;
  inclTax: Money;
}

export interface IEstimatePriceRangeDto {
  displayMode: EstimateDisplayMode;
  subtotal: Money;
  contingencyPercent: number;
  low: IEstimatePriceRangeBoundDto;   // low.exclTax === totals.subTotal; low.tax === totals.taxTotal
  high: IEstimatePriceRangeBoundDto;  // high.exclTax = low.exclTax × (1 + c/100); high.tax = low.tax × (1 + c/100)
}
```

No `vat*` fields. No `taxRegistered` boolean. No effective `taxRate` field at the range level. A business that doesn't collect tax has `taxRate: 0` on all its line items, which naturally yields `tax: Money.zero()` and `exclTax === inclTax`.

**`src/estimate/data-transfer-objects/estimate-response-summary.dto.ts`**
```typescript
import { DateTime } from "luxon";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";

export interface IEstimateResponseSummaryDto {
  type: EstimateResponseType;
  respondedAt: DateTime;
  message?: string;
  declineReason?: EstimateDeclineReason;     // only when type === "decline"
  siteVisitAvailability?: string;             // only when type === "site_visit_request"
}
```

**`src/estimate/entities/estimate-line-item.entity.ts`**
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ItemType } from "@item/enums/item-type.enum";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";
import { ObjectId } from "mongodb";

export interface IEstimateLineItemEntity extends IBaseEntity {
  estimateId: ObjectId;
  businessId: ObjectId;
  itemId: ObjectId;
  quantity: number;
  unit: string;
  unitPrice: number;
  lineTotal: number;
  originalUnitPrice: number;
  originalLineTotal: number;
  discountAmount: number;
  taxRate: number;
  type: ItemType;
  status: EstimateLineItemStatus;
  parentLineItemId?: ObjectId | null;
}
```

Literal 1:1 field mirror of `IQuoteLineItemEntity` except `quoteId → estimateId` and the enum type.

**`src/estimate/data-transfer-objects/estimate-line-item.dto.ts`**
```typescript
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { Money } from "@core/value-objects/money.value-object";
import { ItemType } from "@item/enums/item-type.enum";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";

export interface IEstimateLineItemDto extends IBaseResourceDto {
  id: string;
  estimateId: string;
  itemId: string;
  businessId: string;
  quantity: number;
  unit: string;
  unitPrice: Money;
  lineTotal: Money;
  originalUnitPrice: Money;
  originalLineTotal: Money;
  discountAmount: Money;
  taxRate: number;
  type: ItemType;
  status: EstimateLineItemStatus;
  parentLineItemId?: string | null;
}
```

### Enums (locked)

```typescript
// src/estimate/enums/estimate-status.enum.ts
export enum EstimateStatus {
  DRAFT = "draft",
  SENT = "sent",
  VIEWED = "viewed",
  RESPONDED = "responded",
  SITE_VISIT_REQUESTED = "site_visit_requested",
  CONVERTED = "converted",
  DECLINED = "declined",
  EXPIRED = "expired",
  LOST = "lost",
  DELETED = "deleted",
}

// src/estimate/enums/estimate-display-mode.enum.ts
export enum EstimateDisplayMode {
  RANGE = "range",
  FROM = "from",
}

// src/estimate/enums/estimate-line-item-status.enum.ts
export enum EstimateLineItemStatus {
  PENDING = "PENDING",
  APPROVED = "APPROVED",
  REJECTED = "REJECTED",
  DELETED = "DELETED",
}

// src/estimate/enums/estimate-response-type.enum.ts
export enum EstimateResponseType {
  SITE_VISIT_REQUEST = "site_visit_request",
  SEND_ME_QUOTE = "send_me_quote",
  QUESTION = "question",
  DECLINE = "decline",
}

// src/estimate/enums/estimate-decline-reason.enum.ts
export enum EstimateDeclineReason {
  TOO_EXPENSIVE = "too_expensive",
  DECIDED_NOT_TO_DO_WORK = "decided_not_to_do_work",
  GOING_WITH_ANOTHER_TRADESPERSON = "going_with_another_tradesperson",
  JUST_GETTING_IDEA = "just_getting_idea",
  TIMING_NOT_RIGHT = "timing_not_right",
  OTHER = "other",
}
```

### Transition map (locked)

```typescript
// src/estimate/enums/estimate-transitions.ts
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";

export const ALLOWED_TRANSITIONS: ReadonlyMap<EstimateStatus, readonly EstimateStatus[]> = new Map([
  [EstimateStatus.DRAFT, [EstimateStatus.SENT, EstimateStatus.DELETED]],
  [EstimateStatus.SENT, [
    EstimateStatus.VIEWED,
    EstimateStatus.RESPONDED,
    EstimateStatus.EXPIRED,
    EstimateStatus.LOST,
    EstimateStatus.CONVERTED,
    EstimateStatus.SENT,                  // re-send no-op (Phase 44)
  ]],
  [EstimateStatus.VIEWED, [
    EstimateStatus.RESPONDED,
    EstimateStatus.EXPIRED,
    EstimateStatus.LOST,
    EstimateStatus.CONVERTED,
  ]],
  [EstimateStatus.RESPONDED, [
    EstimateStatus.SITE_VISIT_REQUESTED,
    EstimateStatus.CONVERTED,
    EstimateStatus.DECLINED,
    EstimateStatus.LOST,
    EstimateStatus.EXPIRED,
  ]],
  [EstimateStatus.SITE_VISIT_REQUESTED, [
    EstimateStatus.CONVERTED,
    EstimateStatus.DECLINED,
    EstimateStatus.LOST,
    EstimateStatus.EXPIRED,
  ]],
  [EstimateStatus.CONVERTED, []],
  [EstimateStatus.DECLINED, []],
  [EstimateStatus.EXPIRED, []],
  [EstimateStatus.LOST, []],
  [EstimateStatus.DELETED, []],
]);

export const isValidTransition = (from: EstimateStatus, to: EstimateStatus): boolean => {
  const allowed = ALLOWED_TRANSITIONS.get(from);
  return allowed !== undefined && allowed.includes(to);
};

export const getValidTransitions = (from: EstimateStatus): readonly EstimateStatus[] => {
  return ALLOWED_TRANSITIONS.get(from) ?? [];
};
```

### Collection naming

| Collection | Name | Convention |
|---|---|---|
| Estimates | `estimates` | plural noun, lowercase |
| Estimate line items | `estimatelineitems` | no underscore — mirrors existing `quotelineitems` |
| Estimate counters | `estimate_counters` | underscore — mirrors existing `quote_counters` |
| Document tokens | `document_tokens` | underscore — renamed from `quote_tokens` |

The ROADMAP.md prose referring to `estimate_line_items` (with underscore) is wrong and should be updated to `estimatelineitems` for internal consistency. The planner owns that ROADMAP edit.

### Required indexes (Phase 41 only)

```javascript
db.estimates.createIndex({ businessId: 1, createdAt: -1 });
db.estimates.createIndex({ jobId: 1, createdAt: -1 });
db.estimates.createIndex(
  { businessId: 1, number: 1 },
  { unique: true, partialFilterExpression: { deletedAt: null } }
);
```

Revision indexes (`{ parentEstimateId, revisionNumber }` and the partial unique `{ parentEstimateId, isCurrent }`) belong to Phase 42.

</specifics>

<deferred>
## Deferred Ideas

- **Dual-read fallback for token rename** — explicitly rejected. Left here so the decision is traceable.
- **Empty audit-trail migration file** — explicitly rejected.
- **Effective `taxRate` / `taxRegistered` fields on the price range DTO** — rejected as UK-specific and redundant. Caller can derive effective rate from `tax / exclTax` if ever needed.
- **Symmetric range formula** (`base × (1 - c%)` to `base × (1 + c%)`) — rejected in favour of "base is the floor".
- **Single-mode simplification (range-only or from-only)** — rejected; both modes serve structurally different uncertainty types.
- **Full `as unknown as T` pattern for type narrowing** — excluded from this phase by project rule; use type guards.
- **`estimate_responses` history collection** — rejected; customer responses are persisted inline on the estimate per RESP-08.
- **Cross-module reuse of `@quote/services/` from estimate module** — rejected; shared helpers must live in `@item/services/`.
- **Public transition endpoint `POST /v1/estimates/:id/transition`** — rejected; Phase 41 exposes CRUD only. All non-CRUD transitions go through purpose-built endpoints in Phases 44/45/47.
- **Uncertainty reasons enum in Phase 41** — deferred to Phase 43 where the UI owns the preset chip vocabulary.
- **Legacy-shape document-token integration test** — rejected (no legacy data exists to guard).
- **`vatRegistered` + effective `vatRate` on Business entity lookup** — moot; dropped when the price-range DTO shape simplified to country-agnostic fields.

### Reviewed Todos (not folded)

None — `.planning/STATE.md` reports no pending todos.

</deferred>

---

*Phase: 41-estimate-module-crud-backend*
*Context gathered: 2026-04-11*
