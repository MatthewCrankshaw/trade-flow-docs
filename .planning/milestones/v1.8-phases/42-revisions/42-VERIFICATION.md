---
phase: 42-revisions
verified: 2026-04-12T19:30:00Z
status: human_needed
score: 3/4 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Execute the Phase 42 concurrent-revise smoke procedure at trade-flow-api/docs/smoke/phase-42-concurrent-revise.md"
    expected: "Exactly one of two parallel POST /v1/estimates/:id/revisions calls succeeds (201) and the other returns 409 Conflict; no duplicate isCurrent: true rows exist in the estimates collection after both calls complete"
    why_human: "SC #4 requires a real MongoDB with partial unique index enforcement. Per user decision Q1 (2026-04-11), mongodb-memory-server was not added; only a docker-compose live run can exercise the index-enforced 409 path. Unit tests mock the repository layer and cannot prove the index fires."
---

# Phase 42: Revisions Verification Report

**Phase Goal:** A trader can invisibly revise a Sent estimate -- the new revision becomes current, the previous becomes non-current, history is queryable -- and the data model guarantees exactly one current revision per estimate chain.
**Verified:** 2026-04-12T19:30:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `POST /v1/estimates/:id/revisions` creates a new revision setting parentEstimateId, incrementing revisionNumber, and enforcing isCurrent via partial unique index on (rootEstimateId, isCurrent: true) | VERIFIED | Controller at line 116 `@Post("estimates/:id/revisions")` with `@UseGuards(JwtAuthGuard)` calls `estimateReviser.revise()`; repository has 5 indexes including `{rootEstimateId, isCurrent: 1}` partial unique; 8 controller tests + 18 reviser tests pass |
| 2 | `GET /v1/estimates/:id/revisions` returns the full revision chain in revisionNumber order (oldest first) suitable for History section | VERIFIED | Controller at line 127 `@Get("estimates/:id/revisions")` calls `estimateRetriever.findRevisionsByIdOrFail()`; `findRevisionsByRootId` query uses `sort({revisionNumber: 1})`; retriever resolves any chain member (root or revision) to the full chain; spec tests at describe block "findRevisionsByIdOrFail (D-HIST-01)" confirmed |
| 3 | Creating a revision atomically transitions the chain: new Draft becomes current, previous keeps status with isCurrent: false; IEstimateFollowupCanceller is injected but Phase 44 owns the call; ROADMAP.md and v1.8-ROADMAP.md SC #3 rewritten per D-HOOK-05 | VERIFIED | `downgradeCurrent` uses atomic `findOneAndUpdate` with `{isCurrent: true}` filter; `ESTIMATE_FOLLOWUP_CANCELLER` injected in reviser but never called (`cancelSpy.not.toHaveBeenCalled()` assertion in spec line 130); both roadmap files contain the D-HOOK-05 wording at ROADMAP.md:164 and v1.8-ROADMAP.md:77 |
| 4 | Attempting to concurrently create two revisions results in exactly one success and one 409 Conflict (index-enforced), with no duplicate isCurrent: true rows | NEEDS HUMAN | Partial unique index `{rootEstimateId, isCurrent: 1}` declared in repository; `ConflictError` wired to HTTP 409 via `createHttpError`; smoke procedure exists at `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md`; but no live MongoDB run has been executed — per user decision Q1, this requires manual smoke verification |

