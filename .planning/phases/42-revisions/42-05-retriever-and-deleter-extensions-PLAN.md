---
phase: 42
plan: 05
type: execute
wave: 3
depends_on: ["42-02", "42-03"]
files_modified:
  - trade-flow-api/src/estimate/services/estimate-retriever.service.ts
  - trade-flow-api/src/estimate/services/estimate-deleter.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts
autonomous: true
requirements: [REV-02, REV-04]
estimate: 55min
tags: [service, retriever, deleter, phase-42]

must_haves:
  truths:
    - "`EstimateRetriever.findByIdOrFail(authUser, id)` returns the current row in the chain when `:id` resolves to a non-current row (D-DET-01), with a single-step lookup — no recursion."
    - "When the chain has zero current rows (ephemeral window between downgrade and insert), `findByIdOrFail` returns the target row as-is (fallback)."
    - "`EstimateRetriever.findPaginatedByBusinessId` (or equivalent) filters on `isCurrent: true` at the Mongo query level (D-DET-02)."
    - "`EstimateRetriever.findRevisionsByIdOrFail(authUser, id)` resolves `:id` to its chain (root or revision), applies `EstimatePolicy.canRead` on the root, and returns the full chain as `IEstimateDto[]` ordered by `revisionNumber` ascending (D-HIST-01)."
    - "`findRevisionsByIdOrFail` throws `ResourceNotFoundError` when `:id` does not exist (matches Q5 user decision)."
    - "`EstimateDeleter.delete(authUser, id)` on a non-root revision (`parentEstimateId !== null`) atomically restores the predecessor's `isCurrent: true` BEFORE soft-deleting the target (D-REV-05/06)."
    - "Deleting a Draft non-root revision where the predecessor is no longer restorable (edge race) throws `ConflictError`."
    - "Deleting a root revision (parentEstimateId null) proceeds exactly as Phase 41 specced — no predecessor restoration path."
  artifacts:
    - path: "trade-flow-api/src/estimate/services/estimate-retriever.service.ts"
      provides: "Extended findByIdOrFail + list filter + new findRevisionsByIdOrFail"
    - path: "trade-flow-api/src/estimate/services/estimate-deleter.service.ts"
      provides: "Extended delete path with predecessor restoration for non-root revisions"
  key_links:
    - from: "trade-flow-api/src/estimate/services/estimate-retriever.service.ts"
      to: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      via: "`findCurrentInChainByRootId`, `findRevisionsByRootId` calls (plan 42-03 methods)"
      pattern: "findCurrentInChainByRootId|findRevisionsByRootId"
    - from: "trade-flow-api/src/estimate/services/estimate-deleter.service.ts"
      to: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      via: "`restoreCurrent` for predecessor (plan 42-03 method)"
      pattern: "restoreCurrent"
---

<objective>
Extend two existing Phase 41 services with Phase 42 behaviors: `EstimateRetriever` gains D-DET-01 (non-current target → chain current row), D-DET-02 (list filters on isCurrent), and `findRevisionsByIdOrFail` (history endpoint support); `EstimateDeleter` gains D-REV-05/06 (predecessor restoration on non-root Draft delete). This plan is isolated from plan 42-04 (reviser) — all file paths touched here are existing Phase 41 files, not new files, and there is zero overlap with plan 42-04's `estimate-reviser.service.ts`.

This plan runs in wave 3 alongside plan 42-04 because they both depend on plan 42-03 (repository methods) and plan 42-02 (ConflictError), and they touch disjoint file sets. Plan 42-06 (controller + module wiring) depends on both completing.

Purpose: make the Phase 41 retriever and deleter "revision-aware" without Phase 41 having known about revisions at all. The Phase 41 files are the Phase-41-produced originals; this plan amends them in place under Phase 42's commit. This mirrors the philosophical pattern of plan 42-01 (amend Phase 41 plan docs) but at the code level — Phase 41's plans are amended in place for cleanliness, not as a separate migration.

Output: four files amended (two services + two specs), one commit, full coverage of the five new behaviors.
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
@.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
@trade-flow-api/src/estimate/services/estimate-retriever.service.ts
@trade-flow-api/src/estimate/services/estimate-deleter.service.ts
@trade-flow-api/src/estimate/repositories/estimate.repository.ts
@trade-flow-api/src/core/errors/conflict.error.ts
@trade-flow-api/src/core/errors/error-codes.enum.ts
@trade-flow-api/src/core/errors/resource-not-found.error.ts

<interfaces>
<!-- Target extended EstimateRetriever surface -->

```typescript
// Extensions Phase 42 adds to the Phase-41-produced EstimateRetriever

// EXTENDED from Phase 41: now resolves non-current :id to the chain's current row
public async findByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto>;

// EXTENDED from Phase 41: list query filter now includes `isCurrent: true` at the repository level
// (the repository's findPaginatedByBusinessId already accepts a filter object; Phase 42 adds isCurrent: true)
public async findPaginatedByBusinessId(authUser: IUserDto, businessId: string, /* ... */): Promise<...>;

// NEW: history endpoint support
public async findRevisionsByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto[]>;
```

