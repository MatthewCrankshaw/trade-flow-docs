# Phase 42: Revisions - Research

**Researched:** 2026-04-11
**Domain:** NestJS service/repository extension, MongoDB partial unique indexes, atomic two-step writes, DI token indirection
**Confidence:** HIGH for code mirrors and driver semantics; MEDIUM for concurrency test strategy (test infra gap identified)

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

All sections of CONTEXT.md `<decisions>` are locked and NOT subject to research alternatives. Summary for traceability:

- **D-REV (Revise endpoint shape):**
  - D-REV-01: `POST /v1/estimates/:id/revisions` is clone-only; no body; deep-copies source to new row with `status: DRAFT`, `isCurrent: true`, `revisionNumber: prev+1`, `parentEstimateId: prev._id`, `rootEstimateId: prev.rootEstimateId`. Edits via existing PATCH.
  - D-REV-02: Allowed source statuses `SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED`. Check lives in the atomic `findOneAndUpdate` filter.
  - D-REV-03: Source must be `isCurrent: true`. Non-current source returns 409 `ESTIMATE_REVISION_CONFLICT`.
  - D-REV-04: Previous revision keeps existing status, flips `isCurrent: false`. No new `SUPERSEDED` enum value.
  - D-REV-05: DELETE on non-root Draft revision atomically flips `isCurrent: true` on `parentEstimateId` (rollback path).
  - D-REV-06: Two-step guarded-write for delete: restore predecessor first, soft-delete the row second; compensate if step 2 fails.

- **D-CHAIN (Chain identity & index rework):**
  - D-CHAIN-01..02: `rootEstimateId` is the chain identity; `parentEstimateId` is the immediate predecessor. Root: `rootEstimateId === self._id`, `parentEstimateId: null`.
  - D-CHAIN-03..04: Retrofit `EstimateCreator` so root rows write `rootEstimateId = inserted._id` (insert + immediate updateOne). Fold into Phase 41 if not yet executed; otherwise ship a one-shot `IMigration`.
  - D-CHAIN-05: New partial unique index `{rootEstimateId:1, isCurrent:1}` partial on `{isCurrent: true}`.
  - D-CHAIN-06: Phase 41's `{businessId:1, number:1}` partial unique index is DROPPED and recreated with partial filter `{deletedAt: null, isCurrent: true}`.
  - D-CHAIN-07: Index drop-recreate must happen before any reviser code path runs.
  - D-CHAIN-08: Phase 41's `{businessId, createdAt:-1}` and `{jobId, createdAt:-1}` remain; `isCurrent` is in-memory filter.

- **D-CONC (Concurrency, ordering, 409):**
  - D-CONC-01: Downgrade-old-first, then insert-new. Brief zero-current window is acceptable.
  - D-CONC-02: Step 1 is atomic `findOneAndUpdate` with filter `{_id, isCurrent: true, status: {$in: [...allowed]}}`. Null return ⇒ 409.
  - D-CONC-03: Step 2 inserts the new row with fully populated clone (full field inventory in CONTEXT).
  - D-CONC-04: Step 3 clones non-deleted line items via `EstimateLineItemRepository`, forcing `status: PENDING`, remapping `parentLineItemId` via a bundle-parent map.
  - D-CONC-05: Compensating rollback restores old `isCurrent: true` on failure in step 2 or 3. Log at `error`. Best-effort, not a transaction.
  - D-CONC-06: No silent retry on 409.
  - D-CONC-07: New `ConflictError` class at `src/core/errors/conflict.error.ts`; extend `createHttpError` to map to `HttpStatus.CONFLICT` (409).
  - D-CONC-08: New error codes `ESTIMATE_REVISION_CONFLICT` and `ESTIMATE_NOT_REVISABLE`. Message intentionally generic.

- **D-LI (Line-item clone semantics):**
  - D-LI-01: Every cloned line item forced to `status: PENDING`. DELETED items skipped.
  - D-LI-02: Bundle parents inserted first, IDs captured, children remapped via `Map<oldParentId, newParentId>`.
  - D-LI-03: Pricing fields copied verbatim. `EstimateTotalsCalculator` recomputes on read.
  - D-LI-04: No inline line-item edits in the POST body. Edits happen via PATCH afterward.
  - D-LI-05: Step 3 is NOT transactional. Step-3 failure ⇒ soft-delete new row + restore old `isCurrent: true`.

- **D-HOOK (Follow-up cancellation hook):**
  - D-HOOK-01: `IEstimateFollowupCanceller` interface in `src/estimate/services/estimate-followup-canceller.interface.ts` with signature `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>`.
  - D-HOOK-02: String provider token `ESTIMATE_FOLLOWUP_CANCELLER`. `NoopEstimateFollowupCanceller` default binding. `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)`.
  - D-HOOK-03: Phase 42 wires but does NOT call the hook from `EstimateReviser`. Phase 44 calls it on the `DRAFT → SENT` transition.
  - D-HOOK-04: Cancellation happens at the moment the new revision is sent (Phase 44).
  - D-HOOK-05: ROADMAP SC #3 is rewritten. MANDATORY ROADMAP.md and v1.8-ROADMAP.md update in Phase 42 execution.

- **D-HIST (History query):**
  - D-HIST-01..05: `GET /v1/estimates/:id/revisions` returns full `IEstimateDto[]`, oldest-first, excludes soft-deleted. Accepts either root or revision id. Same policy as `GET /v1/estimates/:id`.

- **D-DET (Detail-by-id behaviour):**
  - D-DET-01: `EstimateRetriever.findByIdOrFail` resolves non-current target → current row in chain via `{rootEstimateId, isCurrent: true}`.
  - D-DET-02: List query filters on `isCurrent: true` at the Mongo query level.
  - D-DET-03: Status-tab filter continues to work via in-memory filtering on `{businessId, createdAt:-1}` index.

### Claude's Discretion

Copied verbatim from CONTEXT.md:

