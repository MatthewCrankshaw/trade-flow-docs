---
phase: 41
slug: estimate-module-crud-backend
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-11
---

# Phase 41 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.2.0 (`ts-jest` transformer, `@nestjs/testing`, `supertest`) |
| **Config file** | `trade-flow-api/jest.config.js` (existing; no changes needed) |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern=estimate` |
| **Full suite command** | `cd trade-flow-api && npm run ci` |
| **Estimated runtime** | ~45 seconds full suite, <5s per `--testPathPattern` slice |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern=<task-area>`
- **After every plan wave:** Run `cd trade-flow-api && npm run test -- --passWithNoTests`
- **Before `/gsd-verify-work`:** `cd trade-flow-api && npm run ci` must exit 0 (test + lint:check + format:check + typecheck)
- **Max feedback latency:** ~5 seconds per slice, ~45 seconds full suite

---

## Per-Task Verification Map

> One row per source-modifying `auto` or `tdd` task across all 8 plans. Checkpoint gates and pure CI-only verification tasks are excluded (they have no source-file delta to validate). `File Exists` column references Wave 0 spec files listed below.

| Task ID | Plan | Wave | Requirement(s) | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|----------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 41-01-02 | 01 | 1 | EST-09 | T-41-01-03 | Path aliases resolve only intended source trees | typecheck | `cd trade-flow-api && npm run typecheck` | n/a (config) | ⬜ pending |
| 41-01-03 | 01 | 1 | EST-02, EST-04, EST-05, EST-08, EST-09 | T-41-01-04, T-41-01-05 | Typed domain error codes prevent unsafe HTTP status defaults and leak-prone messages | typecheck + lint | `cd trade-flow-api && npm run typecheck && npm run lint:check` | n/a (enum) | ⬜ pending |
| 41-01-04 | 01 | 1 | EST-02, EST-04, EST-05, EST-08, EST-09 | T-41-01-03, T-41-01-04 | Foundation commit passes full CI gate before downstream plans build on it | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-02-01 | 02 | 2 | EST-03 | T-41-02-01 | Bundle tax calculator generalised to `ILineItemTaxInput` without cross-module coupling; existing behavior unchanged | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=item/test/services/bundle` | ❌ W0 (relocated from quote module) | ⬜ pending |
| 41-02-02 | 02 | 2 | EST-03 | T-41-02-02, T-41-02-03 | Full quote test suite stays green after DI rewire (Pitfall 7 guard) | unit (module wiring + regression) | `cd trade-flow-api && npm run test -- --testPathPattern=quote` | ✅ (existing quote suite) | ⬜ pending |
| 41-02-03 | 02 | 2 | EST-03 | T-41-02-03 | Refactor is non-regressive across full repo (test + lint + format + typecheck) | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-03-01 | 03 | 2 | EST-09 | T-41-03 (see plan threat_model) | `document_tokens` collection constant in place; `DocumentSessionAuthGuard` attaches `request.documentToken` with `documentType` field and rejects missing/expired/revoked tokens | unit (repo + guard) | `cd trade-flow-api && npm run test -- --testPathPattern=document-token/test/(repositories\|guards)` | ❌ W0 — `src/document-token/test/repositories/document-token.repository.spec.ts`, `src/document-token/test/guards/document-session-auth.guard.spec.ts` | ⬜ pending |
| 41-03-02 | 03 | 2 | EST-09 | T-41-03 | `PublicQuoteController.findByToken` asserts `documentType === "quote"` and returns 404 on mismatch | unit (controller + service) | `cd trade-flow-api && npm run test -- --testPathPattern=document-token` | ❌ W0 — `src/document-token/test/controllers/public-quote.controller.spec.ts`, `src/document-token/test/services/public-quote-retriever.service.spec.ts`, `document-token-creator.service.spec.ts`, `document-token-retriever.service.spec.ts`, `document-token-revoker.service.spec.ts`, `quote-response-handler.service.spec.ts` | ⬜ pending |
| 41-03-03 | 03 | 2 | EST-09 | T-41-03 | Legacy `src/quote-token/` tree fully removed; all consumers updated | unit (regression) | `cd trade-flow-api && npm run typecheck && npm run test -- --testPathPattern=quote && npm run test -- --testPathPattern=document-token` | ✅ (existing quote/doc-token suites) | ⬜ pending |
| 41-03-04 | 03 | 2 | EST-09 | T-41-03 | Token rename is non-regressive across full repo | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-04-01 | 04 | 3 | EST-08 | T-41-04 (see plan threat_model) | `ALLOWED_TRANSITIONS` map matches locked CONTEXT.md §specifics wording; terminal states reject all transitions; `SENT → SENT` no-op allowed | unit (enum module) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-transitions` | ❌ W0 — `src/estimate/enums/estimate-transitions.spec.ts` | ⬜ pending |
| 41-04-02 | 04 | 3 | EST-01, EST-04, EST-06, EST-07, CONT-02, CONT-05, RESP-08 | T-41-04 | `IEstimateEntity` shape includes all reserved nullable response fields; `IEstimateDto` carries `totals`, `priceRange`, `responseSummary` | compile-time (typecheck) | `cd trade-flow-api && npm run typecheck` | n/a (type-only) | ⬜ pending |
| 41-04-03 | 04 | 3 | EST-01, EST-06, CONT-01, CONT-02, CONT-05 | T-41-04 | `CreateEstimateRequest.contingencyPercent` decorated `@Min(0) @Max(30) @IsIn([0,5,10,15,20,25,30])`; `ListEstimatesRequest` pagination validated | compile-time (typecheck) | `cd trade-flow-api && npm run typecheck` | n/a (type-only; mocks only) | ⬜ pending |
| 41-04-04 | 04 | 3 | EST-01, EST-03, EST-04, EST-06, EST-07, EST-08, CONT-01, CONT-02, CONT-05, RESP-08 | T-41-04 | Scaffold commit passes full CI gate before stateful services build on it | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-05-01 | 05 | 4 | EST-03, EST-07, RESP-08 | T-41-05 (see plan threat_model) | `estimatelineitems` collection constant in `EstimateLineItemRepository`; `quotelineitems` untouched; repositories enforce policy boundary | unit (repo) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/repositories` | ❌ W0 — `src/estimate/test/repositories/estimate.repository.spec.ts`, `src/estimate/test/repositories/estimate-line-item.repository.spec.ts` | ⬜ pending |
| 41-05-02 | 05 | 4 | EST-02, CONT-02, CONT-05 | T-41-05 | `E-YYYY-NNN` number atomic via `estimate_counters` `findOneAndUpdate`; totals calculator math penny-exact for every locked contingency bucket; `priceRange` attached in both display modes | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/services/(estimate-number-generator\|estimate-totals-calculator)` | ❌ W0 — `src/estimate/test/services/estimate-number-generator.service.spec.ts`, `src/estimate/test/services/estimate-totals-calculator.service.spec.ts` | ⬜ pending |
| 41-05-03 | 05 | 4 | EST-04, EST-05, EST-08 | T-41-05 | Transition service rejects invalid transitions with `ESTIMATE_INVALID_TRANSITION`; policies enforce business ownership (V4 access control) | unit (service + policy) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/(services/estimate-transition\|policies)` | ❌ W0 — `src/estimate/test/services/estimate-transition.service.spec.ts`, `src/estimate/test/policies/estimate.policy.spec.ts`, `src/estimate/test/policies/estimate-line-item.policy.spec.ts` | ⬜ pending |
| 41-05-04 | 05 | 4 | EST-02, EST-03, EST-04, EST-05, EST-07, EST-08, CONT-02, CONT-05, RESP-08 | T-41-05 | Wave 4 is non-regressive across full repo | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-06-01 | 06 | 5 | EST-03 | T-41-06 (see plan threat_model) | `EstimateStandardLineItemFactory.create()` returns `IEstimateLineItemDto`; `EstimateBundleLineItemFactory.createLineItems()` consumes `@item/services` bundle helpers | unit (factory) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-(standard\|bundle)-line-item-factory` | ❌ W0 — `src/estimate/test/services/estimate-standard-line-item-factory.service.spec.ts`, `src/estimate/test/services/estimate-bundle-line-item-factory.service.spec.ts` | ⬜ pending |
| 41-06-02 | 06 | 5 | EST-03 | T-41-06 | `EstimateLineItemCreator`/`EstimateLineItemRetriever` enforce policy boundary and respect Draft-only edit rule | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item-(creator\|retriever)` | ❌ W0 — `src/estimate/test/services/estimate-line-item-creator.service.spec.ts`, `src/estimate/test/services/estimate-line-item-retriever.service.spec.ts` | ⬜ pending |
| 41-06-03 | 06 | 5 | EST-03 | T-41-06 | Wave 5 is non-regressive across full repo | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-07-01 | 07 | 6 | EST-01, CONT-02, CONT-05, RESP-08 | T-41-07 (see plan threat_model) | `EstimateCreator` generates `E-YYYY-NNN`, attaches totals/priceRange, initialises reserved response fields to null | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-creator` | ❌ W0 — `src/estimate/test/services/estimate-creator.service.spec.ts` | ⬜ pending |
| 41-07-02 | 07 | 6 | EST-06, EST-07 | T-41-07 | `EstimateRetriever.findByIdOrFail` returns full detail; `findPaginated` filters by status and emits `IResponse<T>.pagination` envelope | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever` | ❌ W0 — `src/estimate/test/services/estimate-retriever.service.spec.ts` | ⬜ pending |
| 41-07-03 | 07 | 6 | EST-04 | T-41-07 | `EstimateUpdater.update()` and line-item add/update/delete methods reject non-Draft estimates with `ESTIMATE_NOT_EDITABLE` | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-updater` | ❌ W0 — `src/estimate/test/services/estimate-updater.service.spec.ts` | ⬜ pending |
| 41-07-04 | 07 | 6 | EST-05 | T-41-07 | `EstimateDeleter.softDelete()` sets `status = DELETED` via transition service; rejects non-Draft with `ESTIMATE_NOT_EDITABLE`; does not touch line items | unit (service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter` | ❌ W0 — `src/estimate/test/services/estimate-deleter.service.spec.ts` | ⬜ pending |
| 41-07-05 | 07 | 6 | EST-01, EST-04, EST-05, EST-06, EST-07, CONT-02, CONT-05, RESP-08 | T-41-07 | Wave 6 is non-regressive across full repo | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |
| 41-08-01 | 08 | 7 | EST-01, EST-04, EST-05, EST-06, EST-07, EST-08, CONT-01, CONT-02, CONT-05 | T-41-08 (see plan threat_model) | Every controller handler wrapped with `JwtAuthGuard` and `createHttpError(error)`; no non-CRUD transition endpoint; `findAll` emits pagination envelope | unit (controller) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller` | ❌ W0 — `src/estimate/test/controllers/estimate.controller.spec.ts` | ⬜ pending |
| 41-08-02 | 08 | 7 | EST-01, EST-04, EST-05, EST-06, EST-07, EST-08, EST-09 | T-41-08 | `EstimateModule` wires every provider from Plans 05-07; `AppModule` imports `EstimateModule` and `DocumentTokenModule`; `QuoteTokenModule` removed | typecheck + full suite | `cd trade-flow-api && npm run typecheck && npm run test` | n/a (module wiring) | ⬜ pending |
| 41-08-03 | 08 | 7 | EST-01, EST-04, EST-05, EST-06, EST-07 | T-41-08 | `openapi.yaml` declares all 8 estimate endpoints with full request/response schemas; no stale `quote-token` internal types; `/v1/public/quote/{token}` preserved | grep assertion | `grep -c "operationId: createEstimate" trade-flow-api/openapi.yaml && grep -c "operationId: deleteEstimateLineItem" trade-flow-api/openapi.yaml && grep -c "/v1/public/quote/{token}" trade-flow-api/openapi.yaml` | n/a (spec file) | ⬜ pending |
| 41-08-04 | 08 | 7 | EST-09 | T-41-08 | ROADMAP.md / v1.8-ROADMAP.md / STATE.md all reflect D-TKN-07 locked success criterion #5; `estimate_line_items` prose drift replaced with `estimatelineitems` (D-ENT-02) | grep assertion | `grep -c "No data migration is required" .planning/ROADMAP.md` | n/a (planning docs) | ⬜ pending |
| 41-08-05 | 08 | 7 | EST-01, EST-04, EST-05, EST-06, EST-07, EST-08, EST-09, CONT-01, CONT-02, CONT-05 | T-41-08 | Phase 41 FINAL gate: every test, lint, format, typecheck passes across the full repo | CI gate | `cd trade-flow-api && npm run ci` | n/a (gate) | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Per RESEARCH.md §Validation Architecture → Wave 0 Gaps. Framework install: **none** (jest, @nestjs/testing, ts-jest already installed).

**Estimate module specs (new):**
- [ ] `src/estimate/test/services/estimate-number-generator.service.spec.ts` — EST-02
- [ ] `src/estimate/test/services/estimate-totals-calculator.service.spec.ts` — CONT-02, CONT-05
- [ ] `src/estimate/test/services/estimate-transition.service.spec.ts` — EST-08
- [ ] `src/estimate/test/services/estimate-creator.service.spec.ts` — EST-01
- [ ] `src/estimate/test/services/estimate-retriever.service.spec.ts` — EST-06, EST-07
- [ ] `src/estimate/test/services/estimate-updater.service.spec.ts` — EST-04
- [ ] `src/estimate/test/services/estimate-deleter.service.spec.ts` — EST-05
- [ ] `src/estimate/test/services/estimate-standard-line-item-factory.service.spec.ts` — EST-03
- [ ] `src/estimate/test/services/estimate-bundle-line-item-factory.service.spec.ts` — EST-03
- [ ] `src/estimate/test/services/estimate-line-item-creator.service.spec.ts` — EST-03
- [ ] `src/estimate/test/services/estimate-line-item-retriever.service.spec.ts` — EST-03
- [ ] `src/estimate/test/controllers/estimate.controller.spec.ts` — EST-01, EST-06, EST-07, EST-08
- [ ] `src/estimate/test/repositories/estimate.repository.spec.ts` — RESP-08
- [ ] `src/estimate/test/repositories/estimate-line-item.repository.spec.ts` — EST-03
- [ ] `src/estimate/test/policies/estimate.policy.spec.ts` — V4 access control
- [ ] `src/estimate/test/policies/estimate-line-item.policy.spec.ts` — V4 access control
- [ ] `src/estimate/test/mocks/estimate-mock-generator.ts` — shared fixture
- [ ] `src/estimate/test/mocks/estimate-line-item-mock-generator.ts` — shared fixture
- [ ] `src/estimate/enums/estimate-transitions.spec.ts` — EST-08 transition map

**Document-token module specs (refactored from `quote-token/test/*`):**
- [ ] `src/document-token/test/repositories/document-token.repository.spec.ts` — EST-09
- [ ] `src/document-token/test/guards/document-session-auth.guard.spec.ts` — EST-09
- [ ] `src/document-token/test/controllers/public-quote.controller.spec.ts` — EST-09 regression (unchanged behavior)
- [ ] `src/document-token/test/services/public-quote-retriever.service.spec.ts`
- [ ] `src/document-token/test/services/document-token-creator.service.spec.ts`
- [ ] `src/document-token/test/services/document-token-retriever.service.spec.ts`
- [ ] `src/document-token/test/services/document-token-revoker.service.spec.ts`
- [ ] `src/document-token/test/services/quote-response-handler.service.spec.ts`
- [ ] `src/document-token/test/mocks/document-token-mock-generator.ts`

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `openapi.yaml` updated with new `/v1/estimates*` endpoints and renamed document-token internal types | EST-01..07 | OpenAPI docs not covered by unit tests | Inspect `trade-flow-api/openapi.yaml` diff; confirm 8 estimate endpoints (CRUD + line-item sub-resources) and no references to `quote-token` internal types |
| `AppModule.imports` contains `EstimateModule` and `DocumentTokenModule`; removes `QuoteTokenModule` | EST-09 | Module wiring is structural | `grep -n "Module" trade-flow-api/src/app.module.ts` |
| `tsconfig.json` paths include `@estimate/*`, `@estimate-test/*`, `@document-token/*`, `@document-token-test/*`; removes `@quote-token/*`, `@quote-token-test/*` | EST-09 | Path aliases are configuration | `grep paths trade-flow-api/tsconfig.json` |
| ROADMAP.md success criterion #5 rewritten per D-TKN-07 | — | Documentation update | Inspect `.planning/ROADMAP.md` phase 41 section |
| Production `quote_tokens` collection is empty (assumption A1 verification) OR rename migration is absorbed | EST-09 | Runtime data state check | Blocking pre-execution task — see RESEARCH.md Pitfall flagged as A1 |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references (28 spec files listed above)
- [ ] No watch-mode flags
- [ ] Feedback latency < 45s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