<!-- Target extended EstimateDeleter surface -->

```typescript
// EXTENDED from Phase 41: now restores predecessor before soft-deleting non-root Draft revisions
public async delete(authUser: IUserDto, id: string): Promise<void>;
```

<!-- Phase 42 plan-03 repository methods this plan calls -->

- `estimateRepository.findCurrentInChainByRootId(rootEstimateId): Promise<IEstimateDto | null>` — for D-DET-01
- `estimateRepository.findRevisionsByRootId(rootEstimateId): Promise<IEstimateDto[]>` — for findRevisionsByIdOrFail
- `estimateRepository.restoreCurrent(id): Promise<void>` — for D-REV-05 predecessor restoration

<!-- Phase 42 constraint: no recursion on D-DET-01 (Pitfall "infinite loop") -->

```typescript
// CORRECT: single-step lookup with zero-current-window fallback
async findByIdOrFail(authUser, id): Promise<IEstimateDto> {
  const target = await this.repository.findByIdOrFail(id);
  accessController.canRead(authUser, target);
  if (target.isCurrent) return target;
  const current = await this.repository.findCurrentInChainByRootId(target.rootEstimateId);
  if (!current) return target;  // ephemeral zero-current window; return target as-is
  return current;
}
// NO RECURSION. Max 2 DB reads.
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Extend EstimateRetriever with D-DET-01, D-DET-02, and findRevisionsByIdOrFail</name>
  <files>trade-flow-api/src/estimate/services/estimate-retriever.service.ts, trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/services/estimate-retriever.service.ts (entire file — Phase 41 produced; confirm current shape of `findByIdOrFail(authUser, id)` and `findPaginatedByBusinessId` or equivalent list method)
    - trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts (Phase 41 spec — confirm mock pattern)
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (confirm `findCurrentInChainByRootId` and `findRevisionsByRootId` signatures added by plan 42-03)
    - trade-flow-api/src/estimate/policies/estimate.policy.ts (confirm `canRead` method exists and its signature matches `accessController.canRead(authUser, resource)`)
    - trade-flow-api/src/core/factories/access-controller.factory.ts (confirm create() signature)
    - trade-flow-api/src/core/errors/resource-not-found.error.ts (for throwing on missing `:id`)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-DET-01, D-DET-02, D-HIST-01, D-HIST-02, D-HIST-03, D-HIST-04, D-HIST-05; user decision Q5 for 404 on invalid id)
    - .planning/phases/42-revisions/42-RESEARCH.md §8 Pitfall "findByIdOrFail D-DET-01 infinite loop" (the no-recursion guard)
  </read_first>
  <behavior>
    - Test 1: `findByIdOrFail` with a current target returns the target directly (regression — Phase 41 behavior preserved).
    - Test 2: `findByIdOrFail` with a non-current target performs a second lookup via `findCurrentInChainByRootId(target.rootEstimateId)` and returns the result.
    - Test 3: `findByIdOrFail` with a non-current target where `findCurrentInChainByRootId` returns null falls back to returning the target row itself (zero-current-window tolerance).
    - Test 4: `findByIdOrFail` never recurses — max 2 repository reads per call.
    - Test 5: `findPaginatedByBusinessId` (or equivalent list method) passes `isCurrent: true` into the repository filter.
    - Test 6: `findRevisionsByIdOrFail(authUser, rootId)` resolves a root id to the chain, applies `canRead` on the root, calls `findRevisionsByRootId(rootId)`, returns the array.
    - Test 7: `findRevisionsByIdOrFail(authUser, revisionId)` resolves a revision id to the same chain (via `target.rootEstimateId`), returns the same array.
    - Test 8: `findRevisionsByIdOrFail(authUser, nonExistentId)` throws `ResourceNotFoundError` (matches Q5 and Phase 41 `findByIdOrFail` behavior).
    - Test 9: `findRevisionsByIdOrFail` applies `EstimatePolicy.canRead` — a user from another business gets `ForbiddenError`.
    - Test 10: The returned array is ordered by `revisionNumber` ascending (verified by asserting the call was made to `findRevisionsByRootId` which is specced to sort).
    - Test 11: Soft-deleted revisions are excluded (delegated to the repository's `findRevisionsByRootId` filter).
  </behavior>
  <action>
    **Step 0 — Read the current `estimate-retriever.service.ts`** to determine:
    - The exact signature of the existing `findByIdOrFail` method (is it `findByIdOrFail(authUser, id)` or `findByIdOrFail(id)` followed by an access check outside, or a factored `accessController` wrapper?).
    - The exact shape of the list method — Phase 41 PLAN-05 names it `findPaginatedByBusinessId` but Phase 41 PLAN-07 may have renamed it. Confirm by reading the actual file.
    - Whether `accessControllerFactory` is injected into the retriever (likely yes — mirrors the quote retriever).

    **Step 1 — Extend `findByIdOrFail` for D-DET-01.**

    Find the existing method body. The Phase 41 shape likely looks like:

    ```typescript
    public async findByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto> {
      const estimate = await this.estimateRepository.findByIdOrFail(id);
      const accessController = this.accessControllerFactory.create(this.estimatePolicy);
      accessController.canRead(authUser, estimate);
      return this.totalsCalculator.calculateTotals(estimate);
    }
    ```

    Replace with:

    ```typescript
    public async findByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto> {
      const target = await this.estimateRepository.findByIdOrFail(id);
      const accessController = this.accessControllerFactory.create(this.estimatePolicy);
      accessController.canRead(authUser, target);

      // Phase 42 D-DET-01: if target is not the current revision, resolve to the chain's current row.
      // Single-step lookup — NO recursion. Max 2 DB reads.
      if (!target.isCurrent) {
        const current = await this.estimateRepository.findCurrentInChainByRootId(target.rootEstimateId);
        if (current) {
          // Re-check access on the current row in case the chain root's policy differs (defense in depth)
          accessController.canRead(authUser, current);
          return this.totalsCalculator.calculateTotals(current);
        }
        // Ephemeral zero-current window (between downgrade and insert in EstimateReviser).
        // Return the target as-is; its isCurrent=false flag is correct for the moment.
      }

      return this.totalsCalculator.calculateTotals(target);
    }
    ```

    **Critical:** NO recursion. No "if current is also not current, look up again" branch. The chain always has at most one `isCurrent: true` row at steady state; if the lookup returns null, we're in the ephemeral window and returning the target is correct.

    **Step 2 — D-DET-02 list filter (ownership delegated to plan 42-03).**

    The `isCurrent: true` filter on `EstimateRepository.findPaginatedByBusinessId` is added by plan 42-03 Task 1 Step 2.5 (repository-level edit). Plan 42-05 does NOT duplicate that edit — it only verifies the behavior at the retriever-spec level by stubbing the repository and asserting that the retriever's list method passes through the repository's result unchanged (no post-filtering in the service layer).

    The retriever service code at the list-method level is UNCHANGED by Phase 42. The retriever continues to call `estimateRepository.findPaginatedByBusinessId(...)` as Phase 41 produced it; the filter change lives inside the repository. This keeps `files_modified` for plan 42-05 free of `estimate.repository.ts` (no file overlap with plan 42-03).

    Add one spec case to the retriever spec in Step 4 (below) that stubs the repository to return a mixture of current and non-current rows, and asserts the retriever's list output preserves the mixture — proving the retriever does NOT post-filter (the filter is expected to happen at the repository/Mongo layer). See Step 4's `D-DET-02 list query returns only current rows` describe block for the exact shape.

    **Step 3 — Add `findRevisionsByIdOrFail` for D-HIST-01.**

    ```typescript
    public async findRevisionsByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto[]> {
      // Step 1: resolve :id to the chain by reading the target row.
      // The repository's findByIdOrFail throws ResourceNotFoundError when :id doesn't exist (Q5).
      const target = await this.estimateRepository.findByIdOrFail(id);

      // Step 2: apply canRead on the target. Business scoping flows through policy.
      const accessController = this.accessControllerFactory.create(this.estimatePolicy);
      accessController.canRead(authUser, target);

      // Step 3: fetch the chain ordered by revisionNumber ascending, soft-deleted excluded.
      // The repository method (plan 42-03) encapsulates both the filter and the sort.
      const chain = await this.estimateRepository.findRevisionsByRootId(target.rootEstimateId);

      // Step 4: attach totals + priceRange to each revision so the UI can render without extra calls.
      return chain.map((revision) => this.totalsCalculator.calculateTotals(revision));
    }
    ```

    **Step 4 — Amend the spec file.**

    Add these test cases in a new `describe("Phase 42 extensions")` block:

    ```typescript
    describe("Phase 42 extensions", () => {
      describe("findByIdOrFail (D-DET-01)", () => {
        it("returns target directly when target.isCurrent is true (regression)", async () => {
          const current = EstimateMockGenerator.createEstimateDto({ isCurrent: true });
          mockRepo.findByIdOrFail.mockResolvedValue(current);
          const result = await retriever.findByIdOrFail(authUser, current.id);
          expect(result.id).toBe(current.id);
          expect(mockRepo.findCurrentInChainByRootId).not.toHaveBeenCalled();
        });

        it("resolves non-current target to the chain's current row", async () => {
          const target = EstimateMockGenerator.createEstimateDto({
            id: "rev-2-id",
            isCurrent: false,
            rootEstimateId: "root-id",
          });
          const currentInChain = EstimateMockGenerator.createEstimateDto({
            id: "rev-3-id",
            isCurrent: true,
            rootEstimateId: "root-id",
          });
          mockRepo.findByIdOrFail.mockResolvedValue(target);
          mockRepo.findCurrentInChainByRootId.mockResolvedValue(currentInChain);
          const result = await retriever.findByIdOrFail(authUser, target.id);
          expect(result.id).toBe("rev-3-id");
          expect(mockRepo.findCurrentInChainByRootId).toHaveBeenCalledWith("root-id");
        });

        it("falls back to target when zero-current-window (findCurrentInChainByRootId returns null)", async () => {
          const target = EstimateMockGenerator.createEstimateDto({
            id: "rev-2-id",
            isCurrent: false,
            rootEstimateId: "root-id",
          });
          mockRepo.findByIdOrFail.mockResolvedValue(target);
          mockRepo.findCurrentInChainByRootId.mockResolvedValue(null);
          const result = await retriever.findByIdOrFail(authUser, target.id);
          expect(result.id).toBe("rev-2-id");
          expect(result.isCurrent).toBe(false);
        });

        it("never recurses — max 2 repository reads", async () => {
          const target = EstimateMockGenerator.createEstimateDto({ isCurrent: false });
          mockRepo.findByIdOrFail.mockResolvedValue(target);
          mockRepo.findCurrentInChainByRootId.mockResolvedValue(null);
          await retriever.findByIdOrFail(authUser, target.id);
          expect(mockRepo.findByIdOrFail).toHaveBeenCalledTimes(1);
          expect(mockRepo.findCurrentInChainByRootId).toHaveBeenCalledTimes(1);
        });

        it("re-applies canRead on the current row when resolving (defense in depth)", async () => {
          const target = EstimateMockGenerator.createEstimateDto({ isCurrent: false, rootEstimateId: "root-id" });
          const currentInChain = EstimateMockGenerator.createEstimateDto({ isCurrent: true, rootEstimateId: "root-id" });
          mockRepo.findByIdOrFail.mockResolvedValue(target);
          mockRepo.findCurrentInChainByRootId.mockResolvedValue(currentInChain);
          await retriever.findByIdOrFail(authUser, target.id);
          // Expect canRead called twice — once on target, once on current
          expect(mockAccessController.canRead).toHaveBeenCalledTimes(2);
        });
      });

      describe("D-DET-02 list query returns only current rows", () => {
        it("stub: repository returns current rows; retriever passes them through unchanged", async () => {
          // D-DET-02 list filter ownership lives at the repository level (plan 42-03).
          // This test verifies the retriever is a pass-through: when the repository returns
          // only current rows, the retriever returns them unchanged — it does NOT post-filter.
          const currentA = EstimateMockGenerator.createEstimateDto({ id: "a", isCurrent: true });
          const currentB = EstimateMockGenerator.createEstimateDto({ id: "b", isCurrent: true });
          mockRepo.findPaginatedByBusinessId.mockResolvedValue({
            items: [currentA, currentB],
            pagination: { limit: 10, offset: 0, total: 2 },
          });
          const result = await retriever.findPaginatedByBusinessId(authUser, "biz-id", undefined, 10, 0);
          expect(result.items).toHaveLength(2);
          expect(result.items.every((e) => e.isCurrent === true)).toBe(true);
        });

        it("regression: retriever does NOT post-filter — if the repository returned a mixture, the retriever would return the mixture (filter belongs to the repo layer)", async () => {
          // Contract documentation: if plan 42-03's repository filter is ever weakened,
          // the retriever will NOT silently compensate. The failure surface is the repo layer.
          const currentRow = EstimateMockGenerator.createEstimateDto({ id: "cur", isCurrent: true });
          const nonCurrentRow = EstimateMockGenerator.createEstimateDto({ id: "old", isCurrent: false });
          mockRepo.findPaginatedByBusinessId.mockResolvedValue({
            items: [currentRow, nonCurrentRow],
            pagination: { limit: 10, offset: 0, total: 2 },
          });
          const result = await retriever.findPaginatedByBusinessId(authUser, "biz-id", undefined, 10, 0);
          expect(result.items).toHaveLength(2);
        });
      });

      describe("findRevisionsByIdOrFail (D-HIST-01)", () => {
        it("resolves root id to the chain", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({ id: "root-id" });
          const chain = [root, EstimateRevisionMockGenerator.createChildRevision(root)];
          mockRepo.findByIdOrFail.mockResolvedValue(root);
          mockRepo.findRevisionsByRootId.mockResolvedValue(chain);
          const result = await retriever.findRevisionsByIdOrFail(authUser, "root-id");
          expect(result).toHaveLength(2);
          expect(mockRepo.findRevisionsByRootId).toHaveBeenCalledWith("root-id");
        });

        it("resolves revision id to the same chain via target.rootEstimateId", async () => {
          const child = EstimateRevisionMockGenerator.createChildRevision(
            EstimateRevisionMockGenerator.createRoot({ id: "root-id" }),
          );
          mockRepo.findByIdOrFail.mockResolvedValue(child);
          mockRepo.findRevisionsByRootId.mockResolvedValue([/* chain */]);
          await retriever.findRevisionsByIdOrFail(authUser, child.id);
          expect(mockRepo.findRevisionsByRootId).toHaveBeenCalledWith(child.rootEstimateId);
        });

        it("throws ResourceNotFoundError when :id does not exist (Q5)", async () => {
          mockRepo.findByIdOrFail.mockRejectedValue(
            new ResourceNotFoundError(ErrorCodes.ESTIMATE_NOT_FOUND, "not found"),
          );
          await expect(retriever.findRevisionsByIdOrFail(authUser, "missing"))
            .rejects.toBeInstanceOf(ResourceNotFoundError);
        });

        it("applies EstimatePolicy.canRead on the target", async () => {
          const target = EstimateRevisionMockGenerator.createRoot();
          mockRepo.findByIdOrFail.mockResolvedValue(target);
          mockRepo.findRevisionsByRootId.mockResolvedValue([target]);
          await retriever.findRevisionsByIdOrFail(authUser, target.id);
          expect(mockAccessController.canRead).toHaveBeenCalledWith(authUser, target);
        });
      });
    });
    ```

    If the repository filter edit is applied in this task, add spec coverage for the list method filter change as well (or confirm plan 42-03's repository spec already covers it).

    **Step 5 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever
    ```

    **Prohibitions:**
    - No `any`, no `as` (except spec-only `as unknown as jest.Mocked<...>` for DI mocking).
    - No `eslint-disable` / `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck`.
    - No recursion in `findByIdOrFail`. Flat two-step lookup only.
  </action>
  <acceptance_criteria>
    - `grep -c "public async findRevisionsByIdOrFail" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 1
    - `grep -c "findCurrentInChainByRootId" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "findRevisionsByRootId" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "target.isCurrent\\|if (!target.isCurrent)" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "D-DET-01\\|Phase 42 D-DET" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "findRevisionsByIdOrFail\\|findByIdOrFail" trade-flow-api/src/estimate/services/estimate-retriever.service.ts | head -n 1` returns at least 3 (both methods exist)
    - `grep -c "recursion\\|recursive\\|this.findByIdOrFail" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 0 (no recursion allowed)
    - `grep -c "Phase 42 extensions" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns 1
    - `grep -c "D-DET-01" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns at least 1
    - `grep -c "findRevisionsByIdOrFail\\|D-HIST-01" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns at least 2
    - `grep -c "zero-current-window\\|falls back to target" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns at least 1
    - `grep -c "D-DET-02 list query returns only current rows" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns 1 (retriever-level verification of the plan-42-03 repository filter)
    - `grep -c " any\\| as " trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate-retriever</automated>
  </verify>
  <done>EstimateRetriever.findByIdOrFail resolves non-current to chain current with zero-current-window fallback (no recursion); findRevisionsByIdOrFail exists with policy check and chain ordering via repository; 9+ new spec cases green.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Extend EstimateDeleter with D-REV-05/06 predecessor restoration for non-root revisions</name>
  <files>trade-flow-api/src/estimate/services/estimate-deleter.service.ts, trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/services/estimate-deleter.service.ts (entire Phase 41 file — confirm the existing `delete` method shape, likely `delete(authUser, id)` that calls `EstimateTransitionService.transition(..., DELETED)`)
    - trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts (Phase 41 spec — confirm mock pattern)
    - trade-flow-api/src/estimate/services/estimate-transition.service.ts (if it exists — confirm the transition method signature)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-REV-05, D-REV-06 — two-step guarded write for delete-rollback)
    - .planning/phases/42-revisions/42-RESEARCH.md §8 Pitfall "parentEstimateId: null | undefined handling" (coerce with ?? null)
    - trade-flow-api/src/core/errors/conflict.error.ts (the ConflictError class from plan 42-02)
  </read_first>
  <behavior>
    - Test 1: Deleting a ROOT Draft (parentEstimateId null) proceeds via the existing Phase 41 transition path — no predecessor restoration (regression).
    - Test 2: Deleting a non-root Draft revision (parentEstimateId non-null) first restores the predecessor's `isCurrent: true` THEN soft-deletes the target.
    - Test 3: Order of operations: `restoreCurrent` is called BEFORE the soft-delete transition (verified by call order via `jest.fn.mock.invocationCallOrder`).
    - Test 4: If `restoreCurrent` throws (unexpected), the delete does NOT proceed — the target is left untouched so the trader can retry.
    - Test 5: If the soft-delete transition throws AFTER `restoreCurrent` succeeded, the deleter attempts to UN-restore the predecessor (compensating rollback: `findOneAndUpdate({_id, isCurrent: true}, {$set: {isCurrent: false}})`) — OR logs and re-throws without compensation. D-REV-06 says "If the soft-delete fails, compensating update restores `isCurrent: false` on the predecessor." So: compensate.
    - Test 6: `parentEstimateId: null | undefined` coerce — test both shapes behave identically (root path in both cases).
  </behavior>
  <action>
    **Step 0 — Read the current `estimate-deleter.service.ts`** to determine:
    - The exact signature of `delete(authUser, id)`.
    - How it currently performs the soft-delete (likely via `EstimateTransitionService.transition(..., EstimateStatus.DELETED)` per Phase 41 PLAN-07).
    - Whether it injects `EstimateRepository` directly or only goes through the transition service.

    **Step 1 — Extend the delete flow.**

    Amend the `delete` method to branch on `parentEstimateId`:

    ```typescript
    public async delete(authUser: IUserDto, id: string): Promise<void> {
      const target = await this.estimateRetriever.findByIdOrFail(authUser, id);
      // findByIdOrFail already applies canRead via the retriever.
      // Now apply canDelete via the policy.
      const accessController = this.accessControllerFactory.create(this.estimatePolicy);
      accessController.canDelete(authUser, target);

      // Phase 41 rule (preserved): only Draft estimates can be deleted.
      if (target.status !== EstimateStatus.DRAFT) {
        throw new InvalidRequestError(
          ErrorCodes.ESTIMATE_NOT_EDITABLE,
          "Only Draft estimates can be deleted",
        );
      }

      // Phase 42 D-REV-05: non-root revisions restore predecessor BEFORE soft-delete (rollback path).
      // parentEstimateId !== null → this is a revision created by EstimateReviser.
      const parentId = target.parentEstimateId ?? null;
      if (parentId !== null) {
        await this.restorePredecessorAndSoftDelete(authUser, target, parentId);
        return;
      }

      // Phase 41 path: root delete via transition service.
      await this.estimateTransitionService.transition(
        authUser,
        target,
        EstimateStatus.DELETED,
      );
    }

    // Phase 42 D-REV-06: two-step guarded write.
    // Step 1: restore predecessor's isCurrent: true.
    // Step 2: soft-delete the target.
    // If step 2 fails after step 1 succeeded, compensate by re-downgrading the predecessor.
    private async restorePredecessorAndSoftDelete(
      authUser: IUserDto,
      target: IEstimateDto,
      predecessorId: string,
    ): Promise<void> {
      await this.estimateRepository.restoreCurrent(predecessorId);

      try {
        await this.estimateTransitionService.transition(
          authUser,
          target,
          EstimateStatus.DELETED,
        );
      } catch (softDeleteError) {
        // Compensate: re-downgrade the predecessor so the chain isn't left with two current rows.
        // Use the same allowed-statuses filter as the reviser (the predecessor was one of those before
        // the revision was created, so it should still be in one of them).
        this.logger.warn(
          `Soft-delete of non-root revision ${target.id} failed after restoring predecessor ${predecessorId}. Attempting compensation.`,
        );
        try {
          await this.estimateRepository.downgradeCurrent(predecessorId, [
            EstimateStatus.SENT,
            EstimateStatus.VIEWED,
            EstimateStatus.RESPONDED,
            EstimateStatus.SITE_VISIT_REQUESTED,
          ]);
        } catch (compensationError) {
          this.logger.error(
            `Compensation failed for delete-rollback on revision ${target.id}. Chain may be inconsistent. Predecessor: ${predecessorId}`,
            compensationError instanceof Error ? compensationError.stack : String(compensationError),
          );
        }
        throw softDeleteError;
      }
    }
    ```

    **Notes:**
    - The `parentEstimateId ?? null` coercion handles the `null | undefined` ambiguity (Pitfall from research §8).
    - The compensation uses `downgradeCurrent` (the same method plan 42-03 added for the reviser) because it's the exact inverse of `restoreCurrent` for this case.
    - The compensation only runs if the soft-delete failed AFTER the restore succeeded — otherwise there's nothing to compensate.
    - Injection: `EstimateRepository` must be available to the deleter. If Phase 41's deleter only injects `EstimateTransitionService`, add `EstimateRepository` to the constructor and register it in `EstimateModule` (module wiring is plan 42-06, but the constructor edit lives here — plan 42-06 verifies the module exports the repository to the deleter). Actually, `EstimateRepository` is ALREADY registered in `EstimateModule` (Phase 41 does that), so the deleter just needs to add the constructor parameter.
    - The `logger` — use `AppLogger` or `Logger` consistent with whatever Phase 41's deleter uses.

    **Step 2 — Amend the spec file.**

    Add a new `describe("Phase 42 non-root revision delete (D-REV-05)")` block:

    ```typescript
    describe("Phase 42 non-root revision delete (D-REV-05)", () => {
      describe("root Draft delete (regression: Phase 41 behavior)", () => {
        it("deletes a root Draft via transition service without restoring any predecessor", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({
            status: EstimateStatus.DRAFT,
            parentEstimateId: null,
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(root);
          await deleter.delete(authUser, root.id);
          expect(mockTransitionService.transition).toHaveBeenCalledWith(
            authUser,
            root,
            EstimateStatus.DELETED,
          );
          expect(mockRepo.restoreCurrent).not.toHaveBeenCalled();
        });

        it("handles parentEstimateId: undefined same as null", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({
            status: EstimateStatus.DRAFT,
            parentEstimateId: undefined as unknown as null,
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(root);
          await deleter.delete(authUser, root.id);
          expect(mockRepo.restoreCurrent).not.toHaveBeenCalled();
        });
      });

      describe("non-root Draft delete (D-REV-05)", () => {
        it("restores predecessor BEFORE soft-deleting the target", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          const revision = EstimateRevisionMockGenerator.createChildRevision(root, {
            status: EstimateStatus.DRAFT,
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(revision);
          mockRepo.restoreCurrent.mockResolvedValue(undefined);
          mockTransitionService.transition.mockResolvedValue(undefined);

          await deleter.delete(authUser, revision.id);

          expect(mockRepo.restoreCurrent).toHaveBeenCalledWith(root.id);
          expect(mockTransitionService.transition).toHaveBeenCalledWith(
            authUser,
            revision,
            EstimateStatus.DELETED,
          );
          // Assert call order
          const restoreOrder = mockRepo.restoreCurrent.mock.invocationCallOrder[0];
          const transitionOrder = mockTransitionService.transition.mock.invocationCallOrder[0];
          expect(restoreOrder).toBeLessThan(transitionOrder);
        });

        it("does not proceed with soft-delete if restoreCurrent throws", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          const revision = EstimateRevisionMockGenerator.createChildRevision(root, {
            status: EstimateStatus.DRAFT,
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(revision);
          mockRepo.restoreCurrent.mockRejectedValue(new Error("restore failed"));

          await expect(deleter.delete(authUser, revision.id)).rejects.toThrow();
          expect(mockTransitionService.transition).not.toHaveBeenCalled();
        });

        it("compensates (re-downgrades predecessor) when soft-delete fails after restore", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          const revision = EstimateRevisionMockGenerator.createChildRevision(root, {
            status: EstimateStatus.DRAFT,
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(revision);
          mockRepo.restoreCurrent.mockResolvedValue(undefined);
          mockTransitionService.transition.mockRejectedValue(new Error("soft-delete failed"));
          mockRepo.downgradeCurrent.mockResolvedValue(/* some DTO */);

          await expect(deleter.delete(authUser, revision.id)).rejects.toThrow("soft-delete failed");
          expect(mockRepo.downgradeCurrent).toHaveBeenCalledWith(
            root.id,
            expect.arrayContaining([
              EstimateStatus.SENT,
              EstimateStatus.VIEWED,
              EstimateStatus.RESPONDED,
              EstimateStatus.SITE_VISIT_REQUESTED,
            ]),
          );
        });

        it("rejects deletion of non-Draft status (regression: Phase 41 rule preserved)", async () => {
          const root = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          const revision = EstimateRevisionMockGenerator.createChildRevision(root, {
            status: EstimateStatus.SENT, // not Draft
          });
          mockRetriever.findByIdOrFail.mockResolvedValue(revision);
          await expect(deleter.delete(authUser, revision.id)).rejects.toThrow(/Draft/i);
        });
      });
    });
    ```

    **Step 3 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter
    ```

    **Prohibitions:**
    - No `any`, no `as` in service code.
    - `undefined as unknown as null` in spec is the one test-only allowed cast (to simulate the undefined vs null ambiguity).
    - No `eslint-disable` / `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck`.
  </action>
  <acceptance_criteria>
    - `grep -c "Phase 42 D-REV-05\\|D-REV-05\\|restorePredecessor" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "restoreCurrent" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "parentEstimateId" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "downgradeCurrent" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1 (compensation path)
    - `grep -c "parentEstimateId ?? null\\|parentEstimateId === null\\|parentEstimateId !== null" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "EstimateRepository" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 2 (import + constructor)
    - `grep -c " any\\| as " trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns 0
    - `grep -c "Phase 42 non-root revision delete" trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` returns 1
    - `grep -c "invocationCallOrder" trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` returns at least 1 (call order asserted)
    - `grep -c "restoreCurrent" trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` returns at least 3
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate-deleter</automated>
  </verify>
  <done>EstimateDeleter handles non-root revisions via restore-then-soft-delete with compensation on failure; root deletes regression-safe; 6+ spec cases green.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| HTTP-client → EstimateRetriever | Trader passes a crafted `:id` to read another business's chain via history endpoint |
