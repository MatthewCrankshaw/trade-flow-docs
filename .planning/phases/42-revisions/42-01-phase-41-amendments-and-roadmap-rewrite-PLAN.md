---
phase: 42
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
  - .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
  - .planning/ROADMAP.md
  - .planning/milestones/v1.8-ROADMAP.md
autonomous: true
requirements: [REV-01, REV-03, REV-05]
estimate: 25min
tags: [docs, planning, phase-41-retrofit]

must_haves:
  truths:
    - "Phase 41 PLAN-05 declares all three revision-aware indexes from the start (no runtime drop-and-recreate in Phase 42)."
    - "Phase 41 PLAN-07 specifies root rows write `rootEstimateId = inserted._id` after insertOne (not the legacy `rootEstimateId: null`)."
    - "ROADMAP.md Phase 42 SC #3 wording matches the D-HOOK-05 rewrite."
    - ".planning/milestones/v1.8-ROADMAP.md mirrors the SC #3 rewrite."
  artifacts:
    - path: ".planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md"
      provides: "Amended index bootstrap spec for EstimateRepository.createIndexes"
      contains: "partialFilterExpression: { deletedAt: null, isCurrent: true }"
    - path: ".planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md"
      provides: "Amended EstimateCreator two-step root write"
      contains: "rootEstimateId: inserted._id"
    - path: ".planning/ROADMAP.md"
      provides: "Rewritten Phase 42 SC #3 per D-HOOK-05"
      contains: "Phase 44 owns the call via the `IEstimateFollowupCanceller`"
    - path: ".planning/milestones/v1.8-ROADMAP.md"
      provides: "Milestone-level Phase 42 SC #3 mirroring ROADMAP.md"
      contains: "Phase 44 owns the call via the `IEstimateFollowupCanceller`"
  key_links:
    - from: ".planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md"
      to: "future EstimateRepository.createIndexes() implementation"
      via: "spec text declaring the three indexes"
      pattern: "rootEstimateId: 1, isCurrent: 1"
    - from: ".planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md"
      to: "future EstimateCreator.create() implementation"
      via: "spec text declaring the two-step root write"
      pattern: "setRootEstimateId|rootEstimateId: inserted\\._id|rootEstimateId: created\\.id"
---

<objective>
Fold Phase 42's two hard prerequisites (D-CHAIN-03/04/05/06 index topology + EstimateCreator root-write retrofit) into Phase 41's still-unexecuted plans, and rewrite Phase 42 SC #3 per D-HOOK-05 in both roadmap documents. No runtime code is written in this plan — only planning artifacts and roadmap text. This plan is wave 1 because every downstream Phase 42 plan assumes the Phase 41 plans already declare the final index shape and the final creator behavior.

Purpose: avoid shipping a runtime drop-and-recreate migration and avoid a second-commit "fix rootEstimateId on existing roots" backfill. Phase 41 has not executed (verified: `trade-flow-api/src/estimate/` does not exist yet). The cleanest correction is to amend the Phase 41 plans in-place under Phase 42's commit history, crediting D-CHAIN-03 / D-CHAIN-06 / D-HOOK-05 as rationale.

Output: four files modified (two Phase 41 plan files + two roadmap files). Zero source code.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/ROADMAP.md
@.planning/milestones/v1.8-ROADMAP.md
@.planning/phases/42-revisions/42-CONTEXT.md
@.planning/phases/42-revisions/42-RESEARCH.md
@.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
@.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md

<interfaces>
<!-- No new code interfaces. This is a docs-only plan. -->
<!-- Key "contracts" are the spec-text locations that downstream Phase 42 plans assume. -->

Phase 41 PLAN-05 current createIndexes shape (the block that MUST be rewritten):

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

