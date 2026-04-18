---
phase: 42
slug: revisions
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-11
---

# Phase 42 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.
> Seeded from `42-RESEARCH.md` §8 "Validation Architecture". User-approved decisions applied:
> - **No `mongodb-memory-server`** — SC #4 concurrency verified via unit test (atomic filter shape) + manual smoke procedure.
> - **Phase 41 amendments** — Phase 42 plans edit `41-05-*PLAN.md` and `41-07-*PLAN.md` in-place (Phase 41 not yet executed).

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.2.0 with ts-jest, @nestjs/testing 11.1.12, supertest 7.2.2 |
| **Config file** | `trade-flow-api/package.json` (jest config block — existing, no changes) |
| **Quick run command** | `cd trade-flow-api && npm run test -- --testPathPattern=estimate` |
| **Full suite command** | `cd trade-flow-api && npm run ci` (tests + lint + format + typecheck) |
| **Estimated runtime** | ~5s focused slice / ~45s full CI |

**No new dev dependency is introduced.** Concurrency coverage relies on unit-level filter-shape assertions and a documented manual smoke procedure; no real-MongoDB integration test is added to the automated suite.

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npm run test -- --testPathPattern=<task-area>` (~5s)
- **After every plan wave:** Run `cd trade-flow-api && npm run test -- --testPathPattern=estimate` (~15s)
- **Before `/gsd-verify-work`:** Full suite `cd trade-flow-api && npm run ci` must be green (~45s)
- **Max feedback latency:** ~15 seconds at the per-wave sampling rate

---

## Per-Task Verification Map

> This table is filled in by the planner during `/gsd-plan-phase` — each task emitted into a PLAN.md must land here with its specific grep/command verification. Wave 0 fields marked ❌ until the corresponding spec file exists.

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 42-XX-XX | {plan} | {wave} | REV-{id} | — | {expected} | unit / integration / manual | `{command}` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| REV-01 | `IEstimateEntity` has `parentEstimateId`, `rootEstimateId`, `revisionNumber`, `isCurrent`; root rows carry `rootEstimateId === self._id` after Phase 41 PLAN-07 amendment | unit (amended Phase 41 creator spec) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-creator` | ❌ W0 (Phase 41 amendment) |
| REV-02 | `POST /v1/estimates/:id/revisions` creates new row under same `E-YYYY-NNN` with `revisionNumber+1`, `parentEstimateId`, `rootEstimateId` copied, `isCurrent: true`, prev row `isCurrent: false` | unit (controller + reviser service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` | ❌ W0 (new file) |
| REV-03 | Partial unique index `{rootEstimateId:1, isCurrent:1}` partial on `{isCurrent:true}` is declared at repository bootstrap; chain has exactly one current row at steady state | unit (repository spec asserts `createIndex` call shape) + manual smoke | `cd trade-flow-api && npm run test -- --testPathPattern=estimate.repository` | ❌ W0 |
| REV-04 | `GET /v1/estimates/:id/revisions` returns full `IEstimateDto[]` ordered by `revisionNumber` ascending; excludes soft-deleted; resolves both root id and revision id | unit (controller + retriever) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever` | ❌ W0 (extension) |
| REV-05 | `IEstimateFollowupCanceller` interface + token declared; `NoopEstimateFollowupCanceller` bound as default; reviser injects but does NOT call `cancelAllFollowups` at POST time | unit (DI resolution + `jest.spyOn` non-call assertion) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` | ❌ W0 |

---

## Success Criteria → Test Map

| SC | Behavior | Test Type | Coverage Strategy |
|----|----------|-----------|-------------------|
| SC #1 | `POST /v1/estimates/:id/revisions` wires to reviser, returns 201 with new DTO | unit (controller spec + reviser service spec) | Happy path + each allowed source status (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED) + each disallowed status (DRAFT, CONVERTED, DECLINED, EXPIRED, LOST, DELETED) → 409 `ESTIMATE_REVISION_CONFLICT` |
| SC #2 | `GET /v1/estimates/:id/revisions` returns chain oldest-first, both root id and revision id resolve to the same chain | unit (controller + retriever spec) | Three fixtures: query by root id, query by revision-1 id, query by revision-2 id — all return identical chain; assert `revisionNumber: 1` ordering |
| SC #3 (rewritten per D-HOOK-05) | Creating a revision atomically transitions the chain; trader sees only latest via D-DET-01 | unit (retriever spec for findByIdOrFail non-current handling + reviser spec for two-row post-state) | `findByIdOrFail(nonCurrentId)` returns the current row in the chain; `findByIdOrFail(currentId)` returns it directly; zero-current-window fallback returns target row with its non-current flag |
| SC #4 | Concurrent revise: exactly one success, one 409, exactly one row with `isCurrent: true` | unit (filter shape) + manual smoke | **Unit:** reviser spec asserts `downgradeCurrent` is called with filter `{_id, isCurrent: true, status: {$in: [...]}}` and throws `ConflictError` when the repository returns `null` (simulating the "already-revised" race). **Manual smoke (new file `docs/smoke/phase-42-concurrent-revise.md`):** docker-compose MongoDB + seed script + parallel curl recipe + expected output assertion. |

---

## Wave 0 Requirements

> "Wave 0" is the set of files/scaffolding that must exist before any task can be verified. Each item below blocks the first task that references it.

**New source files (Phase 42):**
- [ ] `trade-flow-api/src/core/errors/conflict.error.ts` — `ConflictError` class mirroring `InvalidRequestError` shape
- [ ] `trade-flow-api/src/core/errors/test/conflict.error.spec.ts` — unit spec for `getCode`/`getMessage`/`getDetails`
- [ ] `trade-flow-api/src/core/errors/error-codes.enum.ts` — add `ESTIMATE_REVISION_CONFLICT` and `ESTIMATE_NOT_REVISABLE` entries
- [ ] `trade-flow-api/src/core/errors/errors-map.constant.ts` — add entries for the two new error codes
- [ ] `trade-flow-api/src/core/errors/handle-error.utility.ts` — add `ConflictError → HttpStatus.CONFLICT` branch
- [ ] `trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` — interface + `ESTIMATE_FOLLOWUP_CANCELLER` token
- [ ] `trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` — default no-op implementation
- [ ] `trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts` — minimal injectable + resolves check
- [ ] `trade-flow-api/src/estimate/services/estimate-reviser.service.ts` — new service; injects repository, line-item repository, retriever, policy, `ESTIMATE_FOLLOWUP_CANCELLER`, `AppLogger`
- [ ] `trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` — full coverage per §7.3 of RESEARCH.md (happy path × allowed statuses, disallowed statuses, compensating rollback, line-item clone including bundle parent/child remap, non-call assertion for `IEstimateFollowupCanceller`, revision number monotonicity, filter shape for concurrency)
- [ ] `trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` — fixture builder for revision chains (extends Phase 41 `EstimateMockGenerator`)

**Extensions to Phase 41 files (Phase 42 amends in same wave):**
- [ ] `trade-flow-api/src/estimate/repositories/estimate.repository.ts` — add `downgradeCurrent`, `insertRevision`, `restoreCurrent`, `findRevisionsByRootId`, `findCurrentInChainByRootId`; extend `createIndexes()` to declare the new `{rootEstimateId, isCurrent}` partial unique index and the `{rootEstimateId, revisionNumber}` sort index (both go into the amended PLAN-05 output, not added at runtime)
- [ ] `trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` — add `bulkInsertForRevision`, `findNonDeletedByEstimateId` (or mirror existing conventions)
- [ ] `trade-flow-api/src/estimate/services/estimate-retriever.service.ts` — extend `findByIdOrFail` per D-DET-01 (non-current → chain lookup → fallback); add `findRevisionsByIdOrFail`
- [ ] `trade-flow-api/src/estimate/services/estimate-deleter.service.ts` — extend delete path to restore predecessor `isCurrent` when target has `parentEstimateId !== null`
- [ ] `trade-flow-api/src/estimate/controllers/estimate.controller.ts` — add `POST /v1/estimates/:id/revisions` and `GET /v1/estimates/:id/revisions` handlers
- [ ] `trade-flow-api/src/estimate/estimate.module.ts` — register `EstimateReviser`, `NoopEstimateFollowupCanceller`, `ESTIMATE_FOLLOWUP_CANCELLER` provider
- [ ] `trade-flow-api/openapi.yaml` — document both new endpoints including 409 and 422 responses
- [ ] `trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` — amend for D-DET-01 branch + `findRevisionsByIdOrFail`
- [ ] `trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` — amend for D-REV-05 predecessor restoration
- [ ] `trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts` — amend for new endpoints

**Phase 41 plan amendments (cross-phase edits committed under Phase 42):**
- [ ] `.planning/phases/41-estimate-module-crud-backend/41-05-*-PLAN.md` — amend `createIndexes` to declare `{businessId:1, number:1}` with partial filter `{deletedAt: null, isCurrent: true}` (not just `{deletedAt: null}`); declare `{rootEstimateId:1, isCurrent:1}` partial unique on `{isCurrent: true}`; declare `{rootEstimateId:1, revisionNumber:1}` sort index
- [ ] `.planning/phases/41-estimate-module-crud-backend/41-07-*-PLAN.md` — amend `EstimateCreator.create()` to write `rootEstimateId: inserted._id` via a follow-up `updateOne` after `insertOne`; amend the spec assertions that currently expect `rootEstimateId: null` on root rows
- [ ] `.planning/ROADMAP.md` — rewrite Phase 42 SC #3 per D-HOOK-05 amendment
- [ ] `.planning/milestones/v1.8-ROADMAP.md` — mirror the SC #3 rewrite

**Manual smoke procedure:**
- [ ] `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` — step-by-step procedure using docker-compose MongoDB + parallel curl commands to exercise SC #4 end-to-end against a real database. Documents expected 201 × 1, 409 × 1, and post-state assertion (`db.estimates.countDocuments({rootEstimateId: X, isCurrent: true}) === 1`).

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Concurrent revise → one success, one 409, one current row | SC #4 | Existing test infra is unit-only (mocked MongoDB primitives); user declined adding `mongodb-memory-server`. The partial unique index cannot be exercised by mocks — only real MongoDB enforces it. | See `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md`. Run against the docker-compose Mongo instance before any `/gsd-verify-work` sign-off. |
| Index drop-and-recreate at app bootstrap (if Phase 42 takes the runtime-migration path instead of Phase 41 amendments) | D-CHAIN-06 | Bootstrap state + multi-instance race is not reachable from unit tests. N/A since user selected Phase 41 amendment path — Phase 42 never performs runtime drop-recreate. | N/A |

---

## Validation Sign-Off

- [ ] All tasks in PLAN.md have `<automated>` verify or a Wave 0 dependency entry
- [ ] Sampling continuity: no 3 consecutive tasks without an automated verify step
- [ ] Wave 0 covers every item marked ❌ W0 in the per-task table
- [ ] Manual smoke procedure (`docs/smoke/phase-42-concurrent-revise.md`) is authored before `/gsd-verify-work`
- [ ] No `--watch` flags in any automated command
- [ ] Feedback latency < 15s at per-wave sampling rate
- [ ] `nyquist_compliant: true` set in frontmatter only after all boxes above are ticked

**Approval:** pending