| HTTP-client → EstimateDeleter | Trader uses DELETE-rollback path to manipulate another business's chain |
| service → repository | `findCurrentInChainByRootId` and `findRevisionsByRootId` do not enforce business scoping — caller must |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-05-01 | IDOR | Trader reads another business's revision chain via `findRevisionsByIdOrFail` | mitigate | HIGH | `findRevisionsByIdOrFail` applies `EstimatePolicy.canRead` on the target BEFORE calling `findRevisionsByRootId`. Policy enforces `businessId === authUser.businessId`. Tested by a policy-scoped spec case. |
| T-42-05-02 | IDOR | Trader reads another business's current row via D-DET-01 resolution (`findByIdOrFail(nonCurrentId)` resolves to chain current) | mitigate | HIGH | `findByIdOrFail` applies `canRead` on the target FIRST; if target is in another business, canRead throws ForbiddenError before the chain lookup runs. Additionally, `canRead` is re-applied on the `current` row returned by `findCurrentInChainByRootId` (defense in depth) — the spec includes an explicit "canRead called twice" assertion. |
| T-42-05-03 | IDOR / Elevation of privilege | Trader deletes another business's revision via the delete-rollback path | mitigate | HIGH | `EstimateDeleter.delete` applies `canDelete` on the target BEFORE any repository writes. If target is in another business, the policy throws ForbiddenError before `restoreCurrent` runs. |
| T-42-05-04 | Data integrity / Chain corruption | Delete-rollback soft-deletes the target but fails to restore predecessor, leaving chain with zero current rows | mitigate | MEDIUM | Two-step guarded write: `restoreCurrent(predecessorId)` runs FIRST. If it throws, the soft-delete does not proceed and the target is untouched. If `restoreCurrent` succeeds but the soft-delete fails, `downgradeCurrent(predecessorId, [...])` compensates. Compensation failure is logged at error level with runbook context. |
| T-42-05-05 | Denial of service | Trader creates 1000 revisions then deletes them all to stress the rollback path | accept | LOW | Rate limiting out of Phase 42 scope. Partial unique index still prevents duplicate-current state; worst case is ephemeral zero-current windows that self-correct on retry. |
| T-42-05-06 | Information disclosure | `findByIdOrFail` returns a non-current target's full DTO when the chain lookup returns null (zero-current window) — could this leak data from an in-progress revise by another tab? | accept | LOW | The target was already accessible to the trader (canRead passed). Returning the target itself during the brief window is not a new disclosure — the data is the same data the trader already had access to. The only cost is momentary "you see the old version" UX, which self-corrects on next refresh. |
| T-42-05-07 | Tampering / Recursion | A misconfigured chain with two non-current rows makes `findByIdOrFail` loop | mitigate | MEDIUM | Implementation is explicitly non-recursive — single `findCurrentInChainByRootId` call, single null-fallback. Spec asserts max 2 DB reads (`expect(...).toHaveBeenCalledTimes(1)`). No `while` or `this.findByIdOrFail` recursion in source (grep asserts). |

