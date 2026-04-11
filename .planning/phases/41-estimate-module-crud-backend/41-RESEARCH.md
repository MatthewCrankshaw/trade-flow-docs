# Phase 41: Estimate Module CRUD (Backend) - Research

**Researched:** 2026-04-11
**Domain:** NestJS backend — new `src/estimate/` module + `quote-token → document-token` code-level rename + extract bundle helpers from `@quote/services` to `@item/services`
**Confidence:** HIGH (entire research grounded in direct inspection of `trade-flow-api/src/` — no training-data hypotheses)

## Summary

Phase 41 is a **mirror-and-rename** phase, not a design phase. Every architectural decision is already locked in `41-CONTEXT.md`. Research effort is therefore concentrated on two things: (1) mapping every quote-module file to its estimate-module counterpart with exact paths and patterns, and (2) confirming that the established quote-module conventions (counter, totals, transitions, policy, repository layering, test structure) are exactly as the context claims so the planner can trust literal file-for-file mirroring as the implementation strategy.

The verification is complete. `QuoteNumberGenerator`, `QuoteTotalsCalculator`, `QuoteTransitionService`, `ALLOWED_TRANSITIONS`, `QuoteRepository`, `QuoteLineItemRepository`, `QuotePolicy`, and `QuoteCreator` all exist in the shapes the context describes, and the three bundle helpers (`BundleConfigValidator`, `BundlePricingPlanner`, `BundleTaxRateCalculator`) are confirmed quote-agnostic by implementation (the first two import only `@item/*` and `@core/*`; the third takes `IQuoteLineItemDto[]` but uses only `{ unitPrice, quantity, taxRate }`). The `quote-token` module is self-contained under `src/quote-token/` with no external consumers outside `QuoteTransitionService` (which imports `QuoteTokenRevoker`), `QuoteController` (imports `QuoteTokenRetriever`), and `AppModule`.

One divergence from the context is worth flagging: the current `QuoteController` CRUD surface does **not** implement paginated list-with-tab-filter — `findAll` returns all quotes for a business in a single array with no `pagination`, no status filter query param, and no `take/skip`. The context asks for `GET /v1/estimates` to be paginated with status tab filtering. The planner will therefore need to build this from scratch against `IResponse<T>.pagination` shape rather than mirror a quote pattern that does not exist yet. The `IResponse` interface already declares `pagination?: IQueryResultsDto` so the infrastructure for pagination exists at the response envelope level.

**Primary recommendation:** Plan Phase 41 as five sequential tasks grouped by concern — (1) delete quote-token, create document-token module (code rename only, no migration), refactor PublicQuoteController; (2) lift `BundleConfigValidator`/`BundlePricingPlanner`/`BundleTaxRateCalculator` into `@item/services/` and generalise `BundleTaxRateCalculator` via a new `ILineItemTaxInput` interface in `@item/interfaces/`; (3) scaffold `src/estimate/` with entities, enums, DTOs, requests, responses, repository, policy, number generator, totals calculator (with range math), transition service; (4) scaffold `src/estimate/` line-item sub-stack (repository, creator, retriever, policy, standard factory, bundle factory) as literal mirrors of the quote equivalents; (5) wire the controller, creator/retriever/updater/deleter services, paginated list with status tab, wire `EstimateModule` into `AppModule`, update `openapi.yaml`, add jest specs for every new class. CI gate: `npm run ci` must pass (test + lint:check + format:check + typecheck) with no `@ts-ignore`, no `eslint-disable`, no `any`, no domain-to-domain `as`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Token Rename (D-TKN)**

- **D-TKN-01:** Pure code rename. Delete `src/quote-token/` entirely, create `src/document-token/` from scratch. Collection constant in `DocumentTokenRepository` is `"document_tokens"`. No migration file, no rollback path, no legacy-data handling. Application has no production data to preserve.
- **D-TKN-02:** `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard` renamed in place. Request field `req.quoteToken` → `req.documentToken` renamed in place. No back-compat alias, no deprecated re-export.
- **D-TKN-03:** `PublicQuoteController` reads `req.documentToken` and asserts `documentType === "quote"` at the entry of every handler. If a caller ever hits `/v1/public/quote/:token` with a token whose `documentType === "estimate"` the controller responds with a 404.
- **D-TKN-04:** `IDocumentTokenEntity` and `IDocumentTokenDto` replace the quote-token types. Field shape: `{ token, documentType, documentId, expiresAt, revokedAt?, sentAt, recipientEmail, firstViewedAt? }`. Field rename `quoteId → documentId`.
- **D-TKN-05:** Unit tests on `DocumentTokenRepository` and `DocumentSessionAuthGuard` only. No legacy-shape integration test. Existing `PublicQuoteController` tests cover end-to-end wire-up.
- **D-TKN-06:** Migrations, if ever needed, land as an `IMigration` class in `src/migration/migrations/` and run via authenticated `POST /v1/migrations/run`. Migrations MUST NOT run on app startup.
- **D-TKN-07:** Success criterion #5 in ROADMAP.md must be rewritten as part of this phase.

**Contingency, Totals, and Price Range Shape (D-CONT)**

- **D-CONT-01:** Range formula: **base is the floor, contingency is pure upside**. `low = subtotal`, `high = subtotal × (1 + contingencyPercent / 100)`. Contingency never reduces the floor.
- **D-CONT-02:** Tax computed per line item first. Base subtotal and base tax total are each scaled by `(1 + c%)` to derive the high end. Preserves mixed tax rates via proportional scaling.
- **D-CONT-03:** All tax field names **country-agnostic**. No `vat*` fields. Use `tax`, `exclTax`, `inclTax`, `taxRate`.
- **D-CONT-04:** `IEstimatePriceRangeDto` does NOT carry `taxRegistered` or effective `taxRate`. Zero-tax businesses set `taxRate: 0` on line items.
- **D-CONT-05:** `IEstimatePriceRangeDto` is **not persisted**. Derived by `EstimateTotalsCalculator` on every service-layer read path. Entity stores only `contingencyPercent` and `displayMode`.
- **D-CONT-06:** `IEstimateDto` carries both `totals: IEstimateTotalsDto` (pre-contingency base) AND `priceRange: IEstimatePriceRangeDto` (customer-facing range). Additive, not alternatives.
- **D-CONT-07:** Both display modes (`range`, `from`) retained. API always returns both `low` and `high` regardless of `displayMode`. UI owns labelling.

**EstimateLineItem Mirror Fidelity (D-LI)**

- **D-LI-01:** Three services move from `@quote/services/` to `@item/services/` as a pure refactor:
  - `BundleConfigValidator` (already uses only `@item/` types)
  - `BundlePricingPlanner` (already uses only `@item/` and `@core/` types)
  - `BundleTaxRateCalculator` (generalised to accept `ILineItemTaxInput` from `@item/interfaces/` containing only `{ unitPrice: Money; quantity: number; taxRate: number }`)
  Quote module imports update to `@item/services/...`. Estimate module imports the same three shared helpers.
- **D-LI-02:** All entity-scoped classes in the quote line-item stack are **duplicated 1:1** into the estimate module with `quote → estimate` renames in files/classes/types/collection constants/tests. The duplicated set:
  - `QuoteLineItemRepository` → `EstimateLineItemRepository`
  - `QuoteBundleLineItemFactory` → `EstimateBundleLineItemFactory`
  - `QuoteStandardLineItemFactory` → `EstimateStandardLineItemFactory`
  - `QuoteTotalsCalculator` → `EstimateTotalsCalculator` (with added `priceRange` computation from D-CONT-05)
  - `QuoteLineItemCreator` / `QuoteLineItemRetriever` → estimate equivalents
  - `IQuoteLineItemDto` → `IEstimateLineItemDto`
  - `IQuoteLineItemEntity` → `IEstimateLineItemEntity`
  - `QuoteLineItemStatus` → `EstimateLineItemStatus`
  - `QuoteLineItemPolicy` → `EstimateLineItemPolicy`
  - All corresponding test files 1:1
- **D-LI-03:** File-for-file mirror is preferred over estimate-specific reshaping.
- **D-LI-04:** Bug fixes in bundle helpers happen once (in `@item/services/`). Bug fixes in duplicated entity-scoped classes touch two places forever. Accepted cost.

**Estimate Entity Shape (D-ENT)**

- **D-ENT-01:** Full `IEstimateEntity` inventory is in CONTEXT.md <specifics>. `estimateDate: Date` user-editable, defaults to now on create. Reserved nullable fields for Phases 42/43/44/45/46/47 so no schema changes land mid-milestone. Phase 41 writes `parentEstimateId: null`, `rootEstimateId: null`, `revisionNumber: 1`, `isCurrent: true` on create.
- **D-ENT-02:** Collection naming:
  - Estimates: `estimates`
  - Estimate line items: `estimatelineitems` (no underscore — mirrors `quotelineitems`)
  - Estimate counters: `estimate_counters` (underscore — mirrors `quote_counters`)
  - Document tokens: `document_tokens` (underscore — renamed from `quote_tokens`)
- **D-ENT-03:** Required Phase 41 indexes:
  ```
  db.estimates.createIndex({ businessId: 1, createdAt: -1 });
  db.estimates.createIndex({ jobId: 1, createdAt: -1 });
  db.estimates.createIndex(
    { businessId: 1, number: 1 },
    { unique: true, partialFilterExpression: { deletedAt: null } }
  );
  ```
  Revision indexes belong to Phase 42.
- **D-ENT-04:** `EstimateNumberGenerator` is a literal mirror of `QuoteNumberGenerator`. Same `findOneAndUpdate` + `$inc` + `upsert: true` pattern on `estimate_counters` keyed by `{ businessId, year }`. Format `E-YYYY-NNN` with 3-digit zero-padded sequence.

**Status Lifecycle and Transitions (D-TXN)**

