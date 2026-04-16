---
phase: 41
slug: estimate-module-crud-backend
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-16
---

# Phase 41 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| planner → production MongoDB | Pre-check reads production token collection; operator-only gate | Read-only count query; no credentials logged |
| developer → tsconfig.json / error-codes.enum.ts | Configuration changes consumed transitively by every module | Path aliases, error code strings |
| customer browser → /v1/public/quote/:token | Untrusted token validated by DocumentSessionAuthGuard | Token string, quote data (read) |
| DocumentSessionAuthGuard → PublicQuoteController | Guard loads token DTO; controller asserts documentType discriminator | IDocumentTokenDto |
| HTTP client → EstimateController | Untrusted input; JwtAuthGuard authenticates; ValidationPipe validates | Estimate fields, line-item fields, IDs |
| EstimateCreator ← authUser | AuthorizedCreatorFactory + EstimatePolicy enforce business ownership | IUserDto, IEstimateDto |
| EstimateUpdater / EstimateDeleter ← authUser | Re-fetch + canUpdateOrFail + Draft status check (Pitfall 5) | IUserDto, estimate id |
| EstimateLineItemCreator ← authUser | AuthorizedCreatorFactory + EstimateLineItemPolicy | IUserDto, line-item fields |
| EstimateNumberGenerator ← businessId | Atomic counter prevents race-induced duplicate E-YYYY-NNN | businessId string |
| quote module → shared bundle helpers | QuoteBundleLineItemFactory now consumes helpers from @item/services/ via DI | Bundle config, pricing, tax data |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-41-01-01 | Tampering | Production quote_tokens data loss during code rename | mitigate | BLOCKING pre-check in Plan 01 Task 1: operator verifies `db.quote_tokens.countDocuments({}) === 0` before Plan 03 proceeds; explicit override required to accept data loss | closed |
| T-41-01-02 | Information Disclosure | Production MongoDB connection string during pre-check | accept | Accepted — operator uses Railway-provisioned MONGO_URL; read-only query; no credentials logged to commit | closed |
| T-41-01-03 | Tampering | Partial path-alias update leaves dangling imports | mitigate | Task 2 runs `npm run typecheck`; Task 4 runs full `npm run ci`; verified: `@estimate/*` alias present, `@quote-token/*` alias removed in tsconfig.json | closed |
| T-41-01-04 | Elevation of Privilege | New ESTIMATE_* error codes with wrong HTTP mapping | mitigate | Error codes are string literals only; HTTP mapping handled by existing `createHttpError` utility (422 for InvalidRequest, 404 for ResourceNotFound); all 12 codes confirmed present in error-codes.enum.ts | closed |
| T-41-01-05 | Information Disclosure | New error messages could leak business IDs or internal state | mitigate | Downstream services use short human-readable messages; existing `createErrorResponse` sanitization applies; error messages reference field names not IDs | closed |
| T-41-02-01 | Tampering | Behavioral drift in BundleTaxRateCalculator after type generalisation | mitigate | Spec suite runs unchanged against new `ILineItemTaxInput` signature; structural typing guarantees caller compatibility; full CI gate at Task 3; 439 tests pass | closed |
| T-41-02-02 | Elevation of Privilege | Incorrect module wiring exposes bundle math to unauthenticated code | accept | Accepted — bundle helpers are pure functions with no auth surface; they never appear in a controller; DI wiring changes do not expose new endpoints | closed |
| T-41-02-03 | Tampering | Quote test suite silently disabled to mask regression | mitigate | Task 3 acceptance criterion verifies quote test file count unchanged; 62/62 quote tests pass; full CI gate green | closed |
| T-41-03-01 | Spoofing | Token-type confusion: estimate token accepted at /v1/public/quote/:token | mitigate | PublicQuoteController asserts `documentType !== "quote"` throws DOCUMENT_TOKEN_TYPE_MISMATCH (404) on all handlers; verified: 3 occurrences in controller | closed |
| T-41-03-02 | Tampering | Silent customer-link breakage during rename due to production data loss | mitigate | Plan 01 Task 1 BLOCKING pre-check verifies quote_tokens empty before rename proceeds | closed |
| T-41-03-03 | Information Disclosure | 410 Revoked vs 404 Not-Found leaks token existence | accept | Accepted — existing quote-token flow already uses this distinction; preserving same behavior is not a regression; ThrottlerGuard rate limiting limits enumeration | closed |
| T-41-03-04 | Elevation of Privilege | documentType set at token creation can be tampered with via update | mitigate | Repository updateOne `$set` blocks do not include `documentType`; verified: documentType appears only at create (line 28, 86) and toDto read (line 118) in document-token.repository.ts | closed |
| T-41-03-05 | Denial of Service | Rate limit not preserved during rename | mitigate | @Throttle decorators preserved on PublicQuoteController handlers; verified: 5 @Throttle occurrences in public-quote.controller.ts | closed |
| T-41-03-06 | Tampering | MongoDB injection via user-supplied token string | mitigate | Token route param extracted by NestJS as plain string; global ValidationPipe with forbidNonWhitelisted strips any object payload; string passed directly to findOne | closed |
| T-41-03-07 | Information Disclosure | firstViewedAt written on create rather than lazily | mitigate | DocumentTokenRepository.toEntity omits firstViewedAt when DTO does not provide it; verified: repository uses `$exists: false` filter before setting; spec asserts entity.firstViewedAt undefined on create | closed |
| T-41-04-01 | Tampering | Mass-assignment of privileged fields via create/update request | mitigate | CreateEstimateRequest / UpdateEstimateRequest whitelist only safe fields; global ValidationPipe with `forbidNonWhitelisted: true` confirmed in main.ts; status field not present on UpdateEstimateRequest | closed |
| T-41-04-02 | Tampering | contingencyPercent outside allowed set permits unbounded markup | mitigate | class-validator stack `@Min(0) @Max(30) @IsIn([0, 5, 10, 15, 20, 25, 30])` on CreateEstimateRequest; verified: @IsIn present in create-estimate.request.ts | closed |
| T-41-04-03 | Input Validation | uncertaintyReasons array can contain non-string values | mitigate | `@IsArray() @IsString({ each: true })` on uncertaintyReasons field; verified: IsArray and IsString decorators present in create-estimate.request.ts | closed |
| T-41-04-04 | Denial of Service | Unbounded pagination limit crashes API with large result sets | mitigate | ListEstimatesRequest `@Max(100)` on limit field; verified: Max(100) present in list-estimates.request.ts | closed |
| T-41-04-05 | Input Validation | customerId / jobId accept arbitrary strings permitting enumeration | mitigate | `@IsMongoId()` decorator on both fields; verified: 3 IsMongoId decorators present in create-estimate.request.ts | closed |
| T-41-04-06 | Tampering | Type-level pollution via `as` casts in DTOs | mitigate | Acceptance criteria grep for `as any` / `as unknown as` / `eslint-disable` on all scaffold files; 0 suppressions in estimate DTOs | closed |
| T-41-05-01 | Tampering | Counter race producing duplicate E-YYYY-NNN numbers | mitigate | EstimateNumberGenerator uses `findOneAndUpdate` + `$inc` + `upsert: true`; verified: all three present in estimate-number-generator.service.ts; partial-unique index on (businessId, number) in EstimateRepository | closed |
| T-41-05-02 | Tampering | Stale DTO bypasses status check (Pitfall 5) | mitigate | EstimateTransitionService calls `estimateRepository.findByIdOrFail(id)` before any status logic; re-fetch confirmed in transition service and updater/deleter | closed |
| T-41-05-03 | Information Disclosure | firstViewedAt overwritten on repeated VIEWED transitions | mitigate | Explicit `if (!current.firstViewedAt)` check in EstimateTransitionService; verified: 1 occurrence in estimate-transition.service.ts; spec test asserts preservation | closed |
| T-41-05-04 | Elevation of Privilege | IDOR — user accesses another business's estimate by id | mitigate | EstimatePolicy.canRead/canUpdate check `authUser.businessIds.includes(resource.businessId)`; AccessControllerFactory wraps repository reads; verified: businessIds.includes in estimate.policy.ts | closed |
| T-41-05-05 | Tampering | Soft-delete bypass via PATCH with status=DRAFT | mitigate | UpdateEstimateRequest has no status field; transition map has no DELETED→DRAFT path; index partial filter excludes deleted rows from list queries | closed |
| T-41-05-06 | Input Validation | Contingency multiplier produces fractional minor units | mitigate | Parameterised spec over 7 contingency values × 4 subtotals asserts penny-exact output; contingencyPercent / 100 arithmetic confirmed in estimate-totals-calculator.service.ts | closed |
| T-41-05-07 | Tampering | Counter collection name drift between generator and index | mitigate | EstimateRepository targets "estimates"; EstimateNumberGenerator.COLLECTION targets "estimate_counters"; verified: estimate_counters in number generator, 9 createIndex calls in repository | closed |
| T-41-05-08 | Elevation of Privilege | Policy misconfigured as status-aware allowing non-Draft writes | mitigate | Policy only checks businessIds (who); service checks Draft status (whether); policy spec has no status-based tests; verified: businessIds.includes in policy, ESTIMATE_NOT_EDITABLE in updater/deleter | closed |
| T-41-05-09 | Denial of Service | Unbounded list query crashes API | mitigate | ListEstimatesRequest @Max(100) on limit; EstimateRetriever defaults limit=20; findPaginatedByBusinessId applies .limit(limit) cursor; verified: findPaginatedByBusinessId present in estimate.repository.ts | closed |
| T-41-06-01 | Elevation of Privilege | Line items created under wrong businessId | mitigate | EstimateLineItemCreator uses AuthorizedCreatorFactory + EstimateLineItemPolicy; mirror of proven quote pattern | closed |
| T-41-06-02 | Tampering | Stale bundle helper reference causes DI failure (Pitfall 7) | mitigate | Task 1 acceptance criteria grep for `@quote/services/bundle-` returns 0; verified: no stale references in estimate-bundle-line-item-factory.service.ts | closed |
| T-41-06-03 | Information Disclosure | Retriever returns DELETED line items to caller | mitigate | EstimateLineItemRetriever filters by status !== DELETED; findAllByEstimateId method confirmed present in estimate-line-item-retriever.service.ts | closed |
| T-41-06-04 | Tampering | Bundle math drift from quote equivalent | mitigate | Bundle math centralised in @item/services/*; estimate and quote factories call same helpers; verified: @item/services/bundle-* imports in estimate-bundle-line-item-factory.service.ts | closed |
| T-41-06-05 | Input Validation | ILineItemTaxInput structural type accepts arbitrary object | mitigate | Factory arguments constructed inside service from validated request fields; not from raw user input | closed |
| T-41-07-01 | Tampering | Stale DTO bypasses Draft check (Pitfall 5) | mitigate | EstimateUpdater and EstimateDeleter call `estimateRepository.findByIdOrFail(id)` before any write; 10 findByIdOrFail calls verified in estimate-updater.service.ts | closed |
| T-41-07-02 | Tampering | User patches privileged fields (number, businessId, status, sentAt) via PATCH body | mitigate | UpdateEstimateRequest only declares whitelisted fields; global ValidationPipe forbidNonWhitelisted=true strips extras; updater spreads only validated fields | closed |
| T-41-07-03 | Elevation of Privilege | Create path attaches estimate to a business the user does not own | mitigate | AuthorizedCreatorFactory wraps repository create with EstimatePolicy.canCreate; customer/job validation restricts to accessible resources | closed |
| T-41-07-04 | Elevation of Privilege | Line-item CRUD on non-owned estimate | mitigate | addLineItem/updateLineItem/deleteLineItem each re-fetch estimate and apply access control + DRAFT check; EstimateLineItemPolicy scoped by businessId | closed |
| T-41-07-05 | Denial of Service | findPaginated with abusive limit/offset consumes memory | mitigate | ListEstimatesRequest caps limit at 100; EstimateRetriever defaults 20; repository cursor uses skip/limit | closed |
| T-41-07-06 | Tampering | Counter consumed but create rolls back — gaps in E-YYYY-NNN sequence | accept | Accepted — gaps in counter sequence are cosmetic; atomic counter never produces duplicates; identical semantics to quote module | closed |
| T-41-07-07 | Information Disclosure | Error message exposes internal IDs or business state | mitigate | createHttpError + createErrorResponse sanitize error details; error messages reference field names not IDs; verified: error messages in service code use human-readable strings | closed |
| T-41-07-08 | Tampering | Line-item deletion touches estimate-level status | mitigate | EstimateDeleter has no EstimateLineItemRepository or EstimateLineItemRetriever injection; verified: 0 line-item repo references in estimate-deleter.service.ts | closed |
| T-41-08-01 | Spoofing | Unauthenticated access to estimate endpoints | mitigate | Every controller handler has @UseGuards(JwtAuthGuard); verified: 13 @UseGuards occurrences in estimate.controller.ts (all 13 handlers) | closed |
| T-41-08-02 | Elevation of Privilege | Phase 41 controller accidentally exposes non-CRUD transition endpoints | mitigate | All 13 handlers have JwtAuthGuard; post-Phase-41 handlers (send, convert, mark-lost, revisions) added by later phases are also fully guarded; D-TXN-05 test re-scoped in controller spec to account for Phase 44/47 additions | closed |
| T-41-08-03 | Tampering | Mass-assignment via PATCH body bypasses Draft guard | mitigate | UpdateEstimateRequest whitelists fields; global ValidationPipe forbidNonWhitelisted=true confirmed in main.ts; EstimateUpdater re-fetches and checks Draft at service layer | closed |
| T-41-08-04 | Information Disclosure | OpenAPI schema exposes reserved fields before Phase 45 is ready | accept | Accepted — Phase 41 reserves fields at DTO level per RESP-08; OpenAPI documents them as nullable/read-only; no behavioral impact until Phase 45 populates them | closed |
| T-41-08-05 | Denial of Service | Public list endpoint accepts unbounded pagination | mitigate | ListEstimatesRequest @Max(100) on limit; EstimateRetriever defaults to 20; EstimateRepository cursor applies .limit(n); verified: Max(100) in list-estimates.request.ts | closed |
| T-41-08-06 | Tampering | Incorrect DI wiring lets unauthorized path reach repository directly | mitigate | Full `npm run test` boots every TestingModule at plan completion; DI failure fails a spec; module exports are explicit; 617 tests pass | closed |
| T-41-08-07 | Information Disclosure | Error messages in openapi.yaml expose internal state | mitigate | OpenAPI response schemas reference createErrorResponse format; no raw stack traces; no internal IDs in error response schemas | closed |
| T-41-08-08 | Tampering | ROADMAP rewrite drops an unrelated line by accident | mitigate | Task 4 acceptance criteria greps for "atomically generated" and "paginated list with tab filtering" confirm other success criteria untouched | closed |

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-41-01 | T-41-01-02 | Production MongoDB connection string used for read-only pre-check query via Railway-provisioned MONGO_URL; no credentials logged to commit; operator-only access path | GSD security auditor | 2026-04-16 |
| AR-41-02 | T-41-02-02 | Bundle helpers (BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator) are pure math functions with no HTTP exposure, no authentication surface, and no data persistence; DI wiring changes are internal; no new endpoints created | GSD security auditor | 2026-04-16 |
| AR-41-03 | T-41-03-03 | 410 Revoked vs 404 Not-Found HTTP status distinction for document tokens preserves existing quote-token behavior; change would be a regression; ThrottlerGuard rate limiting at the controller layer limits token enumeration via timing | GSD security auditor | 2026-04-16 |
| AR-41-04 | T-41-07-06 | Gaps in the E-YYYY-NNN estimate number sequence are cosmetic artifacts of the atomic counter pattern (counter increments even if create transaction rolls back); duplicate numbers cannot occur; identical semantics accepted in quote module | GSD security auditor | 2026-04-16 |
| AR-41-05 | T-41-08-04 | Phase 41 IEstimateDto reserves Phase 45 response-tracking fields (lastResponseType, lastResponseAt, lastResponseMessage, declineReason, siteVisitAvailability) as nullable; OpenAPI schema documents them; fields are unpopulated until Phase 45 implements the customer response flow | GSD security auditor | 2026-04-16 |

---

## Unregistered Threat Flags

| Flag | Source | Assessment |
|------|--------|------------|
| T-41-08-02 D-TXN-05 controller spec re-scoped | 41-08-SUMMARY.md Threat Flags section | Later phases (44/47) added send, convert, mark-lost, revisions routes to EstimateController post-Phase-41. All added handlers carry @UseGuards(JwtAuthGuard). The D-TXN-05 spec assertion was re-scoped to confirm Phase 44 send endpoint exists rather than assert absence. The underlying auth threat (T-41-08-01) remains closed since all 13 handlers are guarded. Informational only — maps to T-41-08-02. |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-16 | 51 | 51 | 0 | GSD security auditor (claude-sonnet-4-6) |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-16