**Score:** 3/4 truths verified (SC #4 blocked on human smoke test)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/core/errors/conflict.error.ts` | 409 Conflict domain error class | VERIFIED | Exists; mirrors InvalidRequestError shape; `this.name = "ConflictError"`; 6 unit tests |
| `trade-flow-api/src/core/errors/error-codes.enum.ts` | ESTIMATE_REVISION_CONFLICT + ESTIMATE_NOT_REVISABLE | VERIFIED | Both values at lines 75-76 using string-value-equals-key pattern |
| `trade-flow-api/src/core/errors/handle-error.utility.ts` | ConflictError → HttpStatus.CONFLICT branch | VERIFIED | `instanceof ConflictError` at line 45, returns `HttpStatus.CONFLICT` (409) |
| `trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` | IEstimateFollowupCanceller + ESTIMATE_FOLLOWUP_CANCELLER token | VERIFIED | Exact signature `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>` |
| `trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` | @Injectable, implements interface, debug log | VERIFIED | All conditions met; `Logger.debug()` call present |
| `trade-flow-api/src/estimate/repositories/estimate.repository.ts` | 6 new revision methods + 5-index topology + D-DET-02 list filter | VERIFIED | All 6 methods confirmed (downgradeCurrent, insertRevision, restoreCurrent, softDeleteRow, findRevisionsByRootId, findCurrentInChainByRootId); 7 createIndex calls (5 standard + 2 counted twice for partial options); `isCurrent: true` in findPaginatedByBusinessId filter at line 93 |
| `trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` | findNonDeletedByEstimateId + bulkInsertForRevision | VERIFIED | Both methods confirmed |
| `trade-flow-api/src/estimate/services/estimate-reviser.service.ts` | EstimateReviser with two-write flow + compensating rollback | VERIFIED | Exists; ConflictError thrown on step-1 miss; compensating rollback with softDeleteRow + restoreCurrent; 18 tests |
| `trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` | EstimateRevisionMockGenerator fixture builder | VERIFIED | Exists; `class EstimateRevisionMockGenerator` confirmed |
| `trade-flow-api/src/estimate/services/estimate-retriever.service.ts` | Extended with D-DET-01 + findRevisionsByIdOrFail | VERIFIED | findCurrentInChainByRootId call at line 32; findRevisionsByIdOrFail at line 49 |
| `trade-flow-api/src/estimate/services/estimate-deleter.service.ts` | Extended with D-REV-05/06 predecessor restoration | VERIFIED | restoreCurrent call at line 46; parentEstimateId check for root detection |
| `trade-flow-api/src/estimate/controllers/estimate.controller.ts` | POST + GET /v1/estimates/:id/revisions with guards | VERIFIED | Both routes confirmed at lines 116 and 127; both have `@UseGuards(JwtAuthGuard)` |
| `trade-flow-api/src/estimate/estimate.module.ts` | EstimateReviser + NoopEstimateFollowupCanceller + token provider + export | VERIFIED | EstimateReviser at line 48, NoopEstimateFollowupCanceller at line 49, `{provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller}` at line 50, export at line 59 |
| `trade-flow-api/openapi.yaml` | Both endpoints documented with 201/200/409/404/403 | VERIFIED | `/v1/estimates/{id}/revisions` POST with 201/401/403/404/409/422; GET with 200/401/403/404 confirmed |
| `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` | Manual smoke procedure for SC #4 | VERIFIED | Exists with docker-compose + parallel curl instructions |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `handle-error.utility.ts` | `conflict.error.ts` | `instanceof ConflictError` branch at line 45 | WIRED | Confirmed; returns `HttpStatus.CONFLICT` (409) |
| `errors-map.constant.ts` | `error-codes.enum.ts` | ERRORS_MAP entries for ESTIMATE_REVISION_CONFLICT + ESTIMATE_NOT_REVISABLE | WIRED | Both codes confirmed in enum; map entries confirmed with D-CONC-08 messages |
| `estimate-reviser.service.ts` | `estimate.repository.ts` | `downgradeCurrent`, `insertRevision`, `restoreCurrent` calls | WIRED | All three repository methods called within `revise()` |
| `estimate-reviser.service.ts` | `conflict.error.ts` | `throw new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, ...)` on atomic filter miss | WIRED | ConflictError thrown at line 49 on step-1 null return |
| `estimate-reviser.service.ts` | `estimate-followup-canceller.interface.ts` | `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` constructor injection | WIRED | `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` at line 38; NOT called from `revise()` per D-HOOK-03 |
| `estimate-retriever.service.ts` | `estimate.repository.ts` | `findCurrentInChainByRootId`, `findRevisionsByRootId` calls | WIRED | Both calls confirmed in service |
| `estimate-deleter.service.ts` | `estimate.repository.ts` | `restoreCurrent` for predecessor restoration | WIRED | `restoreCurrent` call confirmed at line 46 |
| `estimate.controller.ts` | `estimate-reviser.service.ts` | `estimateReviser.revise(authUser, id)` in POST handler | WIRED | Call confirmed at controller line 119 |
| `estimate.controller.ts` | `estimate-retriever.service.ts` | `estimateRetriever.findRevisionsByIdOrFail(authUser, id)` in GET handler | WIRED | Call confirmed at controller line 132 |
| `estimate.module.ts` | `noop-estimate-followup-canceller.service.ts` | `{provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller}` | WIRED | Provider registered and ESTIMATE_FOLLOWUP_CANCELLER exported at line 59 |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| `estimate-reviser.service.ts` | `downgraded` from `downgradeCurrent()` | `findOneAndUpdate` atomic MongoDB op | Yes — real DB write with `{$set: {isCurrent: false}}` | FLOWING |
| `estimate-reviser.service.ts` | `inserted` from `insertRevision()` | `insertOne` MongoDB op with fully-formed DTO | Yes — real DB insert | FLOWING |
| `estimate-retriever.service.ts` → `findRevisionsByIdOrFail` | `revisions[]` from `findRevisionsByRootId()` | MongoDB cursor with `{rootEstimateId, deletedAt: null}` sorted by revisionNumber | Yes — real DB query | FLOWING |
| `estimate.repository.ts` | list filter includes `isCurrent: true` | `findPaginatedByBusinessId` query filter | Yes — D-DET-02: filter confirmed at line 93 | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED for the concurrent-revise behavior — requires live MongoDB (no runnable entry point without docker-compose). All other behaviors are covered by unit tests confirmed passing (685 total, 0 errors per 42-06-SUMMARY.md).

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| ConflictError maps to HTTP 409 | Unit spec `conflict.error.spec.ts` + `handle-error.utility.spec.ts` | 6+2 tests passing | PASS |
| EstimateReviser does not call followupCanceller | `estimate-reviser.service.spec.ts` line 130: `expect(cancelSpy).not.toHaveBeenCalled()` | 18 tests passing | PASS |
| Compensating rollback order verified | `invocationCallOrder` assertion in reviser spec | softDeleteRow before restoreCurrent | PASS |
| D-DET-01 non-current resolution | Retriever spec "findByIdOrFail (D-DET-01)" describe block | 20 retriever tests passing | PASS |
| D-REV-05/06 predecessor restoration | Deleter spec "Phase 42 non-root revision delete" describe block | 11 deleter tests passing | PASS |
| Concurrent 409 Conflict | Requires docker-compose live run | Not executed | SKIP — human needed |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| REV-01 | 42-01, 42-03 | Estimate entity stores parentEstimateId, revisionNumber, and isCurrent | SATISFIED | All four fields (parentEstimateId, rootEstimateId, revisionNumber, isCurrent) present in `estimate.entity.ts` lines 41-44 and `estimate.dto.ts` lines 48-51. Note: REQUIREMENTS.md traceability table still shows "Pending" — documentation tracking gap only, code is complete. |
| REV-02 | 42-03, 42-04, 42-05, 42-06 | User can revise a Sent estimate creating new revision under same number, marking previous non-current | SATISFIED | `POST /v1/estimates/:id/revisions` wired end-to-end; EstimateReviser service implements the two-write flow with correct E-YYYY-NNN reuse (same rootEstimateId chain) |
| REV-03 | 42-01, 42-03, 42-06 | Only one revision per chain can have isCurrent: true, enforced by partial unique index | SATISFIED | Index `{rootEstimateId: 1, isCurrent: 1}` with `partialFilterExpression: {isCurrent: true}` confirmed in repository; also: plan 42-01 amended Phase 41 PLAN-05 to declare all 5 indexes upfront |
| REV-04 | 42-03, 42-05, 42-06 | Estimate detail shows collapsed History section with previous revisions | SATISFIED (backend) | `GET /v1/estimates/:id/revisions` returns full chain ordered by revisionNumber ascending; findRevisionsByIdOrFail resolves any chain member to full history. Note: UI implementation is Phase 43 scope — backend contract is complete. |
| REV-05 | 42-01, 42-02, 42-04, 42-06 | Revising cancels pending follow-ups and schedules fresh sequence | PARTIALLY SATISFIED by design | Phase 42 ships IEstimateFollowupCanceller interface + ESTIMATE_FOLLOWUP_CANCELLER token + NoopEstimateFollowupCanceller. Cancellation is intentionally deferred to Phase 44 (Send path) and Phase 46 (BullMQ queue). This is the D-HOOK-05 design decision. ROADMAP.md SC #3 wording explicitly states "Phase 44 owns the call via the IEstimateFollowupCanceller binding from Phase 42." |

### Anti-Patterns Found

No blockers or warnings identified:

- No `TODO`/`FIXME`/`XXX`/placeholder comments in Phase 42 files
- No `return null` / `return {}` / `return []` stub implementations (all methods have real logic or correct return types)
- No hardcoded empty arrays/objects flowing to user-visible output
- No `as` type assertions in new production code (only pre-existing ones in unmodified test helpers left untouched per SCOPE BOUNDARY rule)
- No `eslint-disable`, `@ts-ignore`, or `@ts-nocheck` comments introduced

### Human Verification Required

#### 1. Concurrent Revision 409 Conflict (SC #4)

**Test:** Execute `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` step-by-step:
1. `docker compose up -d mongo` to start a real MongoDB instance
2. Seed one estimate in Sent status via POST /v1/estimates + status transition
3. Fire two parallel `POST /v1/estimates/:id/revisions` requests using the parallel-curl recipe in the smoke document
4. Assert exactly one 201 response and one 409 response
5. Run the MongoDB validation query in Step 5 of the smoke document to confirm zero duplicate `isCurrent: true` rows

**Expected:** One request returns 201 with the new Draft revision DTO; the other returns 409 with `code: "ESTIMATE_REVISION_CONFLICT"`. MongoDB query confirms a single `isCurrent: true` document per rootEstimateId chain.

**Why human:** The partial unique index on `{rootEstimateId: 1, isCurrent: 1}` only fires on a real MongoDB instance. Unit tests mock `downgradeCurrent` to return null (simulating filter miss) but do not prove the index prevents the race at the database level. Per user decision Q1 (2026-04-11), `mongodb-memory-server` was not added as a dependency for Phase 42.

---

### Gaps Summary

No code gaps found. All Phase 42 code artifacts exist, are substantive, are wired, and data flows correctly through them.

The only open item is SC #4 (concurrent 409 Conflict), which requires a manual smoke test against a live MongoDB instance. This is expected per user decision Q1 and is documented in the smoke procedure.

**REQUIREMENTS.md tracking note:** REV-01 is still marked "Pending" in the traceability table even though the fields are fully implemented (parentEstimateId, rootEstimateId, revisionNumber, isCurrent all present in entity and DTO). This is a documentation tracking gap only — no code action required.

---

_Verified: 2026-04-12T19:30:00Z_
_Verifier: Claude (gsd-verifier)_
