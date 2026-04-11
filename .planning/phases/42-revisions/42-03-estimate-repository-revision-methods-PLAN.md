---
phase: 42
plan: 03
type: execute
wave: 2
depends_on: ["42-01", "42-02"]
files_modified:
  - trade-flow-api/src/estimate/repositories/estimate.repository.ts
  - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
  - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
  - trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
autonomous: true
requirements: [REV-01, REV-02, REV-03, REV-04]
estimate: 55min
tags: [repository, mongodb, indexes, phase-42]

must_haves:
  truths:
    - "`EstimateRepository.downgradeCurrent(id, allowedStatuses)` performs a single atomic `findOneAndUpdate` on filter `{_id, isCurrent: true, status: {$in: allowedStatuses}}` and returns `null` when nothing matched."
    - "`EstimateRepository.insertRevision(dto)` inserts a new row and returns the persisted DTO; throws on index collision (caught by caller)."
    - "`EstimateRepository.restoreCurrent(id)` issues `findOneAndUpdate` with filter `{_id, isCurrent: false}` update `{$set: {isCurrent: true}}` — idempotent no-op if the row is already current."
    - "`EstimateRepository.findRevisionsByRootId(rootEstimateId)` returns an array of DTOs filtered by `{rootEstimateId, deletedAt: null}` ordered by `{revisionNumber: 1}` ascending."
    - "`EstimateRepository.findCurrentInChainByRootId(rootEstimateId)` returns the single DTO matching `{rootEstimateId, isCurrent: true, deletedAt: null}` or `null` during the zero-current window."
    - "`EstimateLineItemRepository.findNonDeletedByEstimateId(estimateId)` returns all line items for an estimate excluding `status: DELETED`."
    - "`EstimateLineItemRepository.bulkInsertForRevision(entities)` inserts the given rows sequentially and returns the persisted DTOs (sequential insertOne calls, not a non-existent `insertMany`)."
    - "`createIndexes()` / `ensureIndexes()` declares all five indexes per Phase 41 PLAN-05 amendment (plan 42-01 precondition): `{businessId, createdAt: -1}`, `{jobId, createdAt: -1}`, `{businessId, number}` partial unique on `{deletedAt: null, isCurrent: true}`, `{rootEstimateId, isCurrent: 1}` partial unique on `{isCurrent: true}`, `{rootEstimateId, revisionNumber: 1}`."
  artifacts:
    - path: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      provides: "Five new methods + extended createIndexes"
      exports: ["EstimateRepository"]
    - path: "trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts"
      provides: "Two new methods for the reviser's line-item clone path"
      exports: ["EstimateLineItemRepository"]
  key_links:
    - from: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      to: "trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts"
      via: "`writer.findOneAndUpdate(...)` for atomic downgrade/restore"
      pattern: "findOneAndUpdate"
    - from: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      to: "trade-flow-api/src/core/services/mongo/mongo-connection.service.ts"
      via: "`onModuleInit()` calls `ensureIndexes()` which calls `connection.getDb()`"
      pattern: "onModuleInit"
---

<objective>
Add the five new repository methods Phase 42 needs on `EstimateRepository` and the two new methods on `EstimateLineItemRepository`, and finalize the Phase 42 index topology. This plan is pure repository-layer work — no services, no controllers, no error-throwing logic beyond what the existing `MongoDbWriter` primitives do. It depends on plan 42-01 (because plan 42-01 amended Phase 41 PLAN-05 to declare the correct indexes up-front) and plan 42-02 (because the repository may wrap `findOneAndUpdate` result inspection in a way that the reviser service will later catch as `ConflictError`, though the repository itself never throws `ConflictError` — it returns `null` instead).

Purpose: separate Mongo-touching code from business logic. The reviser (plan 42-04) composes these repository methods; the retriever/deleter extensions (plan 42-05) consume `findCurrentInChainByRootId` and `findRevisionsByRootId`. Isolating the repository in its own plan keeps the next wave's services focused on orchestration.

**CRITICAL PRECONDITION:** this plan assumes plan 42-01 has landed AND Phase 41's PLAN-05 / PLAN-07 amendments have been executed (i.e., `EstimateRepository` exists with the amended `createIndexes` declaration). If Phase 41 has not executed yet, plan 42-03 cannot run — the executor MUST run `/gsd-execute-phase 41` first, THEN return to Phase 42. This is flagged in the plan checker gate.