Phase 41 PLAN-07 current EstimateCreator defaults block (the lines that MUST be rewritten):

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
  rootEstimateId: null,          // ← Phase 42 amendment: set to created.id post-insert
  responseSummary: null,
};
```

ROADMAP.md Phase 42 SC #3 current wording (the line that MUST be rewritten):

> 3. Creating a revision atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision when loading the estimate detail page by root id.
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Amend Phase 41 PLAN-05 index bootstrap spec</name>
  <files>.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md</files>
  <read_first>
    - .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md (read the entire `createIndexes` code block around lines 185-195 AND every grep assertion in the acceptance_criteria that references `createIndex`, `partialFilterExpression`, or `deletedAt: null`)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-CHAIN-05, D-CHAIN-06, D-CHAIN-07, D-CHAIN-08)
    - .planning/phases/42-revisions/42-RESEARCH.md §2.4 (index bootstrap pattern)
    - .planning/phases/42-revisions/42-RESEARCH.md §5.3 (Route 1 — Phase 41 PLAN-05 amendment)
    - trade-flow-api/src/subscription/repositories/subscription.repository.ts (the blessed `OnModuleInit` + `ensureIndexes` precedent since `QuoteRepository` has no index method)
  </read_first>
  <action>
    Apply three edits to `.planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md`.

    **Edit A — Rewrite the `createIndexes` code sketch (currently around lines 185-195).**

    Find the exact block:
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

    Replace with:
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

    Also replace the sentence immediately above the block — "(If `QuoteRepository` uses an `onModuleInit` hook or lifecycle method, mirror that.)" — with the exact text:

    > NOTE (Phase 42 D-CHAIN-07): `QuoteRepository` does NOT create indexes. The blessed precedent is `SubscriptionRepository` which implements `OnModuleInit` and calls `ensureIndexes()` from the lifecycle hook. `EstimateRepository` MUST follow the same pattern: declare `implements OnModuleInit`, inject `MongoConnectionService` in the constructor, and call `await this.createIndexes()` from `onModuleInit()`. Do NOT create indexes in the constructor — the `MongoConnectionService` may not be ready yet.

    **Edit B — Amend the narrative around the `createIndexes` description (the bullet in the "Task 2 action" or equivalent section, originally around line 147).**

    Find the line:
    > `createIndexes()` method (called from constructor or module bootstrap — mirror the quote pattern) creates the three indexes from D-ENT-03.

    Replace with:
    > `createIndexes()` method (called from `onModuleInit()` — mirror the `SubscriptionRepository` pattern, NOT the quote pattern which has no index bootstrap) creates the **five** indexes: Phase 41's `{businessId, createdAt: -1}`, `{jobId, createdAt: -1}`, and `{businessId, number}` partial unique (now filtered on `{deletedAt: null, isCurrent: true}` per Phase 42 D-CHAIN-06), plus Phase 42's new `{rootEstimateId, isCurrent}` partial unique and `{rootEstimateId, revisionNumber}` sort index. `EstimateRepository` MUST declare `implements OnModuleInit` and inject `MongoConnectionService` via constructor. Rationale: folded forward from Phase 42 D-CHAIN-03/05/06 because Phase 41 has not executed — this avoids a runtime drop-and-recreate.

    **Edit C — Update the grep acceptance criteria for the repository task.**

    Find every acceptance_criteria bullet in Task 2 (the repository task) that references index assertions. The current set includes (verify by reading the file):
    - `grep -c "createIndex" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 3
    - `grep -c "partialFilterExpression" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1

    Replace those two lines with these five lines (keep the other criteria untouched):
    - `grep -c "createIndex" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 5
    - `grep -c "partialFilterExpression" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 2
    - `grep -c "rootEstimateId: 1, isCurrent: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "rootEstimateId: 1, revisionNumber: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "deletedAt: null, isCurrent: true" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "implements OnModuleInit" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "onModuleInit" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1

    **Edit D — Append a footer paragraph at the very bottom of `41-05-*-PLAN.md` (below all task blocks, above the frontmatter-terminator if any):**

    ```
    ---

    **Amendment history:**
    - 2026-04-11 — Phase 42 wave 1 (plan 42-01): added `{rootEstimateId, isCurrent}` partial unique index, `{rootEstimateId, revisionNumber}` sort index, and tightened `{businessId, number}` partial filter to `{deletedAt: null, isCurrent: true}`. Switched index bootstrap precedent from `QuoteRepository` (has no indexes) to `SubscriptionRepository` (`OnModuleInit` + `ensureIndexes`). Rationale: Phase 42 D-CHAIN-03 / D-CHAIN-05 / D-CHAIN-06 / D-CHAIN-07. Phase 41 has not executed; folding the final index topology forward avoids a runtime drop-and-recreate in Phase 42.
    ```

    Commit as part of Phase 42 plan 01 (combined with the other three tasks in this plan): `docs(42): amend Phase 41 PLAN-05 + PLAN-07 + ROADMAP SC #3 per Phase 42 chain decisions`.
  </action>
  <acceptance_criteria>
    - `grep -c "rootEstimateId: 1, isCurrent: 1" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "rootEstimateId: 1, revisionNumber: 1" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "deletedAt: null, isCurrent: true" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "implements OnModuleInit" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "SubscriptionRepository" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "Phase 42" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 2
    - `grep -c "D-CHAIN-0[356]" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns at least 1
    - `grep -c "partialFilterExpression: { deletedAt: null }" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns 0 (the old filter must be fully removed, not commented-out)
    - `grep -c "Amendment history" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md` returns 1
  </acceptance_criteria>
  <verify>
    <automated>grep -q "rootEstimateId: 1, isCurrent: 1" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md &amp;&amp; grep -q "implements OnModuleInit" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md &amp;&amp; ! grep -q "partialFilterExpression: { deletedAt: null }" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md</automated>
  </verify>
  <done>Phase 41 PLAN-05 declares all five indexes (including the two new Phase 42 ones and the tightened {businessId, number} filter), points at `SubscriptionRepository` as the bootstrap precedent, and carries an amendment history footer.</done>
</task>

<task type="auto">
  <name>Task 2: Amend Phase 41 PLAN-07 EstimateCreator root-write spec</name>
  <files>.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md</files>
  <read_first>
    - .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md (read the entire EstimateCreator code sketch starting around line 150 through the acceptance_criteria ending around line 290)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-CHAIN-03, D-CHAIN-04)
    - .planning/phases/42-revisions/42-RESEARCH.md §5.2 (Specific amendment needed in Phase 41 PLAN-07)
  </read_first>
  <action>
    Apply four edits to `.planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md`.

    **Edit A — Rewrite the "applies defaults" bullet in the task narrative (originally around line 125).**

    Find:
    > Applies defaults if not provided: `contingencyPercent ??= 10`, `displayMode ??= EstimateDisplayMode.RANGE`, `estimateDate ??= DateTime.now()`, `revisionNumber = 1`, `isCurrent = true`, `parentEstimateId = null`, `rootEstimateId = null`, `status = EstimateStatus.DRAFT`, `responseSummary = null`.

    Replace with:
    > Applies defaults if not provided: `contingencyPercent ??= 10`, `displayMode ??= EstimateDisplayMode.RANGE`, `estimateDate ??= DateTime.now()`, `revisionNumber = 1`, `isCurrent = true`, `parentEstimateId = null`, `status = EstimateStatus.DRAFT`, `responseSummary = null`. **Root rows are their own chain identity**: `rootEstimateId` starts as `null` in the prepared DTO, then `EstimateCreator.create()` issues a follow-up `updateOne` to set `rootEstimateId: created.id` on the newly-inserted row before returning. Amended per Phase 42 D-CHAIN-03 / D-CHAIN-04 (folded forward because Phase 41 has not executed).

    **Edit B — Rewrite the EstimateCreator code sketch.**

    Find the `create()` method body (starts around line 162, ends before line 198 where `}` closes the class). Replace the entire method body with:

    ```typescript
    public async create(authUser: IUserDto, dto: IEstimateDto): Promise<IEstimateDto> {
      const customer = await this.customerRetriever.findByIdOrFail(authUser, dto.customerId);
      if (!customer) {
        throw new ResourceNotFoundError(ErrorCodes.ESTIMATE_CUSTOMER_NOT_FOUND, "Customer not found for estimate");
      }

      const job = await this.jobRetrieverService.findByIdOrFail(authUser, dto.jobId);
      if (!job) {
        throw new ResourceNotFoundError(ErrorCodes.ESTIMATE_JOB_NOT_FOUND, "Job not found for estimate");
      }

      const number = await this.numberGenerator.generateNumber(dto.businessId);

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
    }
    ```

    **Edit C — Add a new `setRootEstimateId` repository method contract note** as a new paragraph directly below the `create()` code block:

    > **Repository contract addition (Phase 42 D-CHAIN-04 fold-forward):** `EstimateRepository` MUST expose `public async setRootEstimateId(id: string): Promise<void>` that wraps `writer.updateOne({_id: new ObjectId(id)}, {$set: {rootEstimateId: new ObjectId(id)}})`. This method has no error handling beyond what `updateOne` already provides; a zero-match result is an internal error because the caller just inserted the row. The acceptance criteria for PLAN-05's repository task is amended by plan 42-01 to include a grep for `setRootEstimateId`.

    **Edit D — Rewrite the spec-file assertions and grep acceptance criteria.**

    Find in the spec code block (around line 218):
    ```typescript
    expect(result.parentEstimateId).toBeNull();
    expect(result.rootEstimateId).toBeNull();
    ```

    Replace with:
    ```typescript
    expect(result.parentEstimateId).toBeNull();
    expect(result.rootEstimateId).toBe(result.id);  // Phase 42 D-CHAIN-03: root is its own chain identity
    expect(mockRepo.setRootEstimateId).toHaveBeenCalledWith(result.id);
    ```

    Find in the acceptance_criteria at line 277:
    - `grep -c "rootEstimateId: null" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1

    Replace with:
    - `grep -c "rootEstimateId: null" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1 (the initial prepared-DTO value)
    - `grep -c "setRootEstimateId" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1
    - `grep -c "rootEstimateId: created\\.id\\|withRoot" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1

    **Edit E — Append a footer paragraph at the very bottom of `41-07-*-PLAN.md`:**

    ```
    ---

    **Amendment history:**
    - 2026-04-11 — Phase 42 wave 1 (plan 42-01): `EstimateCreator.create()` now performs a two-step root write — `insertOne` via `authorizedCreator.create(...)` returns the new `id`, then `EstimateRepository.setRootEstimateId(created.id)` persists `rootEstimateId = created._id` on the row. Spec assertions updated to expect `result.rootEstimateId === result.id` on root creation. Rationale: Phase 42 D-CHAIN-03 / D-CHAIN-04. Phase 41 has not executed; folding the retrofit forward avoids a one-shot backfill migration.
    ```

    Commit as part of the same plan-01 commit.
  </action>
  <acceptance_criteria>
    - `grep -c "setRootEstimateId" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns at least 3
    - `grep -c "rootEstimateId: created.id\\|rootEstimateId: inserted\\._id\\|withRoot" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns at least 1
    - `grep -c "D-CHAIN-0[34]" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns at least 1
    - `grep -c "expect(result.rootEstimateId).toBeNull" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns 0 (the old test expectation must be fully removed)
    - `grep -c "expect(result.rootEstimateId).toBe(result.id)" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns at least 1
    - `grep -c "Amendment history" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns 1
    - `grep -c "Phase 42" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md` returns at least 2
  </acceptance_criteria>
  <verify>
    <automated>grep -q "setRootEstimateId" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md &amp;&amp; ! grep -q "expect(result.rootEstimateId).toBeNull" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md &amp;&amp; grep -q "expect(result.rootEstimateId).toBe(result.id)" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md</automated>
  </verify>
  <done>Phase 41 PLAN-07 specifies the two-step root-write, updated spec assertions, a repository contract addition (`setRootEstimateId`), and an amendment history footer.</done>
</task>

<task type="auto">
  <name>Task 3: Rewrite ROADMAP.md Phase 42 Success Criterion #3 per D-HOOK-05</name>
  <files>.planning/ROADMAP.md</files>
  <read_first>
    - .planning/ROADMAP.md (the entire Phase 42 entry, currently around lines 157-166)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-HOOK-01 through D-HOOK-05, and the literal replacement text in D-HOOK-05)
  </read_first>
  <action>
    Find the exact line in `.planning/ROADMAP.md` (currently line 164):

    > 3. Creating a revision atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision when loading the estimate detail page by root id.

    Replace it with the exact D-HOOK-05 wording:

    > 3. Creating a revision atomically transitions the chain: the new Draft revision becomes current, the previous revision keeps its existing status with `isCurrent: false`, and the trader sees only the latest revision when loading the estimate detail page by root id. Cancellation of the previous revision's pending follow-ups happens at the moment the new revision is actually Sent (Phase 44 owns the call via the `IEstimateFollowupCanceller` binding from Phase 42).

    Do NOT touch Phase 42 SC #1, #2, or #4. Do NOT touch the Phase 42 `**Goal**` or `**Depends on**` or `**Requirements**` lines. Do NOT renumber the success criteria.
  </action>
  <acceptance_criteria>
    - `grep -c "Phase 44 owns the call via the \`IEstimateFollowupCanceller\` binding from Phase 42" .planning/ROADMAP.md` returns 1
    - `grep -c "atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision" .planning/ROADMAP.md` returns 0 (old wording fully replaced)
    - `grep -c "atomically transitions the chain" .planning/ROADMAP.md` returns 1
    - `grep -c "\\*\\*Phase 42: Revisions\\*\\*" .planning/ROADMAP.md` returns at least 1 (the heading is untouched)
    - `grep -c "REV-01, REV-02, REV-03, REV-04, REV-05" .planning/ROADMAP.md` returns at least 1 (requirements list unchanged)
  </acceptance_criteria>
  <verify>
    <automated>grep -q "atomically transitions the chain" .planning/ROADMAP.md &amp;&amp; grep -q "Phase 44 owns the call via the \`IEstimateFollowupCanceller\` binding from Phase 42" .planning/ROADMAP.md &amp;&amp; ! grep -q "atomically cancels all pending follow-ups for the previous revision and the trader sees only the latest revision" .planning/ROADMAP.md</automated>
  </verify>
  <done>ROADMAP.md Phase 42 SC #3 uses the D-HOOK-05 wording verbatim; old wording is fully removed.</done>
</task>

<task type="auto">
  <name>Task 4: Mirror SC #3 rewrite in v1.8-ROADMAP.md</name>
  <files>.planning/milestones/v1.8-ROADMAP.md</files>
  <read_first>
    - .planning/milestones/v1.8-ROADMAP.md (the Phase 42 entry — grep for "Phase 42" to locate)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-HOOK-05 literal replacement text)
  </read_first>
  <action>
    Locate the Phase 42 section in `.planning/milestones/v1.8-ROADMAP.md` (grep `"Phase 42"`). Find the Success Criterion that mentions "atomically cancels all pending follow-ups" (this will be SC #3 or similarly numbered). Replace its text with the D-HOOK-05 wording:

    > Creating a revision atomically transitions the chain: the new Draft revision becomes current, the previous revision keeps its existing status with `isCurrent: false`, and the trader sees only the latest revision when loading the estimate detail page by root id. Cancellation of the previous revision's pending follow-ups happens at the moment the new revision is actually Sent (Phase 44 owns the call via the `IEstimateFollowupCanceller` binding from Phase 42).

    Preserve any surrounding bullet/numbering structure. If the milestone roadmap has a separate "Acceptance criteria" section distinct from "Success criteria", update both if the wording appears in both.

    If the milestone roadmap ALREADY uses the D-HOOK-05 wording (somehow — unlikely), this task is a no-op and the acceptance criteria below still pass. If the milestone roadmap uses a materially different phrasing than ROADMAP.md, replace ONLY the "cancels all pending follow-ups" sentence — do not rewrite the rest.
  </action>
  <acceptance_criteria>
    - `grep -c "atomically transitions the chain" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
    - `grep -c "Phase 44 owns the call via the \`IEstimateFollowupCanceller\` binding from Phase 42" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
    - `grep -c "atomically cancels all pending follow-ups for the previous revision" .planning/milestones/v1.8-ROADMAP.md` returns 0
  </acceptance_criteria>
  <verify>
    <automated>grep -q "atomically transitions the chain" .planning/milestones/v1.8-ROADMAP.md &amp;&amp; grep -q "Phase 44 owns the call via the \`IEstimateFollowupCanceller\` binding from Phase 42" .planning/milestones/v1.8-ROADMAP.md &amp;&amp; ! grep -q "atomically cancels all pending follow-ups for the previous revision" .planning/milestones/v1.8-ROADMAP.md</automated>
  </verify>
  <done>Milestone v1.8-ROADMAP.md reflects the same D-HOOK-05 Phase 42 SC #3 wording as ROADMAP.md.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| planning-docs → future-code | Phase 41 plans are read by the Phase 41 executor; wrong text here leaks into production code semantics |
| planning-docs → roadmap-truth | ROADMAP.md is the source of truth verified by `/gsd-verify-work`; wrong SC #3 wording breaks the verification loop |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-01-01 | Tampering | Phase 41 PLAN-05 index spec | mitigate | MEDIUM | Every index addition/change is grep-asserted in acceptance_criteria with exact-match strings; the old `partialFilterExpression: { deletedAt: null }` form is explicitly asserted to NOT appear (grep returns 0) so the rewrite cannot be accidentally partial |
| T-42-01-02 | Tampering | Phase 41 PLAN-07 creator spec | mitigate | MEDIUM | Spec-line changes are grep-asserted with both "added" and "removed" checks — `expect(result.rootEstimateId).toBeNull` must have 0 matches and `expect(result.rootEstimateId).toBe(result.id)` must have ≥1 |
| T-42-01-03 | Repudiation | Amendment attribution | mitigate | LOW | Every amended file gets an "Amendment history" footer with date, plan id (42-01), rationale D-XX reference, and a one-line summary of the change |
| T-42-01-04 | Information disclosure | SC #3 rewrite leaks "Phase 44 will cancel follow-ups" design detail in public roadmap | accept | LOW | The roadmap is a public planning artifact; cross-phase wiring is already visible in the phase list; no PII or secrets exposed |
| T-42-01-05 | Tampering | Old ROADMAP wording survives in one of the two roadmap files | mitigate | HIGH | Task 3 and Task 4 both grep for the absence of the old wording ("atomically cancels all pending follow-ups for the previous revision") with an expected count of 0 |

**Verification command:**
`grep -rn "atomically cancels all pending follow-ups for the previous revision" .planning/ROADMAP.md .planning/milestones/v1.8-ROADMAP.md` must produce zero output after this plan runs.
</threat_model>

<verification>
After all four tasks complete, run:

```bash
# Old wording is fully removed from both roadmap files:
! grep -r "atomically cancels all pending follow-ups for the previous revision" .planning/ROADMAP.md .planning/milestones/v1.8-ROADMAP.md

