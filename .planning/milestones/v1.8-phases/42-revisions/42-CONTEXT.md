# Phase 42: Revisions - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 42 lets a trader invisibly revise an estimate that has already reached the customer, under the same `E-YYYY-NNN` number. A revision is a brand-new row in `estimates`, born as `DRAFT` with `isCurrent: true`, while the previous revision retains its existing status and has `isCurrent` flipped to `false`. The partial unique index on `{rootEstimateId, isCurrent}` is the correctness guarantee. History is queryable via `GET /v1/estimates/:id/revisions`.

Ships in this phase:
- `POST /v1/estimates/:id/revisions` clone endpoint (Draft birth; no body required beyond auth).
- `GET /v1/estimates/:id/revisions` history query (ordered oldest-first, full `IEstimateDto[]` \u2014 see D-HIST-01).
- `EstimateReviser` service + `EstimateRevisionSeeder` (or whatever the planner names the line-item cloner) with atomic downgrade-then-insert semantics.
- A new `ConflictError` base class + `ErrorCodes.ESTIMATE_REVISION_CONFLICT` mapped to HTTP 409 via `createHttpError`.
- A new partial unique index `{rootEstimateId: 1, isCurrent: 1}` partial on `{isCurrent: true}`.
- A rework of Phase 41's `{businessId, number}` partial unique index: filter becomes `{deletedAt: null, isCurrent: true}`.
- A retrofit of Phase 41's `EstimateCreator` so root rows are written with `rootEstimateId = self._id` on create instead of `null`. Cleanest path: fold this into Phase 41 while Phase 41 is still in planning. Fallback if Phase 41 has already executed: Phase 42 ships a one-shot backfill.
- `IEstimateFollowupCanceller` interface + `NoopEstimateFollowupCanceller` default binding under DI token `"ESTIMATE_FOLLOWUP_CANCELLER"`. Phase 46 replaces the binding. Phase 42 wires the interface but does NOT call it from `EstimateReviser` at POST time \u2014 see D-HOOK-03 and the amendment to ROADMAP SC #3 captured below.
- `EstimateDeleter` is extended so that deleting a non-root Draft revision atomically flips `isCurrent: true` on the immediately previous revision (identified by `parentEstimateId`). This is how a trader rolls back a revision they abandoned.