- **Service naming:** `EstimateReviser` expected. Planner chooses whether revision-count increment is inline or in a helper.
- **Controller placement:** both new endpoints live in `EstimateController` alongside Phase 41 endpoints.
- **Line-item clone helper:** whether the clone logic lives inside `EstimateReviser` or as a private helper method on `EstimateLineItemRepository` is the planner's call. Preferred: private method on `EstimateReviser` building the insert list, repository does the bulk write.
- **Bootstrap order for index reworks:** planner decides whether `{businessId, number}` drop-recreate and `{rootEstimateId, isCurrent}` create happen in `EstimateRepository` `onModuleInit`, in a one-shot migration, or at `AppModule` bootstrap. Mirror Phase 41's pattern.
- **`ConflictError` base class shape:** matches existing error hierarchy; wires `createHttpError` to `HttpStatus.CONFLICT`.
- **Mock generators:** add `EstimateRevisionMockGenerator` or extend `EstimateMockGenerator` with `generateRevision(root)`.
- **Integration test for concurrent revise (SC #4):** planner designs this. Minimum: two parallel calls, exactly one succeeds and one throws `ConflictError`; assert exactly one row has `isCurrent: true` in chain. "Mongo in-memory server is fine (existing pattern)." — NOTE: Research has identified that this is NOT the existing pattern; see §4 Test Strategy for the actual infra state and the recommended substitution.

### Deferred Ideas (OUT OF SCOPE)

Copied verbatim from CONTEXT.md:

- `SUPERSEDED` enum value on `EstimateStatus` — rejected.
- Combined revise+resend single endpoint — rejected.
- Revising a Draft estimate — rejected.
- Trimmed `IEstimateRevisionSummaryDto` — rejected; full DTOs ship.
- Silent retry on 409 — rejected.
- Transaction-based revise — explicitly ruled out.
- `{parentEstimateId, isCurrent}` partial unique index (alternative) — rejected.
- Public latest-revision resolution — deferred to Phase 45.
- `includeDeleted=true` history param — deferred.
- Revision-level events / audit log — deferred.
- Draft-resume detection on list page — deferred to Phase 43.
- `isRevisionOf` denormalisation on line items — rejected.
- Auto-cleanup of abandoned Draft revisions — rejected for Phase 42.

*Note:* "Amending SC #3 literal wording in ROADMAP per D-HOOK-05" is listed in CONTEXT's Deferred block only for traceability — it is NOT deferred; it is MANDATORY in Phase 42 execution.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| REV-01 | Estimate entity stores `parentEstimateId`, `revisionNumber`, `isCurrent` to support versioned revisions | Phase 41 PLAN-04 already declares these fields on `IEstimateEntity` and `IEstimateDto` (41-04 lines 445-449 and 598-602). Phase 42 adds the retrofit so root rows carry `rootEstimateId = self._id`. REV-01 wording says "parentEstimateId" but the data model CONTEXT D-CHAIN-01/02 uses both `parentEstimateId` (predecessor) and `rootEstimateId` (chain). REQUIREMENTS.md wording is out-of-date; CONTEXT overrides. |
| REV-02 | Trader can revise a Sent estimate via "Edit and resend" — new revision under same `E-YYYY-NNN`, previous marked non-current | D-REV-01/02 + D-CONC flow in CONTEXT. `EstimateReviser` service + `POST /v1/estimates/:id/revisions`. |
| REV-03 | Only one revision per chain has `isCurrent: true`, enforced by a partial unique index on `(parentEstimateId, isCurrent: true)` | REQUIREMENTS.md says `parentEstimateId`; CONTEXT D-CHAIN-05 overrides to `{rootEstimateId: 1, isCurrent: 1}` because `parentEstimateId` is null on roots. Partial filter expression: `{isCurrent: true}`. Confirmed possible in MongoDB 7.0 + native driver 7.0.0 (§3). |
| REV-04 | Estimate detail shows collapsed History section listing previous revisions with send/view timestamps (trader-only) | `GET /v1/estimates/:id/revisions` (D-HIST-01). Returns full `IEstimateDto[]` oldest-first. UI lives in Phase 43. |
| REV-05 | Revising cancels pending follow-ups and schedules fresh 3/10/21-day sequence | Satisfied across Phase 42 × Phase 44 × Phase 46 per D-HOOK-03/04/05. Phase 42 ships `IEstimateFollowupCanceller` + `NoopEstimateFollowupCanceller` binding + `ESTIMATE_FOLLOWUP_CANCELLER` token. Cancellation call happens in Phase 44. SC #3 rewritten. |
</phase_requirements>

---

## 1. Summary

Phase 42 is an additive extension to Phase 41 (not yet executed; plans 41-01..41-08 are in "planning complete" state per `.planning/STATE.md`). It adds two new endpoints, a fifth estimate service (`EstimateReviser`), a new error class (`ConflictError`) + 2 new error codes, a new provider token (`ESTIMATE_FOLLOWUP_CANCELLER`) with a no-op default binding, two new indexes (plus a drop-and-recreate of one Phase 41 index), and targeted extensions to `EstimateCreator`, `EstimateDeleter`, and `EstimateRetriever` from Phase 41.

**The big risk is not the shape of the code** — CONTEXT.md locks 35 decisions with surgical precision — **it is three implementation realities the research surfaced:**

1. **Phase 41 is not executed yet.** All 8 plans exist but `ls trade-flow-api/src/estimate/` returns empty. The retrofit for `rootEstimateId = self._id` should be folded into Phase 41 PLAN-07 directly (§5). Phase 41 PLAN-07 currently hard-codes `rootEstimateId: null` with a grep assertion on line 277 of 41-07-estimate-crud-services-PLAN.md — Phase 42's planner must either amend PLAN-07 or ship the migration.

2. **There is no `mongodb-memory-server` in the repo.** Neither `package.json` nor any existing spec file uses it. The entire test suite is pure unit tests with mocked `MongoDbFetcher`/`MongoDbWriter`/`MongoConnectionService` primitives — see `trade-flow-api/src/subscription/test/repositories/subscription.repository.spec.ts` lines 15-52. CONTEXT.md D-discretion says "Mongo in-memory server is fine (existing pattern)" but that is not the existing pattern. The planner must either (a) add `mongodb-memory-server` as a new dev dependency and accept the infra lift, or (b) redesign SC #4 to be verified by a focused integration test that mocks the `MongoConnectionService` to inject a `mongodb-memory-server` instance OR (c) implement SC #4 as a manual smoke test plus unit-level verification that the filter in step 1 is correct by construction. **The planner must decide and flag this in PLAN.md.** Research recommends option (a) because SC #4 is a stated success criterion and the partial-unique-index guarantee is only truly exercised against a real MongoDB instance.

3. **`MongoDbWriter` does not expose `insertMany`.** Phase 42 needs a bulk line-item insert for step 3 of the reviser. Two paths: (a) call `QuoteLineItemCreator`-style sequential `insertOne` calls in a loop (simple, slower, still within one service call), or (b) add an `insertMany` primitive to `MongoDbWriter` (cleaner, but a core-service edit that crosses phase boundaries). Research recommends option (a) for scope discipline — typical line-item counts are small and the compensating rollback logic is the same either way.

**Primary recommendation:** Fold the `rootEstimateId` retrofit into Phase 41 PLAN-07 (path A below). Install `mongodb-memory-server` as a new dev dependency scoped to the one `estimate-reviser.service.spec.ts` concurrency test (§7). Use sequential `insertOne` loops for line-item clone to avoid touching `MongoDbWriter`. Ship `conflict.error.ts` matching the exact shape of `invalid-request.error.ts` (constructor, private fields, three public getters — all four existing error classes are byte-for-byte identical modulo the class name). Add `ConflictError` branch to `createHttpError` between the `InvalidRequestError` branch and the `ResourceNotFoundError` branch. Put the index bootstrap inside `EstimateRepository.onModuleInit()` mirroring `SubscriptionRepository` (§2, the precedent exists).

---

## 2. Code Mirrors

### 2.1 Error class hierarchy (`trade-flow-api/src/core/errors/`)

**All four existing error classes have the exact same shape.** Source:

- `invalid-request.error.ts` lines 4-31
- `forbidden-error.error.ts` lines 4-31
- `resource-not-found.error.ts` lines 4-31
- `internal-server-error.error.ts` lines 4-31

The shape (verbatim from `invalid-request.error.ts`):

```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ERRORS_MAP } from "@core/errors/errors-map.constant";

export class InvalidRequestError extends Error {
  private code: ErrorCodes;
  private details: string;

  constructor(code: ErrorCodes, details: string) {
    const { message } = ERRORS_MAP.get(code) ?? {
      message: "Error code unknown",
      code: ErrorCodes.ERROR_CODE_UNKNOWN,
    };
    super(message);
    this.name = "InvalidRequestError";
    this.details = details;
    this.code = code;
  }

  public getCode() { return this.code; }
  public getMessage() { return this.message; }
  public getDetails() { return this.details; }
}
```

**Phase 42's `conflict.error.ts` MUST match this byte-for-byte** (rename class, rename `this.name`). No base class, no abstract parent — all four existing files are standalone duplicates. The planner MUST NOT introduce a new `BaseError` abstract class just because "four files duplicate the shape" — that is a refactor outside Phase 42 scope.

**Filename:** `conflict.error.ts` (matches `forbidden-error.error.ts` naming — trailing `-error`? Actually `forbidden-error.error.ts` double-suffixes. `invalid-request.error.ts`, `resource-not-found.error.ts`, `internal-server-error.error.ts` follow the same pattern. Planner should follow `conflict.error.ts` (single `.error.ts` suffix). **Verified inconsistency: `forbidden-error.error.ts` is the odd one out.** Planner's call — recommendation: `conflict.error.ts`.)

### 2.2 `createHttpError` utility extension

Source: `trade-flow-api/src/core/errors/handle-error.utility.ts` lines 9-86.

Current order of branches: `HttpException` → `InternalServerError` → `InvalidRequestError` → `ResourceNotFoundError` → `ForbiddenError` → fallthrough.

Phase 42 adds `ConflictError` branch. Best insertion point: **after `InvalidRequestError`, before `ResourceNotFoundError`** (places 409 adjacent to 422 since both are client-side request-state errors). Logger call should be `Logger.warn` (not `Logger.error`) to match the pattern of `InvalidRequestError` and `ResourceNotFoundError`:

```typescript
if (error instanceof ConflictError) {
  const response: IResponse = {
    errors: [{
      code: error.getCode(),
      message: error.getMessage(),
      details: error.getDetails(),
    }],
  };
  Logger.warn("Conflict error encountered:", response);
  return new HttpException(response, HttpStatus.CONFLICT);
}
```

### 2.3 Error codes enum + errors map

Source: `trade-flow-api/src/core/errors/error-codes.enum.ts` (lines 1-69), `errors-map.constant.ts` (lines 1-278).

**Add to `error-codes.enum.ts`** (two new entries, using `ESTIMATE_*` namespace reserved by Phase 41):

```typescript
  // Estimates (Phase 41 reserves ESTIMATE_0..ESTIMATE_N; Phase 42 adds the below)
  ESTIMATE_REVISION_CONFLICT = "ESTIMATE_X",   // X to be assigned after Phase 41 finalises its codes
  ESTIMATE_NOT_REVISABLE     = "ESTIMATE_X+1",
```

**Coordination risk:** Phase 41 PLAN-01 likely reserves the first estimate error codes. Phase 42's planner must read Phase 41's final error-codes list before assigning slots. Fallback: use the next available ordinals after whatever Phase 41 lands on.

**Add to `errors-map.constant.ts`:**

```typescript
  [
    ErrorCodes.ESTIMATE_REVISION_CONFLICT,
    {
      code: ErrorCodes.ESTIMATE_REVISION_CONFLICT,
      message: "Estimate has already been revised or is no longer revisable",
    },
  ],
  [
    ErrorCodes.ESTIMATE_NOT_REVISABLE,
    {
      code: ErrorCodes.ESTIMATE_NOT_REVISABLE,
      message: "Estimate cannot be revised in its current state",
    },
  ],
```

### 2.4 Index bootstrap pattern

**Source:** `trade-flow-api/src/subscription/repositories/subscription.repository.ts` lines 13, 23-25, 142-149.

**KEY FINDING:** `QuoteRepository` (lines 22-31) does **NOT** implement `OnModuleInit` and does **NOT** create any indexes. Phase 41 CONTEXT D-ENT-03 and Phase 41 PLAN-05 mandate a `createIndexes()` method in `EstimateRepository`, and PLAN-05 line 197 says "If `QuoteRepository` uses an `onModuleInit` hook or lifecycle method, mirror that." It doesn't. The blessed precedent is **`SubscriptionRepository`**:

```typescript
@Injectable()
export class SubscriptionRepository implements OnModuleInit {
  // ...
  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
    private readonly connection: MongoConnectionService,
  ) {}

  async onModuleInit(): Promise<void> {
    await this.ensureIndexes();
  }

  private async ensureIndexes(): Promise<void> {
    const db = await this.connection.getDb();
    const collection = db.collection(SubscriptionRepository.COLLECTION);
    await collection.createIndex({ userId: 1 }, { unique: true });
    await collection.createIndex({ stripeSubscriptionId: 1 }, { unique: true, sparse: true });
    await collection.createIndex({ stripeLatestCheckoutSessionId: 1 }, { unique: true, sparse: true });
    this.logger.log("Subscription indexes ensured");
  }
}
```

**Phase 42 implication:** If Phase 41 is folded-amended, `EstimateRepository.onModuleInit` calls a single `ensureIndexes()` method that declares all three Phase 41 indexes AND the two new Phase 42 indexes (plus the reworked `{businessId, number}` filter). No migration file is needed.

**Important driver detail:** `createIndex` in MongoDB native driver 7.0.0 is **idempotent IF the index specification matches**. Calling it on app startup in a multi-replica deployment is safe — the driver silently no-ops when the index already exists with the same spec. However, **if the spec differs (e.g., different `partialFilterExpression`) `createIndex` throws `IndexOptionsConflict` (error code 85).** This is the D-CHAIN-06 drop-and-recreate risk: Phase 41's index has filter `{deletedAt: null}`; Phase 42's has `{deletedAt: null, isCurrent: true}`. If `createIndex` is called without first dropping, it throws.

**Safe drop-and-recreate in `ensureIndexes`:**

```typescript
// Drop the Phase 41 version if it exists (safe no-op if absent)
try {
  await collection.dropIndex("businessId_1_number_1");
} catch (e) {
  // IndexNotFound error code 27; ignore
  if ((e as { codeName?: string }).codeName !== "IndexNotFound") throw e;
}
await collection.createIndex(
  { businessId: 1, number: 1 },
  { unique: true, partialFilterExpression: { deletedAt: null, isCurrent: true } }
);
```

This pattern is safe across concurrent startups (single instance on Railway) but produces a brief "index missing" window under multi-replica bootstrap. See §9 Pitfalls.

### 2.5 `findOneAndUpdate` primitive

**Source:** `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` lines 37-52.

```typescript
async findOneAndUpdate<TSchema extends Record<string, unknown>>(
  collectionName: string,
  filter: Filter<TSchema>,
  update: UpdateFilter<TSchema>,
  options?: FindOneAndUpdateOptions,
): Promise<WithId<TSchema> | null> {
  const db = await this.connection.getDb();
  const result = await db.collection<TSchema>(collectionName).findOneAndUpdate(filter, update, {
    returnDocument: "after",
    ...options,
  });
  if (!result) return null;
  return result as WithId<TSchema>;
}
```

**Semantics:**
- Default `returnDocument: "after"` — the returned doc is the POST-update state.
- On filter miss: returns `null` (no exception).
- `findOneAndUpdate` is atomic at the document level — critical for the D-CONC-02 step-1 guarantee.

**D-CONC-02 implication:** The filter `{_id, isCurrent: true, status: {$in: [...]}}` with update `{$set: {isCurrent: false}}` will:
- **Match:** atomically flip `isCurrent` → `false`, return the updated doc (with `isCurrent: false`).
- **Miss:** return `null` — meaning either (a) row doesn't exist, (b) status not in allowed list, (c) already `isCurrent: false`.

All three miss cases collapse to the same 409 conflict per CONTEXT D-CONC-08 (intentionally generic message). The service layer inspects `=== null` and throws `ConflictError`.

**Pitfall noted:** Because `returnDocument: "after"`, the returned doc has `isCurrent: false`, not the "before" state. This is fine for our flow because we fetched the old-row snapshot via `findByIdOrFail(id)` BEFORE the atomic update (Step 0 in CONTEXT's specifics line 212). The only reason to look at the returned doc is to confirm non-null (match happened).

### 2.6 `QuoteUpdater` findOneAndUpdate idiom

**Source:** `trade-flow-api/src/quote/services/quote-updater.service.ts`, all 166 lines.

**KEY FINDING:** `QuoteUpdater` does **NOT** use atomic filter-gated `findOneAndUpdate`. It uses a read-then-update pattern — `findByIdOrFail(quoteId)` then calls on line items. The atomic-update idiom CONTEXT D-CONC-02 references does not exist in `QuoteUpdater`. The closest existing mirror is `SubscriptionRepository.upsertByUserId` (lines 44-66) which uses `findOneAndUpdate` with an empty-row-safe upsert — not the same shape as what Phase 42 needs.

**Conclusion:** There is no existing "atomic status-gated flip" precedent in the codebase. Phase 42's `EstimateRepository.downgradeCurrent(id, allowedStatuses)` is a NEW pattern. The planner should document this explicitly in PLAN.md as a novel primitive and write extensive unit tests (mocked `MongoDbWriter.findOneAndUpdate` with filter-inspection assertions) to verify the filter shape is correct.

**Repository method shape (recommended):**

```typescript
public async downgradeCurrent(
  id: string,
  allowedSourceStatuses: EstimateStatus[],
): Promise<IEstimateDto | null> {
  const _id = new ObjectId(id);
  const updated = await this.writer.findOneAndUpdate<IEstimateEntity>(
    EstimateRepository.COLLECTION,
    {
      _id,
      isCurrent: true,
      status: { $in: allowedSourceStatuses },
    },
    { $set: { isCurrent: false, ...updateAuditFields() } },
  );
  if (!updated) return null;
  // The caller needs the PRE-update snapshot; fetch separately or rely on the service-layer snapshot taken at step 0.
  // Recommended: return toDto(updated) — it's the "new" state but carries all pre-flip data (number, status, businessId, etc.)
  return this.toDto(updated);
}
```

### 2.7 Controller POST/GET shape

**Source:** `trade-flow-api/src/quote/controllers/quote.controller.ts` lines 30-119.

Pattern:
- Class decorator `@Controller("v1")`.
- Each route handler wraps its body in `try { ... } catch (error) { throw createHttpError(error); }`.
- `@Req() request: { user: IUserDto; params: {...} }` — inline typed request per CLAUDE.md.
- Responses mapped via private `enrichAndMapToResponse` method and wrapped in `createResponse([...])`.
- `@UseGuards(JwtAuthGuard)` on every authenticated route.

**Phase 42 controller additions (mirror to EstimateController from Phase 41 PLAN-08):**

```typescript
@UseGuards(JwtAuthGuard)
@Post("estimates/:id/revisions")
public async revise(
  @Req() request: { user: IUserDto; params: { id: string } },
): Promise<IResponse<IEstimateResponse>> {
  try {
    const estimate = await this.estimateReviser.revise(request.user, request.params.id);
    const response = await this.enrichAndMapToResponse(request.user, estimate);
    return createResponse([response]);
  } catch (error) {
    throw createHttpError(error);
  }
}

@UseGuards(JwtAuthGuard)
@Get("estimates/:id/revisions")
public async findRevisions(
  @Req() request: { user: IUserDto; params: { id: string } },
): Promise<IResponse<IEstimateResponse>> {
  try {
    const chain = await this.estimateRetriever.findRevisionsByIdOrFail(request.user, request.params.id);
    const responses = await Promise.all(chain.map((e) => this.enrichAndMapToResponse(request.user, e)));
    return createResponse(responses);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

**Route prefix confusion:** Phase 41 PLAN-08 will define the estimate routes. Per CLAUDE.md "All endpoints use the `/v1` prefix via `@Controller('v1')`" and per `QuoteController` line 30, the pattern is `@Controller("v1")` at class level with full paths in handlers. So handlers are `@Post("estimates/:id/revisions")`, not `@Post("revisions")` with a nested controller. Planner confirms against Phase 41 PLAN-08 before writing.

### 2.8 Module wiring pattern

**Source:** `trade-flow-api/src/quote/quote.module.ts` lines 33-73.

Pattern:
- `@Module({ imports, controllers, providers, exports })`.
- Providers listed flat (no grouping, though Phase 42 can add a blank-line separator).
- Exports only what other modules need.

**Phase 42 additions to `EstimateModule.providers` (plan-08 already declares most of this):**

```typescript
providers: [
  // ...Phase 41 providers...
  EstimateReviser,                                      // NEW
  { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },  // NEW
  NoopEstimateFollowupCanceller,                        // NEW (useClass requires the class to be registered)
],
exports: [
  // ...existing exports from Phase 41...
  ESTIMATE_FOLLOWUP_CANCELLER,                          // NEW — so Phase 44/46 can re-import
],
```

**DI token pattern precedent:** Searching for `@Inject(` string-token usage in `trade-flow-api/src/` — none yet (all existing DI uses class-based tokens). This will be the first use of a string-token provider in the codebase. The pattern is standard NestJS and documented at https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens — no risk, but the planner should document it as a new pattern for future-phase reference.

### 2.9 Line-item clone — bulk insert path

**Source:** `trade-flow-api/src/quote/repositories/quote-line-item.repository.ts` (full 165 lines) and `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` (full 80 lines).

**KEY FINDING:** `MongoDbWriter` exposes `insertOne`, `updateOne`, `findOneAndUpdate`, `deleteOne`, `updateMany`, `deleteMany` — **but NOT `insertMany`**. The existing `QuoteLineItemRepository.create()` does a single `insertOne` per call. Line-item creation in the quote flow is therefore sequential (see `QuoteUpdater.addBundleLineItems` lines 142-157 — parent insert, then `Promise.all(...)` over components).

**Phase 42 recommendation:** For the step-3 clone (non-transactional anyway), use the same sequential pattern. Line-item counts per estimate are low (typically < 20 rows). Don't add `insertMany` to `MongoDbWriter` in Phase 42 — that's a core-service edit crossing phase boundaries. If the planner wants a tidier contract, add a **private helper** on `EstimateLineItemRepository` named `cloneAllForRevision(sourceEstimateId, targetEstimateId)` that:

1. Calls `findNonDeletedByEstimateId(sourceEstimateId)` (existing method per Phase 41 PLAN-05).
2. Groups by `parentLineItemId` into `[roots, children]`.
3. Loops `roots` with `insertOne`, collecting new `_id` into a `Map<string, ObjectId>`.
4. Loops `children` with `insertOne`, remapping `parentLineItemId` via the map.
5. Forces every row's `status = EstimateLineItemStatus.PENDING` (D-LI-01).
6. Returns the inserted line items as `IEstimateLineItemDto[]` for the caller to attach to the new revision.

The sequential pattern is acceptable because the whole operation is non-transactional anyway and the compensating rollback logic handles partial failure.

### 2.10 Migration infrastructure

**Source:** `trade-flow-api/src/migration/interfaces/migration.interface.ts`, example `trade-flow-api/src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts`, runner entrypoint `trade-flow-api/src/migration/controllers/migration.controller.ts` lines 42-50.

**Shape:** `IMigration` has `migrationName`, `up(db: Db): Promise<void>`, `down(db: Db): Promise<void>`, `isBootstrap(): boolean`. Migrations live in `src/migration/migrations/` with timestamp-prefixed filenames. They run via authenticated `POST /v1/migrations/run`. **Migrations MUST NOT run on app startup** (Phase 41 D-TKN-06).

**Phase 42 implication:** Only needed if Phase 41 executes BEFORE Phase 42 lands the retrofit. Recommended path (§5) is to amend Phase 41 PLAN-07 and skip the migration. If the migration IS needed, it is a one-shot:

```typescript
// src/migration/migrations/20260412000000-backfill-estimate-root-id.migration.ts
export default class BackfillEstimateRootIdMigration implements IMigration {
  migrationName = "20260412000000-backfill-estimate-root-id.migration";

  async up(db: Db): Promise<void> {
    const collection = db.collection("estimates");
    // Set rootEstimateId = _id on every row where rootEstimateId IS NULL AND parentEstimateId IS NULL (i.e., roots).
    const cursor = collection.find({ rootEstimateId: null, parentEstimateId: null });
    for await (const doc of cursor) {
      await collection.updateOne({ _id: doc._id }, { $set: { rootEstimateId: doc._id } });
    }
  }

  async down(db: Db): Promise<void> {
    // Reversible — unset rootEstimateId on roots where it equals _id
    // No-op for safety; production data is not expected.
  }

  isBootstrap(): boolean { return false; }
}
```

---

## 3. MongoDB / Mongoose Technical Notes

### 3.1 Mongoose vs native driver

**CLAIM: `trade-flow-api` uses the MongoDB native driver (`mongodb` 7.0.0), NOT Mongoose.** [VERIFIED: `trade-flow-api/package.json` lists both `mongodb: ^7.0.0` and `mongoose: ^9.1.5`, but every repository in `src/` imports from `mongodb`, not `mongoose`. Verified via `src/quote/repositories/quote.repository.ts` lines 11 and 5-6 (`MongoDbFetcher`/`MongoDbWriter` which wrap `db.collection<T>().{findOne,insertOne,findOneAndUpdate,...}` directly), and `trade-flow-api/CLAUDE.md` "mongodb — Native driver (not Mongoose ODM)".]

**Implication:** CONTEXT.md's reference to "mongoose 9.1.5" in the priority-2 research area is misleading — the planner must ignore Mongoose-specific API considerations. All index operations go through the native driver's `Collection.createIndex()`, `Collection.dropIndex()`, `Collection.indexes()`.

### 3.2 Partial unique index syntax

[VERIFIED: MongoDB docs] Partial unique indexes are supported in MongoDB 3.2+ and fully functional in 7.0. Native driver 6.x/7.x syntax:

```typescript
await collection.createIndex(
  { rootEstimateId: 1, isCurrent: 1 },
  { unique: true, partialFilterExpression: { isCurrent: true } }
);
```

**Critical constraint:** The `partialFilterExpression` supports only: equality (`{field: value}`), `$exists: true`, `$gt/gte/lt/lte/eq`, `$type`, and top-level `$and`. `{isCurrent: true}` is a simple equality and is valid. `{deletedAt: null, isCurrent: true}` is also valid (two equality clauses implicitly ANDed).

**Source:** https://www.mongodb.com/docs/manual/core/index-partial/ (MEDIUM — docs URL referenced from memory; planner should verify against the current version before execution).

### 3.3 Duplicate-key error shape

**Error code:** `11000` (`DuplicateKey`). [VERIFIED: `trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts` line 1 — `const MONGO_DUPLICATE_KEY_ERROR_CODE = 11000`.]

**Existing utility:** `isMongoDuplicateKeyError(error: unknown): boolean` — lives in `src/core/errors/is-mongo-duplicate-key-error.utility.ts`. Type-safe guard that checks `error.code === 11000`.

**Phase 42 usage:** Any place the reviser inserts a new row (step 2) can race against a concurrent writer. If the step-1 atomic filter succeeds for two parallel calls (it can't, because `findOneAndUpdate` is atomic — but paranoid defense-in-depth), the step-2 insert would catch a `11000` error from the `{rootEstimateId, isCurrent}` partial unique index. The planner can use `isMongoDuplicateKeyError(e)` inside the reviser's catch block to distinguish "index conflict" from "everything else" and surface a cleaner 409. In practice, step 1 is atomic so step 2 never sees a concurrent race — but this belt-and-braces check is cheap.

### 3.4 `dropIndex` behavior

[CITED: https://www.mongodb.com/docs/manual/reference/method/db.collection.dropIndex/] `dropIndex("indexName")` throws `IndexNotFound` (error code 27) if the named index doesn't exist. Phase 42's drop-and-recreate logic MUST catch and ignore this (`IndexNotFound` specifically — don't swallow other errors).

**Index naming:** If you create an index without a `name` option, MongoDB derives the name from the field spec: `{businessId: 1, number: 1}` → `"businessId_1_number_1"`. This is stable across recreations. The Phase 42 plan can `dropIndex("businessId_1_number_1")` by exact name.

**Verification step for the plan:** Ship a `repository.spec.ts` test that asserts `collection.indexes()` output contains an entry matching:
```js
{ key: { rootEstimateId: 1, isCurrent: 1 }, unique: true, partialFilterExpression: { isCurrent: true }, name: "rootEstimateId_1_isCurrent_1" }
```

### 3.5 `findOneAndUpdate` — driver-level contract

[CITED: https://mongodb.github.io/node-mongodb-native/6.x/classes/Collection.html#findOneAndUpdate]

- Returns `WithId<T> | null` depending on driver version:
  - Driver ≥ 6.x: returns the document directly (not a wrapped result object).
  - Driver 5.x: returns `ModifyResult<T>` with `.value` field.
- **trade-flow-api uses mongodb 7.0.0** → direct document return. `MongoDbWriter.findOneAndUpdate` wraps this correctly (lines 37-52).
- `returnDocument`: `"before"` (pre-update state) or `"after"` (post-update state). Default in `MongoDbWriter` is `"after"`.
- Atomicity: **document-level atomic**. The filter-match-and-update is guaranteed to be a single atomic operation at the database level. No transaction needed.

**D-CONC-02 correctness proof:** With the filter `{_id, isCurrent: true, status: {$in: [...]}}` and update `{$set: {isCurrent: false}}`, two concurrent calls on the same `_id` will see:
- Call A: matches, flips `isCurrent: true → false`, returns doc.
- Call B: arrives after A's write; filter now sees `isCurrent: false`; no match; returns `null`.

The partial unique index is NOT actually needed to enforce this against two concurrent step-1 calls — `findOneAndUpdate` alone guarantees exactly-one winner. **The partial unique index enforces the invariant against a step-1-and-step-2 race** where writer A completes step 1, and writer B (who somehow has a stale `isCurrent: true` snapshot) attempts step 2 (`insertOne` with `isCurrent: true` in the same chain). The insert would collide with A's newly-inserted current row.

In Phase 42's actual flow, writer B cannot reach step 2 because step 1 gates step 2. So the partial unique index is a **secondary defense**, not the primary lock. That's fine — the index is cheap, correctness-preserving, and matches Pitfall 12 in `.planning/research/PITFALLS.md`.

### 3.6 Partial index coverage for `findCurrentInChainByRootId`

[CITED: https://www.mongodb.com/docs/manual/core/index-partial/] Partial indexes are **only used for queries whose filter is a subset of the partial filter expression**. The partial unique index on `{rootEstimateId:1, isCurrent:1}` with filter `{isCurrent: true}` will be used for queries like `{rootEstimateId: X, isCurrent: true}`. It will NOT be used for `{rootEstimateId: X}` alone (because that matches all rows in the chain, including non-current ones).

**Phase 42 implications:**
- `findCurrentInChainByRootId({rootEstimateId: X, isCurrent: true})` → served by partial unique index. Fast.
- `findRevisionsByRootId({rootEstimateId: X, deletedAt: null})` sorted by `revisionNumber` → NOT served by partial unique index. Needs the separate `{rootEstimateId: 1, revisionNumber: 1}` index from CONTEXT's specifics line 289. Phase 42 MUST create this index.

---

## 4. Atomic Reviser Flow

End-to-end trace of `EstimateReviser.revise(authUser, id)`:

### Step 0 — Load & authorize
```typescript
const old = await this.estimateRetriever.findByIdOrFail(authUser, id);
// findByIdOrFail per D-DET-01 resolves non-current to current — which is correct here:
// if the trader POSTs to a non-current id, the auto-resolve returns the chain's current row,
// then the step-1 filter gates on isCurrent: true so the atomic check still holds.

const accessController = this.accessControllerFactory.create<IEstimateDto>(this.estimatePolicy);
accessController.canUpdate(authUser, old);
// Raises ForbiddenError if the user doesn't own the business.
```

### Step 1 — Atomic downgrade
```typescript
const allowedStatuses = [
  EstimateStatus.SENT,
  EstimateStatus.VIEWED,
  EstimateStatus.RESPONDED,
  EstimateStatus.SITE_VISIT_REQUESTED,
];
const downgraded = await this.estimateRepository.downgradeCurrent(old.id, allowedStatuses);
if (!downgraded) {
  throw new ConflictError(
    ErrorCodes.ESTIMATE_REVISION_CONFLICT,
    "Estimate has already been revised or is no longer revisable",
  );
}
```

- `downgradeCurrent` wraps `findOneAndUpdate` on `estimates` collection with filter `{_id, isCurrent: true, status: {$in: allowedStatuses}}` and update `{$set: {isCurrent: false, ...updateAuditFields()}}`.
- Null return = 409. Non-null = proceed.
- **Post-step-1 state:** exactly zero rows in the chain have `isCurrent: true`. This is the "brief window" D-CONC-01 refers to.

### Step 2 — Insert new revision
```typescript
let insertedId: string;
try {
  const newDto: IEstimateDto = this.buildNewRevisionDto(old);  // private helper, see below
  const inserted = await this.estimateRepository.insertRevision(newDto);
  insertedId = inserted.id;
} catch (error) {
  // Step 2 failed; the old row is still flipped to false.
  await this.compensatingRestore(old.id);
  throw error instanceof ConflictError ? error : this.wrapAsInternalServerError(error);
}
```

- `buildNewRevisionDto(old)` creates a full `IEstimateDto` matching the field inventory in CONTEXT D-CONC-03:
  - Copy: businessId, customerId, jobId, number, title, estimateDate, notes, contingencyPercent, displayMode, uncertaintyReasons, uncertaintyNotes.
  - Override: status=DRAFT, isCurrent=true, revisionNumber=old.revisionNumber+1, parentEstimateId=old.id, rootEstimateId=old.rootEstimateId.
  - Reset to null: sentAt, firstViewedAt, respondedAt, convertedAt, convertedToQuoteId, declinedAt, lostAt, expiresAt, deletedAt, lastResponseType, lastResponseAt, lastResponseMessage, declineReason, siteVisitAvailability.
  - lineItems: empty `DtoCollection` (filled in step 3).
- `insertRevision(dto)` wraps `writer.insertOne` with entity mapping. Throws `MongoServerError` (`code: 11000`) on partial-unique-index collision — this should be impossible given step-1 atomicity but catch-and-wrap-to-ConflictError for defense-in-depth.

### Step 3 — Clone line items
```typescript
try {
  const sourceLineItems = await this.estimateLineItemRepository.findNonDeletedByEstimateId(old.id);
  await this.cloneLineItemsForRevision(sourceLineItems, insertedId);
} catch (error) {
  // Step 3 failed; compensate by soft-deleting the new row AND restoring the old row's isCurrent.
  await this.compensatingRollback(insertedId, old.id);
  throw this.wrapAsInternalServerError(error);
}
```

- `cloneLineItemsForRevision` implements D-LI-02:
  ```typescript
  private async cloneLineItemsForRevision(
    source: IEstimateLineItemDto[],
    newEstimateId: string,
  ): Promise<void> {
    const roots = source.filter((li) => li.parentLineItemId === null || li.parentLineItemId === undefined);
    const children = source.filter((li) => li.parentLineItemId !== null && li.parentLineItemId !== undefined);
    const parentRemap = new Map<string, string>();  // oldParentId → newParentId

    for (const root of roots) {
      const cloned = { ...root, id: undefined, estimateId: newEstimateId,
                       parentLineItemId: null, status: EstimateLineItemStatus.PENDING };
      const inserted = await this.estimateLineItemRepository.create(cloned);
      parentRemap.set(root.id, inserted.id);
    }

    for (const child of children) {
      const oldParent = child.parentLineItemId as string;
      const newParent = parentRemap.get(oldParent);
      if (!newParent) {
        // Orphaned child — data corruption. Log and skip, or throw. Planner's call.
        throw new InternalServerError(ErrorCodes.UNKNOWN_SERVER_ERROR,
          `Orphaned child line item ${child.id} — parent ${oldParent} not found in source set`);
      }
      const cloned = { ...child, id: undefined, estimateId: newEstimateId,
                       parentLineItemId: newParent, status: EstimateLineItemStatus.PENDING };
      await this.estimateLineItemRepository.create(cloned);
    }
  }
  ```

### Step 4 — Compensating rollback (failure paths)
```typescript
private async compensatingRestore(oldId: string): Promise<void> {
  try {
    await this.estimateRepository.restoreCurrent(oldId);
  } catch (rollbackError) {
    this.logger.error("COMPENSATING ROLLBACK FAILED — MANUAL INTERVENTION REQUIRED", {
      oldId,
      error: rollbackError,
    });
  }
}

private async compensatingRollback(newId: string, oldId: string): Promise<void> {
  try {
    // Soft-delete the new row first so the partial unique index is freed.
    await this.estimateRepository.update(/* DTO with status: DELETED, deletedAt: now, isCurrent: false */);
    // Then restore the old row's current flag.
    await this.estimateRepository.restoreCurrent(oldId);
  } catch (rollbackError) {
    this.logger.error("COMPENSATING ROLLBACK FAILED — MANUAL INTERVENTION REQUIRED", {
      newId, oldId, error: rollbackError,
    });
  }
}
```

- `restoreCurrent(id)` wraps `findOneAndUpdate({_id: id, isCurrent: false}, {$set: {isCurrent: true}})`. The `isCurrent: false` filter makes it idempotent — if some other code path has already restored the row, this is a no-op. If the filter matches nothing, we don't throw; we assume the chain is already correct.
- Rollback failure is logged at `error` level. The 500 surfaces to the client; the trader can re-try the POST.

### Step 5 — Return new revision DTO
```typescript
const newRevision = await this.estimateRetriever.findByIdOrFail(authUser, insertedId);
// findByIdOrFail re-computes totals + priceRange via EstimateTotalsCalculator.
return newRevision;
```

---

## 5. Phase 41 Interaction

### 5.1 Execution state

**[VERIFIED]** via `ls /Users/mattc/Documents/projects/agent/trade-flow-docs/trade-flow-api/src/estimate/` returns empty (no directory). `.planning/STATE.md` line 6 says `stopped_at: Phase 42 context gathered` and line 31 says `Status: Ready to execute`. Phase 41 has 8 PLAN files written (41-01 through 41-08) but no execution. **Phase 41 is in "plans written, not executed" state.**

**Recommendation:** Path A — fold the `rootEstimateId` retrofit into Phase 41 PLAN-07 directly. Do NOT ship a migration in Phase 42.

### 5.2 Specific amendment needed in Phase 41 PLAN-07

**File:** `.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md`

**Current text (lines 125 and 185):** `rootEstimateId: null`

**Amended text:** `rootEstimateId: null // Phase 42 amendment: set to self._id after insert via EstimateCreator two-step`

**Amended behavior:** `EstimateCreator.create()` currently calls `authorizedCreatorFactory.create(policy).create(authUser, dtoWithNumber)` which internally calls `estimateRepository.create(dto)`. After this returns the created DTO with `id` populated, `EstimateCreator` issues a second repository call:

```typescript
await this.estimateRepository.setRootEstimateId(createdEstimate.id);
// Or: await this.estimateRepository.update({...createdEstimate, rootEstimateId: createdEstimate.id});
// Prefer a dedicated method so the grep assertions in PLAN-07 stay predictable.
```

Then return `{...createdEstimate, rootEstimateId: createdEstimate.id}` so the calculator-attached response reflects the retrofit.

**Grep assertions to update in PLAN-07:**
- Line 277: `grep -c "rootEstimateId: null"` — UPDATE to allow 1 match (initial DTO value) and verify the retrofit line exists separately.
- Line 234: `expect(result.rootEstimateId).toBeNull()` — UPDATE to `expect(result.rootEstimateId).toBe(result.id)`.

**New grep assertion:** `grep -c "setRootEstimateId\|rootEstimateId: createdEstimate" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1.

**Who amends PLAN-07?** Phase 42's planner, as part of Phase 42's task plan. This is acceptable per GSD workflow because Phase 41 has not executed and the plans are still in draft state. The change must be committed with a clear message crediting Phase 42 context D-CHAIN-03 as the rationale.

**Alternative path (Path B) — ship the migration:** If for any reason Path A is not taken (e.g., Phase 41 executes before Phase 42 plans), Phase 42 ships `20260412000000-backfill-estimate-root-id.migration.ts` as described in §2.10. This migration is idempotent and safe to run in any environment.

**Recommendation:** Path A. Simpler, cleaner, avoids the "migration not run" failure mode.

### 5.3 Phase 41 PLAN-05 index bootstrap

**File:** `.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md`

Phase 42 also needs to rework the `{businessId, number}` partial filter (D-CHAIN-06). There are two routes:

**Route 1:** Amend PLAN-05 so the Phase 41 `createIndexes` method already uses the final `{deletedAt: null, isCurrent: true}` filter. This eliminates the drop-and-recreate entirely.

**Route 2:** Leave PLAN-05 alone and do the drop-and-recreate in a Phase 42 `ensureIndexes` extension (or Phase 42 adds the two new indexes to the same method).

**Recommendation:** **Route 1.** It's cleaner, avoids the drop-and-recreate race window, and Phase 41 hasn't executed yet so there's no "migration over existing data" concern. Phase 42's amendment to PLAN-05 is: change the partialFilterExpression on line 193 from `{ deletedAt: null }` to `{ deletedAt: null, isCurrent: true }`. The Phase 41 list-query code also needs to filter on `isCurrent: true` for the unique index to be semantically complete — but per D-DET-02, the Phase 42 extension already amends `EstimateRetriever.findPaginatedByBusinessId` to filter on `isCurrent: true`. **Both amendments must land together** in Phase 42's task plan.

---

## 6. DI Binding Plan

### 6.1 Interface file

**Location:** `trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts`

```typescript
/**
 * Contract for cancelling pending follow-up jobs for a specific estimate revision.
 * Phase 42 ships a no-op binding; Phase 46 replaces with a BullMQ-backed implementation.
 *
 * Signature takes both estimateId and revisionNumber so implementations can compute
 * the deterministic BullMQ jobId pattern `estimate-followup:{estimateId}:{revisionNumber}:{step}`
 * without an extra database round-trip (see research/PITFALLS.md §Pitfall 2).
 */
export interface IEstimateFollowupCanceller {
  cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
}

/** DI provider token. String-based per NestJS custom-provider pattern. */
export const ESTIMATE_FOLLOWUP_CANCELLER = "ESTIMATE_FOLLOWUP_CANCELLER";
```

### 6.2 No-op implementation

**Location:** `trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts`

```typescript
import { Injectable } from "@nestjs/common";
import { AppLogger } from "@core/services/app-logger.service";
import { IEstimateFollowupCanceller } from "@estimate/services/estimate-followup-canceller.interface";

@Injectable()
export class NoopEstimateFollowupCanceller implements IEstimateFollowupCanceller {
  private readonly logger = new AppLogger(NoopEstimateFollowupCanceller.name);

  public async cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void> {
    this.logger.debug("Noop cancelAllFollowups called (Phase 46 will replace this binding)", {
      estimateId,
      revisionNumber,
    });
  }
}
```

### 6.3 Module wiring (Phase 42)

**File:** `trade-flow-api/src/estimate/estimate.module.ts` (edit Phase 41's file).

```typescript
import { ESTIMATE_FOLLOWUP_CANCELLER } from "@estimate/services/estimate-followup-canceller.interface";
import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";
import { EstimateReviser } from "@estimate/services/estimate-reviser.service";
// ...

@Module({
  imports: [/* unchanged from Phase 41 */],
  controllers: [EstimateController],
  providers: [
    // ...existing Phase 41 providers...
    EstimateReviser,
    NoopEstimateFollowupCanceller,
    {
      provide: ESTIMATE_FOLLOWUP_CANCELLER,
      useClass: NoopEstimateFollowupCanceller,
    },
  ],
  exports: [
    // ...existing Phase 41 exports...
    ESTIMATE_FOLLOWUP_CANCELLER,
  ],
})
export class EstimateModule {}
```

**Why include `NoopEstimateFollowupCanceller` as a separate provider entry?** Not strictly required — `{ provide: TOKEN, useClass: Class }` registers the class internally. But listing it explicitly makes the provider list self-documenting and makes `jest.spyOn(noopInstance, 'cancelAllFollowups')` easier in the spec file that verifies D-HOOK-03 ("reviser does NOT call the canceller").

### 6.4 Phase 46 shadow binding (for reference only — not shipped in Phase 42)

```typescript
// Phase 46's EstimateFollowupsModule
@Module({
  imports: [EstimateModule, BullMQModule],
  providers: [
    BullMQEstimateFollowupCanceller,
    {
      provide: ESTIMATE_FOLLOWUP_CANCELLER,
      useClass: BullMQEstimateFollowupCanceller,
    },
  ],
  exports: [ESTIMATE_FOLLOWUP_CANCELLER],
})
export class EstimateFollowupsModule {}
```

**`useExisting` vs `useClass`:** CONTEXT.md specifics line 349 suggests `useExisting`. `useExisting` is for aliasing an existing token to another; `useClass` instantiates the class. For Phase 46's case (where `BullMQEstimateFollowupCanceller` is a fresh class), `useClass` is correct. `useExisting` would be right only if Phase 46 wanted to alias one token to another existing one — not applicable here. The planner should use `useClass` in Phase 42 and note for Phase 46 that `useClass` is also the right choice there.

### 6.5 Injection at consumer site

```typescript
// EstimateReviser constructor — Phase 42 wires the injection but the reviser does NOT call the canceller.
// This is an intentional design decision per D-HOOK-03.
constructor(
  // ...other deps...
  @Inject(ESTIMATE_FOLLOWUP_CANCELLER)
  private readonly followupCanceller: IEstimateFollowupCanceller,
) {}
```

**Unused-injection warning:** ESLint's `@typescript-eslint/no-unused-vars` may flag `followupCanceller` as unused if the reviser never calls it. Two options: (a) add a test-level assertion that the injection resolves (proves the wiring works), (b) mark the field with a `// Phase 44 consumer` comment and rely on the fact that TypeScript private fields don't trigger the rule. Recommendation: (a) — ship a unit test that confirms `reviser['followupCanceller']` is defined and is an instance of `NoopEstimateFollowupCanceller` in Phase 42, so the DI wiring has test coverage even though no code calls the method.

---

## 7. Test Strategy

### 7.1 Test infrastructure reality check

**[VERIFIED: package.json grep]** The trade-flow-api test suite is:
- `jest` 30.2.0 + `ts-jest` + `@nestjs/testing` + `supertest`.
- Pure unit tests with `jest.fn()` mocks of `MongoDbFetcher`, `MongoDbWriter`, `MongoConnectionService`.
- **No `mongodb-memory-server` in `package.json`**. No existing spec file instantiates a real MongoDB connection.
- Example: `trade-flow-api/src/subscription/test/repositories/subscription.repository.spec.ts` lines 15-52 — mocks `createIndex: jest.fn()`, mocks the whole `db.collection()` return value.

**Implication for SC #4 ("two parallel revise calls → one success, one 409"):** This cannot be meaningfully tested against mocked primitives. Mocked `findOneAndUpdate` will return whatever `jest.fn()` is configured to return — it cannot model real atomicity. The assertion "exactly one row with `isCurrent: true`" requires a real index enforcing uniqueness.

### 7.2 Three options for SC #4

**Option A — Add `mongodb-memory-server`:**
```bash
cd trade-flow-api && npm install --save-dev mongodb-memory-server@^10
```
Write `estimate-reviser.service.spec.ts` with a `beforeAll` that starts a `MongoMemoryServer`, creates a `MongoClient`, wires it into a real `MongoDbWriter`/`MongoDbFetcher`/`MongoConnectionService`, and runs actual concurrent `Promise.allSettled([reviser.revise(id), reviser.revise(id)])`. Assert exactly one result is `status: "fulfilled"` and one is `status: "rejected"` with `reason instanceof ConflictError`.

**Pros:** Definitive verification of SC #4. Aligns with CONTEXT's Discretion section ("Mongo in-memory server is fine"). Future phases benefit from the infra.

**Cons:** New dev dependency. mongodb-memory-server pulls MongoDB binaries on install (~100 MB) and adds ~3-5 seconds to test startup. Requires a separate test file that doesn't use the pure-unit mocked pattern — this is a precedent-setting decision for the repo.

**Option B — Redesign SC #4 as a unit test with filter-inspection:**
Assert that `downgradeCurrent` is called with a filter containing `isCurrent: true` AND `status: {$in: [...]}`. Two parallel calls against a mocked `findOneAndUpdate` that returns the old row on the first call and null on the second will verify the service-level error handling. Does NOT verify the database-level atomicity guarantee — that's tested indirectly via the `repository.spec.ts` assertion that the partial unique index is present after bootstrap.

**Pros:** No new dependency. Stays within the existing pattern.

**Cons:** Cannot claim SC #4 is verified end-to-end. The planner must document this gap in VERIFICATION.md and rely on a manual smoke test (see Option C).

**Option C — Manual smoke test + unit filter-inspection:**
Ship Option B's unit test AND a smoke-test script (`trade-flow-api/scripts/smoke-revise-concurrent.ts`) that the trader runs against a local MongoDB before promoting to production. The smoke test hits a local API instance with two parallel `curl`s and asserts the HTTP response codes.

**Pros:** No new dependency. Gives real-world coverage.

**Cons:** Manual. Easy to forget. Not in CI.

**Recommendation: Option A.** SC #4 is a stated roadmap success criterion and the partial unique index is Phase 42's primary correctness guarantee. The upfront cost of adding `mongodb-memory-server` is a single `npm install` plus ~15 lines of setup code. The precedent it sets ("integration tests with real MongoDB when correctness depends on index enforcement") is good for future phases (41-05 repository tests would also benefit). **Flag to user for approval before executing Phase 42.**

### 7.3 Full test coverage map

Minimum test coverage required by Phase 42's plan checker, mapped to file:

| Spec file | Test cases | Type |
|-----------|-----------|------|
| `src/estimate/test/services/estimate-reviser.service.spec.ts` | Happy path SENT → Draft (unit); each allowed status reaches happy path (unit); each disallowed status throws 409 (unit); concurrent revise → exactly one success, one ConflictError (integration with mongodb-memory-server); compensating rollback restores old isCurrent on step-2 failure (unit with mocked repository); compensating rollback soft-deletes new row AND restores old on step-3 failure (unit); line-item clone preserves bundle parent/child with remapped parentLineItemId and PENDING status (unit); revision number monotonicity (unit); **D-HOOK-03 verification**: `jest.spyOn(noopCanceller, 'cancelAllFollowups')` is NOT called during a reviser invocation (unit). | unit + 1 integration |
| `src/estimate/test/services/estimate-deleter.service.spec.ts` | DELETE on non-root Draft atomically flips predecessor's isCurrent: true (unit); DELETE on non-root where predecessor is not the expected one throws ConflictError (unit — confirms guard); DELETE on root Draft (no parentEstimateId) behaves per Phase 41 semantics (regression unit) | unit |
| `src/estimate/test/services/estimate-retriever.service.spec.ts` | D-DET-01: `findByIdOrFail(nonCurrentId)` returns current row in chain (unit); D-DET-02: list query filters on isCurrent: true (unit); D-HIST-01: `findRevisionsByIdOrFail` returns oldest-first, excludes soft-deleted (unit); `findRevisionsByIdOrFail` resolves root id AND revision id to same chain (unit) | unit |
| `src/estimate/test/controllers/estimate.controller.spec.ts` | POST `/v1/estimates/:id/revisions` happy path → 201 (unit); POST → 409 on disallowed status (unit); POST → 403 on unauthorized (unit); GET `/v1/estimates/:id/revisions` happy path (unit); GET with root id vs revision id both resolve to chain (unit) | unit |
| `src/estimate/test/repositories/estimate.repository.spec.ts` | `downgradeCurrent` calls findOneAndUpdate with correct filter shape (unit); filter miss returns null, does not throw (unit); `insertRevision` maps DTO to entity and calls insertOne (unit); `restoreCurrent` filter is idempotent (unit); `findRevisionsByRootId` sorts by revisionNumber ascending (unit); `findCurrentInChainByRootId` filter uses isCurrent: true (unit); indexes present after onModuleInit (integration with mongodb-memory-server — optional, piggybacks on Option A infra) | unit + optional integration |
| `src/core/errors/test/conflict.error.spec.ts` | Constructs with code + details; getCode/getMessage/getDetails return expected values (unit) | unit |
| `src/core/errors/test/handle-error.utility.spec.ts` (extend Phase 41 spec) | `createHttpError(new ConflictError(...))` returns HttpException with status 409 (unit) | unit |

### 7.4 Line-item clone test

The bundle parent/child remapping is the trickiest logic. Suggested fixture:

```
Source estimate (old id = "old-1") has line items:
  - id="li-A", parentLineItemId=null, type=BUNDLE  (bundle parent)
  - id="li-B", parentLineItemId="li-A", type=STANDARD  (child)
  - id="li-C", parentLineItemId="li-A", type=STANDARD  (child)
  - id="li-D", parentLineItemId=null, type=STANDARD  (standalone)
  - id="li-E", parentLineItemId=null, type=STANDARD, status=DELETED  (should be skipped)

After clone to new estimate (id = "new-1"), expect:
  - 4 rows in estimatelineitems (not 5 — li-E skipped)
  - All 4 have estimateId="new-1"
  - All 4 have status=PENDING (even if source was APPROVED)
  - New bundle parent (mirrors li-A) has parentLineItemId=null, fresh _id (not "li-A")
  - New children (mirror li-B, li-C) have parentLineItemId = new bundle parent's _id
  - New standalone (mirrors li-D) has parentLineItemId=null
  - Pricing fields (unitPrice, lineTotal, etc.) are byte-identical to source
```

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | jest 30.2.0 with ts-jest transformer, @nestjs/testing, supertest |
| Config file | `trade-flow-api/jest.config.js` (inline in package.json — existing; no changes needed) |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern=estimate` |
| Full suite command | `cd trade-flow-api && npm run ci` |
| Estimated runtime | ~45 seconds full suite, ~5s per focused slice |
| **New dev dependency** | `mongodb-memory-server@^10` (REQUIRED for SC #4 concurrent-revise integration test — see §7.2 Option A. Planner MUST flag this in PLAN.md as a user-approval gate.) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| REV-01 | `IEstimateEntity` has `parentEstimateId`, `rootEstimateId`, `revisionNumber`, `isCurrent` fields (already added by Phase 41 PLAN-04; Phase 42 retrofits `rootEstimateId` default via amended PLAN-07) | typecheck + unit (creator) | `cd trade-flow-api && npm run typecheck && npm run test -- --testPathPattern=estimate-creator` | ❌ Wave 0 (Phase 41 scaffolding) |
| REV-02 | `POST /v1/estimates/:id/revisions` creates new revision under same E-YYYY-NNN, flips prev to non-current | unit (controller + reviser service) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` | ❌ Wave 0 (new file) |
| REV-03 | Partial unique index `{rootEstimateId, isCurrent}` enforces exactly one current per chain | integration (repository onModuleInit + concurrent reviser) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser.*concurrent` | ❌ Wave 0 (mongodb-memory-server setup) |
| REV-04 | `GET /v1/estimates/:id/revisions` returns chain oldest-first with full DTOs | unit (controller + retriever) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever` | ❌ Wave 0 (extension to Phase 41 retriever) |
| REV-05 | `IEstimateFollowupCanceller` injection resolves to `NoopEstimateFollowupCanceller` in Phase 42; reviser does NOT call it | unit (DI resolution + reviser spec with jest.spyOn assertion) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` | ❌ Wave 0 (new file) |
| SC #1 | `POST .../revisions` wires to reviser; response is 201 with new revision DTO | unit (controller spec) | same as REV-02 | ❌ Wave 0 |
| SC #2 | `GET .../revisions` returns chain oldest-first for both root id and revision id inputs | unit (controller + retriever spec) | same as REV-04 | ❌ Wave 0 |
| SC #3 (rewritten per D-HOOK-05) | Creating a revision atomically transitions the chain; trader sees only latest via D-DET-01 | unit (retriever spec for findByIdOrFail non-current handling + reviser integration) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever\|estimate-reviser` | ❌ Wave 0 |
| SC #4 | Concurrent revise: exactly one success, one 409, exactly one row with isCurrent: true | integration (mongodb-memory-server) | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser.*concurrent` | ❌ Wave 0 (blocked on Option A approval) |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npm run test -- --testPathPattern=<task-area>` (~5s)
- **Per wave merge:** `cd trade-flow-api && npm run test -- --testPathPattern=estimate` (~15s)
- **Phase gate:** `cd trade-flow-api && npm run ci` green before `/gsd-verify-work` (~45s)

### Wave 0 Gaps
- [ ] `src/core/errors/conflict.error.ts` — new error class
- [ ] `src/core/errors/test/conflict.error.spec.ts` — unit test
- [ ] `src/estimate/services/estimate-followup-canceller.interface.ts` — interface + token
- [ ] `src/estimate/services/noop-estimate-followup-canceller.service.ts` — default implementation
- [ ] `src/estimate/services/estimate-reviser.service.ts` — new service
- [ ] `src/estimate/test/services/estimate-reviser.service.spec.ts` — full coverage per §7.3
- [ ] `src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts` — minimal "is injectable" check
- [ ] `src/estimate/test/mocks/estimate-revision-mock-generator.ts` — fixture builder for revision chains
- [ ] **`mongodb-memory-server` dev dependency install** (gated on user approval per §7.2 Option A)
- [ ] `src/estimate/test/setup/mongo-memory.ts` — shared setup/teardown helper for integration tests (if Option A approved)

---

## 8. Pitfalls & Edge Cases

### Pitfall: Index drop-and-recreate race in multi-replica bootstrap
**What goes wrong:** Two app instances start simultaneously. Both call `dropIndex("businessId_1_number_1")`. Instance A drops, instance B gets `IndexNotFound` (catches it), instance A calls `createIndex(new filter)`, instance B calls `createIndex(new filter)` — second call no-ops because spec matches. **BUT:** between A's drop and B's re-create, any incoming WRITE on the `estimates` collection bypasses the uniqueness guarantee.
**Applicability to Trade Flow:** Railway deployment is currently single-instance per CLAUDE.md. So this race is not active. **But** the planner MUST document this as a known limitation in the PLAN.md "deployment notes" section so that if multi-replica is ever enabled, a migration-based drop-recreate is used instead of bootstrap-based.
**Mitigation:** Path B (Route 1 in §5.3) — amend Phase 41 PLAN-05 so the index is created with the final filter from the start. Then Phase 42 never needs to drop.

### Pitfall: Brief zero-current window during step 1 → step 2
**What goes wrong:** After `downgradeCurrent` succeeds (step 1) and before `insertRevision` completes (step 2), the chain has zero rows with `isCurrent: true`. If another read path queries `findCurrentInChainByRootId(rootId)` in that window, it finds nothing.
**Impact:** `EstimateRetriever.findByIdOrFail(nonCurrentId)` per D-DET-01 would fall through — the `target.isCurrent === false` branch looks up the "current row" and gets null. Must throw `ResourceNotFoundError` or retry.
**Mitigation:** In `findByIdOrFail`'s D-DET-01 branch, if the chain lookup returns null, fall back to returning the target row itself (with its `isCurrent: false` flag). The trader sees the pre-revise state briefly — not incorrect, just momentarily out of date. Document this in the spec as the "ephemeral zero-current window tolerance."

### Pitfall: Compensating rollback leaks a new row if step 3 fails
**What goes wrong:** Step 2 inserts new row with `isCurrent: true`. Step 3 (line-item clone) throws. Compensation runs: (a) soft-delete new row (`status: DELETED, isCurrent: false`), (b) restore old row (`isCurrent: true`). If compensation step (a) fails (transient Mongo error), the new row is still `isCurrent: true` in the chain. When compensation step (b) tries to restore the old row, the partial unique index REJECTS because two rows in the chain would have `isCurrent: true`.
**Impact:** Compensation step (b) throws. Final state: new row is still current (with no line items). Old row is non-current. Chain is usable but broken.
**Mitigation:** Ensure compensation ordering is (a) soft-delete new row FIRST, (b) restore old row SECOND. This is CONTEXT D-REV-06 and it's correct. The planner should add a defensive retry-once on compensation step (a) with a 100ms backoff. If it still fails, log at `error` and surface 500. The trader can manually recover (delete the broken new row and re-POST the revise).

### Pitfall: Parent-remap dictionary order-of-operations
**What goes wrong:** Line item clone loop processes all items in arbitrary order. A child is inserted before its parent. `parentRemap.get(oldParentId)` returns undefined. Insert fails or produces orphan.
**Mitigation:** §2.9's `cloneLineItemsForRevision` explicitly filters roots first (`parentLineItemId === null || undefined`), processes them, then processes children. Covered. Add a spec test with shuffled input order to confirm.

### Pitfall: `EstimateRetriever.findByIdOrFail` D-DET-01 infinite loop
**What goes wrong:** `findByIdOrFail(id)` loads row. Row has `isCurrent: false`. Service performs chain lookup `{rootEstimateId, isCurrent: true}` and finds another row. Returns it. **But what if that "current row" also has `isCurrent: false` due to a race?** Recursion or explicit null handling?
**Mitigation:** D-DET-01 is a single-step lookup, not recursive. Implementation:
```typescript
async findByIdOrFail(authUser, id): Promise<IEstimateDto> {
  const target = await this.repository.findByIdOrFail(id);
  accessController.canRead(authUser, target);
  if (target.isCurrent) return target;  // most common case
  const current = await this.repository.findCurrentInChainByRootId(target.rootEstimateId);
  if (!current) {
    // Zero-current window or chain corruption. Return the target row with its non-current flag.
    return target;
  }
  return current;
}
```
No recursion. Max 2 database reads. Safe.

### Pitfall: `EstimateDeleter` concurrency with a competing `EstimateReviser`
**What goes wrong:** Trader has Draft revision N open. In tab A they click "Delete" (rollback). In tab B they click "Revise again" on the root, expecting a new revision N+1. Race:
- Tab A step 1: `restoreCurrent(parentEstimateId)` → succeeds, parent row now `isCurrent: true`.
- Tab B step 1: `downgradeCurrent(parent.id, allowedStatuses)` — filter matches because parent is now `isCurrent: true`, flips to `false`.
- Tab A step 2: attempts to soft-delete the Draft revision N. Wait — parent is now `isCurrent: false` (tab B moved on). Tab A's delete proceeds as planned; the Draft is marked DELETED.
- Tab B step 2: inserts new revision N+1 (with `parentEstimateId = old Draft's id? Or the parent?`). This is where it gets confused.

**Analysis:** The race produces: old Draft marked DELETED (correct), new revision N+1 inserted with `parentEstimateId = old Draft._id` (incorrect — the parent chain now skips the deleted draft). But because D-HIST-03 excludes soft-deleted revisions from history, the UI won't show this ambiguity.
**Mitigation:** Phase 42 SHOULD add an integration test for this race, but SC #4 already covers the primary concurrent-revise case. This secondary case is acceptable if the planner documents it as "known-tolerated" in PLAN.md. Full fix would require a per-chain mutex which is explicitly out-of-scope.

### Pitfall: Phase 42 retrofit breaks Phase 41 tests
**What goes wrong:** Phase 41 PLAN-07's spec file asserts `expect(result.rootEstimateId).toBeNull()` on line 234 (`41-07-estimate-crud-services-PLAN.md`). Amending the retrofit breaks this assertion.
**Mitigation:** §5.2's amendment explicitly updates the grep assertions. The planner for Phase 42 MUST include the PLAN-07 amendment as a task in Phase 42's first plan (e.g., `42-01-amend-phase-41-plans-PLAN.md`). This amendment commits to `.planning/phases/41-estimate-module-crud-backend/` even though Phase 42 owns the commit.

### Pitfall: `parentEstimateId: null | undefined` handling
**What goes wrong:** Phase 41 PLAN-04 declares the entity field as `parentEstimateId?: ObjectId | null`. The `?` plus `| null` means three possible values: `undefined`, `null`, `ObjectId`. Phase 42 code that checks `parentEstimateId === null` misses the `undefined` case.
**Mitigation:** Use `parentEstimateId ?? null` coercion at the service boundary, OR normalize on read in `toDto`. Phase 41 repository `toDto` should produce exactly one representation. Add a lint-level check or at least a runtime assertion in `EstimateDeleter`'s rollback path.

---

## 9. Project Constraints (from CLAUDE.md)

The planner MUST honor every directive below. Every one is enforced by `npm run ci`.

- **Zero `any` usage** — use `unknown`, generics, type guards. `ConflictError` constructor signature must match the existing pattern (no `any`).
- **Avoid `as` type assertions** — prefer type guards. `isMongoDuplicateKeyError(error)` is the blessed pattern for narrowing Mongo errors.
- **No `eslint-disable`, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`** — fix root cause.
- **Path aliases** — every import in Phase 42 code uses `@core/*`, `@estimate/*`, `@auth/*`, etc. No relative imports across modules.
- **Service naming:** `EstimateReviser` (matches `-r` / `-er` convention — `EstimateCreator`, `EstimateRetriever`, `EstimateUpdater`, `EstimateDeleter`, `EstimateReviser`).
- **File naming:** `estimate-reviser.service.ts`, `conflict.error.ts`, `estimate-followup-canceller.interface.ts`, `noop-estimate-followup-canceller.service.ts`.
- **DateTime in DTOs, Date in entities.** Repository is the conversion boundary. Phase 42 extends `EstimateRepository` entity ↔ DTO conversions; use existing `toDateTime` / `toOptionalDateTime` utilities.
- **All endpoints use `@Controller("v1")`** class-level prefix + full path at handler level.
- **Response format:** `createResponse([item])`, `createHttpError(error)` in controller catch.
- **Test layout:** `src/estimate/test/services/`, `src/estimate/test/controllers/`, `src/estimate/test/repositories/`, `src/estimate/test/mocks/`.
- **Mock generator pattern:** `EstimateRevisionMockGenerator` with static methods accepting `overrides?: Partial<IEstimateDto>`.
- **Prefer self-documenting code over comments.** Acceptable comments: TODOs with context, non-obvious race warnings, runbook notes for compensating rollback.
- **CI gate:** `npm run ci` (tests + lint:check + format:check + typecheck) exits 0 before any merge/deploy.
- **Controller `@Req()` typing:** inline `{ user: IUserDto; params: { id: string } }`, never `Request` from express.
- **Print Width 125, Tab 2 spaces, double quotes, trailing commas all, LF line endings.**

---

## 10. Open Questions (RESOLVED 2026-04-11)

All six research questions have been answered by the user before planning. Plans 42-01..42-06 encode these resolutions.

### Q1: `mongodb-memory-server` approval — RESOLVED: **Rejected**
**Decision:** Do NOT add `mongodb-memory-server` as a dev dependency. SC #4 is verified via (a) unit test asserting the exact atomic filter shape used by `EstimateRepository.downgradeCurrent`, (b) unit test asserting reviser throws `ConflictError` when `downgradeCurrent` returns `null`, and (c) a new manual smoke procedure at `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` with docker-compose + parallel-curl steps. The existing test infra is pure unit tests with mocked MongoDB primitives and that pattern is preserved.

### Q2: Phase 41 amendment ownership — RESOLVED: **Amend in place**
**Decision:** Phase 42's plans edit `.planning/phases/41-estimate-module-crud-backend/41-05-*-PLAN.md` and `41-07-*-PLAN.md` directly (Phase 41 is still in the "plans written, not executed" state). Phase 42 does NOT ship a runtime migration and does NOT perform runtime index drop-and-recreate. Plan 42-01 owns the amendments and commits them under Phase 42 with rationale crediting D-CHAIN-03/04/06.

### Q3: Error code ordinal allocation — RESOLVED: **String-value-equals-key fallback**
**Decision:** Use `ESTIMATE_REVISION_CONFLICT = "ESTIMATE_REVISION_CONFLICT"` and `ESTIMATE_NOT_REVISABLE = "ESTIMATE_NOT_REVISABLE"` to sidestep coordination with Phase 41's ordinal namespace. The enum string value is what's persisted in API responses; ordinals are internal.

### Q4: `NoopEstimateFollowupCanceller` unit test — RESOLVED: **Ship minimal spec**
**Decision:** ~10-line spec asserting `@Injectable()` + `cancelAllFollowups` returns a resolved promise. Real test value lives in `estimate-reviser.service.spec.ts` via the `jest.spyOn(..., "cancelAllFollowups")` non-call assertion.

### Q5: `GET /v1/estimates/:id/revisions` with invalid id — RESOLVED: **404 ResourceNotFoundError**
**Decision:** Throw `ResourceNotFoundError` — same as `GET /v1/estimates/:id`. Achieved via the underlying `findByIdOrFail` call that precedes the chain lookup.

### Q6: Multiple in-flight Drafts per chain — RESOLVED: **Already blocked by D-REV-02**
**Decision:** No new code. D-REV-02 already blocks Draft-source revisions (returns 422 `ESTIMATE_NOT_REVISABLE`). Plan 42-04 includes a spec case "POST `/revisions` on a Draft source returns 422".

---

## 11. Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Adding `mongodb-memory-server` as dev dep won't break existing Jest setup | §7.2 | LOW — it's a standard, widely-used package. Mitigated by running `npm run ci` immediately after install. |
| A2 | `createIndex` in mongodb 7.0.0 driver no-ops on identical spec | §2.4 | LOW — this is documented driver behavior. Planner can verify with a one-off script before relying on it. |
| A3 | `dropIndex` throws error code 27 with `codeName: "IndexNotFound"` when the index is absent | §2.4 | MEDIUM — the exact `codeName` string may vary by driver version. Planner should verify by running a throw-catch in a spec and logging `e.codeName`. |
| A4 | Phase 41 PLAN-01 reserves `ESTIMATE_0..ESTIMATE_N` error-code slots | §2.3, Q3 | MEDIUM — didn't verify PLAN-01 directly. If PLAN-01 uses a different namespace, Phase 42 must match. |
| A5 | The existing `AppLogger` supports structured object logging via `logger.error("msg", {context})` | §4 step 4, Pitfall `EstimateRetriever` | LOW — this is the pattern in `subscription.repository.ts` and other existing services. |
| A6 | NestJS `OnModuleInit` fires AFTER the module graph is fully instantiated but BEFORE the HTTP server starts accepting requests | §2.4 | LOW — documented NestJS lifecycle behavior; confirmed by `SubscriptionRepository` precedent. |
| A7 | Partial unique index with `{isCurrent: true}` filter will prevent exactly the insertion race described in SC #4 | §3.5 | LOW — this is the canonical use case for partial unique indexes per MongoDB docs. |
| A8 | `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` string-token pattern works with NestJS 11 | §6 | LOW — standard NestJS feature documented at https://docs.nestjs.com/fundamentals/custom-providers. |
| A9 | Phase 41 PLAN-08 will export `EstimateController`, `EstimateRetriever`, `EstimateDeleter` with the method signatures Phase 42 needs | §2.7, §4 | HIGH — if Phase 41's final plans diverge, Phase 42's amendments to retriever/deleter won't have the expected method surface. Mitigation: planner reads ALL Phase 41 PLANs before writing Phase 42 PLANs. |
| A10 | Phase 41 `EstimateRepository` extends `MongoConnectionService` (to get `connection.getDb()` for index bootstrap) — same as `SubscriptionRepository` | §2.4 | MEDIUM — Phase 41 PLAN-05 line 147 says "createIndexes method (called from constructor or module bootstrap — mirror the quote pattern) creates the three indexes". The quote repository does NOT call `connection.getDb()`. Planner must ensure Phase 41's `EstimateRepository` constructor includes `MongoConnectionService` as a dependency. This may require an amendment to PLAN-05 as part of Phase 42's scope. |

---

## 12. Sources

### Primary (HIGH confidence — verified in repo this session)
- `trade-flow-api/src/core/errors/invalid-request.error.ts` (lines 4-31)
- `trade-flow-api/src/core/errors/forbidden-error.error.ts` (lines 4-31)
- `trade-flow-api/src/core/errors/resource-not-found.error.ts` (lines 4-31)
- `trade-flow-api/src/core/errors/internal-server-error.error.ts` (lines 4-31)
- `trade-flow-api/src/core/errors/handle-error.utility.ts` (lines 9-86)
- `trade-flow-api/src/core/errors/error-codes.enum.ts` (lines 1-69)
- `trade-flow-api/src/core/errors/errors-map.constant.ts` (lines 1-278)
- `trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts` (lines 1-13)
- `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` (lines 1-80)
- `trade-flow-api/src/quote/quote.module.ts` (lines 1-73)
- `trade-flow-api/src/quote/controllers/quote.controller.ts` (lines 1-120)
- `trade-flow-api/src/quote/repositories/quote.repository.ts` (lines 1-150)
- `trade-flow-api/src/quote/repositories/quote-line-item.repository.ts` (lines 1-165)
- `trade-flow-api/src/quote/services/quote-creator.service.ts` (lines 1-58)
- `trade-flow-api/src/quote/services/quote-updater.service.ts` (lines 1-166)
- `trade-flow-api/src/subscription/repositories/subscription.repository.ts` (lines 1-150)
- `trade-flow-api/src/migration/interfaces/migration.interface.ts` (lines 1-26)
- `trade-flow-api/src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts` (lines 1-82)
- `trade-flow-api/src/migration/controllers/migration.controller.ts` (lines 42-50)
- `trade-flow-api/CLAUDE.md` (full contents — NestJS conventions, naming standards, date/time rules)
- `.planning/phases/42-revisions/42-CONTEXT.md` (full — all 35 decisions)
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` (first 150 lines — entity shape, collection names, Phase 41 indexes)
- `.planning/phases/41-estimate-module-crud-backend/41-04-estimate-scaffold-PLAN.md` (grep: entity field inventory)
- `.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` (grep: createIndex pattern)
- `.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` (grep: creator defaults + grep assertions)
- `.planning/REQUIREMENTS.md` (REV-01..REV-05 lines 33-37)
- `.planning/STATE.md` (lines 1-50 — phase status, roadmap summary)
- `.planning/ROADMAP.md` (Phase 42 entry lines 157-166)
- `.planning/research/PITFALLS.md` (Pitfalls 2, 3, 12)
- `.planning/phases/41-estimate-module-crud-backend/41-VALIDATION.md` (lines 1-50 — shape template)

### Secondary (MEDIUM confidence — verified against documentation, not run locally)
- MongoDB partial index filter restrictions (equality, `$exists`, `$gt/gte/lt/lte/eq`, `$type`, `$and`) — [CITED: https://www.mongodb.com/docs/manual/core/index-partial/]
- `findOneAndUpdate` returns direct document in native driver 6.x+ — [CITED: https://mongodb.github.io/node-mongodb-native/6.x/classes/Collection.html#findOneAndUpdate]
- NestJS `useClass` vs `useExisting` semantics — [CITED: https://docs.nestjs.com/fundamentals/custom-providers]

### Tertiary (LOW confidence — assumed from training, flagged in §11)
- Exact `codeName` string for IndexNotFound error — see A3.
- `OnModuleInit` lifecycle ordering relative to HTTP server startup — see A6.

---

## Metadata

**Confidence breakdown:**
- Code mirrors (§2): HIGH — every referenced file read in session with line numbers
- MongoDB technical notes (§3): HIGH for driver version + duplicate-key utility (verified in-repo); MEDIUM for partial-index docs (cited but not locally re-verified)
- Atomic reviser flow (§4): HIGH — constructed from CONTEXT decisions + verified driver primitive
- Phase 41 interaction (§5): HIGH — verified empty `src/estimate/` and read PLAN files directly
- DI binding (§6): HIGH for wiring shape; MEDIUM for `useExisting` alternative (not used in-repo yet)
- Test strategy (§7): HIGH for infra audit (verified no `mongodb-memory-server` in package.json); MEDIUM for Option A recommendation (depends on user approval)
- Pitfalls (§8): MEDIUM — analytical reasoning over verified data
- Project constraints (§9): HIGH — direct from CLAUDE.md

**Research date:** 2026-04-11
**Valid until:** 2026-05-11 (30 days — stable tech stack, no fast-moving dependencies)

---

## RESEARCH COMPLETE

**Phase:** 42 - Revisions
**Confidence:** HIGH for locked decisions and code mirrors; MEDIUM for concurrency test strategy (blocked on Q1 user approval)

### Key Findings
- **Phase 41 not yet executed** — `src/estimate/` directory empty. Retrofit path A (fold into Phase 41 PLAN-05 and PLAN-07 amendments) is clearly the right choice over shipping a migration.
- **No `mongodb-memory-server` in repo** — CONTEXT says "existing pattern"; research confirms it is NOT. Q1 asks user for approval to add it (recommended Option A).
- **No `insertMany` primitive in `MongoDbWriter`** — line-item clone uses sequential `insertOne` calls; no core edit required.
- **`QuoteUpdater` does NOT use atomic status-gated `findOneAndUpdate`** — Phase 42's `downgradeCurrent` is a novel pattern in the codebase. Needs extra test rigor.
- **`ConflictError` shape is byte-for-byte identical** to the 4 existing error classes; no base class extraction in scope.
- **Index bootstrap precedent is `SubscriptionRepository.OnModuleInit`**, NOT `QuoteRepository` (which does no index creation).
- **`createHttpError` extension** adds one branch between `InvalidRequestError` and `ResourceNotFoundError` branches, using `Logger.warn`.
- **D-HOOK-03 means the injected `ESTIMATE_FOLLOWUP_CANCELLER` is unused code in Phase 42** — the spec test MUST use `jest.spyOn` to prove it's not called, AND must prove the DI resolution works.
- **Line-item clone order:** bundle parents inserted first, children second — `Map<oldParentId, newParentId>` — DELETED items skipped, status forced to PENDING on every row.

### File Created
`/Users/mattc/Documents/projects/agent/trade-flow-docs/.planning/phases/42-revisions/42-RESEARCH.md`

### Confidence Assessment
| Area | Level | Reason |
|------|-------|--------|
| Locked decisions (CONTEXT summary) | HIGH | Copied verbatim from CONTEXT.md |
| Code mirrors | HIGH | Every file read in-session with line numbers |
| Driver semantics | HIGH | Verified against `mongo-db-writer.service.ts` + type signatures |
| Concurrency test strategy | MEDIUM | Blocks on Q1 user approval for mongodb-memory-server |
| Phase 41 retrofit path | HIGH | Verified Phase 41 is unexecuted; plan files examined |
| Error codes ordinal allocation | MEDIUM | Flagged in Q3 — coordination risk with Phase 41 PLAN-01 |
| DI string-token pattern | MEDIUM | First use in codebase; standard NestJS per docs |

### Open Questions (require user input before planner proceeds)
1. **Q1** (CRITICAL): Approve `mongodb-memory-server` dev dependency for SC #4 integration test?
2. **Q2**: Allow Phase 42 planner to amend Phase 41 PLAN-05 and PLAN-07 files as part of Phase 42's task plan?
3. **Q3**: Error-code ordinals — coordinate with Phase 41 PLAN-01 before final assignment.
4. **Q4**: Is a minimal `NoopEstimateFollowupCanceller` spec required?
5. **Q5**: 404 vs empty array when `GET /v1/estimates/:id/revisions` target id doesn't exist?
6. **Q6**: No action — the "can't revise a Draft" constraint is correct by construction; just add a regression spec.

### Ready for Planning
Research complete. Planner MAY proceed to write PLANs, **with the explicit caveats that Q1 and Q2 need user sign-off before the SC #4 integration test infrastructure is scaffolded and before the Phase 41 PLAN amendments land.** All other decisions are locked.