**Verification commands:**
- `! grep -q "this.findByIdOrFail" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` — no recursion.
- `cd trade-flow-api && npm run test -- --testPathPattern="estimate-retriever|estimate-deleter"` — specs green.
</threat_model>

<verification>
After both tasks complete:

```bash
cd trade-flow-api && npm run test -- --testPathPattern="estimate-retriever|estimate-deleter"
cd trade-flow-api && npm run ci
```

Both must exit 0. This plan runs in parallel with plan 42-04 (wave 3); both must be green before plan 42-06 (wave 4) can start.
</verification>

<success_criteria>
- `EstimateRetriever.findByIdOrFail` resolves non-current `:id` to chain current row without recursion.
- `EstimateRetriever.findRevisionsByIdOrFail` exists, applies policy, returns chain ordered ascending, throws 404 on missing `:id`.
- List query filters on `isCurrent: true` (via repository — either already in plan 42-03 or added here).
- `EstimateDeleter.delete` handles non-root revisions via restore-then-soft-delete with compensation on soft-delete failure.
- Root delete regression-safe.
- `npm run ci` exits 0.
- Single commit: `feat(42): extend EstimateRetriever + EstimateDeleter for revision-aware behavior (Phase 42 wave 3)`.
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-05-SUMMARY.md` documenting:
- The D-DET-01 flow and confirmation of no-recursion (grep result).
- The D-REV-05/06 flow with the exact call order (restoreCurrent → transition → optional compensate).
- The list filter location (which file was edited for `isCurrent: true`).
- Spec test counts and pass summary.
- Commit hash.
</output>
