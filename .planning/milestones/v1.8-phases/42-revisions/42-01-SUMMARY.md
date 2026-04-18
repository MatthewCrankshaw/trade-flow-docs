---
phase: 42
plan: 01
subsystem: planning-docs
tags: [docs, planning, phase-41-retrofit, roadmap-rewrite]
requires:
  - Phase 42 D-CHAIN-03 (root rows carry rootEstimateId === self.id)
  - Phase 42 D-CHAIN-04 (two-step insert + updateOne root write)
  - Phase 42 D-CHAIN-05 (partial unique on rootEstimateId, isCurrent)
  - Phase 42 D-CHAIN-06 (tighten businessId/number partial filter to include isCurrent)
  - Phase 42 D-CHAIN-07 (index bootstrap precedent switches from QuoteRepository to SubscriptionRepository)
  - Phase 42 D-HOOK-05 (SC #3 rewrite: follow-up cancellation deferred to Phase 44 Send path)
provides:
  - Phase 41 PLAN-05 with final Phase 42 index topology declared upfront
  - Phase 41 PLAN-07 with EstimateCreator two-step root write specified upfront
  - ROADMAP.md Phase 42 SC #3 reflecting D-HOOK-05 wording
  - v1.8-ROADMAP.md Phase 42 SC #3 mirroring ROADMAP.md
affects:
  - All downstream Phase 42 plans (42-02 through 42-06) can assume the Phase 41 plans already declare the final index topology and the final creator behavior
tech-stack:
  added: []
  patterns:
    - "Fold-forward retrofit: amend unexecuted earlier-phase plans in-place under the later phase's commit history"
key-files:
  created:
    - .planning/phases/42-revisions/42-01-SUMMARY.md
  modified:
    - .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
    - .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
    - .planning/ROADMAP.md
    - .planning/milestones/v1.8-ROADMAP.md
decisions:
  - "Fold Phase 42's index topology and creator retrofit into Phase 41's unexecuted plans instead of writing a runtime migration"
  - "Switch index bootstrap precedent from QuoteRepository (has no indexes) to SubscriptionRepository (OnModuleInit + ensureIndexes)"
  - "Rewrite Phase 42 SC #3 wording to reflect that follow-up cancellation lives in the Phase 44 Send path, not in the revise flow"
metrics:
  duration: "~10min"
  completed: 2026-04-11
  tasks: 4
  files: 4
---

# Phase 42 Plan 01: Phase 41 Amendments and Roadmap Rewrite Summary

**One-liner:** Fold Phase 42's D-CHAIN-03/04/05/06/07 index and creator retrofits into Phase 41's still-unexecuted plans, and rewrite Phase 42 SC #3 per D-HOOK-05 in both roadmap files. Zero source code touched.

## What Changed

Four files amended under Phase 42 commit history:

1. **`.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md`** — index bootstrap spec now declares all five indexes and uses `SubscriptionRepository` as the `OnModuleInit` precedent.
2. **`.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md`** — `EstimateCreator.create()` now specifies the two-step root write (`insertOne` → `setRootEstimateId(created.id)`).
3. **`.planning/ROADMAP.md`** — Phase 42 SC #3 rewritten verbatim per D-HOOK-05.
4. **`.planning/milestones/v1.8-ROADMAP.md`** — same rewrite mirrored at milestone level.

No source code in `trade-flow-api/` or `trade-flow-ui/` was touched (verified: `trade-flow-api/src/estimate/` does not yet exist).

## Task Execution

| Task | Name                                                        | Commit    | Files                                                                                        |
| ---- | ----------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------- |
| 1    | Amend Phase 41 PLAN-05 index bootstrap spec                 | `ac50c44` | `.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` |
| 2    | Amend Phase 41 PLAN-07 EstimateCreator root-write spec      | `0c92515` | `.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md`      |
| 3    | Rewrite ROADMAP.md Phase 42 SC #3 per D-HOOK-05             | `3bb2f85` | `.planning/ROADMAP.md`                                                                       |
| 4    | Mirror SC #3 rewrite in v1.8-ROADMAP.md                     | `2a0ede5` | `.planning/milestones/v1.8-ROADMAP.md`                                                       |

## Before / After Diffs

### 1. Phase 41 PLAN-05 (`createIndexes` block)

**Before (3 indexes, quote-mirroring language):**
```typescript
public async createIndexes(): Promise<void> {
  const db = await this.connection.getDb();
  const collection = db.collection(EstimateRepository.COLLECTION);
  await collection.createIndex({ businessId: 1, createdAt: -1 });
  await collection.createIndex({ jobId: 1, createdAt: -1 });
  await collection.createIndex(
    { businessId: 1, number: 1 },
    { unique: true, partialFilterExpression: { deletedAt: null } },
  );
}
```
Followed by: "(If `QuoteRepository` uses an `onModuleInit` hook or lifecycle method, mirror that.)"

**After (5 indexes, `SubscriptionRepository` precedent):**
```typescript
public async createIndexes(): Promise<void> {
  const db = await this.connection.getDb();
  const collection = db.collection(EstimateRepository.COLLECTION);
  await collection.createIndex({ businessId: 1, createdAt: -1 });
  await collection.createIndex({ jobId: 1, createdAt: -1 });
  // Partial unique on the "visible" number: a historical revision of the same chain can carry the
  // same E-YYYY-NNN because only the current non-deleted row is included in the partial filter.
  // Revised to add `isCurrent: true` per Phase 42 D-CHAIN-06 (folded forward because Phase 41 has not executed).
  await collection.createIndex(
    { businessId: 1, number: 1 },
    { unique: true, partialFilterExpression: { deletedAt: null, isCurrent: true } },
  );
  // Phase 42 D-CHAIN-05: exactly one current revision per chain.
  await collection.createIndex(
    { rootEstimateId: 1, isCurrent: 1 },
    { unique: true, partialFilterExpression: { isCurrent: true } },
  );
  // Phase 42: ordered chain lookup for the history endpoint.
  await collection.createIndex({ rootEstimateId: 1, revisionNumber: 1 });
}
```
Followed by an explicit note that `EstimateRepository` MUST implement `OnModuleInit` following the `SubscriptionRepository` precedent.

Task 2 narrative bullet rewritten from "three indexes per D-ENT-03" → "five indexes per Phase 42 D-CHAIN-03/05/06". Grep acceptance criteria rewritten from 2 lines to 7 lines covering all five indexes plus `OnModuleInit` assertions. Amendment history footer appended.

### 2. Phase 41 PLAN-07 (`EstimateCreator.create`)

**Before (one-step insert, rootEstimateId stays null):**
```typescript
const prepared: IEstimateDto = {
  ...dto,
  number,
  status: EstimateStatus.DRAFT,
  contingencyPercent: dto.contingencyPercent ?? 10,
  displayMode: dto.displayMode ?? EstimateDisplayMode.RANGE,
  estimateDate: dto.estimateDate ?? DateTime.now(),
  revisionNumber: 1,
  isCurrent: true,
  parentEstimateId: null,
  rootEstimateId: null,
  responseSummary: null,
};

const authorizedCreator = this.authorizedCreatorFactory.create(this.estimatePolicy);
const created = await authorizedCreator.create(
  authUser,
  prepared,
  (toPersist) => this.estimateRepository.create(toPersist),
);

return this.totalsCalculator.calculateTotals(created);
```
Spec assertion: `expect(result.rootEstimateId).toBeNull();`

**After (two-step insert + root-write, rootEstimateId === created.id):**
```typescript
const prepared: IEstimateDto = {
  ...dto,
  number,
  status: EstimateStatus.DRAFT,
  contingencyPercent: dto.contingencyPercent ?? 10,
  displayMode: dto.displayMode ?? EstimateDisplayMode.RANGE,
  estimateDate: dto.estimateDate ?? DateTime.now(),
  revisionNumber: 1,
  isCurrent: true,
  parentEstimateId: null,
  rootEstimateId: null, // overwritten by setRootEstimateId below
  responseSummary: null,
};

const authorizedCreator = this.authorizedCreatorFactory.create(this.estimatePolicy);
const created = await authorizedCreator.create(
  authUser,
  prepared,
  (toPersist) => this.estimateRepository.create(toPersist),
);

// Phase 42 D-CHAIN-03/04: root rows carry rootEstimateId === self.id.
// Two-write pattern (no transactions in this codebase): insert first, then set chain identity.
// The brief window where rootEstimateId is null is tolerable because no other read path
// depends on rootEstimateId until a revision is created (Phase 42+).
await this.estimateRepository.setRootEstimateId(created.id);
const withRoot: IEstimateDto = { ...created, rootEstimateId: created.id };

return this.totalsCalculator.calculateTotals(withRoot);
```
Spec assertions:
```typescript
expect(result.rootEstimateId).toBe(result.id);  // Phase 42 D-CHAIN-03: root is its own chain identity
expect(mockRepo.setRootEstimateId).toHaveBeenCalledWith(result.id);
```

Plus a new repository-contract note requiring `EstimateRepository.setRootEstimateId(id: string): Promise<void>`. Defaults narrative bullet rewritten to document the post-insert chain-identity write. Grep acceptance criteria extended by 2 lines (`setRootEstimateId` + `rootEstimateId: created.id|withRoot`). Amendment history footer appended.

### 3. ROADMAP.md — Phase 42 SC #3

**Before:**
> 3. Creating a revision atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision when loading the estimate detail page by root id.

**After (D-HOOK-05 wording verbatim):**
> 3. Creating a revision atomically transitions the chain: the new Draft revision becomes current, the previous revision keeps its existing status with `isCurrent: false`, and the trader sees only the latest revision when loading the estimate detail page by root id. Cancellation of the previous revision's pending follow-ups happens at the moment the new revision is actually Sent (Phase 44 owns the call via the `IEstimateFollowupCanceller` binding from Phase 42).

SC #1, #2, #4 untouched. Phase heading, Goal, Depends on, Requirements, and Plans list all untouched.

### 4. v1.8-ROADMAP.md — Phase 42 SC #3

Identical rewrite as ROADMAP.md. The single "pending follow-ups" sentence on line 127 (Phase 43's description, unrelated to Phase 42 SC #3) was left untouched by design — Task 4 only updated the Phase 42 block per plan instructions.

## Verification

All eleven plan-defined verification checks pass:

| # | Check                                                                             | Result |
| - | --------------------------------------------------------------------------------- | ------ |
| 1 | Old wording removed from both roadmaps                                            | PASS   |
| 2 | `atomically transitions the chain` present in ROADMAP.md                          | PASS   |
| 3 | `atomically transitions the chain` present in v1.8-ROADMAP.md                     | PASS   |
| 4 | `rootEstimateId: 1, isCurrent: 1` present in PLAN-05                              | PASS   |
| 5 | `rootEstimateId: 1, revisionNumber: 1` present in PLAN-05                         | PASS   |
| 6 | `deletedAt: null, isCurrent: true` present in PLAN-05                             | PASS   |
| 7 | `implements OnModuleInit` present in PLAN-05                                      | PASS   |
| 8 | `setRootEstimateId` present in PLAN-07                                            | PASS   |
| 9 | `expect(result.rootEstimateId).toBe(result.id)` present in PLAN-07                | PASS   |
| 10| `expect(result.rootEstimateId).toBeNull` removed from PLAN-07                     | PASS   |
| 11| All four tasks committed individually under `docs(42-01):` commits                | PASS   |

## No Source Code Touched

Confirmation: `trade-flow-api/src/estimate/` does not exist (Phase 41 has not executed). This plan amended planning artifacts only — zero `.ts`, `.tsx`, `.js`, or `.json` source files modified.

## Deviations from Plan

None — plan executed exactly as written. Tasks 1-4 were applied with the exact edit text from the plan's `<action>` blocks. No Rule 1/2/3 auto-fixes were needed.

## Commit History

```
2a0ede5 docs(42-01): mirror Phase 42 SC #3 rewrite in v1.8-ROADMAP.md per D-HOOK-05
3bb2f85 docs(42-01): rewrite ROADMAP.md Phase 42 SC #3 per D-HOOK-05
0c92515 docs(42-01): amend Phase 41 PLAN-07 EstimateCreator root-write retrofit per D-CHAIN-03/04
ac50c44 docs(42-01): amend Phase 41 PLAN-05 index bootstrap per D-CHAIN-03/05/06/07
```

Per the plan's output spec, each task was committed individually rather than as a single combined commit, so the commit history clearly records each amendment's rationale and scope.

## Self-Check: PASSED

- File `.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` exists — FOUND
- File `.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` exists — FOUND
- File `.planning/ROADMAP.md` exists — FOUND
- File `.planning/milestones/v1.8-ROADMAP.md` exists — FOUND
- Commit `ac50c44` — FOUND
- Commit `0c92515` — FOUND
- Commit `3bb2f85` — FOUND
- Commit `2a0ede5` — FOUND
- All 11 plan-defined grep checks pass — PASS