Output: four files (two repository files amended, two spec files amended), one commit, unit tests green.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/42-revisions/42-CONTEXT.md
@.planning/phases/42-revisions/42-RESEARCH.md
@.planning/phases/42-revisions/42-VALIDATION.md
@trade-flow-api/CLAUDE.md
@.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
@.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
@trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts
@trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts
@trade-flow-api/src/subscription/repositories/subscription.repository.ts
@trade-flow-api/src/quote/repositories/quote.repository.ts
@trade-flow-api/src/quote/repositories/quote-line-item.repository.ts
@trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts

<interfaces>
<!-- Target new EstimateRepository methods -->

```typescript
// EstimateRepository — new methods Phase 42 adds

public async downgradeCurrent(
  id: string,
  allowedSourceStatuses: EstimateStatus[],
): Promise<IEstimateDto | null>;

public async insertRevision(dto: IEstimateDto): Promise<IEstimateDto>;

public async restoreCurrent(id: string): Promise<void>;

public async findRevisionsByRootId(rootEstimateId: string): Promise<IEstimateDto[]>;

public async findCurrentInChainByRootId(rootEstimateId: string): Promise<IEstimateDto | null>;
```

<!-- Target new EstimateLineItemRepository methods -->

```typescript
// EstimateLineItemRepository — new methods Phase 42 adds

public async findNonDeletedByEstimateId(estimateId: string): Promise<IEstimateLineItemDto[]>;

public async bulkInsertForRevision(
  entities: IEstimateLineItemDto[],
): Promise<IEstimateLineItemDto[]>;
```

<!-- Existing MongoDbWriter shape (from trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts) -->

```typescript
async findOneAndUpdate<TSchema extends Record<string, unknown>>(
  collectionName: string,
  filter: Filter<TSchema>,
  update: UpdateFilter<TSchema>,
  options?: FindOneAndUpdateOptions,
): Promise<WithId<TSchema> | null>;

async insertOne<TSchema>(...): Promise<InsertOneResult<TSchema>>;
```

<!-- Precondition: Phase 41 PLAN-05 has been executed AND the indexes declared per the plan 42-01 amendment -->

Phase 41 PLAN-05 (post plan-42-01) createIndexes block (target state):

```typescript
public async createIndexes(): Promise<void> {
  const db = await this.connection.getDb();
  const collection = db.collection(EstimateRepository.COLLECTION);
  await collection.createIndex({ businessId: 1, createdAt: -1 });
  await collection.createIndex({ jobId: 1, createdAt: -1 });
  await collection.createIndex(
    { businessId: 1, number: 1 },
    { unique: true, partialFilterExpression: { deletedAt: null, isCurrent: true } },
  );
  await collection.createIndex(
    { rootEstimateId: 1, isCurrent: 1 },
    { unique: true, partialFilterExpression: { isCurrent: true } },
  );
  await collection.createIndex({ rootEstimateId: 1, revisionNumber: 1 });
}
```