- **D-TXN-01:** `EstimateStatus` enum values lowercase: `draft`, `sent`, `viewed`, `responded`, `site_visit_requested`, `converted`, `declined`, `expired`, `lost`, `deleted`.
- **D-TXN-02:** `ALLOWED_TRANSITIONS` map mirrors `src/quote/enums/quote-transitions.ts`. Full map in CONTEXT <specifics>. `isValidTransition` and `getValidTransitions` helpers mirrored.
- **D-TXN-03:** Map includes `SENT → SENT` as a valid no-op for Phase 44's re-send flow.
- **D-TXN-04:** Phase 41 ships `EstimateTransitionService` with new error code `ESTIMATE_INVALID_TRANSITION` (exact naming planner's call against existing `ErrorCodes` convention).
- **D-TXN-05:** **Phase 41 exposes NO HTTP endpoint that triggers a non-CRUD transition.** Only CRUD endpoints. Draft is the only writable status. The transition service exists but is unreachable from a Phase 41 HTTP route.
- **D-TXN-06:** Transition map unit-tested in `src/estimate/enums/estimate-transitions.spec.ts` — every allowed transition has positive assertion; every disallowed transition has negative assertion; at least one terminal-state test.

**Draft-Only Enforcement (D-DRAFT)**

- **D-DRAFT-01:** `EstimateUpdater` and `EstimateDeleter` re-fetch current status and throw `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_EDITABLE, "Estimate can only be edited in Draft status")` if `status !== EstimateStatus.DRAFT`.
- **D-DRAFT-02:** `EstimatePolicy.canWrite` handles ownership only (business scoping). Does NOT handle status-based gating. **Policy checks who; service checks whether.**
- **D-DRAFT-03:** Soft-delete semantics: `EstimateDeleter` sets `status = EstimateStatus.DELETED` and `deletedAt = DateTime.now()`. Line items NOT touched (history preserved per EST-05). Disappears from list queries via partial unique index filtering on `deletedAt: null` + list query filter on `deletedAt: null`.

**Response Summary Reservation (D-RESP)**

- **D-RESP-01:** `IEstimateResponseSummaryDto` fully defined in Phase 41. Always `null` on `IEstimateDto.responseSummary` in Phase 41. Shape: `{ type, respondedAt, message?, declineReason?, siteVisitAvailability? }`.
- **D-RESP-02:** Entity reserves `lastResponseType?`, `lastResponseAt?`, `lastResponseMessage?`, `declineReason?`, `siteVisitAvailability?` — Phase 45 writes these inline on the estimate entity (not a separate collection).
- **D-RESP-03:** `EstimateResponseType` and `EstimateDeclineReason` enums defined in Phase 41 even though no code sets them yet.

**Uncertainty Field Reservation (D-UNC)**

- **D-UNC-01:** `uncertaintyReasons?: string[]` and `uncertaintyNotes?: string` reserved on entity/DTO. `CreateEstimateRequest` and `UpdateEstimateRequest` accept these as optional fields. Phase 41 never populates them but accepts them through CRUD so Phase 43 wires into existing fields.
- **D-UNC-02:** No enum for uncertainty reason values in Phase 41. Validation: "must be a string array of strings".

### Claude's Discretion (planner may override with rationale)

- **Validator for `contingencyPercent`:** class-validator decorator enforcing `Min(0)`, `Max(30)`, and `IsIn([0, 5, 10, 15, 20, 25, 30])`.
- **Default `displayMode`:** `"range"` on create if not specified.
- **Default `contingencyPercent`:** `10` on create if not specified.
- **Default `estimateDate`:** `DateTime.now()` on create if not specified (UTC).
- **Default revision fields on create:** `parentEstimateId: null`, `rootEstimateId: null`, `revisionNumber: 1`, `isCurrent: true`.
- **Request DTO classes:** `CreateEstimateRequest`, `UpdateEstimateRequest`, `ListEstimatesRequest` (query params for pagination, status filter tab).
- **Response DTO classes:** `EstimateResponse`, `EstimateListResponse`. Response mapping done in private controller methods.
- **Endpoint list:**
  - `POST /v1/estimates` (authenticated, `JwtAuthGuard`)
  - `GET /v1/estimates` (paginated, status filter via query param)
  - `GET /v1/estimates/:id`
  - `PATCH /v1/estimates/:id`
  - `DELETE /v1/estimates/:id`
- **Pagination pattern:** mirrors existing quote list endpoint. **NOTE: the existing `QuoteController.findAll` does NOT paginate** — see Pitfall 6 below. Planner must build pagination from scratch against `IResponse<T>.pagination` envelope.
- **Line-item write patterns on PATCH:** mirrors `QuoteUpdater` (add/update/delete line items through the same single update request).

### Deferred Ideas (OUT OF SCOPE)

- Dual-read fallback for token rename — rejected.
- Empty audit-trail migration file — rejected.
- Effective `taxRate` / `taxRegistered` fields on the price range DTO — rejected as UK-specific and redundant.
- Symmetric range formula (`base × (1 - c%)` to `base × (1 + c%)`) — rejected.
- Single-mode simplification (range-only or from-only) — rejected.
- Full `as unknown as T` pattern for type narrowing — use type guards.
- `estimate_responses` history collection — rejected (customer responses inline on estimate per RESP-08).
- Cross-module reuse of `@quote/services/` from estimate module — rejected (shared helpers must live in `@item/services/`).
- Public transition endpoint `POST /v1/estimates/:id/transition` — rejected. Phase 41 exposes CRUD only.
- Uncertainty reasons enum in Phase 41 — deferred to Phase 43.
- Legacy-shape document-token integration test — rejected.
- `vatRegistered` + effective `vatRate` on Business entity lookup — moot.
- Frontend work (React/Redux) — Phase 43 scope.
- Email/SendGrid integration — Phase 44 scope.
- Customer-facing accept/reject page — Phase 45 scope.
- Any changes to `quote_line_items` collection — untouched.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| EST-01 | User can create an estimate via a document-type toggle (Quote / Estimate) on the shared create dialog | Phase 41 backend exposes `POST /v1/estimates` — the UI toggle lands in Phase 43 |
| EST-02 | Estimates numbered E-YYYY-NNN via a new atomic `estimate_counters` collection (mirror of v1.2 `quote_counters` pattern) | `EstimateNumberGenerator` literally mirrors `QuoteNumberGenerator` — see Code Examples §Counter |
| EST-03 | Estimate creation uses its own dedicated `estimatelineitems` collection and module (mirroring the quote line-item stack), with separate repository, services, bundle factories, and tax-rate integration | Full duplicated `EstimateLineItem*` stack per D-LI-02; `quote_line_items` untouched |
| EST-04 | User can edit an estimate in Draft status (scope, line items, contingency, notes, customer, job) | `EstimateUpdater` with D-DRAFT-01 status re-check at service layer |
| EST-05 | User can soft-delete an estimate from Draft status (DELETED enum; preserves line item history) | `EstimateDeleter` sets status + deletedAt; line items untouched per D-DRAFT-03 |
| EST-06 | User can view a list of all estimates with tab filtering by status | `GET /v1/estimates` with `?status=` query param + pagination; built from scratch (see Pitfall 6) |
| EST-07 | User can view estimate detail with line items, contingency range, totals, status, customer info, and response summary | `GET /v1/estimates/:id` + `EstimateRetriever` + `EstimateTotalsCalculator` attaches totals and priceRange; `responseSummary: null` in Phase 41 |
| EST-08 | Estimate has status lifecycle enforced by API | `EstimateTransitionService` + `ALLOWED_TRANSITIONS` map. **Phase 41 ships the state machine but no HTTP endpoint that triggers a non-CRUD transition** (D-TXN-05) |
| EST-09 | `quote-token` module renamed to `document-token` with `documentType` discriminator; existing quote tokens continue to validate | Pure code rename per D-TKN-01..07; no data migration since no prod data exists |
| CONT-01 | User can set contingency percentage via a slider (0-30% in 5% increments, default 10%) | Request DTO accepts `contingencyPercent: number` with `Min(0) Max(30) IsIn([0,5,10,15,20,25,30])`; slider itself is Phase 43 |
| CONT-02 | Estimate displays price as a range (£X-£Y) by default | API returns `priceRange.low/high`; display logic is Phase 43 |
| CONT-05 | Contingency range math is computed API-side only | `EstimateTotalsCalculator` scales subtotal and tax total by `(1 + c/100)` on every read path per D-CONT-01/02/05 |
| RESP-08 | Customer response data (type, reason, message, timestamp) is persisted on the estimate | Phase 41 reserves nullable fields on entity per D-RESP-02; Phase 45 writes them |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

Extracted from `./CLAUDE.md` (root) and `./trade-flow-api/CLAUDE.md`:

- **Layering is strict:** Controller → Service → Repository → Database. Services NEVER access DB directly. Entities NEVER passed between layers. Controllers NEVER touch entities.
- **DTOs use Luxon `DateTime`; entities use native JS `Date`. Conversion happens in the repository layer** via `toDateTime` / `toOptionalDateTime` from `@core/utilities/to-date-time.utility`.
- **No `any`, no `eslint-disable`, no `@ts-ignore`, no `@ts-expect-error`, no `@ts-nocheck`.** Use `unknown`, generics, or proper types.
- **Avoid `as` type assertions** except for third-party SDK quirks. Domain-to-domain `as` is forbidden — create mapping/type-guard functions instead.
- **Use `Record<string, unknown>` instead of `Record<string, any>`.**
- **Prefer `const`. Use `let` only when intentionally accumulating state.**
- **Use path aliases** (`@estimate/*`, `@core/*`, etc.) — never relative imports across modules. Planner must add `@estimate/*` and `@document-token/*` entries to `tsconfig.json`.
- **Country-agnostic tax field naming.** Never introduce `vat`, `VAT`, `exclVat`, `inclVat`, `vatRate`, `vatRegistered` — use `tax`, `exclTax`, `inclTax`, `taxRate`.
- **Comments:** prefer self-documenting code. Only legal/TODO-with-ticket/non-obvious-consequence comments allowed. Never commit commented-out code.
- **Separation over DRY at entity boundaries** (user feedback memory): duplicate modules/collections over shared-with-discriminator for distinct document types. D-LI-02 is the canonical application of this rule.
- **CI gate policy:** `npm run ci` must pass (unit tests + lint:check + format:check + typecheck) before any phase ships. CI command in API repo: `npm run test -- --passWithNoTests && npm run lint:check && npm run format:check && npm run typecheck`.
- **Response format:** always `createResponse([item])` even for single items. Use `createHttpError(error)` in controller catch blocks.
- **Authentication:** `@UseGuards(JwtAuthGuard)` on authenticated routes. `@Req() request: { user: IUserDto; params: { ... } }` inline typing. Never import `Request` from express.
- **API versioning:** `@Controller("v1")` prefix. All new routes land under `/v1`.
- **OpenAPI contract:** `openapi.yaml` at repo root is canonical. Update when adding/modifying endpoints.
- **Test structure:** `src/{module}/test/{controllers,services,repositories,policies,mocks}/*.spec.ts`. Mock generators in `test/mocks/` use static methods with optional `overrides` parameter.

## Standard Stack

### Core (already installed in trade-flow-api — no new runtime dependencies needed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/common | 11.1.12 | Module/controller/service DI primitives | Project-standard framework [VERIFIED: package.json] |
| @nestjs/jwt | 11.0.0 | JWT auth guard | Auth already set up; Phase 41 reuses `JwtAuthGuard` [VERIFIED: existing code] |
| mongodb | 7.0.0 | Native driver for `findOneAndUpdate`, `$inc`, partial indexes | No Mongoose ODM in this codebase [VERIFIED: package.json + QuoteRepository uses raw driver] |
| luxon | 3.5.1 | DateTime for DTOs | Project DTO standard per CLAUDE.md [VERIFIED: `@core/utilities/to-date-time.utility`] |
| class-validator | 0.14.1 | Request DTO validation decorators (`@IsString`, `@IsNumber`, `@Min`, `@Max`, `@IsIn`) | Global `ValidationPipe` already enforces whitelist + forbidNonWhitelisted [VERIFIED: existing code] |
| class-transformer | 0.5.1 | Transform plain request objects into DTO classes | Auto-applied by `ValidationPipe` [VERIFIED: package.json] |
| jest | 30.2.0 | Test runner | Project-standard; `npm run test` with coverage [VERIFIED: package.json scripts] |
| @nestjs/testing | 11.1.12 | TestingModule with DI | Used in all existing `*.spec.ts` files [VERIFIED: quote-transition.service.spec.ts] |

**Installation:** None. Every dependency required by Phase 41 is already installed. The phase adds no new packages.

### Supporting (existing codebase services — import via path alias)

| Module/Service | Path | Purpose |
|----------------|------|---------|
| `CoreModule` | `@core/core.module` | Global; provides MongoDB connection, fetchers, writers, loggers, audit utilities |
| `AuthorizedCreatorFactory` | `@core/factories/authorized-creator.factory` | Wraps creator repos with policy enforcement. Used by `QuoteCreator`; reuse in `EstimateCreator`. |
| `AccessControllerFactory` | `@core/factories/access-controller.factory` | Wraps read/update with policy `canRead`/`canUpdate` checks. |
| `MongoDbFetcher` / `MongoDbWriter` | `@core/services/mongo/*` | Thin wrappers over `db.collection().findOne/findMany/insertOne/findOneAndUpdate/updateOne/updateMany`. Injected into all repositories. |
| `AppLogger` | `@core/services/app-logger.service` | Pino-backed structured logger. Pattern: `private readonly logger = new AppLogger(ClassName.name)`. |
| `Money` value object | `@core/value-objects/money.value-object` | Immutable money with `add`/`subtract`/`multiply`/`percentage`/`divide`/`percentageOf`/`allocate`/`fromMajorUnits`/`fromMinorUnits`/`toMajorUnits`/`toMinorUnits`/`isZero`/`isNegative` operations. |
| `DtoCollection` / `EntityCollection` | `@core/collections/*` | Wraps arrays with `queryResults` + `queryOptions` for paginated lists. `DtoCollection.empty()` / `DtoCollection.create(data, queryResults, queryOptions)`. |
| `createAuditFields` / `updateAuditFields` | `@core/utilities/*` | Attaches `{ createdAt, updatedAt, createdBy, updatedBy }` fields on entities. |
| `toDateTime` / `toOptionalDateTime` | `@core/utilities/to-date-time.utility` | Date ↔ Luxon DateTime conversion at repository boundary. |
| `DEFAULT_CURRENCY` | `@core/constants/currency.constant` | GBP. Used as the currency argument for `Money.zero()`. |
| `ErrorCodes` enum | `@core/errors/error-codes.enum` | Central enum. **Planner must add new entries for estimate errors.** |
| `InvalidRequestError` / `ResourceNotFoundError` / `ForbiddenError` | `@core/errors/*` | Domain error classes; each has `getCode()`, `getMessage()`, `getDetails()`. Caught by controllers' `createHttpError(error)` utility. |
| `createHttpError` | `@core/errors/handle-error.utility` | Maps domain errors to NestJS HTTP exceptions. Used in every controller catch block. |
| `createResponse` / `createErrorResponse` | `@core/response/create-response.utility` | Wraps response data in `IResponse<T>` envelope. |
| `IResponse<T>` | `@core/response/response.interface` | `{ data?: T[], errors?: IResponseError[], pagination?: IQueryResultsDto }`. |
| `BasePolicy<T>` | `@core/policies/base.policy` | Abstract base for policies; provides `logger` property. Policies extend with `canCreate`/`canRead`/`canUpdate`/`canDelete`. |
| `ICreatorService` / `ICreatorRepository` / `IByIdRetrieverService` / `IByIdRetrieverRepository` | `@core/interfaces/*` | Layer contracts implemented by creator/retriever classes. |
| `IBaseEntity` / `IBaseResourceDto` | `@core/entities/*`, `@core/data-transfer-objects/*` | Base types for entities and DTOs. |
| `CustomerRetriever` | `@customer/services/customer-retriever.service` | Validates customer resolves + is active. Throw `CUSTOMER_INACTIVE` if not. |
| `JobRetrieverService` | `@job/services/job-retriever.service` | Validates job resolves. |
| `ItemRetrieverService` | `@item/services/item-retriever.service` | Looks up items by id; returns `IItemDto` including `bundleConfig` for bundle items. |
| `TaxRateRepository` | `@tax-rate/repositories/tax-rate.repository` | Resolves `taxRateId` → rate percent. |
| `BusinessRepository` | `@business/repositories/business.repository` | Used by session-auth guards to resolve business name for token error responses. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff | Recommendation |
|------------|-----------|----------|----------------|
| Duplicated `EstimateLineItem*` stack | Shared `LineItem` collection with `parentType` discriminator | Less code, but breaks separation-over-DRY, introduces polymorphic queries, couples quote and estimate lifecycles | **Duplicate** (D-LI-02; user's locked preference captured in `feedback_separation_over_dry.md` memory) |
| Store `priceRange` on entity | Derive on every read via calculator | Cache invalidation risk; stale range on contingency edit | **Derive** (D-CONT-05) |
| Single `DocumentStatus` enum | Separate `QuoteStatus` and `EstimateStatus` | Share string values but different allowed transitions; union typing risks cross-domain casts | **Separate** (Pitfall 7) |
| Polymorphic `parentType` on `quote_line_items` | Dedicated `estimatelineitems` collection | See line 1 | **Dedicated collection** (D-LI-02) |

## Architecture Patterns

### Recommended Project Structure

```
trade-flow-api/src/
├── document-token/                        # NEW — renamed from quote-token
│   ├── document-token.module.ts
│   ├── controllers/
│   │   └── public-quote.controller.ts     # MOVED/refactored: reads req.documentToken, asserts documentType === "quote"
│   ├── data-transfer-objects/
│   │   └── document-token.dto.ts          # IDocumentTokenDto — field shape: { token, documentType, documentId, expiresAt, revokedAt?, sentAt, recipientEmail, firstViewedAt? }
│   ├── entities/
│   │   └── document-token.entity.ts       # IDocumentTokenEntity
│   ├── guards/
│   │   └── document-session-auth.guard.ts # DocumentSessionAuthGuard; attaches req.documentToken
│   ├── repositories/
│   │   └── document-token.repository.ts   # COLLECTION = "document_tokens"
│   ├── requests/
│   │   └── decline-quote.request.ts       # preserved
│   ├── responses/
│   │   └── public-quote.response.ts       # preserved
│   └── services/
│       ├── public-quote-retriever.service.ts
│       ├── document-token-creator.service.ts
│       ├── document-token-retriever.service.ts
│       ├── document-token-revoker.service.ts
│       └── quote-response-handler.service.ts
│
├── estimate/                              # NEW
│   ├── estimate.module.ts
│   ├── controllers/
│   │   └── estimate.controller.ts         # POST/GET/GET/PATCH/DELETE — CRUD only per D-TXN-05
│   ├── services/
│   │   ├── estimate-creator.service.ts
│   │   ├── estimate-retriever.service.ts
│   │   ├── estimate-updater.service.ts
│   │   ├── estimate-deleter.service.ts
│   │   ├── estimate-number-generator.service.ts
│   │   ├── estimate-totals-calculator.service.ts
│   │   ├── estimate-transition.service.ts
│   │   ├── estimate-standard-line-item-factory.service.ts
│   │   ├── estimate-bundle-line-item-factory.service.ts
│   │   ├── estimate-line-item-creator.service.ts
│   │   └── estimate-line-item-retriever.service.ts
│   ├── repositories/
│   │   ├── estimate.repository.ts         # COLLECTION = "estimates"
│   │   └── estimate-line-item.repository.ts  # COLLECTION = "estimatelineitems"
│   ├── entities/
│   │   ├── estimate.entity.ts
│   │   └── estimate-line-item.entity.ts
│   ├── data-transfer-objects/
│   │   ├── estimate.dto.ts
│   │   ├── estimate-line-item.dto.ts
│   │   ├── estimate-totals.dto.ts
│   │   ├── estimate-price-range.dto.ts
│   │   ├── estimate-response-summary.dto.ts
│   │   ├── bundle-line-item-group.dto.ts  # mirror quote version
│   │   └── component-blueprint.dto.ts     # mirror quote version
│   ├── requests/
│   │   ├── create-estimate.request.ts
│   │   ├── update-estimate.request.ts
│   │   ├── list-estimates.request.ts      # query params for pagination + status filter
│   │   ├── create-estimate-line-item.request.ts
│   │   └── update-estimate-line-item.request.ts
│   ├── responses/
│   │   └── estimate.responses.ts          # IEstimateResponse, IEstimateLineItemResponse
│   ├── policies/
│   │   ├── estimate.policy.ts
│   │   └── estimate-line-item.policy.ts
│   ├── enums/
│   │   ├── estimate-status.enum.ts
│   │   ├── estimate-line-item-status.enum.ts
│   │   ├── estimate-display-mode.enum.ts
│   │   ├── estimate-response-type.enum.ts
│   │   ├── estimate-decline-reason.enum.ts
│   │   ├── estimate-transitions.ts
│   │   └── estimate-transitions.spec.ts
│   └── test/
│       ├── controllers/estimate.controller.spec.ts
│       ├── services/
│       │   ├── estimate-creator.service.spec.ts
│       │   ├── estimate-retriever.service.spec.ts
│       │   ├── estimate-updater.service.spec.ts
│       │   ├── estimate-deleter.service.spec.ts
│       │   ├── estimate-number-generator.service.spec.ts
│       │   ├── estimate-totals-calculator.service.spec.ts
│       │   ├── estimate-transition.service.spec.ts
│       │   ├── estimate-standard-line-item-factory.service.spec.ts
│       │   ├── estimate-bundle-line-item-factory.service.spec.ts
│       │   ├── estimate-line-item-creator.service.spec.ts
│       │   └── estimate-line-item-retriever.service.spec.ts
│       ├── repositories/
│       │   ├── estimate.repository.spec.ts
│       │   └── estimate-line-item.repository.spec.ts
│       ├── policies/
│       │   ├── estimate.policy.spec.ts
│       │   └── estimate-line-item.policy.spec.ts
│       └── mocks/
│           ├── estimate-mock-generator.ts
│           └── estimate-line-item-mock-generator.ts
│
├── item/
│   ├── services/                          # MODIFIED — lift bundle helpers here
│   │   ├── bundle-config-validator.service.ts   # MOVED from @quote/services
│   │   ├── bundle-pricing-planner.service.ts    # MOVED from @quote/services
│   │   ├── bundle-tax-rate-calculator.service.ts # MOVED + generalised to ILineItemTaxInput
│   │   └── (existing) item-creator.service.ts, item-retriever.service.ts, item-updater.service.ts
│   └── interfaces/                        # NEW directory
│       └── line-item-tax-input.interface.ts  # NEW — structural { unitPrice: Money; quantity: number; taxRate: number }
│
├── quote/                                 # MODIFIED
│   ├── quote.module.ts                    # remove BundleConfigValidator/BundlePricingPlanner/BundleTaxRateCalculator from providers (they're now in ItemModule)
│   ├── services/                          # DELETE: bundle-config-validator/bundle-pricing-planner/bundle-tax-rate-calculator
│   │   └── quote-bundle-line-item-factory.service.ts  # UPDATE imports: @quote/services → @item/services
│   └── controllers/quote.controller.ts    # UPDATE: import QuoteTokenRetriever from @document-token/services (or rename to DocumentTokenRetriever)
│
├── quote-token/                           # DELETED ENTIRELY
│
├── app.module.ts                          # MODIFIED: import QuoteTokenModule → DocumentTokenModule, add EstimateModule
└── tsconfig.json                          # MODIFIED: add @estimate/*, @estimate-test/*, @document-token/*, @document-token-test/*; remove @quote-token/*, @quote-token-test/*
```

### Pattern 1: Counter-Based Atomic Numbering

**What:** Generate `E-YYYY-NNN` sequential numbers per `{ businessId, year }` using MongoDB's `findOneAndUpdate` with `$inc` and `upsert: true`. No race condition, no read-then-write.

**When to use:** Every estimate creation. Called exactly once inside `EstimateCreator.create` before passing the DTO to `AuthorizedCreatorFactory`.

**Example (verified, verbatim from `trade-flow-api/src/quote/services/quote-number-generator.service.ts`):**

```typescript
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { MongoConnectionService } from "@core/services/mongo/mongo-connection.service";
import { ObjectId } from "mongodb";

interface IEstimateCounter {
  _id: ObjectId;
  businessId: ObjectId;
  year: number;
  lastNumber: number;
}

@Injectable()
export class EstimateNumberGenerator {
  private static readonly COLLECTION = "estimate_counters";

  constructor(private readonly connection: MongoConnectionService) {}

  public async generateNumber(businessId: string): Promise<string> {
    const year = DateTime.now().year;
    const db = await this.connection.getDb();

    const result = await db
      .collection<IEstimateCounter>(EstimateNumberGenerator.COLLECTION)
      .findOneAndUpdate(
        { businessId: new ObjectId(businessId), year },
        { $inc: { lastNumber: 1 } },
        { upsert: true, returnDocument: "after" },
      );

    const lastNumber = result?.lastNumber ?? 1;
    return `E-${year}-${String(lastNumber).padStart(3, "0")}`;
  }
}
```

This is a literal mirror of `QuoteNumberGenerator` with `quote_counters → estimate_counters` and `Q- → E-`. Note that `MongoConnectionService` (not `MongoDbFetcher`/`MongoDbWriter`) is used here because the counter needs a raw `db.collection(...)` handle for the `findOneAndUpdate` options shape. Mirror this exactly.

### Pattern 2: Totals Calculator with Contingency Range

**What:** Pure service that takes an `IEstimateDto` (with `lineItems` populated), iterates non-deleted non-child line items, computes per-line `unitPrice × quantity` and `taxRate` percentage, returns a new DTO with `totals` AND `priceRange` attached. Called on every service-layer read path (creator, retriever, updater).

**Example (extended from `QuoteTotalsCalculator`):**

```typescript
import { DEFAULT_CURRENCY } from "@core/constants/currency.constant";
import { Money } from "@core/value-objects/money.value-object";
import { Injectable } from "@nestjs/common";
import { IEstimateTotalsDto } from "@estimate/data-transfer-objects/estimate-totals.dto";
import { IEstimatePriceRangeDto } from "@estimate/data-transfer-objects/estimate-price-range.dto";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";

@Injectable()
export class EstimateTotalsCalculator {
  public calculateTotals(estimate: IEstimateDto): IEstimateDto {
    let subTotal = Money.zero(DEFAULT_CURRENCY);
    let taxTotal = Money.zero(DEFAULT_CURRENCY);

    for (const lineItem of estimate.lineItems) {
      if (lineItem.parentLineItemId) continue;
      if (lineItem.status === EstimateLineItemStatus.DELETED) continue;

      const lineItemTotal = lineItem.unitPrice.multiply(lineItem.quantity);
      const lineItemTax = lineItemTotal.percentage(lineItem.taxRate);

      subTotal = subTotal.add(lineItemTotal);
      taxTotal = taxTotal.add(lineItemTax);
    }

    const totals: IEstimateTotalsDto = {
      subTotal,
      taxTotal,
      total: subTotal.add(taxTotal),
    };

    const priceRange = this.buildPriceRange(estimate, totals);

    return { ...estimate, totals, priceRange };
  }

  private buildPriceRange(estimate: IEstimateDto, totals: IEstimateTotalsDto): IEstimatePriceRangeDto {
    // D-CONT-01: base is the floor; contingency is pure upside.
    // D-CONT-02: scale both subtotal and tax total by (1 + c%) for the high end.
    const multiplier = 1 + estimate.contingencyPercent / 100;
    const highSubTotal = totals.subTotal.multiply(multiplier);
    const highTaxTotal = totals.taxTotal.multiply(multiplier);

    return {
      displayMode: estimate.displayMode,
      subtotal: totals.subTotal,
      contingencyPercent: estimate.contingencyPercent,
      low: {
        exclTax: totals.subTotal,
        tax: totals.taxTotal,
        inclTax: totals.subTotal.add(totals.taxTotal),
      },
      high: {
        exclTax: highSubTotal,
        tax: highTaxTotal,
        inclTax: highSubTotal.add(highTaxTotal),
      },
    };
  }
}
```

**Important:** `Money.multiply(number)` exists — verified in `QuoteTotalsCalculator` (line 22: `lineItem.unitPrice.multiply(lineItem.quantity)`) and `QuoteBundleLineItemFactory` (line 135). Multiplying by `1 + c/100` is a valid call. If precision concerns arise (e.g., 17.5% contingency would produce fractional minor units), the planner may prefer a whole-integer scaling approach — but since `contingencyPercent` is constrained to `[0, 5, 10, 15, 20, 25, 30]` the multipliers are always `{1, 1.05, 1.10, 1.15, 1.20, 1.25, 1.30}`. **Validate in a unit test that every multiplier produces penny-exact values.**

### Pattern 3: Transition Service + Static Transition Map

**What:** A static readonly `Map<EstimateStatus, EstimateStatus[]>` of allowed transitions, two pure helper functions (`isValidTransition`, `getValidTransitions`), and an injectable `EstimateTransitionService` that calls the helper, enforces ownership via `AccessControllerFactory`, throws `InvalidRequestError(ESTIMATE_INVALID_TRANSITION, ...)` on rejection, updates timestamps (`sentAt`, `firstViewedAt`, `respondedAt`, `convertedAt`, `declinedAt`, `lostAt`, `deletedAt`), and persists via repository.

**Verified from `src/quote/enums/quote-transitions.ts` and `quote-transition.service.ts`:** this pattern is already proven. Mirror it exactly. See the full locked `ALLOWED_TRANSITIONS` map in CONTEXT.md <specifics> lines 540-584.

**Timestamp-setting lives in the transition service**, not in the controller. Example (current quote implementation):

```typescript
const updated: IQuoteDto = { ...existing, status: targetStatus };
if (targetStatus === QuoteStatus.SENT) updated.sentAt = DateTime.now();
else if (targetStatus === QuoteStatus.ACCEPTED) updated.acceptedAt = DateTime.now();
else if (targetStatus === QuoteStatus.DELETED) {
  updated.deletedAt = DateTime.now();
  await this.quoteTokenRevoker.revokeAllForQuote(existing.id);
}
return this.quoteRepository.update(updated);
```

For `EstimateTransitionService` the timestamps to set on each status are:
- `SENT` → `sentAt`
- `VIEWED` → `firstViewedAt` (only if not already set — Pitfall 9)
- `RESPONDED` → `respondedAt`
- `SITE_VISIT_REQUESTED` → `respondedAt` (if not already set)
- `CONVERTED` → `convertedAt` (Phase 47 sets `convertedToQuoteId`)
- `DECLINED` → `declinedAt`
- `EXPIRED` → (no new timestamp; expiry is relative to `sentAt`)
- `LOST` → `lostAt`
- `DELETED` → `deletedAt` (Phase 41's soft delete)

**IMPORTANT:** Phase 41 only exercises the `DRAFT → DELETED` path through HTTP (via `EstimateDeleter`). The service methods for all other transitions exist and are wired into DI, but there is no controller endpoint that calls them. Phase 44/45/46/47 later import `EstimateTransitionService` from `EstimateModule.exports` and call it internally.

### Pattern 4: Soft-Delete via Status + deletedAt + Partial Unique Index

**What:** A soft-deleted estimate has `status = DELETED` AND `deletedAt = DateTime.now()`. List queries filter by `{ status: { $ne: DELETED } }` OR `{ deletedAt: null }` (the current `QuoteRepository.findQuotesByBusinessId` uses the `$ne: DELETED` filter — line 50). The unique index on `(businessId, number)` has `partialFilterExpression: { deletedAt: null }` so a deleted estimate's number is implicitly free for reuse — although in practice counters never revert, so collisions are impossible anyway.

**Line items are NOT soft-deleted alongside the estimate** (D-DRAFT-03). Rationale: history preservation for audit / future revisions. The `estimatelineitems` query filter is applied per-line-item by checking `status !== EstimateLineItemStatus.DELETED` within the line-item repository.

### Pattern 5: Policy Separates "Who" from "Whether"

**What:** `EstimatePolicy extends BasePolicy<IEstimateDto>` only checks business-ownership (`authUser.businessIds.includes(resource.businessId)`). `canCreate`, `canRead`, `canUpdate` all return `true` for same-business users and support users. `canDelete` returns `false` by default (matches `QuotePolicy.canDelete = false` — verified line 51-53).

**Status-based edit/delete gates live in `EstimateUpdater` and `EstimateDeleter` services, not the policy** — they re-fetch the estimate and throw `InvalidRequestError(ESTIMATE_NOT_EDITABLE)` if `status !== DRAFT`. This is D-DRAFT-02.

### Pattern 6: Paginated List with Status Tab Filtering (BUILD FROM SCRATCH)

**⚠️ CRITICAL:** The existing `QuoteController.findAll` does NOT paginate. It returns a flat array via `findAllByBusinessId` with no `take/skip/limit/offset` and no pagination metadata in the response. The context claim "mirror existing quote list endpoint" is misleading — **there is no pattern to mirror for paginated estimates**. The planner must build this from scratch.

**Recommended approach (uses existing infrastructure):**

1. `ListEstimatesRequest` class in `src/estimate/requests/list-estimates.request.ts`:
   ```typescript
   import { IsEnum, IsInt, IsOptional, Max, Min } from "class-validator";
   import { Type } from "class-transformer";
   import { EstimateStatus } from "@estimate/enums/estimate-status.enum";

   export class ListEstimatesRequest {
     @IsOptional()
     @IsEnum(EstimateStatus)
     status?: EstimateStatus;

     @IsOptional()
     @Type(() => Number)
     @IsInt()
     @Min(1)
     @Max(100)
     limit?: number;

     @IsOptional()
     @Type(() => Number)
     @IsInt()
     @Min(0)
     offset?: number;
   }
   ```

2. `EstimateRepository.findPaginatedByBusinessId(businessId, status?, limit, offset)` uses `MongoDbFetcher.findMany` with a MongoDB `{ skip, limit, sort: { createdAt: -1 } }` options object. (Verify `MongoDbFetcher.findMany` signature — it already returns an `EntityCollection` which carries `queryResults` / `queryOptions`, so pagination metadata is already plumbed through the wrapper layer. The planner should read `src/core/services/mongo/mongo-db-fetcher.service.ts` to confirm exact param shape before writing the repository method.)

3. Return `IResponse<IEstimateResponse>` with both `data` and `pagination` populated. The `pagination: IQueryResultsDto` envelope is already a declared field of `IResponse<T>`.

4. Tab filtering is simply `{ businessId, status: { $ne: DELETED }, ...(status ? { status } : {}) }` in the filter expression.

**If the MongoDbFetcher does NOT support pagination natively**, the planner should either (a) extend `MongoDbFetcher.findMany` to accept options `{ limit, skip, sort }` as a small core-layer change, or (b) access the `MongoConnectionService.getDb()` directly in the estimate repository for the paginated query (similar to how `QuoteNumberGenerator` bypasses the wrappers for `findOneAndUpdate`). **Planner must verify which path is viable before committing.**

### Pattern 7: Session-Auth Guard with documentType Discriminator

**What:** `DocumentSessionAuthGuard` extracts `:token` from route params, looks up via `DocumentTokenRepository.findByToken`, rejects with 404 if missing, 410 GONE if expired/revoked, attaches `request.documentToken: IDocumentTokenDto`. The controller handler then asserts the expected `documentType`:

```typescript
@UseGuards(DocumentSessionAuthGuard)
@Get("quote/:token")
public async findByToken(@Req() request: { documentToken: IDocumentTokenDto }): Promise<IResponse<IPublicQuoteResponse>> {
  if (request.documentToken.documentType !== "quote") {
    throw createHttpError(new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "Quote not found"));
  }
  try {
    const response = await this.publicQuoteRetriever.getPublicQuote(request.documentToken.documentId);
    return createResponse([response]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

Note the guard is unaware of the discriminator — it simply loads the token. Discrimination is a controller-level concern so future public estimate controllers in Phase 45 reuse the same guard without branching inside it.

### Anti-Patterns to Avoid

- **Polymorphic line-item collection.** Rejected in D-LI-02. Do not add `parentType` to `quote_line_items`. Do not query `{ parentType: "estimate" }`.
- **Storing `priceRange` on the entity.** Rejected in D-CONT-05. Compute on every read path. Any persisted range will drift the moment `contingencyPercent` changes and a revision reads it without recomputation.
- **Casting `EstimateStatus` to/from `QuoteStatus`.** Rejected by Pitfall 7. Both enums include `draft` and `sent` string values but they are different domain types. Use two separate typed variables.
- **`as unknown as T`.** Rejected by project rule. Use type guards or mapping functions.
- **Reading quote helpers from the estimate module** (`import { BundleConfigValidator } from "@quote/services/..."`). Rejected in D-01. Move them to `@item/services` first.
- **Timestamp-setting in controllers.** Transition timestamps belong in `EstimateTransitionService`, not in controller handlers. The controller doesn't know which side-effects accompany a transition.
- **Running the token rename as a migration.** Rejected in D-TKN-01. Pure code rename. No `IMigration` file for Phase 41.
- **Dropping `quote_tokens` at startup.** Migrations never run on startup (D-TKN-06). If a stale collection ever needs dropping in the future, it lands as `POST /v1/migrations/run`.
- **Importing `Request` from express.** Project rule. Use inline typed `@Req()` objects.
- **Forgetting to update `openapi.yaml`.** CLAUDE.md is explicit — any new endpoint must have an OpenAPI entry. Planner should include an OpenAPI update task alongside the controller task.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Atomic numbering | `findOne → lastNumber+1 → updateOne` read-then-write | `findOneAndUpdate` with `$inc` + `upsert: true` (Pattern 1) | Race condition under concurrent creates. Pitfall 8. |
| JWT verification | Manual header parse + Firebase public key fetch | `@UseGuards(JwtAuthGuard)` from `@auth/auth.guard` | Already set up with JWKS refresh and user resolution |
| Business-scope access control | `if (user.businessIds.includes(resource.businessId))` inline checks | `AccessControllerFactory.create(policy).canRead/canUpdate/canCreate(user, resource)` | Centralises policy, produces consistent `ForbiddenError`, integrates with `AuthorizedCreatorFactory` |
| Error-to-HTTP mapping | Custom `try/catch` that builds `HttpException` manually | `createHttpError(error)` utility | Consistent status-code mapping across all controllers |
| Money arithmetic | Raw `number * number` with rounding | `Money.multiply/percentage/add/subtract/divide/allocate` from `@core/value-objects` | Precision bugs on GBP/pence; existing value object handles minor-unit conversion |
| Date/time arithmetic | `new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)` | `DateTime.now().plus({ days: 30 })` | Project standard Luxon; DST safety; timezone clarity |
| Status-machine validation | Imperative `if (from === X && to === Y) ...` chains | Static `Map` of allowed targets + pure `isValidTransition(from, to)` helper (Pattern 3) | Testable in isolation; mirrors existing quote convention; diff-readable when adding new states |
| Soft-delete semantics | Hard `deleteOne` + audit log | Status-based `{ status: DELETED, deletedAt: now() }` + partial unique index | Preserves history; matches EST-05 |
| Audit fields on create/update | Manual `{ createdAt: new Date(), updatedAt: new Date(), createdBy: user.id }` | `createAuditFields()` / `updateAuditFields()` from `@core/utilities` | Ensures consistent shape across entities; centralised change if audit schema evolves |
| Pagination envelope | Custom response wrapper | `IResponse<T>` with `pagination?: IQueryResultsDto` (already declared) | Envelope already consumed by frontend; consistent contract |
| Rate limiting on public endpoints | Custom middleware | `@Throttle({ default: { limit: N, ttl: 60000 } })` + `ThrottlerGuard` (already applied on `PublicQuoteController`) | Already wired in `QuoteTokenModule`; mirror in `DocumentTokenModule` |
| Request body validation | Manual `if (!body.customerId) throw ...` | class-validator decorators on `CreateEstimateRequest` / `UpdateEstimateRequest` | Global `ValidationPipe` auto-applies; consistent error format |
| Test fixtures | Per-test ad-hoc object literals | `EstimateMockGenerator.createEstimateDto(overrides?)` in `test/mocks/` | Matches existing convention (`QuoteTransitionServiceSpec` uses an inline helper — the project pattern is a static class method) |

**Key insight:** Phase 41 is pure scaffolding plus one refactor (lifting bundle helpers). Every primitive the planner needs is already in the codebase. Writing new "helper" or "util" code is a smell — if a helper seems missing, it either already exists under `@core/` or is a mirror of an existing quote-module file.

## Runtime State Inventory

This phase is a **rename + scaffolding** phase. State categories audited below.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | `quote_tokens` MongoDB collection exists in production? | **No migration needed** — D-TKN-01 explicitly states "application has no production data". Verify this assumption holds by checking the Railway MongoDB before executing the rename. If production data exists, the plan must absorb a data migration. |
| Live service config | None. No external services carry the string `"quote-token"` or `"quote_tokens"` outside this repo (no n8n, no external webhook URL includes the string, no scheduled job references it). | None. |
| OS-registered state | None. No cron, systemd, pm2, or Task Scheduler entry references `quote-token`. `npm run` scripts in `package.json` do not reference it. | None — verified by grepping `trade-flow-api/package.json` and `railway.json`. |
| Secrets/env vars | None. No `.env` or SOPS-encrypted secret contains `QUOTE_TOKEN_*` or `DOCUMENT_TOKEN_*`. `JWT_SECRET`, `FIREBASE_*`, `MONGO_URL`, `REDIS_URL` are the only auth-adjacent env vars and none reference the quote-token module. | None. |
| Build artifacts / installed packages | `trade-flow-api/dist/` rebuild on next `npm run build` will regenerate the module graph. No npm package carries `quote-token` in its name. TypeScript path aliases need updating in `tsconfig.json` (`@quote-token/*` → `@document-token/*`). | Update `tsconfig.json` paths. Add `@estimate/*` / `@estimate-test/*` and `@document-token/*` / `@document-token-test/*` entries; remove `@quote-token/*` / `@quote-token-test/*`. Drop stale `.js.map` / `.d.ts` from `dist/` on next build (will happen automatically). |

**Canonical question answered:** *After every file in the repo is updated, what runtime systems still have the old string cached, stored, or registered?* **Answer: only the MongoDB `quote_tokens` collection, and only if production data already exists.** D-TKN-01 asserts it does not. Planner task: add a pre-execution verification step that runs `mongo --eval 'db.quote_tokens.countDocuments({})'` or equivalent against the production connection string and aborts if the count is non-zero (unless the human operator overrides with a one-shot rename migration).

## Common Pitfalls

### Pitfall 1: Counter Race Under Concurrent Create

**What goes wrong:** Two simultaneous `POST /v1/estimates` calls to the same business produce the same `E-YYYY-001` number, violating the partial unique index. The second insert throws an 11000 duplicate key error and returns 500 to the user.

**Why it happens:** Developer reads `lastNumber` from counter, increments in memory, writes back. Two reads see the same value.

**How to avoid:** Use `findOneAndUpdate` with `$inc` and `upsert: true` — this is atomic at the MongoDB level. Never split the read and the increment. See Pattern 1.

**Warning signs:** Any code that does `findOne(counter) → counter.lastNumber + 1 → updateOne(counter, { lastNumber: ... })` is wrong. Reject in code review.

### Pitfall 2: Contingency Math Drift Between API and UI

**What goes wrong:** Frontend re-computes `high = subtotal * (1 + c/100)` in JavaScript. Rounding differs from API. Customer-facing range `£1000-£1100` on API matches backend but frontend rounds to `£1000-£1099.99`, and a golden-file test fails.

**Why it happens:** Fractional pennies when multiplying `Money × 1.175` (at the mid-multiplier 17.5%, not in Phase 41's allowed set). Or the UI developer assumes the multiplier and the backend assumes the bounds.

**How to avoid:** CONT-05 is explicit — API computes, UI displays. `priceRange.low` and `priceRange.high` are API-authoritative. Phase 43 must consume these fields unchanged. For Phase 41's unit tests, write a parameterised test over `contingencyPercent ∈ [0, 5, 10, 15, 20, 25, 30]` that asserts `high = low × multiplier` penny-exact and every multiplier produces a valid Money value (no fractional minor units). Also write an "API is the only computer" integration test later in Phase 43 that asserts a golden JSON response.

**Warning signs:** Any frontend code that imports `contingencyPercent` and does arithmetic with it. Any API code that ships `low` without `high` (or vice versa).

### Pitfall 3: Token-Type Confusion After Rename

**What goes wrong:** A Phase 45 public estimate controller naively reads `req.documentToken.documentId` expecting an estimate id, but the same token guard let through a token with `documentType: "quote"` (because someone called `/v1/public/estimate/:token` with a quote token). The estimate retriever throws `ResourceNotFoundError` because no estimate has that id. Unclear error message to customer.

**Why it happens:** Shared guard, different controllers, developer forgets the discriminator assertion.

**How to avoid:** Every public controller that uses `DocumentSessionAuthGuard` MUST start with `if (request.documentToken.documentType !== "quote"/"estimate") throw 404`. Add a comment: `// Token-type assertion: this controller only handles <quote|estimate> tokens.` Add a unit test per controller that passes a wrong-type token and asserts 404.

**Warning signs:** Any handler method that reads `request.documentToken.documentId` without a preceding type check.

### Pitfall 4: Partial Unique Index Not Actually Enforcing

**What goes wrong:** The partial unique index on `(businessId, number)` with `partialFilterExpression: { deletedAt: null }` is created, but a soft-deleted estimate's document has `deletedAt: undefined` (not stored) rather than `deletedAt: null`. MongoDB's partial filter with `{ deletedAt: null }` treats a missing field the same as `null`, so the index still applies and a soft-deleted + new-with-same-number combo is rejected.

**Why it happens:** `IQuoteEntity.deletedAt?: Date` is optional in TypeScript, so `createAuditFields()` never writes `deletedAt: null` on create. MongoDB partial indexes with `{ field: null }` match both missing and explicit `null` by default, which happens to be the behaviour we want — but only if the soft-delete code actually writes `deletedAt: <some Date>` on delete. If it writes `deletedAt: null` (clearing the field) the index treats it as active.

**How to avoid:** `EstimateDeleter.softDelete(id)` must write `{ $set: { status: DELETED, deletedAt: new Date() } }` — never `null`. Unit test the deleter to confirm `deletedAt instanceof Date` after the call. Verify the index definition in the repository migration or constructor matches.

**Warning signs:** Any code that explicitly writes `deletedAt: null`. Any index query that uses `$exists: false` on `deletedAt` (inconsistent with the partial filter).

### Pitfall 5: DRAFT-Only Check Bypassed by Stale DTO

**What goes wrong:** `EstimateUpdater` receives a DTO with `status = DRAFT` from the controller (pass-through from the request). Meanwhile the estimate has been transitioned to `SENT` by a concurrent Phase 44 call. The updater only reads the DTO's `status`, sees `DRAFT`, and overwrites the SENT estimate's line items.

**Why it happens:** Controller passes request through as DTO; service trusts it; never re-fetches.

**How to avoid:** D-DRAFT-01 is explicit: **`EstimateUpdater` and `EstimateDeleter` re-fetch the current status of the estimate from the repository** before making any writes, then check against the freshly loaded status. The pattern mirrors `QuoteUpdater.addLineItem` (line 47: `const quote = await this.quoteRepository.findByIdOrFail(quoteId);`).

**Warning signs:** Any service method that reads `dto.status` without a preceding `findByIdOrFail(dto.id)`.

### Pitfall 6: "Mirror existing quote pagination" Is a Trap

**What goes wrong:** Planner reads "mirror existing quote list endpoint" in the context and assumes `QuoteController.findAll` already has pagination. It does not. Planner writes a PLAN that says "copy the pagination pattern from `quote-retriever.service.ts`" — but there is nothing to copy.

**Why it happens:** CONTEXT.md doesn't flag this gap (it was written under the assumption the pattern existed). This research is the correction.

**How to avoid:** Plan pagination from scratch. Read `src/core/services/mongo/mongo-db-fetcher.service.ts` and `src/core/collections/dto.collection.ts` before writing the repository method so the pagination envelope flows through the existing `DtoCollection` mechanism. See Pattern 6. Consider whether to extend `MongoDbFetcher.findMany` to accept `{ limit, skip, sort }` options, or bypass it for paginated queries.

**Warning signs:** A plan task that says "copy `findAllByBusinessId` from `QuoteRepository`". That method returns a flat array.

### Pitfall 7: Bundle Helper Refactor Breaks Quote Module Tests

**What goes wrong:** Lifting `BundleConfigValidator` / `BundlePricingPlanner` / `BundleTaxRateCalculator` from `@quote/services/` to `@item/services/` changes the import path. If the quote module's `QuoteBundleLineItemFactory` still imports from the old path, the test suite compiles but injection fails at runtime. Or worse, a `quote-bundle-line-item-factory.service.spec.ts` mocks the old `@quote/services/bundle-config-validator.service` path, passes locally, but the runtime wiring is already broken.

**Why it happens:** TypeScript path aliases don't error-check import-from-deleted-file if the compiled output has been cached.

**How to avoid:** Three-step refactor as a single atomic commit:
1. Move files (preserve git history via `git mv`).
2. Update every `import` statement in `QuoteBundleLineItemFactory` and `QuoteModule.providers` array (remove the three services from quote providers — they'll come from `ItemModule`).
3. Add the three services to `ItemModule.providers` and `ItemModule.exports`. `EstimateModule` imports `ItemModule` (already does by the context) and gets them transitively.

After the move, run `npm run typecheck` AND `npm run test -- --testPathPattern=quote`. The tests must still pass without modification (because the class names and public shapes are unchanged; only the import path moved). If `BundleTaxRateCalculator` is generalised to accept `ILineItemTaxInput` instead of `IQuoteLineItemDto[]`, the quote bundle factory callsite also needs updating — add this as a fourth sub-task.

**Warning signs:** `Nest can't resolve dependencies of the QuoteBundleLineItemFactory` at startup. A test spec that imports from `@quote/services/bundle-*` after the move.

### Pitfall 8: `firstViewedAt` Double-Write

**What goes wrong:** Phase 45 will wire customer-facing view tracking. Phase 41 reserves the entity field. But the existing `QuoteTokenRepository.updateFirstViewedAt` uses a conditional filter `{ _id: tokenId, firstViewedAt: { $exists: false } }` (line 52). The same pattern must carry over to `DocumentTokenRepository`. If the field is reserved with an initial value of `undefined` / missing, the conditional filter works. If it's explicitly `null`, the filter `{ firstViewedAt: { $exists: false } }` no longer matches and the timestamp gets rewritten every view.

**Why it happens:** Migrated / mismatched initialization.

**How to avoid:** `DocumentTokenRepository.toEntity` must NOT write `firstViewedAt: null` / `firstViewedAt: undefined` explicitly on create. Just omit the field entirely. Write a unit test: create a document token, assert `entity.firstViewedAt === undefined` (not `null`, not set).

**Warning signs:** Any repo mapper that sets `firstViewedAt` when the DTO doesn't provide it.

### Pitfall 9: CI Gate Failure on `any` or Type Assertion

**What goes wrong:** A quick test passes, `npm run test` passes, but `npm run ci` fails on `lint:check` because the developer used `as unknown as IEstimateDto` to satisfy a tricky union. Trade Flow's ESLint flags `@typescript-eslint/no-explicit-any` as warn and `prettier/prettier` as error — a lint warning doesn't fail CI, but unused variables and `any` without underscore prefix do.

**Why it happens:** Shortcut during scaffolding. The strict-check CI target (`typecheck:strict`) runs `tsconfig-check.json` which adds `strict`, `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`.

**How to avoid:** Every new file must compile clean under BOTH `npm run typecheck` AND `npm run typecheck:strict`. The planner should add a task-level verification step: `npm run ci` must pass after each sub-task, not only at phase end.

**Warning signs:** Any `as unknown as T`, any unused local, any missing return statement in a non-void function.

### Pitfall 10: Forgetting `createResponse([item])` Wrapping

**What goes wrong:** Controller returns `{ data: estimate }` instead of `createResponse([estimate])`. Frontend receives `{ data: { id, ... } }` instead of `{ data: [{ id, ... }] }`. Every consumer breaks silently.

**Why it happens:** Developer fatigue / single-item optimism.

**How to avoid:** Every `return` in every new controller handler uses `createResponse([...])`. Code review rule. See `QuoteController.findOne` line 50 for the canonical pattern.

**Warning signs:** A controller response type of `IResponse<IEstimateResponse>` with a `return { data: ... }` instead of `return createResponse([...])`.

## Code Examples

### Module Wiring (Verified Pattern from `QuoteModule`)

```typescript
// Source: trade-flow-api/src/quote/quote.module.ts (adapted)
import { BusinessModule } from "@business/business.module";
import { CoreModule } from "@core/core.module";
import { CustomerModule } from "@customer/customer.module";
import { DocumentTokenModule } from "@document-token/document-token.module";
import { ItemModule } from "@item/item.module";
import { JobModule } from "@job/job.module";
import { forwardRef, Module } from "@nestjs/common";
import { EstimateController } from "@estimate/controllers/estimate.controller";
import { EstimatePolicy } from "@estimate/policies/estimate.policy";
import { EstimateLineItemPolicy } from "@estimate/policies/estimate-line-item.policy";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateLineItemRepository } from "@estimate/repositories/estimate-line-item.repository";
import { EstimateCreator } from "@estimate/services/estimate-creator.service";
import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
import { EstimateUpdater } from "@estimate/services/estimate-updater.service";
import { EstimateDeleter } from "@estimate/services/estimate-deleter.service";
import { EstimateNumberGenerator } from "@estimate/services/estimate-number-generator.service";
import { EstimateTotalsCalculator } from "@estimate/services/estimate-totals-calculator.service";
import { EstimateTransitionService } from "@estimate/services/estimate-transition.service";
import { EstimateStandardLineItemFactory } from "@estimate/services/estimate-standard-line-item-factory.service";
import { EstimateBundleLineItemFactory } from "@estimate/services/estimate-bundle-line-item-factory.service";
import { EstimateLineItemCreator } from "@estimate/services/estimate-line-item-creator.service";
import { EstimateLineItemRetriever } from "@estimate/services/estimate-line-item-retriever.service";
import { TaxRateModule } from "@tax-rate/tax-rate.module";
import { UserModule } from "@user/user.module";

@Module({
  imports: [
    CoreModule,
    forwardRef(() => UserModule),
    CustomerModule,
    BusinessModule,
    ItemModule,
    TaxRateModule,
    JobModule,
    forwardRef(() => DocumentTokenModule),
  ],
  controllers: [EstimateController],
  providers: [
    EstimateRepository,
    EstimateRetriever,
    EstimateCreator,
    EstimateUpdater,
    EstimateDeleter,
    EstimatePolicy,
    EstimateTotalsCalculator,
    EstimateNumberGenerator,
    EstimateTransitionService,
    EstimateStandardLineItemFactory,
    EstimateBundleLineItemFactory,
    EstimateLineItemCreator,
    EstimateLineItemRetriever,
    EstimateLineItemRepository,
    EstimateLineItemPolicy,
  ],
  exports: [EstimateRepository, EstimateTransitionService, EstimateTotalsCalculator],
})
export class EstimateModule {}
```

### Controller Handler (Verified Pattern from `QuoteController`)

```typescript
// Source: trade-flow-api/src/quote/controllers/quote.controller.ts (adapted)
import { JwtAuthGuard } from "@auth/auth.guard";
import { createHttpError } from "@core/errors/handle-error.utility";
import { createResponse } from "@core/response/create-response.utility";
import { IResponse } from "@core/response/response.interface";
import { Body, Controller, Delete, Get, Patch, Post, Query, Req, UseGuards } from "@nestjs/common";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { CreateEstimateRequest } from "@estimate/requests/create-estimate.request";
import { UpdateEstimateRequest } from "@estimate/requests/update-estimate.request";
import { ListEstimatesRequest } from "@estimate/requests/list-estimates.request";
import { IEstimateResponse } from "@estimate/responses/estimate.responses";
import { EstimateCreator } from "@estimate/services/estimate-creator.service";
import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
import { EstimateUpdater } from "@estimate/services/estimate-updater.service";
import { EstimateDeleter } from "@estimate/services/estimate-deleter.service";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Controller("v1")
export class EstimateController {
  constructor(
    private readonly estimateCreator: EstimateCreator,
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateUpdater: EstimateUpdater,
    private readonly estimateDeleter: EstimateDeleter,
  ) {}

  @UseGuards(JwtAuthGuard)
  @Post("estimates")
  public async create(
    @Req() request: { user: IUserDto },
    @Body() body: CreateEstimateRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const created = await this.estimateCreator.create(request.user, this.mapCreateToDto(body, request.user));
      return createResponse([this.mapToResponse(created)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Get("estimates")
  public async findAll(
    @Req() request: { user: IUserDto },
    @Query() query: ListEstimatesRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const { items, pagination } = await this.estimateRetriever.findPaginated(request.user, query);
      return {
        data: items.map((e) => this.mapToResponse(e)),
        pagination,
      };
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Get("estimates/:estimateId")
  public async findOne(
    @Req() request: { user: IUserDto; params: { estimateId: string } },
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const estimate = await this.estimateRetriever.findByIdOrFail(request.user, request.params.estimateId);
      return createResponse([this.mapToResponse(estimate)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Patch("estimates/:estimateId")
  public async update(
    @Req() request: { user: IUserDto; params: { estimateId: string } },
    @Body() body: UpdateEstimateRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const updated = await this.estimateUpdater.update(request.user, request.params.estimateId, body);
      return createResponse([this.mapToResponse(updated)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Delete("estimates/:estimateId")
  public async softDelete(
    @Req() request: { user: IUserDto; params: { estimateId: string } },
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const deleted = await this.estimateDeleter.softDelete(request.user, request.params.estimateId);
      return createResponse([this.mapToResponse(deleted)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  private mapCreateToDto(body: CreateEstimateRequest, authUser: IUserDto): IEstimateDto { /* ... */ }
  private mapToResponse(estimate: IEstimateDto): IEstimateResponse { /* ... */ }
}
```

### Transition Test (Verified Pattern from `quote-transitions.spec.ts`)

```typescript
// Source: trade-flow-api/src/quote/enums/quote-transitions.spec.ts (adapted)
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { isValidTransition, getValidTransitions } from "@estimate/enums/estimate-transitions";

