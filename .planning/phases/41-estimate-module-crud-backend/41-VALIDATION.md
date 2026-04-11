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

> Populated during plan creation. Each PLAN.md task with a source-file change MUST have a matching row here. See RESEARCH.md §Validation Architecture for the requirement-to-test matrix that seeds this table.

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| _pending planner_ | _tbd_ | _tbd_ | EST-01..EST-09, CONT-01/02/05, RESP-08 | see RESEARCH §Security Domain | see RESEARCH §Test Map | unit | `npm run test -- --testPathPattern=<area>` | ❌ W0 | ⬜ pending |

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