If Phase 41 executed BEFORE plan 42-01 landed (unusual but possible), this plan's Task 1 also applies the index declaration edits directly to the source file (not just the plan doc). The executor verifies by reading the source file at the start of Task 1.
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Extend EstimateRepository with five revision methods + verify index declarations</name>
  <files>trade-flow-api/src/estimate/repositories/estimate.repository.ts, trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (the entire current file — this is the Phase-41-produced file; confirm the `onModuleInit()` + `createIndexes()` / `ensureIndexes()` shape, the existing method names like `findByIdOrFail`, `create`, `update`, and the `toDto(entity)` mapping helper)
    - trade-flow-api/src/estimate/repositories/estimate.repository.spec.ts OR trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts (whichever Phase 41 produced; confirm the mock shape of MongoDbWriter/MongoDbFetcher)
    - trade-flow-api/src/subscription/repositories/subscription.repository.ts (the blessed `OnModuleInit` + `ensureIndexes` precedent; the `updateOne` and `findOneAndUpdate` idioms)
    - trade-flow-api/src/quote/repositories/quote.repository.ts (the DTO/entity conversion pattern — `toDto`, `toEntity`, ObjectId handling)
    - trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts (the primitive signatures — particularly `findOneAndUpdate`'s return type `WithId<TSchema> | null`)
    - trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts (for `find` + `sort` signatures used by `findRevisionsByRootId`)
    - .planning/phases/42-revisions/42-RESEARCH.md §2.5 (`findOneAndUpdate` semantics — returnDocument default, null-on-miss behavior, atomicity guarantee)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-CONC-02 for the exact filter shape, D-CHAIN-05 for the index spec, D-HIST-01 for the history query)
  </read_first>
  <behavior>
    - **downgradeCurrent:** calls `writer.findOneAndUpdate("estimates", {_id: new ObjectId(id), isCurrent: true, status: {$in: allowedSourceStatuses}}, {$set: {isCurrent: false, updatedAt: new Date()}})`. Returns the `toDto(updated)` if the writer returned a doc; returns `null` if the writer returned null. Does NOT throw.
    - **insertRevision:** takes a fully-formed `IEstimateDto` (with `revisionNumber`, `parentEstimateId`, `rootEstimateId`, `isCurrent: true` already set by the caller), maps to entity via existing `toEntity`/equivalent, calls `writer.insertOne("estimates", entity)`, returns the inserted DTO with `id` populated.
    - **restoreCurrent:** calls `writer.findOneAndUpdate("estimates", {_id: new ObjectId(id), isCurrent: false}, {$set: {isCurrent: true, updatedAt: new Date()}})`. Returns `void`. Idempotent: a null result (no match) is NOT an error — it means some other code path already restored the row, which is fine.
    - **findRevisionsByRootId:** calls `fetcher.find("estimates", {rootEstimateId: new ObjectId(rootEstimateId), deletedAt: null}, {sort: {revisionNumber: 1}})` (or the equivalent based on the existing `MongoDbFetcher` primitive). Maps each entity to DTO. Returns an empty array if no matches.
    - **findCurrentInChainByRootId:** calls `fetcher.findOne("estimates", {rootEstimateId: new ObjectId(rootEstimateId), isCurrent: true, deletedAt: null})`. Returns `toDto(entity)` or `null`.
    - **Index topology check:** `createIndexes()` (or `ensureIndexes()` — whichever Phase 41 produced) declares the five indexes. If plan 42-01 already ran and Phase 41 PLAN-05 was amended, this was already done when Phase 41 executed. Task verifies the source file and ADDS the missing declarations if absent.
  </behavior>
  <action>
    **Step 0 — Read the current `estimate.repository.ts` and determine its exact shape.** Do NOT assume Phase 41 PLAN-05 was followed literally. Grep for:
    - `class EstimateRepository` — confirm class name and `implements OnModuleInit` (should be present post plan-42-01).
    - `createIndexes\|ensureIndexes` — confirm the method name.
    - `toDto\|mapToDto` — confirm the DTO mapping helper name.
    - `private readonly writer` / `private readonly fetcher` / `private readonly connection` — confirm constructor dependencies.
    - `findByIdOrFail` — confirm the existing pattern for `_id` to `ObjectId` conversion.

    Use the result to determine which helpers to reuse vs write from scratch. The method implementations below are templates — adjust parameter names and mapping helper names to match the actual Phase-41-produced file.

    **Step 1 — Verify `createIndexes()` / `ensureIndexes()` declares all five indexes.**

    Read the method body. If ALL FIVE of these `createIndex` calls are present, skip to Step 2:

    ```typescript
    await collection.createIndex({ businessId: 1, createdAt: -1 });
    await collection.createIndex({ jobId: 1, createdAt: -1 });
    await collection.createIndex(
      { businessId: 1, number: 1 },
      { unique: true, partialFilterExpression: { deletedAt: null, isCurrent: true } },
    );
    await collection.createIndex(
      { rootEstimateId: 1, isCurrent: 1 },
      { unique: true, partialFilterExpression: { isCurrent: true } },
    );
    await collection.createIndex({ rootEstimateId: 1, revisionNumber: 1 });
    ```

    If ANY are missing (e.g., the Phase 41 executor went off-script), ADD them in this task. Do NOT drop and recreate the `{businessId, number}` index at runtime — if the partial filter in the source doesn't match, the `createIndex` call will throw `IndexOptionsConflict`; in that case, the executor MUST log this as a blocker and stop, not ship a runtime drop-recreate (D-CHAIN-06/07 Phase 41 amendment is the only supported path per user decision Q2).

    **Step 2 — Add the five new methods to the class body.** Insert them grouped together after the existing Phase 41 methods (suggested: after `findByIdOrFail` and `findPaginatedByBusinessId`, before the private helpers). Use this reference implementation, adapted to match the file's existing conventions:

    ```typescript
    public async downgradeCurrent(
      id: string,
      allowedSourceStatuses: EstimateStatus[],
    ): Promise<IEstimateDto | null> {
      const updated = await this.writer.findOneAndUpdate<IEstimateEntity>(
        EstimateRepository.COLLECTION,
        {
          _id: new ObjectId(id),
          isCurrent: true,
          status: { $in: allowedSourceStatuses },
        },
        { $set: { isCurrent: false, updatedAt: new Date() } },
      );
      if (!updated) {
        return null;
      }
      return this.toDto(updated);
    }

    public async insertRevision(dto: IEstimateDto): Promise<IEstimateDto> {
      const entity = this.toEntity(dto);
      const result = await this.writer.insertOne<IEstimateEntity>(
        EstimateRepository.COLLECTION,
        entity,
      );
      const persisted: IEstimateEntity = { ...entity, _id: result.insertedId };
      return this.toDto(persisted);
    }

    public async restoreCurrent(id: string): Promise<void> {
      await this.writer.findOneAndUpdate<IEstimateEntity>(
        EstimateRepository.COLLECTION,
        {
          _id: new ObjectId(id),
          isCurrent: false,
        },
        { $set: { isCurrent: true, updatedAt: new Date() } },
      );
    }

    public async findRevisionsByRootId(rootEstimateId: string): Promise<IEstimateDto[]> {
      const db = await this.connection.getDb();
      const collection = db.collection<IEstimateEntity>(EstimateRepository.COLLECTION);
      const cursor = collection
        .find({ rootEstimateId: new ObjectId(rootEstimateId), deletedAt: null })
        .sort({ revisionNumber: 1 });
      const entities = await cursor.toArray();
      return entities.map((entity) => this.toDto(entity));
    }

    public async findCurrentInChainByRootId(rootEstimateId: string): Promise<IEstimateDto | null> {
      const db = await this.connection.getDb();
      const collection = db.collection<IEstimateEntity>(EstimateRepository.COLLECTION);
      const entity = await collection.findOne({
        rootEstimateId: new ObjectId(rootEstimateId),
        isCurrent: true,
        deletedAt: null,
      });
      if (!entity) {
        return null;
      }
      return this.toDto(entity);
    }
    ```

    **Convention notes:**
    - Uses the same `this.writer.findOneAndUpdate` primitive that `MongoDbWriter` exposes (verified in research §2.5).
    - Uses direct `db.collection(...).find(...)` for the two methods that need cursor-based sorting — if `MongoDbFetcher` exposes a `find` primitive with sort support, use that instead; mirror the existing `findPaginatedByBusinessId` Phase 41 pattern.
    - `toDto` and `toEntity` are the existing Phase 41 helpers. If the helpers are named differently (e.g., `mapToDto`), match the actual file.
    - `new ObjectId(id)` is the existing pattern in every repository in this codebase.
    - `updatedAt: new Date()` mirrors the existing `update` method's audit-field pattern.

    **Step 3 — Amend `estimate.repository.spec.ts`** to add unit coverage. Use the Phase 41 mock shape (`jest.fn()` on `writer.findOneAndUpdate`, `writer.insertOne`, `connection.getDb`, etc.). Add these test cases grouped in a new `describe("Phase 42 revision methods")` block:

    ```typescript
    describe("Phase 42 revision methods", () => {
      describe("downgradeCurrent", () => {
        it("calls findOneAndUpdate with filter {_id, isCurrent: true, status: {$in}}", async () => {
          mockWriter.findOneAndUpdate.mockResolvedValue(mockEntity);
          await repository.downgradeCurrent("507f1f77bcf86cd799439011", [
            EstimateStatus.SENT,
            EstimateStatus.VIEWED,
          ]);
          expect(mockWriter.findOneAndUpdate).toHaveBeenCalledWith(
            "estimates",
            expect.objectContaining({
              _id: expect.any(ObjectId),
              isCurrent: true,
              status: { $in: [EstimateStatus.SENT, EstimateStatus.VIEWED] },
            }),
            { $set: expect.objectContaining({ isCurrent: false }) },
          );
        });

        it("returns null when findOneAndUpdate returns null (filter miss)", async () => {
          mockWriter.findOneAndUpdate.mockResolvedValue(null);
          const result = await repository.downgradeCurrent("507f1f77bcf86cd799439011", [EstimateStatus.SENT]);
          expect(result).toBeNull();
        });

        it("returns the mapped DTO when findOneAndUpdate returns a doc", async () => {
          mockWriter.findOneAndUpdate.mockResolvedValue(mockEntity);
          const result = await repository.downgradeCurrent("507f1f77bcf86cd799439011", [EstimateStatus.SENT]);
          expect(result).not.toBeNull();
          expect(result?.id).toBe(mockEntity._id.toString());
        });
      });

      describe("insertRevision", () => {
        it("calls insertOne with the entity-mapped DTO and returns the persisted DTO", async () => {
          const newDto = EstimateMockGenerator.createEstimateDto({ revisionNumber: 2, isCurrent: true });
          const insertedId = new ObjectId();
          mockWriter.insertOne.mockResolvedValue({ acknowledged: true, insertedId });
          const result = await repository.insertRevision(newDto);
          expect(mockWriter.insertOne).toHaveBeenCalledWith("estimates", expect.any(Object));
          expect(result.id).toBe(insertedId.toString());
          expect(result.revisionNumber).toBe(2);
        });
      });

      describe("restoreCurrent", () => {
        it("calls findOneAndUpdate with filter {_id, isCurrent: false} and update {$set: {isCurrent: true}}", async () => {
          mockWriter.findOneAndUpdate.mockResolvedValue(mockEntity);
          await repository.restoreCurrent("507f1f77bcf86cd799439011");
          expect(mockWriter.findOneAndUpdate).toHaveBeenCalledWith(
            "estimates",
            expect.objectContaining({ _id: expect.any(ObjectId), isCurrent: false }),
            { $set: expect.objectContaining({ isCurrent: true }) },
          );
        });

        it("does not throw when findOneAndUpdate returns null (idempotent no-op)", async () => {
          mockWriter.findOneAndUpdate.mockResolvedValue(null);
          await expect(repository.restoreCurrent("507f1f77bcf86cd799439011")).resolves.toBeUndefined();
        });
      });

      describe("findRevisionsByRootId", () => {
        it("returns DTOs ordered by revisionNumber ascending", async () => {
          // Use whatever cursor-mock shape the Phase 41 spec uses; the assertion is that
          // the filter is {rootEstimateId, deletedAt: null} and the sort is {revisionNumber: 1}.
          // Mirror findPaginatedByBusinessId mock setup from the Phase 41 spec file.
        });

        it("excludes soft-deleted rows via deletedAt: null filter", async () => {
          // Assert the filter object passed to collection.find
        });
      });

      describe("findCurrentInChainByRootId", () => {
        it("returns the DTO matching {rootEstimateId, isCurrent: true, deletedAt: null}", async () => {
          // Mirror findByIdOrFail mock setup
        });

        it("returns null when no current row exists in the chain (zero-current window)", async () => {
          // Mock findOne to return null
        });
      });

      describe("index declarations (Phase 42 D-CHAIN-05/06)", () => {
        it("createIndexes declares partial unique on {rootEstimateId, isCurrent} with filter {isCurrent: true}", async () => {
          // Spy on collection.createIndex calls from the MongoConnectionService mock.
          // Assert one call matches { rootEstimateId: 1, isCurrent: 1 } with unique + partialFilterExpression: { isCurrent: true }.
        });

        it("createIndexes declares the businessId+number unique with filter {deletedAt: null, isCurrent: true}", async () => {
          // Assert the tightened partial filter.
        });

        it("createIndexes declares the sort index on {rootEstimateId, revisionNumber}", async () => {
          // Assert the non-unique sort index is created.
        });
      });
    });
    ```

    For the cursor-based tests (`findRevisionsByRootId`, `findCurrentInChainByRootId`, index declarations), mirror the Phase 41 spec's mock pattern exactly. If Phase 41's spec mocks `connection.getDb()` to return an object with `collection: jest.fn().mockReturnValue({find: jest.fn(), createIndex: jest.fn(), ...})`, reuse that structure.

    **Step 4 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate.repository
    ```

    **Prohibitions:**
    - No `any`, no `as` (prefer type guards or explicit interface parameters).
    - No `eslint-disable` / `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck`.
    - Path aliases only: `@core/*`, `@estimate/*`. No relative cross-module imports.
    - No `eslint-disable` in the spec file either.
    - Mock pattern MUST mirror Phase 41's spec file — do NOT introduce `mongodb-memory-server` or a real `MongoClient` (user-declined Q1).
  </action>
  <acceptance_criteria>
    - `grep -c "public async downgradeCurrent" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "public async insertRevision" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "public async restoreCurrent" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "public async findRevisionsByRootId" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "public async findCurrentInChainByRootId" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "isCurrent: true" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 4 (filters + index declarations)
    - `grep -c "status: { \\\$in: allowedSourceStatuses }" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "rootEstimateId: 1, isCurrent: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "rootEstimateId: 1, revisionNumber: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "deletedAt: null, isCurrent: true" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "createIndex" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 5
    - `grep -c " any\\| as " trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 0
    - `grep -c "Phase 42 revision methods" trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` returns 1
    - `grep -c "downgradeCurrent" trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` returns at least 3
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate.repository` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate.repository</automated>
  </verify>
  <done>EstimateRepository has all five new methods with matching spec coverage; index topology verified (5 createIndex calls with the Phase 42 partial filter expressions); slice is green.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Extend EstimateLineItemRepository with findNonDeletedByEstimateId + bulkInsertForRevision</name>
  <files>trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts, trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (the Phase-41-produced file; confirm existing method names like `create`, `findByEstimateId`, and the entity/DTO shape including `parentLineItemId`, `status`, `estimateId`, and whether `EstimateLineItemStatus.DELETED` exists in the enum)
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.spec.ts OR trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts (Phase 41 spec — confirm mock shape)
    - trade-flow-api/src/quote/repositories/quote-line-item.repository.ts (the 1:1 mirror for line-item repository idioms)
    - trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts (confirm `DELETED` value exists and what status values exist; if the enum is named differently, grep `trade-flow-api/src/estimate/enums/*.enum.ts` for line-item status)
    - .planning/phases/42-revisions/42-RESEARCH.md §2.9 (line-item bulk insert — MongoDbWriter has NO insertMany, use sequential insertOne)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-LI-01 forces status PENDING, D-LI-02 bundle parent/child remap)
  </read_first>
  <behavior>
    - **findNonDeletedByEstimateId:** queries `{estimateId: new ObjectId(estimateId), status: {$ne: EstimateLineItemStatus.DELETED}}` (or equivalent based on how the enum is defined), returns `IEstimateLineItemDto[]`.
    - **bulkInsertForRevision:** takes an array of DTOs (already prepared by the caller with new `estimateId`, remapped `parentLineItemId`, and forced `status: PENDING`), loops with `this.create(dto)` (or `this.writer.insertOne`), collects the results, returns them as an array. Sequential, not parallel. Non-transactional — the caller is responsible for compensating rollback on partial failure.
  </behavior>
  <action>
    **Step 0 — Read the current `estimate-line-item.repository.ts`** to determine:
    - The existing `create(dto)` method signature — use it if present, otherwise drop to `writer.insertOne` directly.
    - Whether there's already a `findByEstimateId` method — if so, `findNonDeletedByEstimateId` can delegate to it with a post-filter OR add a separate filter at the query level (prefer query-level).
    - The enum import path for `EstimateLineItemStatus` and the `DELETED` value.

    **Step 1 — Add `findNonDeletedByEstimateId`:**

    ```typescript
    public async findNonDeletedByEstimateId(estimateId: string): Promise<IEstimateLineItemDto[]> {
      const db = await this.connection.getDb();
      const collection = db.collection<IEstimateLineItemEntity>(EstimateLineItemRepository.COLLECTION);
      const entities = await collection
        .find({
          estimateId: new ObjectId(estimateId),
          status: { $ne: EstimateLineItemStatus.DELETED },
        })
        .toArray();
      return entities.map((entity) => this.toDto(entity));
    }
    ```

    If an existing `findByEstimateId` method already filters on `deletedAt: null` or similar, adapt the new method to use the same primitive. Exact query shape varies by Phase 41 convention.

    **Step 2 — Add `bulkInsertForRevision`:**

    ```typescript
    public async bulkInsertForRevision(
      entities: IEstimateLineItemDto[],
    ): Promise<IEstimateLineItemDto[]> {
      const inserted: IEstimateLineItemDto[] = [];
      for (const entity of entities) {
        const result = await this.create(entity);
        inserted.push(result);
      }
      return inserted;
    }
    ```

    **Why sequential:** `MongoDbWriter` has no `insertMany` primitive (research §2.9 verified). `Promise.all` would be marginally faster but complicates compensating rollback because partial failures leave inconsistent state. Sequential insertOne is simpler, the failure surface is smaller, and line-item counts per estimate are low (< 20 typically). Research explicitly recommends sequential.

    **No transaction.** The caller (plan 42-04's reviser) owns compensating rollback. This method just iterates and returns.

    **Step 3 — Amend `estimate-line-item.repository.spec.ts`** to add coverage in a new `describe` block:

    ```typescript
    describe("Phase 42 revision support methods", () => {
      describe("findNonDeletedByEstimateId", () => {
        it("queries with {estimateId, status: {$ne: DELETED}} filter", async () => {
          // Mock collection.find to return a cursor with .toArray
          // Assert the filter passed to find
        });

        it("returns mapped DTOs", async () => {
          // Mock cursor returns two entities; assert result array has two DTOs with expected ids
        });

        it("returns empty array when no matches", async () => {
          // Mock cursor returns []
        });
      });

      describe("bulkInsertForRevision", () => {
        it("calls create sequentially for each input dto and returns the collected results", async () => {
          const inputs = [
            EstimateLineItemMockGenerator.createLineItemDto({ status: EstimateLineItemStatus.PENDING }),
            EstimateLineItemMockGenerator.createLineItemDto({ status: EstimateLineItemStatus.PENDING }),
            EstimateLineItemMockGenerator.createLineItemDto({ status: EstimateLineItemStatus.PENDING }),
          ];
          // Mock this.create to return each input with a new id
          // Assert create was called 3 times in order
          // Assert returned array has 3 items
        });

        it("returns an empty array when given an empty input", async () => {
          const result = await repository.bulkInsertForRevision([]);
          expect(result).toEqual([]);
        });

        it("propagates the error if any insert throws (no partial rollback at repository level)", async () => {
          // Mock this.create to throw on the second call
          // Assert bulkInsertForRevision rejects with the thrown error
          // (The reviser service owns compensating rollback, not the repository)
        });
      });
    });
    ```

    **Step 4 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item.repository
    ```

    **Prohibitions:**
    - No `any`, no `as`.
    - No suppression comments.
    - No `Promise.all` parallelism — sequential `for...of` only.
    - No reference to `mongodb-memory-server`.
  </action>
  <acceptance_criteria>
    - `grep -c "public async findNonDeletedByEstimateId" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 1
    - `grep -c "public async bulkInsertForRevision" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 1
    - `grep -c "\\\$ne" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns at least 1 (the status ne-DELETED filter)
    - `grep -c "for (const " trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns at least 1 (sequential loop in bulkInsertForRevision)
    - `grep -c "Promise.all" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 0 (must be sequential, not parallel)
    - `grep -c " any\\| as " trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 0
    - `grep -c "Phase 42 revision support methods" trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts` returns 1
    - `grep -c "bulkInsertForRevision" trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts` returns at least 3
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item.repository` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate-line-item.repository</automated>
  </verify>
  <done>EstimateLineItemRepository has both new methods with sequential insertOne loop, spec coverage for the ne-DELETED filter and the sequential insert contract, slice green.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| repository → MongoDB | Filter correctness determines whether the atomic guarantee holds; a typo in `{_id, isCurrent: true, status: {$in}}` silently breaks D-CONC-02 |
| repository → service | `downgradeCurrent` returning `null` on miss is the contract the reviser depends on for its 409 handling |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-03-01 | Tampering | `downgradeCurrent` filter shape drift | mitigate | HIGH | Task 1 grep asserts `status: { $in: allowedSourceStatuses }` and `isCurrent: true` literally in the source; spec asserts `expect.objectContaining({isCurrent: true, status: {$in: [...]}})` |
| T-42-03-02 | Tampering | `downgradeCurrent` accidentally returns the old doc (before flip) and mis-leads the caller | mitigate | MEDIUM | `MongoDbWriter.findOneAndUpdate` defaults to `returnDocument: "after"` (research §2.5) — the returned doc has `isCurrent: false` which correctly indicates the flip succeeded; the reviser doesn't rely on the returned doc's field values for reconstructing the clone (it uses the pre-flip snapshot from Step 0) |
| T-42-03-03 | Elevation of privilege | `insertRevision` bypasses policy checks (service-layer concern) | accept | LOW | Repository layer does not enforce policies — by design. Plan 42-04's `EstimateReviser` enforces `EstimatePolicy.canUpdate` via `AccessControllerFactory` before calling `insertRevision` |
| T-42-03-04 | Denial of service | `bulkInsertForRevision` sequential loop blocks on a large line-item set | accept | LOW | Typical line-item counts are < 20; sequential insert is O(n) at ~5-10ms each = ~100-200ms worst case; well within HTTP timeout |
| T-42-03-05 | Information disclosure | `findRevisionsByRootId` returns DTOs from another business if the caller passes a crafted rootEstimateId | mitigate | HIGH | Policy enforcement is the caller's responsibility — plan 42-05's `EstimateRetriever.findRevisionsByIdOrFail` applies `EstimatePolicy.canRead` on the chain root BEFORE calling this repository method. Tested in plan 42-05 |
| T-42-03-06 | Tampering | Index topology not declared (e.g., `{rootEstimateId, isCurrent}` missing), causing SC #4 to silently not hold | mitigate | HIGH | Task 1 acceptance criteria grep-assert all five createIndex calls with exact-match strings for the partial filter expressions. Additionally, the `index declarations` spec describe block asserts the createIndex calls at unit-test level by spying on `collection.createIndex`. |
| T-42-03-07 | Tampering | `restoreCurrent` accidentally flips a row that wasn't previously current, violating chain invariants | mitigate | MEDIUM | The filter `{_id, isCurrent: false}` makes the operation idempotent and safe — if the row is already current (some other compensating write beat us), the filter misses and `findOneAndUpdate` returns `null`, which the method treats as a no-op |

**Verification commands:**
- `grep -q "rootEstimateId: 1, isCurrent: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` — partial unique index present.
- `grep -q "status: { \\\$in: allowedSourceStatuses }" trade-flow-api/src/estimate/repositories/estimate.repository.ts` — filter shape correct.
- `cd trade-flow-api && npm run test -- --testPathPattern=estimate.repository` — spec green.
- `cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item.repository` — line-item spec green.
</threat_model>

<verification>
After both tasks complete:

```bash
cd trade-flow-api && npm run test -- --testPathPattern="estimate.repository|estimate-line-item.repository"
cd trade-flow-api && npm run ci
```

Both must exit 0.
</verification>

<success_criteria>
- `EstimateRepository` has five new methods with Phase 42 D-CONC-02 / D-CHAIN-05 / D-HIST-01 semantics.
- `EstimateLineItemRepository` has two new methods with sequential-insert semantics.
- `createIndexes()` declares all five indexes verified by grep + spec.
- All new specs green; regression suite for Phase 41 tests still green.
- `npm run ci` exits 0.
- Single commit: `feat(42): add repository revision methods and line-item clone support (Phase 42 wave 2)`.
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-03-SUMMARY.md` documenting:
- Exact method signatures added (line numbers from the source file).
- The verified index declarations (the five `createIndex` calls with their exact partial filter expressions).
- Confirmation that sequential insertOne is used (not `Promise.all` or a non-existent `insertMany`).
- Confirmation that `downgradeCurrent` returns `null` on miss without throwing.
- Commit hash.
</output>