Does NOT ship in this phase:
- Sending the new revision via email (Phase 44).
- Actually cancelling follow-ups in BullMQ (Phase 46).
- Customer-facing resolution of the latest revision via the public token (Phase 45 \u2014 Pitfall 3).
- Any UI (Phase 43 for CRUD; a collapsed History panel is part of Phase 43's detail page).
- New enum values on `EstimateStatus`. The status lifecycle from Phase 41 is unchanged \u2014 revisions flip `isCurrent`, they do not add a `SUPERSEDED` status.

</domain>

<decisions>
## Implementation Decisions

### Revise Endpoint Shape (D-REV)

- **D-REV-01:** `POST /v1/estimates/:id/revisions` is a **clone-only** endpoint. It takes no body beyond auth. It deep-copies the source revision into a new `estimates` row with `status: DRAFT`, `isCurrent: true`, `revisionNumber: prev + 1`, `parentEstimateId: prev._id`, `rootEstimateId: prev.rootEstimateId`. The trader then edits the new revision via the existing `PATCH /v1/estimates/:id` (Phase 41) and sends it via Phase 44's send endpoint. No combined revise+resend path ships in Phase 42.
- **D-REV-02:** Allowed source statuses are `SENT`, `VIEWED`, `RESPONDED`, `SITE_VISIT_REQUESTED`. Any other source status returns `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_REVISABLE)` (422). `DRAFT` is blocked explicitly: Drafts are edited in place via PATCH, never revised. Terminal states (`CONVERTED`, `DECLINED`, `EXPIRED`, `LOST`, `DELETED`) are blocked. The status check lives in the atomic `findOneAndUpdate` filter (see D-CONC-03), not in a read-then-check block.
- **D-REV-03:** The source revision must also have `isCurrent: true`. Attempting to revise a non-current row (e.g. trader paginated backwards through history and POSTed to an old revision id) is blocked by the same atomic filter. Returns 409 `ESTIMATE_REVISION_CONFLICT` because the race interpretation is "someone else already revised this chain."
- **D-REV-04:** During the Draft editing window, the previous revision keeps its existing status value (`SENT`, `VIEWED`, etc.) and has `isCurrent: false`. Two rows live in the collection at once: one Sent-ish-with-isCurrent-false, one Draft-with-isCurrent-true. The status lifecycle from Phase 41 is untouched \u2014 no new `SUPERSEDED` value is added to the enum. The `isCurrent` bit carries the "this is the visible one" meaning.
- **D-REV-05:** If the trader abandons the new Draft revision, they call `DELETE /v1/estimates/:id` on it (same endpoint as Phase 41's Draft soft-delete). `EstimateDeleter` is extended: when the target row has `parentEstimateId !== null`, the deleter **also** atomically flips `isCurrent: true` on the row at `parentEstimateId` as part of the same service call. This is the rollback path. No `/revisions/rollback` or similar dedicated endpoint ships.
- **D-REV-06:** The DELETE-on-revision flow uses the same two-step guarded-write shape as the reviser: first re-set the predecessor's `isCurrent: true` (filter: `{_id: prev._id, isCurrent: false}`; if 0 matched, another writer has already acted \u2014 return 409), then soft-delete the new row (`status: DELETED`, `deletedAt: now`, `isCurrent: false`). If the soft-delete fails, compensating update restores `isCurrent: false` on the predecessor.

### Chain Identity & Index Rework (D-CHAIN)

- **D-CHAIN-01:** `rootEstimateId` is the **chain identity**. Every row in every chain \u2014 root and revisions \u2014 carries a non-null `rootEstimateId`. Root rows have `rootEstimateId === self._id`. Revisions have `rootEstimateId === root._id`.
- **D-CHAIN-02:** `parentEstimateId` is the **immediately previous revision's id**. Root rows have `parentEstimateId: null`. Revision N has `parentEstimateId = revision (N-1)._id`. This gives us both (a) linked-list predecessor lookup for the DELETE-rollback path and (b) denormalised chain identity for partial unique enforcement.
- **D-CHAIN-03:** Phase 42 retrofits Phase 41's `EstimateCreator` so root rows are written with `rootEstimateId: self._id` instead of the `null` value reserved in Phase 41 D-ENT-01. Cleanest path: since Phase 41 is still in "context gathered, no plans" state per `.planning/STATE.md`, fold this amendment into Phase 41's plan directly. The planner for Phase 42 checks Phase 41's execution state; if Phase 41 has already executed, Phase 42 adds a one-shot backfill migration (`IMigration` class in `src/migration/migrations/` run via `POST /v1/migrations/run`) that sets `rootEstimateId = _id` on every existing root row where `rootEstimateId IS NULL`.
- **D-CHAIN-04:** The creator-level write pattern for the retrofit: `insertOne` returns the inserted `_id`; immediately `updateOne({_id: inserted._id}, {$set: {rootEstimateId: inserted._id}})`. No transactions. Brief window where `rootEstimateId` is null is acceptable because (a) no other read path cares about the root row until the next revision is created and (b) any GET by id still returns the row. The service method blocks until both writes complete before returning the DTO.
- **D-CHAIN-05:** Partial unique index `{rootEstimateId: 1, isCurrent: 1}` with partial filter expression `{isCurrent: true}`. Because every row now has a non-null `rootEstimateId`, the null-collision issue is avoided. This index is the correctness guarantee for "exactly one current revision per chain" (SC #4).
- **D-CHAIN-06:** Phase 41's inherited `{businessId: 1, number: 1}` partial unique index is **dropped and recreated** with partial filter `{deletedAt: null, isCurrent: true}`. This permits historical revisions (non-current, same number) to coexist with the current revision in the same chain while still preventing two current non-deleted rows from sharing an `E-YYYY-NNN` number. The index drop-and-recreate ships as part of Phase 42's `EstimateRepository` bootstrap block. Rollback strategy is "recreate the old filter" via a manual Mongo command if Phase 42 is ever rolled back \u2014 not automated.
- **D-CHAIN-07:** The drop-and-recreate in D-CHAIN-06 must happen BEFORE any Phase 42 code path that could write a second non-deleted row with the same number \u2014 i.e. before the reviser runs for the first time in any environment. The planner verifies index state at app bootstrap or ships a targeted migration. `QuoteRepository`'s existing index-creation pattern is the mirror target \u2014 check `trade-flow-api/src/quote/repositories/quote.repository.ts` for how it handles index bootstrap.
- **D-CHAIN-08:** The existing `{businessId: 1, createdAt: -1}` and `{jobId: 1, createdAt: -1}` indexes from Phase 41 remain untouched. List queries that filter on `isCurrent: true` use these indexes for the `businessId`/`jobId` prefix; the `isCurrent` condition is an in-memory filter step since adding it to the index would hurt the `createdAt` sort. Fine for the expected cardinality (a few revisions per estimate).

### Concurrency, Ordering, and 409 (D-CONC)

- **D-CONC-01:** `EstimateReviser` uses **downgrade-old-first, then insert-new** ordering. There is a brief window of "zero current rows in this chain" between the two writes. That window is acceptable because (a) reads of the chain during the window still return the old row (via `_id` lookup) and (b) the alternative "insert-first" is incompatible with the partial unique index without transactions.
- **D-CONC-02:** Step 1: `findOneAndUpdate` on the old row with filter
  ```
  {
    _id: sourceId,
    isCurrent: true,
    status: { $in: [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED] }
  }
  ```
  and update `{ $set: { isCurrent: false } }`. If the returned doc is null (0 matched), throw `ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "Estimate has already been revised or is no longer revisable")`.
- **D-CONC-03:** Step 2: `insertOne` on a new `estimates` document built from the old row (via `EstimateRetriever.findByIdOrFail` at the start of the service call, BEFORE step 1, so the clone has the old field values in memory; the findOneAndUpdate in step 1 is only the `isCurrent` flip, not a read). Insert fields: copy business/customer/job/number/title/estimateDate/notes/contingencyPercent/displayMode/uncertaintyReasons/uncertaintyNotes/lineItems (as a new empty array, filled in step 3) verbatim; `status: DRAFT`; `revisionNumber: old.revisionNumber + 1`; `parentEstimateId: old._id`; `rootEstimateId: old.rootEstimateId`; `isCurrent: true`; `sentAt: null`; `firstViewedAt: null`; `respondedAt: null`; `convertedAt: null`; `convertedToQuoteId: null`; `declinedAt: null`; `lostAt: null`; `expiresAt: null`; `deletedAt: null`; `lastResponseType: null`; `lastResponseAt: null`; `lastResponseMessage: null`; `declineReason: null`; `siteVisitAvailability: null`; `createdAt`/`updatedAt` from the existing base-entity create pattern.
- **D-CONC-04:** Step 3: line-item clone. `EstimateLineItemRepository` is used to fetch all non-deleted line items of the old revision (filter: `estimateId: old._id, status: {$ne: DELETED}`) and then insert fresh rows pointing at the new revision. Each cloned row keeps `itemId`, `quantity`, `unit`, `unitPrice`, `lineTotal`, `originalUnitPrice`, `originalLineTotal`, `discountAmount`, `taxRate`, `type` verbatim. `status` is forced to `PENDING` on every new row regardless of the source status (per D-LI-01 below). `parentLineItemId` is remapped: old line items inside a bundle point to the new bundle-parent's `_id` in the new revision. The planner writes a parent-remap dictionary during the bulk insert.
- **D-CONC-05:** Compensating rollback: if step 2 or step 3 fails for any reason, the service issues `updateOne({_id: sourceId, isCurrent: false}, { $set: { isCurrent: true } })` to restore the old row's current flag. Compensation failure is logged at `AppLogger.error` level with full context (source id, new id if step 2 succeeded but step 3 failed, stack trace) and surfaces to the client as 500. This is an explicit "best-effort rollback; alert on failure" pattern, not a distributed-transaction guarantee.
- **D-CONC-06:** No silent retry on 409. A losing concurrent writer sees the conflict immediately. Client UX (Phase 43) owns the "revise collided with another tab's revise \u2014 refresh and try again" messaging.
- **D-CONC-07:** New `ConflictError` class lives in `src/core/errors/conflict.error.ts` extending the same base as `InvalidRequestError`/`ResourceNotFoundError`/`ForbiddenError`/`InternalServerError`. Its `getCode()` returns the specific `ErrorCodes` value. `createHttpError` in `src/core/errors/handle-error.utility.ts` is extended to map `ConflictError` \u2192 `HttpStatus.CONFLICT` (409). This is the first 409 error in the codebase; the planner verifies the existing error hierarchy and matches it.
- **D-CONC-08:** New error code `ESTIMATE_REVISION_CONFLICT` added to `src/core/errors/error-codes.enum.ts`. Error message string is user-facing but intentionally generic: "Estimate has already been revised or is no longer revisable." Does not leak which exact condition tripped (concurrent revise vs status changed vs isCurrent was already false).

### Line-Item Clone Semantics (D-LI)

- **D-LI-01:** Every cloned line-item row is inserted with `status: EstimateLineItemStatus.PENDING` regardless of the source line item's status. Rationale: the customer has not yet seen the new revision; any `APPROVED`/`REJECTED` state carried forward would be meaningless until the new revision is Sent. `DELETED` line items are not cloned at all (they are skipped in the step-3 fetch filter).
- **D-LI-02:** Bundle parent-child structure is preserved. Step 3 groups fetched line items by `parentLineItemId`: bundle parents (where `parentLineItemId === null` and the item type indicates a bundle parent) are inserted first; their newly-generated `_id` values are recorded in a local `Map<oldParentId, newParentId>`; then bundle children are inserted with `parentLineItemId` remapped via the map. Non-bundle standalone items are inserted with `parentLineItemId: null`.
- **D-LI-03:** Pricing fields \u2014 `unitPrice`, `lineTotal`, `originalUnitPrice`, `originalLineTotal`, `discountAmount`, `taxRate` \u2014 are copied verbatim. No recomputation, no snapshot adjustment. The `EstimateTotalsCalculator` (from Phase 41 D-CONT-05) recomputes `totals` and `priceRange` on every read path, so the cloned rows automatically produce matching totals/range on the new revision.
- **D-LI-04:** `POST /v1/estimates/:id/revisions` does not accept line-item edits in the body. Line-item changes on the new revision are always a follow-up `PATCH /v1/estimates/:id` call (Phase 41's existing update path). This keeps the reviser service small and the clone atomic \u2014 no interleaved "clone some, edit some" semantics.
- **D-LI-05:** The clone happens in the same service call as the two estimate-row writes but is NOT guarded by a transaction. If step 3 fails after step 2 succeeded, the new revision row exists but has no line items. Compensating rollback (D-CONC-05) also soft-deletes the new revision row (`status: DELETED`, `deletedAt: now`) AND restores the old row's `isCurrent: true`, leaving no visible artefacts. The planner writes the rollback logic carefully: a failure in step 3 is more likely than step 2 (line-item clone is a multi-row bulk operation).

### Follow-up Cancellation Hook (D-HOOK)

- **D-HOOK-01:** `IEstimateFollowupCanceller` interface is declared in `src/estimate/services/estimate-followup-canceller.interface.ts`:
  ```typescript
  export interface IEstimateFollowupCanceller {
    cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
  }
  ```
  The method is intentionally side-effect-only (no return value). Takes both `estimateId` and `revisionNumber` so Phase 46 can build the deterministic BullMQ jobId `estimate-followup:{estimateId}:{revisionNumber}:{step}` (per research PITFALLS.md \u00a7Pitfall 2) without a round-trip.
- **D-HOOK-02:** Provider token is the exported string constant `ESTIMATE_FOLLOWUP_CANCELLER` declared in the same interface file. `EstimateModule` registers `NoopEstimateFollowupCanceller` (a one-method class that logs at `debug` level and returns `Promise.resolve()`) under this token. `EstimateReviser` (and, later, `EstimateDeleter` if needed) injects the interface via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)`. Phase 46 ships `BullMQEstimateFollowupCanceller` in a new estimate-followups module and rebinds the provider.
- **D-HOOK-03:** **Phase 42's `EstimateReviser` does NOT call `cancelAllFollowups` at POST time.** Rationale: the new revision is born as DRAFT, not SENT. The old revision's scheduled follow-ups remain meaningful until the trader actually sends the new revision (via Phase 44's send flow). If the trader abandons the new Draft (via DELETE per D-REV-05), the old revision's follow-ups were never cancelled and therefore keep firing on their original schedule \u2014 which is correct behaviour: the customer's experience reverts exactly.
- **D-HOOK-04:** The cancellation call lives in Phase 44's `EstimateEmailSender` service at the moment it transitions the NEW revision from `DRAFT` to `SENT`. At that point, Phase 44 also iterates the chain via `EstimateRepository.findOneByRootId` or similar to find the old revision and calls `cancelAllFollowups(oldRevision._id, oldRevision.revisionNumber)`. Phase 42 ships the wiring (interface, token, injection) so Phase 44 only has to call the method.
- **D-HOOK-05:** **SC #3 amendment (MANDATORY for the planner to reflect in ROADMAP.md):** The original wording "Creating a revision atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision when loading the estimate detail page by root id." is rewritten as:
  > "Creating a revision atomically transitions the chain: the new Draft revision becomes current, the previous revision keeps its existing status with `isCurrent: false`, and the trader sees only the latest revision when loading the estimate detail page by root id. Cancellation of the previous revision's pending follow-ups happens at the moment the new revision is actually Sent (Phase 44 owns the call via the `IEstimateFollowupCanceller` binding from Phase 42)."
  Planner MUST update ROADMAP.md and the Phase 42 entry in `.planning/milestones/v1.8-ROADMAP.md` as part of Phase 42 execution. REV-05 is unchanged \u2014 it's a milestone-level requirement that is satisfied by the Phase 42 \u00d7 Phase 44 \u00d7 Phase 46 chain, not Phase 42 alone.

### History Query (D-HIST)

- **D-HIST-01:** `GET /v1/estimates/:id/revisions` returns the full revision chain as `IEstimateDto[]` (not a trimmed summary DTO). `:id` is resolved to a chain: the controller looks up the estimate by id, extracts `rootEstimateId`, then fetches `{rootEstimateId: X, deletedAt: null}` ordered by `revisionNumber: 1`. Oldest first, per SC #2. Each entry is the full DTO with totals, priceRange, line items, and response summary per Phase 41 D-ENT / D-CONT.
- **D-HIST-02:** Rationale for full DTO over trimmed: the History panel in Phase 43 shows per-revision send/view timestamps, status, and (on click-to-expand) full line-item detail with prices. Serving full DTOs on first load avoids a second round-trip and avoids RTK Query cache-key splintering between `listEstimates`, `getEstimate`, and `getRevisions`. At the expected cardinality (\u2264 5 revisions per chain in practice) the payload cost is trivial.
- **D-HIST-03:** Soft-deleted revisions (`status: DELETED`) are excluded from the history response by default. The query filter is `{rootEstimateId: X, deletedAt: null}`. No query-param option to include soft-deleted revisions in Phase 42. This means if the trader abandons a Draft revision via DELETE-rollback (D-REV-05), that abandoned Draft never appears in the history panel \u2014 which matches the intended "rollback = never happened" semantics.
- **D-HIST-04:** Policy: same as `GET /v1/estimates/:id`. `EstimatePolicy.canRead` on the root row gates the entire chain \u2014 if you can read the root, you can read every revision. Business scoping flows through `rootEstimateId` lookup.
- **D-HIST-05:** The endpoint accepts either a root id or a revision id as `:id`. Both resolve to the same chain because the first DB read extracts `rootEstimateId` from the target row before querying the chain. This matches the Phase 43 UI usage: the trader might have any revision's id in the URL but always sees the full chain.

### Detail-by-id Behaviour (D-DET)

- **D-DET-01:** `GET /v1/estimates/:id` (Phase 41 endpoint) is **extended** in Phase 42 so that when `:id` resolves to a non-current row, the service returns the current row in the same chain instead. Implementation: `EstimateRetriever.findByIdOrFail` loads the target; if `isCurrent === false`, it performs a second lookup `{rootEstimateId: target.rootEstimateId, isCurrent: true}` and returns that row. This satisfies SC #3's requirement that "the trader sees only the latest revision when loading the estimate detail page by root id" while keeping old links/bookmarks working.
- **D-DET-02:** List queries (`GET /v1/estimates`) filter on `isCurrent: true` so that the list shows exactly one row per chain. This is a Phase 42 amendment to `EstimateRetriever.findAll` (or equivalent). The pagination math still works because the filter happens at the Mongo query level, not post-fetch.
- **D-DET-03:** Status tab filtering from Phase 41 SC #2 continues to work: `{businessId: X, isCurrent: true, status: Y}`. No second index needed \u2014 the existing `{businessId: 1, createdAt: -1}` index covers the query with `isCurrent` and `status` as in-memory filters. At the expected cardinality (low thousands of estimates per business over the product's lifetime) this is acceptable.

### Claude's Discretion (planner may override with rationale)

- **Service naming:** `EstimateReviser` is the expected name, mirroring the `EstimateCreator`/`EstimateRetriever`/`EstimateUpdater`/`EstimateDeleter` quartet from Phase 41. Planner chooses whether the revision count increment lives inline or in a shared helper.
- **Controller placement:** `POST /v1/estimates/:id/revisions` and `GET /v1/estimates/:id/revisions` live in `EstimateController` alongside the existing Phase 41 endpoints. Route handlers call `EstimateReviser.revise` and `EstimateRetriever.findRevisionsByIdOrFail` (or similar) respectively.
- **Line-item clone helper:** whether the clone logic lives inside `EstimateReviser` or in a private helper method on `EstimateLineItemRepository` is the planner's call. Preferred: a private method on `EstimateReviser` that takes the old revision's line items and the new revision's `_id`, returns the list of inserts, and delegates the bulk write to the repository.
- **Bootstrap order for index reworks:** planner decides whether the `{businessId, number}` drop-and-recreate and the `{rootEstimateId, isCurrent}` create happen in `EstimateRepository` constructor, in a one-shot migration, or at `AppModule` bootstrap. Mirror whatever pattern Phase 41 picked for its indexes.
- **`ConflictError` base class shape:** planner matches the existing error hierarchy in `src/core/errors/` and wires `createHttpError` to return `HttpStatus.CONFLICT`. The error class file is `conflict.error.ts`.
- **Mock generators:** add `EstimateRevisionMockGenerator` or extend `EstimateMockGenerator` with a `generateRevision(root)` helper. Test files for the reviser service live under `src/estimate/test/services/`.
- **Integration test for concurrent revise (SC #4):** unit-level filter-shape assertion + manual smoke doc per user decision Q1 (resolved 2026-04-11). Minimum: unit test asserting `EstimateRepository.downgradeCurrent` is called with the exact atomic filter `{_id, isCurrent: true, status: {$in: [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED]}}`; unit test asserting reviser throws `ConflictError(ESTIMATE_REVISION_CONFLICT)` when the repository returns `null`; end-to-end two-parallel-caller assertion lives in the manual smoke procedure at `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md`. **Do NOT add `mongodb-memory-server` as a dev dependency.** The existing test infra is pure unit tests with mocked MongoDB primitives and that is intentional.

### Folded Todos

None \u2014 `.planning/STATE.md` shows "Pending Todos: None" and `todo match-phase 42` returned 0 matches.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Milestone roadmap and prior phase context
- `.planning/ROADMAP.md` \u00a7Phase 42 \u2014 goal, requirements, success criteria. Note: SC #3 is being amended by Phase 42 per D-HOOK-05; the planner MUST update ROADMAP.md and `.planning/milestones/v1.8-ROADMAP.md` as part of Phase 42 execution.
- `.planning/milestones/v1.8-ROADMAP.md` \u2014 milestone-level Phase 42 detail, inter-phase dependencies
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` \u2014 REQUIRED READING. Authoritative source for the entity shape, enums, transition map, totals/range calculator contract, collection names, and the reserved revision fields. Phase 42 is a direct extension of these decisions.
- `.planning/notes/2026-04-11-v1.8-restructure-decisions.md` \u2014 D-01 (line-item separation), D-02 (token unification), D-04 (milestone restructure). Authoritative over research docs where they conflict.

### Requirements
- `.planning/REQUIREMENTS.md` \u2014 REV-01, REV-02, REV-03, REV-04, REV-05. Note: REV-05 ("cancels pending follow-ups and schedules fresh sequence") is satisfied across Phase 42 \u00d7 Phase 44 \u00d7 Phase 46, not Phase 42 alone. See D-HOOK-03, D-HOOK-04, D-HOOK-05.

### Research
- `.planning/research/ARCHITECTURE.md` \u00a73 "Versioned Revisions" \u2014 flat collection rationale, index design, `EstimateReviser` flow sketch. Note: the research says to downgrade the old row first (matches D-CONC-01) and explicitly calls out the no-transaction constraint.
- `.planning/research/PITFALLS.md` \u00a7Pitfall 2 (orphaned follow-ups on revise, deterministic jobId pattern \u2014 informs D-HOOK-01 signature), \u00a7Pitfall 3 (stale token resolves to old revision \u2014 Phase 45 owns, but mental model applies), \u00a7Pitfall 12 (compound unique index prevents revision conflicts \u2014 applied as D-CHAIN-05).

### Existing code to mirror (in trade-flow-api)
- `src/quote/quote.module.ts` \u2014 module wiring pattern (how to register new providers and DI tokens)
- `src/quote/controllers/quote.controller.ts` \u2014 controller shape for adding new POST/GET routes to an existing controller
- `src/quote/services/quote-updater.service.ts` \u2014 atomic `findOneAndUpdate` filter patterns, error throwing
- `src/quote/services/quote-creator.service.ts` \u2014 insert + totals attachment pattern (D-CHAIN-04 mirrors the creator retrofit here)
- `src/quote/repositories/quote.repository.ts` \u2014 index bootstrap pattern (whether indexes are created in constructor or at app bootstrap \u2014 planner must check)
- `src/quote/repositories/quote-line-item.repository.ts` \u2014 line-item bulk insert pattern (used for D-LI-02 bundle clone)
- `src/core/errors/` \u2014 existing error class hierarchy, `createHttpError` utility. Phase 42 adds `conflict.error.ts` matching this shape.
- `src/core/errors/error-codes.enum.ts` \u2014 add `ESTIMATE_REVISION_CONFLICT` and `ESTIMATE_NOT_REVISABLE` here
- `src/migration/` \u2014 migration infrastructure if a Phase 41 backfill is needed; example `src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts`

### API contract
- `trade-flow-api/openapi.yaml` \u2014 update with `POST /v1/estimates/:id/revisions` and `GET /v1/estimates/:id/revisions`; document the new 409 response shape
- `trade-flow-api/CLAUDE.md` \u2014 NestJS conventions, layering rules, error handling patterns

### Country-agnostic tax reminder
- Phase 41 D-CONT-03: no `vat*` fields, no `taxRegistered` boolean. This applies to every new DTO field Phase 42 introduces (trivially \u2014 Phase 42 introduces no new tax-related fields).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- **`EstimateCreator` / `EstimateRetriever` / `EstimateUpdater` / `EstimateDeleter`** (Phase 41): the four-service quartet in `src/estimate/services/`. Phase 42 adds a fifth: `EstimateReviser`. Phase 42 also extends `EstimateDeleter` to handle revision-rollback (D-REV-05) and extends `EstimateRetriever` to resolve non-current targets to the chain's current row (D-DET-01).
- **`EstimateTotalsCalculator`** (Phase 41 D-CONT-05): attaches `totals` and `priceRange` on every read path. Phase 42 doesn't change this service \u2014 the cloned revision's totals come out correct automatically because the cloned line items carry identical pricing fields.
- **`EstimateRepository` MongoDB driver primitives** (Phase 41): used for the two-step downgrade-then-insert and for the index drop-and-recreate. Phase 42 adds (a) a `downgradeCurrent` repository method wrapping `findOneAndUpdate` with the atomic status filter, (b) an `insertRevision` method for the new row insert, (c) a `restoreCurrent` method for the compensating rollback, (d) a `findRevisionsByRootId` method for the history query.
- **`EstimateLineItemRepository`** (Phase 41): used in the line-item clone path. The planner may add a `bulkInsertForRevision` method that takes a list of input rows plus a parent-remap dictionary, or clone inline inside `EstimateReviser` \u2014 the planner decides.
- **`QuoteUpdater` `findOneAndUpdate` idioms** (pre-existing): the closest mirror for the atomic status-guarded update. Check `trade-flow-api/src/quote/services/quote-updater.service.ts` for the exact filter/projection shape used in Trade Flow conventions.
- **`AccessControllerFactory` / `AuthorizedCreatorFactory`** (pre-existing `@core/` services): `EstimateReviser` uses these the same way `EstimateCreator` does. Policy enforcement for "can this business revise this estimate?" flows through `EstimatePolicy.canWrite` on the root row.
- **`ErrorCodes` enum and `createHttpError` utility**: Phase 42 extends both with `ESTIMATE_REVISION_CONFLICT` and a new `ConflictError` \u2192 `HttpStatus.CONFLICT` mapping.

### Established Patterns

- **Strict Controller \u2192 Service \u2192 Repository layering.** `EstimateReviser` lives in the service layer and never touches the MongoDB driver directly. All writes flow through `EstimateRepository` methods.
- **No MongoDB transactions in this codebase.** Correctness lives in partial unique indexes and careful ordering. Phase 42's reviser is designed around this constraint: the partial unique on `{rootEstimateId, isCurrent}` is the hard guarantee, not an application-level lock.
- **Entities keep native `Date`; DTOs use Luxon `DateTime`.** Conversion is the repository's job. The cloned revision picks up fresh `createdAt`/`updatedAt` via the base-entity create pattern, not by copying the old values.
- **Error handling** is the Phase 41 pattern: service throws domain error, controller wraps in try/catch, `createHttpError` maps to HTTP. The new `ConflictError` class slots into this pattern \u2014 no controller changes needed beyond the existing try/catch.
- **Test layout:** `src/estimate/test/services/estimate-reviser.service.spec.ts`, `src/estimate/test/controllers/estimate.controller.spec.ts` (extended), `src/estimate/test/repositories/estimate.repository.spec.ts` (extended). Mock generators extend the Phase 41 `EstimateMockGenerator` with a revision helper.
- **Response format:** `createResponse([revision])` / `createResponse(revisions)`. Standard single-or-list wrapping.

### Integration Points

- **`EstimateModule` already imports `CoreModule`, `AuthModule`, `ItemModule`, `BusinessModule`, `CustomerModule`, `JobModule`, `TaxRateModule`** (Phase 41). Phase 42 adds no new module imports. The `IEstimateFollowupCanceller` provider lives inside `EstimateModule` under the string token; Phase 46 rebinds via module composition without Phase 42 needing to know about the follow-ups module.
- **`AppModule` registers `EstimateModule`** (Phase 41). No change.
- **`DocumentTokenModule`** (Phase 41 D-TKN): no interaction with Phase 42. Phase 45 owns the latest-revision token resolution.
- **`openapi.yaml`** gets two new entries: `POST /v1/estimates/:id/revisions` (201 Created with `EstimateResponse`, 409 Conflict, 422 Unprocessable Entity) and `GET /v1/estimates/:id/revisions` (200 with `EstimateListResponse` semantics but unpaginated \u2014 the full chain). Revision count is expected to be small; no pagination needed.

### Known Pitfalls That Apply

- **Pitfall 2 (orphaned follow-ups)**: the deterministic jobId pattern `estimate-followup:{estimateId}:{revisionNumber}:{step}` is already documented in the research. Phase 42's `IEstimateFollowupCanceller.cancelAllFollowups(estimateId, revisionNumber)` signature matches this pattern so Phase 46 can compute target jobIds from the interface arguments.
- **Pitfall 3 (stale token)**: not Phase 42's concern directly, but Phase 42's D-DET-01 (extend `findByIdOrFail` to resolve non-current ids to the chain's current row) is the server-side counterpart. Phase 45's public endpoint applies the same pattern at the public-token layer.
- **Pitfall 12 (concurrent revise \u2192 duplicate `isCurrent`)**: mitigated by D-CHAIN-05 partial unique index + D-CONC-01..08 atomic filter pattern. SC #4 ("exactly one success and one 409") is tested end-to-end in the reviser spec file.
- **Pitfall 8 (counter race)**: not Phase 42's concern \u2014 revisions share the same `E-YYYY-NNN` as the root, so no new counter row is written. But the planner should NOT accidentally call `EstimateNumberGenerator.next()` in the reviser \u2014 that would allocate a new number.
- **Two-write non-transactional rollback risk**: if the compensating `updateOne` in D-CONC-05 fails after a step-3 failure, the chain is left with zero current rows until manual intervention. This is acceptable because (a) the partial unique index still prevents bad state, (b) reads by chain fail cleanly (no current row found), (c) the failure is logged loudly via `AppLogger.error`, and (d) the trader can manually recover by deleting the orphaned new row and re-POSTing the revise. The planner SHOULD include a runbook comment on `EstimateReviser` explaining this recovery path.

</code_context>

<specifics>
## Specific Ideas

### End-to-end revise flow (trader's perspective)

1. Trader opens estimate `E-2026-017` on the detail page. Status: `SENT`. `isCurrent: true`. `revisionNumber: 1`.
2. Trader clicks "Edit and resend" (Phase 43 UI). Frontend calls `POST /v1/estimates/:id/revisions` with no body.
3. `EstimateReviser.revise(id)` executes:
   - Step 0: `findByIdOrFail(id)` loads the source row into memory (old field values).
   - Step 1: `EstimateRepository.downgradeCurrent(id, [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED])` \u2014 atomic `findOneAndUpdate`. Returns `null` \u2192 throw `ConflictError`. Returns doc \u2192 continue.
   - Step 2: build new row, `EstimateRepository.insertRevision(newRow)`. Returns inserted `_id`.
   - Step 3: fetch old line items via `EstimateLineItemRepository.findByEstimateId(old._id)` filtered to non-deleted. Build cloned list with new `estimateId`, `status: PENDING`, remapped `parentLineItemId`. `EstimateLineItemRepository.bulkInsert(cloned)`.
   - Step 4: on any exception in steps 2 or 3, run compensating rollback (restore old `isCurrent: true`; soft-delete new row if step 2 succeeded). Log at error level. Re-throw the original exception wrapped as `InternalServerError` unless it was already a `ConflictError`.
   - Step 5: return the new revision as `IEstimateDto` via `EstimateRetriever.findByIdOrFail(newId)` so `totals`/`priceRange` are freshly calculated.
4. Controller returns `201 Created` with the new revision DTO. Frontend navigates to the new revision's id (or stays on the root id \u2014 both work per D-DET-01).
5. Trader edits the new Draft revision via existing `PATCH /v1/estimates/:id` (Phase 41). Can edit line items, contingency, display mode, notes, uncertainty.
6. Trader clicks "Send" (Phase 44 UI). Phase 44's `EstimateEmailSender` transitions the new row from `DRAFT` to `SENT`, calls `IEstimateFollowupCanceller.cancelAllFollowups(oldRevisionId, oldRevisionNumber)` to cancel the old revision's pending follow-ups, schedules new follow-ups for the new revision, and sends the email. All of this is Phase 44 scope.

If the trader abandons the Draft between steps 5 and 6:
- Trader calls `DELETE /v1/estimates/:id` on the new Draft revision.
- `EstimateDeleter` (extended per D-REV-05) atomically: restores old row's `isCurrent: true`, then soft-deletes the new row.
- Old revision's follow-ups were never cancelled \u2192 they keep firing on the original schedule. Chain is fully restored.

### Repository methods added in Phase 42

**`EstimateRepository`:**
```typescript
// Downgrade the current revision of a chain, atomically gating on status.
// Returns the pre-update document if it matched; null if no row matched the filter
// (meaning: it's already been revised, or status is no longer revisable).
downgradeCurrent(
  id: string,
  allowedSourceStatuses: EstimateStatus[],
): Promise<IEstimateEntity | null>;

// Insert a new revision row. Throws MongoServerError on index violation.
insertRevision(entity: Omit<IEstimateEntity, "_id">): Promise<IEstimateEntity>;

// Compensating: set isCurrent=true on the old row. Used only in rollback paths.
// Filter includes isCurrent: false so it's a no-op if the old row was already
// restored by another code path.
restoreCurrent(id: string): Promise<void>;

// Fetch all non-deleted rows in a chain, ordered by revisionNumber ascending.
findRevisionsByRootId(rootEstimateId: string): Promise<IEstimateEntity[]>;

// Resolve a given id to the chain's current row. Used by EstimateRetriever.findByIdOrFail
// per D-DET-01 so old revision ids still navigate to the current revision.
findCurrentInChainByRootId(rootEstimateId: string): Promise<IEstimateEntity>;
```

**`EstimateLineItemRepository`** (Phase 42 additions):
```typescript
// Bulk insert line items for a new revision. Expects the rows to already have
// their estimateId set to the new revision's _id and parentLineItemId remapped.
bulkInsert(entities: Array<Omit<IEstimateLineItemEntity, "_id">>): Promise<IEstimateLineItemEntity[]>;

// Find non-deleted line items for a given estimate id. Used by the reviser to
// source the clone.
findNonDeletedByEstimateId(estimateId: string): Promise<IEstimateLineItemEntity[]>;
```

(Planner may fold these into existing methods or rename them to match Phase 41's naming conventions. Exact shape is flexible; the contract is what matters.)

### Index changes (Phase 42)

```javascript
// DROP Phase 41's index and recreate with the new partial filter.
// Phase 41 version (to drop):
//   db.estimates.createIndex({ businessId: 1, number: 1 },
//     { unique: true, partialFilterExpression: { deletedAt: null } });

db.estimates.dropIndex("businessId_1_number_1"); // exact name depends on Phase 41's naming
db.estimates.createIndex(
  { businessId: 1, number: 1 },
  { unique: true, partialFilterExpression: { deletedAt: null, isCurrent: true } }
);

// NEW: exactly one current revision per chain.
db.estimates.createIndex(
  { rootEstimateId: 1, isCurrent: 1 },
  { unique: true, partialFilterExpression: { isCurrent: true } }
);

// NEW: fast "get the full chain" queries for the history endpoint.
db.estimates.createIndex({ rootEstimateId: 1, revisionNumber: 1 });
```

The `findCurrentInChainByRootId` query (`{rootEstimateId: X, isCurrent: true}`) is served by the partial unique index. The `findRevisionsByRootId` query (`{rootEstimateId: X, deletedAt: null}` sorted by `revisionNumber`) is served by the `{rootEstimateId: 1, revisionNumber: 1}` index.

### Entity-field defaults for root rows (retrofit from Phase 41)

**Before Phase 42 (Phase 41 as currently specced in 41-CONTEXT.md):**
```
parentEstimateId: null
rootEstimateId: null
revisionNumber: 1
isCurrent: true
```

**After Phase 42 retrofit:**
```
parentEstimateId: null              // root has no predecessor
rootEstimateId: <self._id>          // root is its own chain identity
revisionNumber: 1                   // 1 on root
isCurrent: true                     // true on root
```

The retrofit is done by amending `EstimateCreator.create()` to perform `insertOne` followed by an immediate `updateOne({_id: inserted._id}, {$set: {rootEstimateId: inserted._id}})`. The base-entity base class stays unchanged; the update is explicit in the service. If Phase 41 has already executed when Phase 42 plans, a one-shot `IMigration` backfills `rootEstimateId` from `_id` on every row where `rootEstimateId IS NULL`.

### Wire diagram of the DI binding for `IEstimateFollowupCanceller`

**Phase 42's EstimateModule:**
```typescript
import { NoopEstimateFollowupCanceller } from "./services/noop-estimate-followup-canceller.service";
import { ESTIMATE_FOLLOWUP_CANCELLER } from "./services/estimate-followup-canceller.interface";

@Module({
  // ...imports, controllers...
  providers: [
    EstimateCreator,
    EstimateRetriever,
    EstimateUpdater,
    EstimateDeleter,
    EstimateReviser,
    EstimateTotalsCalculator,
    EstimateNumberGenerator,
    EstimateTransitionService,
    EstimatePolicy,
    EstimateRepository,
    EstimateLineItemRepository,
    // ...factories...
    { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },
  ],
  exports: [/* ...service list... */, ESTIMATE_FOLLOWUP_CANCELLER],
})
export class EstimateModule {}
```

**Phase 46's module (sketched):**
```typescript
@Module({
  imports: [EstimateModule, /* ...queue module... */],
  providers: [
    BullMQEstimateFollowupCanceller,
    { provide: ESTIMATE_FOLLOWUP_CANCELLER, useExisting: BullMQEstimateFollowupCanceller },
  ],
  exports: [ESTIMATE_FOLLOWUP_CANCELLER],
})
export class EstimateFollowupsModule {}
```

Phase 46 imports `EstimateModule` for everything else it needs, then shadows the `ESTIMATE_FOLLOWUP_CANCELLER` token at the module level. Phase 44's `EstimateEmailSender` imports `EstimateFollowupsModule` (or `EstimateModule` pre-Phase-46, with the no-op binding) and injects via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)`.

### Test plan shape (for the planner)

Minimum test coverage expected by Phase 42's plan checker:

1. `estimate-reviser.service.spec.ts` \u2014 happy path: SENT \u2192 new Draft row is current, old row has isCurrent=false, line items cloned with status PENDING.
2. `estimate-reviser.service.spec.ts` \u2014 each allowed-from status (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED) reaches the happy path.
3. `estimate-reviser.service.spec.ts` \u2014 each disallowed-from status (DRAFT, CONVERTED, DECLINED, EXPIRED, LOST, DELETED) throws 409 `ESTIMATE_REVISION_CONFLICT` (because the atomic filter misses).
4. `estimate-reviser.service.spec.ts` \u2014 concurrent revise: two parallel calls, exactly one succeeds and one throws `ConflictError`. Uses a real MongoDB in-memory server (existing test infra) to exercise the partial unique index.
5. `estimate-reviser.service.spec.ts` \u2014 compensating rollback: mock `insertRevision` to throw, assert the old row's `isCurrent` is restored to `true` before the exception is re-thrown.
6. `estimate-reviser.service.spec.ts` \u2014 line-item clone: old revision has bundle parent + 2 children + 1 standalone; new revision has the same shape with fresh `_id`s and remapped `parentLineItemId`; all have `status: PENDING`.
7. `estimate-reviser.service.spec.ts` \u2014 revision number monotonicity: revising N creates N+1; revising N+1 creates N+2.
8. `estimate-reviser.service.spec.ts` \u2014 IEstimateFollowupCanceller is NOT called from the reviser (explicit jest.fn assertion). Phase 42 verifies this is Phase 44's responsibility, not Phase 42's.
9. `estimate-deleter.service.spec.ts` \u2014 DELETE on a non-root Draft revision atomically flips `isCurrent: true` on the predecessor (D-REV-05).
10. `estimate-deleter.service.spec.ts` \u2014 DELETE on a non-root Draft revision where the predecessor is no longer the latest non-deleted row (edge case \u2014 is this even reachable? Only if a concurrent writer intervened) throws `ConflictError`.
11. `estimate-retriever.service.spec.ts` \u2014 `findByIdOrFail(nonCurrentRevisionId)` returns the current row in the chain (D-DET-01).
12. `estimate-retriever.service.spec.ts` \u2014 list query filters to `isCurrent: true` only (D-DET-02).
13. `estimate-retriever.service.spec.ts` \u2014 `findRevisionsByRootId` returns oldest-first, excludes soft-deleted.
14. `estimate.controller.spec.ts` \u2014 `POST /v1/estimates/:id/revisions` happy path + 409 + 422 branches.
15. `estimate.controller.spec.ts` \u2014 `GET /v1/estimates/:id/revisions` happy path (root id) and happy path (revision id both resolve to the same chain).
16. `estimate.repository.spec.ts` \u2014 the partial unique index is in place after repository bootstrap (assert `collection.indexes()` output contains the expected entry).
17. `estimate.repository.spec.ts` \u2014 attempt to write a second row with `isCurrent: true` + same `rootEstimateId` throws `MongoServerError` with duplicate key code.

### 409 Conflict response shape

```json
{
  "data": [],
  "errors": [
    {
      "code": "ESTIMATE_REVISION_CONFLICT",
      "message": "Estimate has already been revised or is no longer revisable",
      "details": null
    }
  ]
}
```

Status: `409 Conflict`. Matches existing Trade Flow error response shape (empty `data` array, single entry `errors[]`).

</specifics>

<deferred>
## Deferred Ideas

- **Adding a `SUPERSEDED` value to the `EstimateStatus` enum** \u2014 rejected. The `isCurrent` boolean carries the "this is the visible revision" meaning; the status lifecycle stays exactly as Phase 41 locked it.
- **Combined revise+resend single-call endpoint** \u2014 rejected in favour of clone-then-PATCH-then-send symmetry with the Phase 41 create flow.
- **Revising a Draft estimate** \u2014 rejected. Drafts are edited in place via PATCH; creating a revision of a Draft would be semantically meaningless and violates the "revision = invisible customer-facing change" premise.
- **Trimmed `IEstimateRevisionSummaryDto`** for the history endpoint \u2014 considered, rejected. Full DTOs ship \u2014 cardinality is low, and the Phase 43 history panel needs expandable line-item detail.
- **Silent retry on 409** \u2014 rejected. The 409 surfaces immediately so client UX can distinguish "I revised twice" from "someone else revised while I was editing."
- **Transaction-based revise** \u2014 explicitly ruled out because the codebase does not use MongoDB transactions (research note \u00a73 "Transaction semantics for revisions"). The partial unique index is the correctness guarantee.
- **`{parentEstimateId: 1, isCurrent: 1}` partial unique index** (alternative to `{rootEstimateId, isCurrent}`) \u2014 rejected. `parentEstimateId` is null on root rows; the partial filter would have to exclude them, which is exactly the case we already considered for `rootEstimateId = null` and rejected.
- **Public `GET /v1/public/estimates/:token` latest-revision resolution** \u2014 deferred to Phase 45. Phase 42's D-DET-01 only covers the authenticated trader path.
- **Query-param `includeDeleted=true`** on the history endpoint \u2014 deferred. No known use case in v1.8.
- **Revision-level events / audit log** \u2014 deferred. Phase 42 does not emit domain events; Phase 46's cancel hook is the only cross-phase integration point.
- **Draft-resume detection** on the list page (surfacing "you have an unsent Draft revision for E-2026-017") \u2014 deferred to Phase 43 UI.
- **`isRevisionOf` denormalisation on line items** \u2014 rejected. Line items carry `estimateId` only; the revision relationship lives on the estimate, not the line item.
- **Amending SC #3 literal wording in ROADMAP** (per D-HOOK-05) \u2014 this is NOT deferred, it is MANDATORY for the planner to update ROADMAP.md during Phase 42 execution. Listed here only for traceability of the alternative "leave the wording as-is" path.
- **Auto-cleanup of abandoned Draft revisions** (e.g. a scheduled sweep deleting Drafts older than 30 days) \u2014 rejected for Phase 42. Drafts persist indefinitely until the trader DELETEs them.

### Reviewed Todos (not folded)

None \u2014 `todo match-phase 42` returned 0 matches and `.planning/STATE.md` reports no pending todos.

</deferred>

---

*Phase: 42-revisions*
*Context gathered: 2026-04-11*