describe("estimate-transitions", () => {
  describe("isValidTransition", () => {
    it("should allow DRAFT -> SENT", () => {
      expect(isValidTransition(EstimateStatus.DRAFT, EstimateStatus.SENT)).toBe(true);
    });

    it("should allow DRAFT -> DELETED", () => {
      expect(isValidTransition(EstimateStatus.DRAFT, EstimateStatus.DELETED)).toBe(true);
    });

    it("should allow SENT -> SENT (re-send no-op)", () => {
      expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.SENT)).toBe(true);
    });

    it("should not allow SENT -> DELETED", () => {
      expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.DELETED)).toBe(false);
    });

    it("should not allow CONVERTED -> anything (terminal)", () => {
      expect(isValidTransition(EstimateStatus.CONVERTED, EstimateStatus.SENT)).toBe(false);
      expect(isValidTransition(EstimateStatus.CONVERTED, EstimateStatus.DECLINED)).toBe(false);
    });
  });

  describe("getValidTransitions", () => {
    it("returns [SENT, DELETED] for DRAFT", () => {
      const transitions = getValidTransitions(EstimateStatus.DRAFT);
      expect(transitions).toContain(EstimateStatus.SENT);
      expect(transitions).toContain(EstimateStatus.DELETED);
    });

    it("returns empty array for terminal states", () => {
      expect(getValidTransitions(EstimateStatus.CONVERTED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.DECLINED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.EXPIRED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.LOST)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.DELETED)).toEqual([]);
    });
  });
});
```

### Mock Generator (New Pattern — Based on Existing Convention)

```typescript
// trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts
import { DateTime } from "luxon";
import { ObjectId } from "mongodb";
import { DtoCollection } from "@core/collections/dto.collection";
import { DEFAULT_CURRENCY } from "@core/constants/currency.constant";
import { Money } from "@core/value-objects/money.value-object";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export class EstimateMockGenerator {
  public static createEstimateDto(overrides?: Partial<IEstimateDto>): IEstimateDto {
    const businessId = new ObjectId().toString();
    const subTotal = Money.zero(DEFAULT_CURRENCY);
    return {
      id: new ObjectId().toString(),
      businessId,
      customerId: new ObjectId().toString(),
      jobId: new ObjectId().toString(),
      number: "E-2026-001",
      title: "Test Estimate",
      estimateDate: DateTime.now(),
      status: EstimateStatus.DRAFT,
      contingencyPercent: 10,
      displayMode: EstimateDisplayMode.RANGE,
      revisionNumber: 1,
      isCurrent: true,
      parentEstimateId: null,
      rootEstimateId: null,
      lineItems: DtoCollection.empty(),
      totals: {
        subTotal,
        taxTotal: Money.zero(DEFAULT_CURRENCY),
        total: Money.zero(DEFAULT_CURRENCY),
      },
      priceRange: {
        displayMode: EstimateDisplayMode.RANGE,
        subtotal: subTotal,
        contingencyPercent: 10,
        low: { exclTax: subTotal, tax: Money.zero(DEFAULT_CURRENCY), inclTax: subTotal },
        high: { exclTax: subTotal, tax: Money.zero(DEFAULT_CURRENCY), inclTax: subTotal },
      },
      responseSummary: null,
      ...overrides,
    };
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `quote-token` module | `document-token` module with `documentType` discriminator | Phase 41 (this phase) | All quote-token imports must update; `req.quoteToken` → `req.documentToken`; `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard` |
| Bundle helpers in `@quote/services` | Bundle helpers in `@item/services` | Phase 41 (this phase) | `QuoteBundleLineItemFactory` import path changes; estimate module reuses |
| `QuoteController.findAll` returns flat array | Paginated list with status tab filter | New in Phase 41 for estimates; Phase 43+ may retrofit quotes | Frontend `estimates` API calls use `IResponse.pagination`; quote UI unchanged |

**Deprecated/outdated:** None. This is a greenfield module addition on top of a healthy codebase.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Production MongoDB has zero documents in `quote_tokens` collection (so token rename can be a code-only rename with no data migration) | D-TKN-01, Runtime State Inventory | **HIGH** — if prod data exists, existing customer quote links break silently on next deploy. Mitigation: add pre-execution verification step that queries `db.quote_tokens.countDocuments({})` against the prod connection string. |
| A2 | `Money.multiply(numberLiteral)` produces penny-exact values for all multipliers in `{1, 1.05, 1.10, 1.15, 1.20, 1.25, 1.30}` | Pattern 2, Pitfall 2 | **MEDIUM** — if `Money` rounds inconsistently, `low * 1.30 ≠ high` on certain inputs. Mitigation: parameterised unit test over every allowed multiplier with odd subtotal values like `£0.01`, `£99.99`, `£1234.56`. |
| A3 | `MongoDbFetcher.findMany` supports `{ limit, skip, sort }` options, OR the planner can bypass it via `MongoConnectionService.getDb()` | Pattern 6, Pitfall 6 | **MEDIUM** — if neither works, pagination requires extending `MongoDbFetcher` as a core-layer change. Mitigation: planner reads `src/core/services/mongo/mongo-db-fetcher.service.ts` before writing the repository method. |
| A4 | The `PublicQuoteController` can relocate inside `document-token/controllers/` without breaking the `/v1/public/quote/:token` URL path (because the path is driven by `@Controller("v1/public")` + `@Get("quote/:token")`, not the file location) | D-TKN + Project Structure | **LOW** — NestJS routing is decorator-based, not filesystem-based. Verified by reading `quote-token/controllers/public-quote.controller.ts` line 13 (`@Controller("v1/public")`). |
| A5 | The three bundle helpers (`BundleConfigValidator`, `BundlePricingPlanner`, `BundleTaxRateCalculator`) have zero estimate-specific logic and can move to `@item/services/` unchanged (except `BundleTaxRateCalculator` which needs generalisation) | D-LI-01 | **LOW** — verified by direct inspection. `BundleConfigValidator` imports only `@item/*` and `@core/*`. `BundlePricingPlanner` imports only `@item/*` and `@core/*`. `BundleTaxRateCalculator` imports `IQuoteLineItemDto` but uses only `{ unitPrice, quantity, taxRate }` — generalisation is mechanical. |
| A6 | There is no existing `@estimate/*` path alias in `tsconfig.json` (so adding it is free) | Architecture — tsconfig | **LOW** — verified by reading `tsconfig.json`; no `@estimate/*` entry present. |
| A7 | Jest test runner scans the `test/` subdirectory in each module (so new spec files under `src/estimate/test/` will be picked up automatically) | Testing | **LOW** — verified by existing `QuoteTransitionServiceSpec` location under `src/quote/test/services/`. |
| A8 | `ErrorCodes` enum can be extended with new entries (`ESTIMATE_INVALID_TRANSITION`, `ESTIMATE_NOT_EDITABLE`, `ESTIMATE_CUSTOMER_NOT_FOUND`, `ESTIMATE_JOB_NOT_FOUND`, etc.) without breaking existing callers | CLAUDE.md: Error codes | **LOW** — verified by `src/core/errors/error-codes.enum.ts`; each domain has its own prefix (`QUOTE_*`, `SCHEDULE_*`, `ITEM_*`). Planner adds `ESTIMATE_0`..`ESTIMATE_N` as needed. |

**If A1 turns out false**, the token rename becomes a two-step deploy: (1) ship code that writes to `document_tokens` and reads from both; (2) run the one-shot rename migration to move existing `quote_tokens` documents into `document_tokens` with `documentType: "quote"` added; (3) remove the dual-read path. This is the rejected "dual-read fallback" in CONTEXT.md <deferred> — but it resurfaces if production data exists. **Planner must verify A1 as the very first task in Phase 41 execution.**

## Open Questions

1. **Pre-execution check: does prod `quote_tokens` have data?**
   - What we know: CONTEXT.md asserts production has no data; Trade Flow shipped v1.3 (Send Quotes) to production on 2026-03-21 and has been live for 21 days, so real customer quote tokens almost certainly exist in production.
   - What's unclear: Whether the "no production data" claim is strictly about Phase 41's own state (no estimate data exists yet, which is trivially true) or also about the token collection (which would be surprising).
   - Recommendation: **BLOCKING.** Before executing Phase 41, run `mongosh "$MONGO_URL" --eval 'db.quote_tokens.countDocuments({})'` against production. If count > 0, escalate to the user: either accept that existing customer links break OR add a one-shot migration task (currently in CONTEXT.md <deferred>, would need un-deferring). The author of CONTEXT.md explicitly stated the "no production data" assumption, and it is the planner/executor's job to verify or reject it.

2. **Should the existing quote module be migrated to use pagination in the same PR?**
   - What we know: `QuoteController.findAll` is unpaginated. Phase 43 (estimate frontend) may surface the inconsistency when the estimate list page has pagination but the quote list page does not.
   - What's unclear: Whether retrofitting quote pagination is in scope for Phase 41 or a separate concern.
   - Recommendation: **Out of scope for Phase 41.** Flag as a follow-up quick task after Phase 43 ships. Phase 41 focuses on estimates only; quote pagination is a parallel concern.

3. **Should `EstimateUpdater.update(authUser, id, UpdateEstimateRequest)` accept a full-replace body or per-field patches?**
   - What we know: The quote module does NOT have a full-quote update — it has `addLineItem`, `updateLineItem`, `deleteLineItem` as three separate endpoints on `QuoteController`. CONTEXT.md recommends "mirror `QuoteUpdater` (add/update/delete line items through the same single update request)" which is slightly contradictory.
   - What's unclear: Whether Phase 41 should expose three sub-endpoints (`PATCH /estimates/:id`, `POST /estimates/:id/line-item`, `PATCH /estimates/:id/line-item/:lineItemId`, `DELETE /estimates/:id/line-item/:lineItemId`) to match quote, or a single `PATCH /estimates/:id` that accepts `{ lineItemsToAdd, lineItemsToUpdate, lineItemsToDelete, ...otherFields }`.
   - Recommendation: **Mirror the quote module's multi-endpoint pattern** for consistency. The single `PATCH /estimates/:id` endpoint should handle non-line-item fields (scope/title/notes/customer/job/contingency/display mode/uncertainty). Separate `POST`/`PATCH`/`DELETE` sub-resource endpoints handle line items. This lets the planner reuse `QuoteUpdater` patterns literally. **Update the endpoint list under Claude's Discretion accordingly.** This changes the 5-endpoint list to 8 endpoints: `POST /estimates`, `GET /estimates`, `GET /estimates/:id`, `PATCH /estimates/:id`, `DELETE /estimates/:id`, `POST /estimates/:id/line-item`, `PATCH /estimates/:id/line-item/:lineItemId`, `DELETE /estimates/:id/line-item/:lineItemId`.

4. **Where does `PublicQuoteController` live after the rename?**
   - What we know: CONTEXT.md recommends `src/quote/controllers/public-quote.controller.ts`.
   - What's unclear: Whether this creates a circular dependency between `QuoteModule` and `DocumentTokenModule` (quote module provides the controller that uses the document-token guard).
   - Recommendation: **Keep `PublicQuoteController` in `DocumentTokenModule` for Phase 41 to avoid a circular dependency.** The existing `quote-token/controllers/public-quote.controller.ts` already demonstrates this works (the file is colocated with the guard and depends on `QuoteModule` via a `forwardRef`). Rename the directory `quote-token/` → `document-token/` and keep the controller in place under `document-token/controllers/public-quote.controller.ts`. Future Phase 45 will add a sibling `public-estimate.controller.ts` under the same `document-token/controllers/` directory. This actually matches the existing pattern more cleanly than CONTEXT's recommendation.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Build + test | Assumed ✓ | 22.x required (per trade-flow-api/Dockerfile) | — |
| npm | Dependency install + CI | Assumed ✓ | 10+ | — |
| MongoDB | Repository tests / dev server | Required for manual QA; unit tests mock the driver | 7.0 (prod) | Unit tests don't need it; integration tests would |
| TypeScript | Compilation | ✓ (dev dep 5.9.3) | 5.9.3 / 5.1.x per CLAUDE.md — note discrepancy | — |
| jest | Test runner | ✓ (dev dep 30.2.0) | — | — |

**Missing dependencies with no fallback:** None — Phase 41 is a code-only scaffolding phase. No new tools or services required.

**Missing dependencies with fallback:** None.

**Note on TypeScript version:** The root CLAUDE.md claims "TypeScript 5.9.3" in the API stack table, while `trade-flow-api/CLAUDE.md` claims "TypeScript 5.1.x". Verify the installed version from `trade-flow-api/package.json` devDependencies before execution — the delta does not affect the plan (both versions support every feature used by this phase) but it is a documentation inconsistency the planner may want to flag as a follow-up.

## Validation Architecture

Phase 41 is a pure backend phase with no UI and no E2E flow (the customer-facing estimate page lands in Phase 45, the UI in Phase 43). Validation is therefore **entirely unit-test + repository-test driven**, which is the same bar applied to every prior backend phase in trade-flow-api.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | jest 30.2.0 (with `ts-jest` transformer) |
| Config file | `trade-flow-api/jest.config.js` (existing; no changes needed) — confirmed by `npm run test` invocation and the existence of `*.spec.ts` files under `src/*/test/` |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern=estimate` |
| Full suite command | `cd trade-flow-api && npm run ci` (runs test + lint:check + format:check + typecheck) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| EST-01 | `POST /v1/estimates` creates a Draft estimate with valid payload | unit (controller) | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-02 | `EstimateNumberGenerator.generateNumber(businessId)` returns `E-YYYY-NNN` format; two concurrent calls for same business in same year produce different sequential numbers | unit (service) | `npm run test -- --testPathPattern=estimate-number-generator` | ❌ Wave 0 |
| EST-02 | `estimate_counters` atomic `findOneAndUpdate` increments correctly | unit (repo-level) — may use in-memory MongoDB or mock the `MongoConnectionService.getDb()` | `npm run test -- --testPathPattern=estimate-number-generator` | ❌ Wave 0 |
| EST-03 | `estimatelineitems` collection constant used in `EstimateLineItemRepository`; `quotelineitems` untouched | unit (repo) | `npm run test -- --testPathPattern=estimate-line-item.repository` | ❌ Wave 0 |
| EST-03 | `EstimateStandardLineItemFactory.create()` returns `IEstimateLineItemDto` with expected fields | unit (factory) | `npm run test -- --testPathPattern=estimate-standard-line-item-factory` | ❌ Wave 0 |
| EST-03 | `EstimateBundleLineItemFactory.createLineItems()` uses `@item/services` bundle helpers | unit (factory) | `npm run test -- --testPathPattern=estimate-bundle-line-item-factory` | ❌ Wave 0 |
| EST-04 | `EstimateUpdater.update()` rejects non-Draft estimates with `ESTIMATE_NOT_EDITABLE` | unit (service) | `npm run test -- --testPathPattern=estimate-updater` | ❌ Wave 0 |
| EST-04 | Line-item add/update/delete endpoints reject non-Draft estimates | unit (service) | `npm run test -- --testPathPattern=estimate-updater` | ❌ Wave 0 |
| EST-05 | `EstimateDeleter.softDelete()` sets `status = DELETED` and `deletedAt`; does not touch line items | unit (service) | `npm run test -- --testPathPattern=estimate-deleter` | ❌ Wave 0 |
| EST-05 | `EstimateDeleter.softDelete()` rejects non-Draft estimates with `ESTIMATE_NOT_EDITABLE` | unit (service) | `npm run test -- --testPathPattern=estimate-deleter` | ❌ Wave 0 |
| EST-06 | `EstimateController.findAll` with no filter returns all non-deleted estimates paginated | unit (controller) | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-06 | `EstimateController.findAll?status=draft` returns only Draft estimates | unit (controller) | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-06 | Pagination metadata `pagination: { total, limit, offset }` present in response | unit (controller) | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-07 | `GET /v1/estimates/:id` returns full detail with `totals`, `priceRange`, `responseSummary: null`, `lineItems` | unit (controller) + (service) | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-08 | `ALLOWED_TRANSITIONS` map: every allowed transition asserted positive; every disallowed asserted negative; terminal states reject all transitions; `SENT → SENT` no-op allowed | unit (enum module) | `npm run test -- --testPathPattern=estimate-transitions` | ❌ Wave 0 |
| EST-08 | `EstimateTransitionService.transition()` rejects invalid transitions with `ESTIMATE_INVALID_TRANSITION` and sets correct timestamps on valid transitions | unit (service) | `npm run test -- --testPathPattern=estimate-transition` | ❌ Wave 0 |
| EST-08 | Phase 41: no HTTP endpoint triggers non-CRUD transitions (verified structurally by absence of `@Post("estimates/:id/transition")` route) | unit (controller) — assert no such route exists | `npm run test -- --testPathPattern=estimate.controller` | ❌ Wave 0 |
| EST-09 | `DocumentTokenRepository` uses `document_tokens` collection constant | unit (repo) | `npm run test -- --testPathPattern=document-token.repository` | ❌ Wave 0 |
| EST-09 | `DocumentSessionAuthGuard` attaches `request.documentToken` with `documentType` field; rejects missing token (404), expired (410), revoked (410) | unit (guard) | `npm run test -- --testPathPattern=document-session-auth.guard` | ❌ Wave 0 |
| EST-09 | `PublicQuoteController.findByToken` asserts `documentType === "quote"` and returns 404 on mismatch | unit (controller) | `npm run test -- --testPathPattern=public-quote.controller` | ❌ Wave 0 |
| EST-09 | Existing `public-quote.controller.spec.ts` tests still pass unchanged (regression) | unit (controller) | `npm run test -- --testPathPattern=public-quote.controller` | ✅ (existing; update imports only) |
| CONT-01 | `CreateEstimateRequest` accepts `contingencyPercent` with `@Min(0) @Max(30) @IsIn([0,5,10,15,20,25,30])` validation | unit (request DTO validation via class-validator; may do runtime assertion) | `npm run test -- --testPathPattern=create-estimate.request` (optional) OR enforce via integration test at controller level | ❌ Wave 0 |
| CONT-02 | `EstimateTotalsCalculator` attaches `priceRange.low` and `priceRange.high` with both `range` and `from` display modes | unit (service) | `npm run test -- --testPathPattern=estimate-totals-calculator` | ❌ Wave 0 |
| CONT-05 | `EstimateTotalsCalculator` math: for every `contingencyPercent ∈ [0, 5, 10, 15, 20, 25, 30]`, `high = low × (1 + c/100)` penny-exact across varied subtotals including odd values | unit (parameterised service test) | `npm run test -- --testPathPattern=estimate-totals-calculator` | ❌ Wave 0 |
| RESP-08 | `IEstimateEntity` contains reserved nullable fields `lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason`, `siteVisitAvailability` | unit (entity shape — compile-time check OR repo spec) | `npm run typecheck` + `npm run test -- --testPathPattern=estimate.repository` | ❌ Wave 0 |
| RESP-08 | `IEstimateResponseSummaryDto` type exists and is exported from `@estimate/data-transfer-objects/estimate-response-summary.dto` | compile-time (typecheck) | `npm run typecheck` | ❌ Wave 0 |

**Manual verification items (cannot automate in Phase 41, confirmed by inspection):**

- `openapi.yaml` updated with 5 new estimate endpoints and all removed `quote-token` internal types (the public quote URL path is unchanged).
- `AppModule.imports` contains `EstimateModule` and `DocumentTokenModule`; no longer contains `QuoteTokenModule`.
- `tsconfig.json` paths: `@estimate/*`, `@estimate-test/*`, `@document-token/*`, `@document-token-test/*` added; `@quote-token/*`, `@quote-token-test/*` removed.
- ROADMAP.md success criterion #5 rewritten per D-TKN-07.

### Sampling Rate

- **Per task commit:** `cd trade-flow-api && npm run test -- --testPathPattern=<task-area>` (fast, seconds).
- **Per wave merge:** `cd trade-flow-api && npm run test -- --passWithNoTests` (full suite).
- **Phase gate:** `cd trade-flow-api && npm run ci` (test + lint:check + format:check + typecheck) must exit 0 before `/gsd-verify-work`.

### Wave 0 Gaps

Every file listed below must be created in Phase 41. The test files are tightly coupled to the source files (same wave as their subjects); the planner should plan each source-file task to include its companion spec file in the same commit.

- [ ] `src/estimate/test/services/estimate-number-generator.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-totals-calculator.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-transition.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-creator.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-retriever.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-updater.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-deleter.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-standard-line-item-factory.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-bundle-line-item-factory.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-line-item-creator.service.spec.ts`
- [ ] `src/estimate/test/services/estimate-line-item-retriever.service.spec.ts`
- [ ] `src/estimate/test/controllers/estimate.controller.spec.ts`
- [ ] `src/estimate/test/repositories/estimate.repository.spec.ts`
- [ ] `src/estimate/test/repositories/estimate-line-item.repository.spec.ts`
- [ ] `src/estimate/test/policies/estimate.policy.spec.ts`
- [ ] `src/estimate/test/policies/estimate-line-item.policy.spec.ts`
- [ ] `src/estimate/test/mocks/estimate-mock-generator.ts`
- [ ] `src/estimate/test/mocks/estimate-line-item-mock-generator.ts`
- [ ] `src/estimate/enums/estimate-transitions.spec.ts`
- [ ] `src/document-token/test/repositories/document-token.repository.spec.ts`
- [ ] `src/document-token/test/guards/document-session-auth.guard.spec.ts`
- [ ] `src/document-token/test/controllers/public-quote.controller.spec.ts` (refactored from `src/quote-token/test/controllers/public-quote.controller.spec.ts`)
- [ ] `src/document-token/test/services/public-quote-retriever.service.spec.ts` (refactored)
- [ ] `src/document-token/test/services/document-token-creator.service.spec.ts` (refactored)
- [ ] `src/document-token/test/services/document-token-retriever.service.spec.ts` (refactored)
- [ ] `src/document-token/test/services/document-token-revoker.service.spec.ts` (refactored)
- [ ] `src/document-token/test/services/quote-response-handler.service.spec.ts` (refactored)
- [ ] `src/document-token/test/mocks/document-token-mock-generator.ts` (refactored)

Framework install: **none**. jest, @nestjs/testing, ts-jest are already installed.

## Security Domain

`.planning/config.json` does not specify `security_enforcement`, so it is treated as enabled.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | `JwtAuthGuard` (Firebase RS256 JWT via JWKS) for all `/v1/estimates/*` routes. Verified existing and reused. |
| V3 Session Management | yes (public endpoints only — existing quote flow) | `DocumentSessionAuthGuard` validates token existence, expiry, revocation. Rate limited via `ThrottlerGuard` (60/min for reads, 10/min for writes). No session cookies. |
| V4 Access Control | yes | `EstimatePolicy` scopes every read/write to `authUser.businessIds.includes(resource.businessId)`. `AccessControllerFactory` + `AuthorizedCreatorFactory` enforce at every layer. Policy checks "who"; services check "whether" (D-DRAFT-02). |
| V5 Input Validation | yes | class-validator decorators on every request class; global `ValidationPipe` with `whitelist: true` + `forbidNonWhitelisted: true`. Strict types on `contingencyPercent`, `displayMode`, `EstimateStatus`. |
| V6 Cryptography | no (Phase 41) | No new crypto introduced. Token generation (random bytes) already exists in `quote-token-creator.service.ts`; carried over unchanged. |
| V7 Error Handling / Logging | yes | Domain errors → `createHttpError(error)` → HTTP exceptions. `AppLogger` (Pino) logs at service boundaries. No error-message leakage: `createErrorResponse` returns sanitized strings. |
| V8 Data Protection | yes (RESP-08 reservation) | Customer response fields (`lastResponseMessage`, `siteVisitAvailability`) stored in MongoDB with standard Atlas at-rest encryption. No PII beyond what quote module already stores. |
| V9 Communications | yes | HTTPS enforced at Railway layer. CORS restricted via `CORS_ORIGIN` env var. |
| V10 Malicious Code | yes | No new external libraries introduced. CI gate enforces linting/typecheck; no eslint-disable allowed. |
| V13 API and Web Service | yes | `@Controller("v1")` prefix; REST conventions; structured `IResponse<T>` envelope; OpenAPI spec update required. |

### Known Threat Patterns for NestJS/MongoDB

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| MongoDB injection via user-supplied query operators (`$where`, `$gt`) | Tampering | class-validator strips unknown fields; all filter construction inside repositories uses hand-built `Filter<Entity>` objects, never spreads request bodies. Verified existing pattern in `QuoteRepository.findQuotesByBusinessId`. |
| IDOR (accessing another business's estimate via sequential ObjectId) | Elevation of Privilege | `EstimatePolicy.canRead/canUpdate/canCreate` always check `authUser.businessIds.includes(resource.businessId)`. `AccessControllerFactory` wraps every repo read. Uniform 403 on mismatch. |
| Counter race producing duplicate `E-YYYY-NNN` numbers | Tampering | Atomic `findOneAndUpdate` + `$inc` + `upsert: true` + partial unique index on `(businessId, number) where deletedAt null`. Pitfall 1 documented. |
| Soft-delete bypass (user undeletes by PATCH) | Elevation of Privilege | `EstimateUpdater.update` re-fetches current status; rejects if not `DRAFT`. DELETED is terminal (D-TXN-02 map); no transition back to DRAFT. |
| Mass-assignment of privileged fields via create/update request | Tampering | Global `ValidationPipe` with `forbidNonWhitelisted: true` rejects any field not declared on the request class. `sentAt`, `firstViewedAt`, `convertedAt`, `lostAt`, `deletedAt`, `number`, `status` (beyond DRAFT) are never in `CreateEstimateRequest` or `UpdateEstimateRequest`. |
| Token-type confusion (estimate token accepted at quote endpoint, and vice versa) | Spoofing | `PublicQuoteController` asserts `documentType === "quote"` on every handler entry. See Pitfall 3. |
| Timing attack on token lookup | Information Disclosure | `DocumentTokenRepository.findByToken` is a direct MongoDB lookup with no branching on token content. Same-length tokens always take the same code path. Rate limited to 60/min per IP. |
| Rate-limit bypass on public endpoints | Denial of Service | `@Throttle({ default: { limit: 60, ttl: 60000 } })` + `ThrottlerGuard` at controller level. Preserved from existing quote module. |

No new threat surface introduced by Phase 41 beyond what the quote module already has. The token rename mechanically preserves existing controls.

## Sources

### Primary (HIGH confidence)

- `trade-flow-api/src/quote/quote.module.ts` — module wiring verified
- `trade-flow-api/src/quote/controllers/quote.controller.ts` — controller pattern verified (including the pagination gap flagged in Pitfall 6)
- `trade-flow-api/src/quote/services/quote-number-generator.service.ts` — counter pattern verified
- `trade-flow-api/src/quote/services/quote-totals-calculator.service.ts` — totals calculator pattern verified
- `trade-flow-api/src/quote/services/quote-transition.service.ts` — transition service pattern verified
- `trade-flow-api/src/quote/services/quote-creator.service.ts`, `quote-retriever.service.ts`, `quote-updater.service.ts` — service layer verified
- `trade-flow-api/src/quote/services/bundle-config-validator.service.ts`, `bundle-pricing-planner.service.ts`, `bundle-tax-rate-calculator.service.ts` — verified quote-agnostic by inspection (imports only `@item/*`, `@core/*`, and `IQuoteLineItemDto` structurally)
- `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts`, `quote-standard-line-item-factory.service.ts` — factory pattern verified
- `trade-flow-api/src/quote/services/quote-line-item-creator.service.ts`, `quote-line-item-retriever.service.ts` — line-item service pattern verified
- `trade-flow-api/src/quote/enums/quote-transitions.ts`, `quote-transitions.spec.ts` — transition map and test pattern verified
- `trade-flow-api/src/quote/enums/quote-status.enum.ts`, `quote-line-item-status.enum.ts` — enum conventions verified
- `trade-flow-api/src/quote/repositories/quote.repository.ts`, `quote-line-item.repository.ts` — repository layering and collection names verified (`quotes`, `quotelineitems`)
- `trade-flow-api/src/quote/entities/quote.entity.ts`, `quote-line-item.entity.ts` — entity field conventions verified
- `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts`, `quote-line-item.dto.ts` — DTO conventions with JSDoc verified
- `trade-flow-api/src/quote/policies/quote.policy.ts`, `quote-line-item.policy.ts` — policy pattern verified
- `trade-flow-api/src/quote/requests/create-quote.request.ts` — request DTO shape verified (note: simpler than expected; line items are added via separate endpoints)
- `trade-flow-api/src/quote/responses/quote.responses.ts` — response shape verified
- `trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts` — test structure and mock pattern verified
- `trade-flow-api/src/quote-token/quote-token.module.ts` — module shape verified
- `trade-flow-api/src/quote-token/repositories/quote-token.repository.ts` — token repository and `quote_tokens` collection constant verified
- `trade-flow-api/src/quote-token/guards/quote-session-auth.guard.ts` — session guard pattern verified
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` — public controller with `@Controller("v1/public")` verified
- `trade-flow-api/src/app.module.ts` — verified imports include `QuoteTokenModule` and `QuoteModule` side-by-side
- `trade-flow-api/src/core/errors/error-codes.enum.ts` — error code naming convention verified (`QUOTE_*`, `SCHEDULE_*`, domain-prefixed)
- `trade-flow-api/src/migration/interfaces/migration.interface.ts`, `migrations/20260325120000-add-users-unique-indexes.migration.ts` — migration pattern verified (authenticated `POST /v1/migrations/run` endpoint, not startup hook)
- `trade-flow-api/tsconfig.json` — path alias structure verified
- `trade-flow-api/package.json` — CI gate command verified: `"ci": "npm run test -- --passWithNoTests && npm run lint:check && npm run format:check && npm run typecheck"`
- `trade-flow-api/CLAUDE.md` — API conventions, layering rules, error handling, auth patterns, naming standards, testing standards

### Secondary (MEDIUM confidence)

- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` — user decision record; treated as authoritative for domain constraints but cross-checked against actual codebase state (discovered the pagination gap in Pitfall 6)
- `.planning/REQUIREMENTS.md` — v1.8 requirement set; ID-to-phase mapping confirmed via traceability table
- `.planning/STATE.md` — current project state including pending position at Phase 41
- `.planning/ROADMAP.md` — phase ordering, dependency graph, success criteria
- Project-root `CLAUDE.md` — project-level conventions, CI gate policy, separation-over-DRY directive, comments standard

### Tertiary (LOW confidence)

None. Every claim in this research document is either verified by direct file inspection or flagged as an assumption in the Assumptions Log.

## Metadata

**Confidence breakdown:**

- **Standard stack:** HIGH — every listed package is already installed; versions verified against `package.json`
- **Architecture:** HIGH — every pattern verified by direct inspection of existing quote module
- **Pitfalls:** HIGH — Pitfall 6 (pagination gap) and Pitfall 7 (bundle helper refactor) were discovered by direct code inspection, not derived from context
- **Runtime state inventory:** MEDIUM — assumption A1 (production `quote_tokens` is empty) is the main risk and is explicitly flagged for pre-execution verification
- **Validation architecture:** HIGH — every test file path follows the existing `src/{module}/test/{layer}/*.spec.ts` convention verified in `quote/test/`
- **Security:** HIGH — no new threat surface; existing controls reused

**Research date:** 2026-04-11
**Valid until:** 2026-05-11 (30 days for a stable NestJS codebase with no expected dependency churn)