# New wording is present in both roadmap files:
grep -q "atomically transitions the chain" .planning/ROADMAP.md
grep -q "atomically transitions the chain" .planning/milestones/v1.8-ROADMAP.md

# Phase 41 PLAN-05 declares the new index topology:
grep -q "rootEstimateId: 1, isCurrent: 1" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
grep -q "rootEstimateId: 1, revisionNumber: 1" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
grep -q "deletedAt: null, isCurrent: true" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md
grep -q "implements OnModuleInit" .planning/phases/41-estimate-module-crud-backend/41-05-estimate-repositories-and-stateless-services-PLAN.md

# Phase 41 PLAN-07 specifies the two-step root-write:
grep -q "setRootEstimateId" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
grep -q "expect(result.rootEstimateId).toBe(result.id)" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
! grep -q "expect(result.rootEstimateId).toBeNull" .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md
```

All eleven checks must exit 0.
</verification>

<success_criteria>
- `.planning/phases/41-estimate-module-crud-backend/41-05-*-PLAN.md` declares all five indexes with the correct partial filter expressions, points at `SubscriptionRepository` as the bootstrap precedent, and carries an amendment footer crediting D-CHAIN-03/05/06.
- `.planning/phases/41-estimate-module-crud-backend/41-07-*-PLAN.md` specifies the two-step root-write (`insertOne` → `setRootEstimateId`), updated spec assertions, and an amendment footer crediting D-CHAIN-03/04.
- `.planning/ROADMAP.md` Phase 42 SC #3 matches the D-HOOK-05 wording verbatim.
- `.planning/milestones/v1.8-ROADMAP.md` Phase 42 SC #3 mirrors ROADMAP.md.
- A single commit under Phase 42: `docs(42): amend Phase 41 plans + ROADMAP SC #3 per D-CHAIN and D-HOOK-05`.
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-01-SUMMARY.md` documenting:
- Exact before/after diff of each amendment (4 files).
- Commit hash.
- Confirmation that no source code was touched.
- Confirmation that all eleven verification grep checks pass.
</output>
