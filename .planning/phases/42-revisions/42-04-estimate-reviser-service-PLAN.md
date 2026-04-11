---
phase: 42
plan: 04
type: execute
wave: 3
depends_on: ["42-02", "42-03"]
files_modified:
  - trade-flow-api/src/estimate/services/estimate-reviser.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
  - trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts
autonomous: true
requirements: [REV-02, REV-03, REV-05]
estimate: 60min
tags: [service, reviser, concurrency, phase-42]

must_haves:
  truths:
    - "`EstimateReviser.revise(authUser, id)` returns the new revision DTO on success (201-ready)."
    - "On filter miss at step 1 (source not current / disallowed status), `revise` throws `ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, ...)`."
    - "On step-2 failure after step-1 succeeded, compensating `restoreCurrent(oldId)` runs and the original exception re-throws."
    - "On step-3 failure after step-2 succeeded, compensating rollback soft-deletes the new row AND restores `oldId` current, then re-throws as `InternalServerError`."
    - "Line-item clone preserves bundle parent/child structure via the old→new `parentLineItemId` remap; every cloned row has `status: PENDING` regardless of source status; `DELETED` source items are skipped."
    - "`IEstimateFollowupCanceller` is injected but NEVER called from the reviser (D-HOOK-03). Verified with `jest.spyOn` non-call assertion in the spec."
    - "Revision number monotonicity: revising N creates N+1."
    - "Atomic filter shape verified at the spec level: `{_id, isCurrent: true, status: {$in: allowedStatuses}}`."
  artifacts:
    - path: "trade-flow-api/src/estimate/services/estimate-reviser.service.ts"
      provides: "EstimateReviser service implementing two-write revise flow with compensating rollback"
      exports: ["EstimateReviser"]
    - path: "trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts"
      provides: "Full coverage per research §7.3 test plan shape"
    - path: "trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts"
      provides: "Fixture builder for revision chains"
      exports: ["EstimateRevisionMockGenerator"]
  key_links:
    - from: "trade-flow-api/src/estimate/services/estimate-reviser.service.ts"
      to: "trade-flow-api/src/estimate/repositories/estimate.repository.ts"
      via: "`downgradeCurrent`, `insertRevision`, `restoreCurrent` calls"
      pattern: "estimateRepository\\.(downgradeCurrent|insertRevision|restoreCurrent)"
    - from: "trade-flow-api/src/estimate/services/estimate-reviser.service.ts"
      to: "trade-flow-api/src/core/errors/conflict.error.ts"
      via: "`throw new ConflictError(...)` on atomic filter miss"
      pattern: "new ConflictError\\(ErrorCodes\\.ESTIMATE_REVISION_CONFLICT"
    - from: "trade-flow-api/src/estimate/services/estimate-reviser.service.ts"
      to: "trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts"
      via: "`@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` constructor injection — NOT called from revise()"
      pattern: "@Inject\\(ESTIMATE_FOLLOWUP_CANCELLER"
---

<objective>
Ship the core Phase 42 service: `EstimateReviser`. This service orchestrates the clone-only revise flow end-to-end — policy check → atomic downgrade → insert new row → clone line items → return new revision DTO — with compensating rollback on any failure in the two-write window. It depends on plan 42-02 (ConflictError + error codes + IEstimateFollowupCanceller interface + token) and plan 42-03 (repository methods: downgradeCurrent, insertRevision, restoreCurrent, + line-item clone helpers).

This plan is explicitly isolated from plan 42-05 (retriever/deleter extensions) so both wave-3 plans can run in parallel without file overlap. `estimate-reviser.service.ts` is a brand-new file; plan 42-05 only touches existing retriever/deleter files.

**CRITICAL constraint from D-HOOK-03:** `EstimateReviser` injects `IEstimateFollowupCanceller` but does NOT call it. Phase 44 owns the call at the `DRAFT → SENT` transition. The spec has an explicit `jest.spyOn(canceller, 'cancelAllFollowups').not.toHaveBeenCalled()` assertion to prevent drift.

Output: three files (one new service + one new spec + one new mock generator), one commit, full test coverage per the research §7.3 plan.
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
@trade-flow-api/src/quote/services/quote-creator.service.ts
@trade-flow-api/src/quote/services/quote-updater.service.ts
@trade-flow-api/src/core/errors/conflict.error.ts
@trade-flow-api/src/core/errors/internal-server-error.error.ts
@trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts
@trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts
@trade-flow-api/src/estimate/repositories/estimate.repository.ts
@trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts

<interfaces>
<!-- Existing Phase 41 symbols that EstimateReviser uses -->

- `IEstimateDto` — `@estimate/data-transfer-objects/estimate.dto`
- `IEstimateLineItemDto` — `@estimate/data-transfer-objects/estimate-line-item.dto` (confirm file name)
- `EstimateStatus` enum — `@estimate/enums/estimate-status.enum`: values SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED, DRAFT, CONVERTED, DECLINED, EXPIRED, LOST, DELETED
- `EstimateLineItemStatus` enum — `@estimate/enums/estimate-line-item-status.enum`: PENDING, APPROVED, REJECTED, DELETED
- `EstimateRepository` — `@estimate/repositories/estimate.repository`
- `EstimateLineItemRepository` — `@estimate/repositories/estimate-line-item.repository`
- `EstimateRetriever` (from Phase 41 — or will be created in plan 42-05's amendment) — `@estimate/services/estimate-retriever.service`; method `findByIdOrFail(authUser, id): Promise<IEstimateDto>` returns the DTO with access checks already applied
- `EstimatePolicy` — `@estimate/policies/estimate.policy`
- `AccessControllerFactory` — `@core/factories/access-controller.factory`
- `IUserDto` — `@user/data-transfer-objects/user.dto`
- `ErrorCodes` — `@core/errors/error-codes.enum` (Phase 42 added ESTIMATE_REVISION_CONFLICT, ESTIMATE_NOT_REVISABLE)
- `ConflictError` — `@core/errors/conflict.error` (Phase 42 plan 42-02)
- `InternalServerError` — `@core/errors/internal-server-error.error`
- `IEstimateFollowupCanceller` + `ESTIMATE_FOLLOWUP_CANCELLER` — `@estimate/services/estimate-followup-canceller.interface`
- `AppLogger` — `@core/services/app-logger.service` (or `Logger` from `@nestjs/common` if the Phase 41 estimate services use `Logger`)

<!-- Target EstimateReviser skeleton -->

```typescript
import { Inject, Injectable, Logger } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { AccessControllerFactory } from "@core/factories/access-controller.factory";
import { ConflictError } from "@core/errors/conflict.error";
import { InternalServerError } from "@core/errors/internal-server-error.error";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { IEstimateLineItemDto } from "@estimate/data-transfer-objects/estimate-line-item.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";
import { EstimatePolicy } from "@estimate/policies/estimate.policy";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateLineItemRepository } from "@estimate/repositories/estimate-line-item.repository";
import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
import {
  ESTIMATE_FOLLOWUP_CANCELLER,
  IEstimateFollowupCanceller,
} from "@estimate/services/estimate-followup-canceller.interface";

const REVISABLE_STATUSES: EstimateStatus[] = [
  EstimateStatus.SENT,
  EstimateStatus.VIEWED,
  EstimateStatus.RESPONDED,
  EstimateStatus.SITE_VISIT_REQUESTED,
];

@Injectable()
export class EstimateReviser {
  private readonly logger = new Logger(EstimateReviser.name);

  constructor(
    private readonly estimateRepository: EstimateRepository,
    private readonly estimateLineItemRepository: EstimateLineItemRepository,
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimatePolicy: EstimatePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
    @Inject(ESTIMATE_FOLLOWUP_CANCELLER)
    private readonly followupCanceller: IEstimateFollowupCanceller,
  ) {}

  public async revise(authUser: IUserDto, id: string): Promise<IEstimateDto> {
    // Step 0: load + authorize
    const source = await this.estimateRetriever.findByIdOrFail(authUser, id);
    const accessController = this.accessControllerFactory.create(this.estimatePolicy);
    accessController.canUpdate(authUser, source);

    // Step 1: atomic downgrade-old (filter gated on isCurrent + allowed statuses)
    const downgraded = await this.estimateRepository.downgradeCurrent(source.id, REVISABLE_STATUSES);
    if (!downgraded) {
      throw new ConflictError(
        ErrorCodes.ESTIMATE_REVISION_CONFLICT,
        "Estimate has already been revised or is no longer revisable",
      );
    }

    // Step 2: insert new revision row
    let insertedId: string | null = null;
    try {
      const newDto = this.buildNewRevisionDto(source);
      const inserted = await this.estimateRepository.insertRevision(newDto);
      insertedId = inserted.id;

      // Step 3: clone line items
      const sourceLineItems = await this.estimateLineItemRepository.findNonDeletedByEstimateId(source.id);
      const clonedItems = this.buildClonedLineItems(sourceLineItems, inserted.id);
      await this.estimateLineItemRepository.bulkInsertForRevision(clonedItems);

      // Step 5: return fresh DTO via retriever so totals + priceRange are recalculated
      return await this.estimateRetriever.findByIdOrFail(authUser, inserted.id);
    } catch (error) {
      await this.compensatingRollback(source.id, insertedId);
      throw this.wrapFailureForThrow(error);
    }
  }

  private buildNewRevisionDto(source: IEstimateDto): IEstimateDto {
    return {
      ...source,
      id: "", // repository overrides on insert
      status: EstimateStatus.DRAFT,
      isCurrent: true,
      revisionNumber: source.revisionNumber + 1,
      parentEstimateId: source.id,
      rootEstimateId: source.rootEstimateId,
      sentAt: null,
      firstViewedAt: null,
      respondedAt: null,
      convertedAt: null,
      convertedToQuoteId: null,
      declinedAt: null,
      lostAt: null,
      expiresAt: null,
      deletedAt: null,
      lastResponseType: null,
      lastResponseAt: null,
      lastResponseMessage: null,
      declineReason: null,
      siteVisitAvailability: null,
      lineItems: [],
      // totals and priceRange will be recalculated by retriever on read
    };
  }

  private buildClonedLineItems(
    source: IEstimateLineItemDto[],
    newEstimateId: string,
  ): IEstimateLineItemDto[] {
    const roots = source.filter((item) => !item.parentLineItemId);
    const children = source.filter((item) => item.parentLineItemId);

    // Build roots first, collecting old→new id map
    const parentRemap = new Map<string, string>();
    const cloned: IEstimateLineItemDto[] = [];

    for (const root of roots) {
      const newRoot: IEstimateLineItemDto = {
        ...root,
        id: "",
        estimateId: newEstimateId,
        parentLineItemId: null,
        status: EstimateLineItemStatus.PENDING,
      };
      cloned.push(newRoot);
      // parentRemap is populated after bulkInsertForRevision runs — see the note below
      parentRemap.set(root.id, "placeholder-for-insertion-order");
    }

    // NOTE: because bulkInsertForRevision inserts sequentially and returns IDs in order,
    // the caller (revise) must insert roots first, capture their new IDs, THEN build children
    // with the remapped parentLineItemId. See the actual revise() flow for the two-pass insertion.
    return cloned;
  }
}
```

**Important correction to the skeleton above:** because the reviser needs the NEW root IDs to remap children, the line-item insertion is a TWO-PASS operation, not a single `bulkInsertForRevision` call. The `revise()` method needs to:

1. Build the roots list (no parentLineItemId), call `bulkInsertForRevision(roots)`, capture returned IDs.
2. Build the children list using the parentRemap populated from step 1's returned IDs.
3. Call `bulkInsertForRevision(children)`.

This is why plan 42-03's `bulkInsertForRevision` returns the persisted DTOs (with populated IDs) — so step 1 can build the remap.

Here is the correct `revise` flow for step 3:

```typescript
// Step 3 (corrected): two-pass line-item clone
const sourceLineItems = await this.estimateLineItemRepository.findNonDeletedByEstimateId(source.id);

const sourceRoots = sourceLineItems.filter((item) => !item.parentLineItemId);
const sourceChildren = sourceLineItems.filter((item) => item.parentLineItemId);

const clonedRoots: IEstimateLineItemDto[] = sourceRoots.map((root) => ({
  ...root,
  id: "",
  estimateId: inserted.id,
  parentLineItemId: null,
  status: EstimateLineItemStatus.PENDING,
}));

const insertedRoots = await this.estimateLineItemRepository.bulkInsertForRevision(clonedRoots);

// Build parent remap: old root id → new root id, in insertion order
const parentRemap = new Map<string, string>();
sourceRoots.forEach((sourceRoot, index) => {
  parentRemap.set(sourceRoot.id, insertedRoots[index].id);
});

const clonedChildren: IEstimateLineItemDto[] = sourceChildren.map((child) => {
  const newParentId = parentRemap.get(child.parentLineItemId ?? "");
  if (!newParentId) {
    throw new InternalServerError(
      ErrorCodes.UNKNOWN_SERVER_ERROR,
      `Orphaned child line item ${child.id}: parent ${child.parentLineItemId ?? "null"} not found in source roots`,
    );
  }
  return {
    ...child,
    id: "",
    estimateId: inserted.id,
    parentLineItemId: newParentId,
    status: EstimateLineItemStatus.PENDING,
  };
});

if (clonedChildren.length > 0) {
  await this.estimateLineItemRepository.bulkInsertForRevision(clonedChildren);
}
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create EstimateRevisionMockGenerator fixture builder</name>
  <files>trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts (the Phase 41 Estimate mock generator — confirm static method patterns, ObjectId usage, default field values)
    - trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts (Phase 41 line-item mock generator)
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts (IEstimateDto field inventory — especially the nullable revision/lifecycle fields)
    - .planning/phases/42-revisions/42-CONTEXT.md §Specifics line-item-clone test fixture
  </read_first>
  <behavior>
    - `EstimateRevisionMockGenerator.createRoot(overrides?)` returns an `IEstimateDto` with `revisionNumber: 1`, `isCurrent: true`, `parentEstimateId: null`, `rootEstimateId === self.id` (a stable ObjectId string), `status: EstimateStatus.SENT` by default.
    - `EstimateRevisionMockGenerator.createChildRevision(root, overrides?)` returns an `IEstimateDto` derived from `root` with `revisionNumber: root.revisionNumber + 1`, `parentEstimateId: root.id`, `rootEstimateId: root.rootEstimateId`, `isCurrent: true`, `status: EstimateStatus.DRAFT` by default.
    - `EstimateRevisionMockGenerator.createBundleLineItemSet(estimateId)` returns a 5-item `IEstimateLineItemDto[]` matching the research §7.4 fixture: one bundle parent (li-A, `parentLineItemId: null`), two children (li-B, li-C with `parentLineItemId: li-A.id`), one standalone (li-D, `parentLineItemId: null`), one DELETED (li-E, `status: DELETED`, `parentLineItemId: null`).
  </behavior>
  <action>
    Create `trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts`:

    ```typescript
    import { ObjectId } from "mongodb";
    import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
    import { IEstimateLineItemDto } from "@estimate/data-transfer-objects/estimate-line-item.dto";
    import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
    import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";
    import { EstimateMockGenerator } from "@estimate/test/mocks/estimate-mock-generator";

    export class EstimateRevisionMockGenerator {
      public static createRoot(overrides: Partial<IEstimateDto> = {}): IEstimateDto {
        const id = new ObjectId().toString();
        const base = EstimateMockGenerator.createEstimateDto({
          ...overrides,
          id,
          revisionNumber: 1,
          isCurrent: true,
          parentEstimateId: null,
          rootEstimateId: id,
          status: overrides.status ?? EstimateStatus.SENT,
        });
        return base;
      }

      public static createChildRevision(
        parent: IEstimateDto,
        overrides: Partial<IEstimateDto> = {},
      ): IEstimateDto {
        return EstimateMockGenerator.createEstimateDto({
          ...overrides,
          id: new ObjectId().toString(),
          revisionNumber: parent.revisionNumber + 1,
          isCurrent: true,
          parentEstimateId: parent.id,
          rootEstimateId: parent.rootEstimateId,
          status: overrides.status ?? EstimateStatus.DRAFT,
          businessId: parent.businessId,
          customerId: parent.customerId,
          jobId: parent.jobId,
          number: parent.number,
        });
      }

      public static createBundleLineItemSet(estimateId: string): IEstimateLineItemDto[] {
        const bundleParentId = new ObjectId().toString();
        // Matches research §7.4 fixture: li-A (bundle parent), li-B / li-C (children of li-A),
        // li-D (standalone), li-E (DELETED, should be skipped by findNonDeletedByEstimateId).
        return [
          {
            id: bundleParentId,
            estimateId,
            parentLineItemId: null,
            status: EstimateLineItemStatus.APPROVED,
            quantity: 1,
            unit: "bundle",
            unitPrice: { amount: 10000, currency: "GBP" },
            lineTotal: { amount: 10000, currency: "GBP" },
          } as unknown as IEstimateLineItemDto,
          // ... the remaining four items; refer to the research §7.4 for exact shape
        ];
      }
    }
    ```

    **Adapt the line-item shape to the actual `IEstimateLineItemDto` interface** — the fields above are examples. Read `trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts` (the Phase-41-produced file) to determine the exact field names, then populate the fixture accordingly. Include `originalUnitPrice`, `originalLineTotal`, `discountAmount`, `taxRate`, `type` per D-CONC-04.

    **The `as unknown as IEstimateLineItemDto` cast is the one allowed exception** — it's a test-only fixture and the full field inventory is tedious. If the existing `EstimateLineItemMockGenerator` has a `createLineItemDto(overrides)` helper that fills defaults, use it: `EstimateLineItemMockGenerator.createLineItemDto({estimateId, parentLineItemId: null, ...})`. That's cleaner than the `as unknown as` cast and aligns with CLAUDE.md.

    **Prohibitions:**
    - No `any`.
    - No `eslint-disable` / `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck`.
    - Use `EstimateMockGenerator.createEstimateDto` if it exists, don't re-invent defaults.

    No spec file is needed for the mock generator itself — its correctness is tested indirectly by the reviser spec in Task 2.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts`
    - `grep -c "export class EstimateRevisionMockGenerator" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "public static createRoot" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "public static createChildRevision" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "public static createBundleLineItemSet" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "rootEstimateId: id" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "revisionNumber: parent.revisionNumber + 1" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 1
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts` returns 0
    - `cd trade-flow-api && npm run typecheck` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run typecheck</automated>
  </verify>
  <done>EstimateRevisionMockGenerator exists with createRoot, createChildRevision, createBundleLineItemSet; typechecks cleanly; used by Task 2 spec.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Create EstimateReviser service with full revise flow, compensating rollback, and spec coverage</name>
  <files>trade-flow-api/src/estimate/services/estimate-reviser.service.ts, trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts</files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-creator.service.ts (existing service pattern for constructor DI + service-layer flow)
    - trade-flow-api/src/quote/services/quote-updater.service.ts (read-then-fetch pattern — the reviser diverges here by using atomic downgradeCurrent at step 1)
    - trade-flow-api/src/estimate/services/estimate-retriever.service.ts (Phase-41-produced; confirm `findByIdOrFail(authUser, id)` signature and return shape; Phase 42 plan 42-05 extends this further for D-DET-01 non-current resolution, but the reviser calls it BEFORE any revision exists, so the Phase 41 shape is fine at step 0)
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (confirm method signatures added by plan 42-03: downgradeCurrent, insertRevision, restoreCurrent)
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (confirm bulkInsertForRevision + findNonDeletedByEstimateId signatures added by plan 42-03)
    - trade-flow-api/src/core/errors/conflict.error.ts (plan 42-02 output — the error class the reviser throws on filter miss)
    - trade-flow-api/src/core/errors/error-codes.enum.ts (plan 42-02 added ESTIMATE_REVISION_CONFLICT + ESTIMATE_NOT_REVISABLE)
    - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts (plan 42-02 output — interface + ESTIMATE_FOLLOWUP_CANCELLER token)
    - trade-flow-api/src/core/factories/access-controller.factory.ts (existing factory — confirm method names: create(), canUpdate(), canRead())
    - .planning/phases/42-revisions/42-CONTEXT.md (D-REV-01..06, D-CONC-01..08, D-LI-01..05, D-HOOK-03)
    - .planning/phases/42-revisions/42-RESEARCH.md §4 (Atomic Reviser Flow step-by-step), §7.3 (full test coverage map), §8 (pitfalls)
  </read_first>
  <behavior>
    **Spec cases (mapped to research §7.3 minimum coverage):**

    - Test 1: Happy path — source SENT → new row is DRAFT, isCurrent true, revisionNumber = source+1, parentEstimateId = source.id, rootEstimateId = source.rootEstimateId; returns the new DTO.
    - Test 2: Each allowed source status reaches happy path: SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED.
    - Test 3: Each disallowed source status throws ConflictError ESTIMATE_REVISION_CONFLICT (because downgradeCurrent returns null on filter miss): DRAFT, CONVERTED, DECLINED, EXPIRED, LOST, DELETED.
    - Test 4: **D-HOOK-03 non-call assertion** — `jest.spyOn(canceller, 'cancelAllFollowups')` is NOT called during a successful revise.
    - Test 5: Compensating rollback on step-2 (insertRevision) failure — `restoreCurrent(source.id)` is called and the original error re-throws.
    - Test 6: Compensating rollback on step-3 (line-item clone) failure — the new row is soft-deleted (status DELETED) AND `restoreCurrent(source.id)` is called and the original error re-throws (wrapped as InternalServerError).
    - Test 7: Line-item clone preserves bundle parent/child — using the research §7.4 fixture (li-A bundle parent, li-B/li-C children, li-D standalone, li-E DELETED); assert:
      - `bulkInsertForRevision` called twice (roots then children).
      - First call has 2 items (li-A equivalent + li-D equivalent), both with `parentLineItemId: null`.
      - Second call has 2 items (li-B equivalent + li-C equivalent), both with `parentLineItemId` remapped to the new root's id.
      - DELETED item (li-E) never appears in either call.
      - Every cloned item has `status: EstimateLineItemStatus.PENDING`.
    - Test 8: Revision number monotonicity — revising N=2 creates N=3; revising a chain at N=5 creates N=6.
    - Test 9: Atomic filter shape — `downgradeCurrent` is called with `(source.id, [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED])`, verified via `expect(mockRepo.downgradeCurrent).toHaveBeenCalledWith(source.id, expect.arrayContaining([...]))`.
    - Test 10: DI injection — the `followupCanceller` field is defined and is the NoopEstimateFollowupCanceller instance in the test module.
  </behavior>
  <action>
    **Step 1 — Create `trade-flow-api/src/estimate/services/estimate-reviser.service.ts`.**

    Use the skeleton from the `<interfaces>` block above, with the corrected two-pass line-item clone, and adapt to the actual Phase 41 symbol names. The full service implementation should be approximately 180-220 lines. Structure:

    1. Imports (all via path aliases).
    2. Module-level constant: `REVISABLE_STATUSES = [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED]`.
    3. `@Injectable()` class with constructor injecting 6 deps: `estimateRepository`, `estimateLineItemRepository`, `estimateRetriever`, `estimatePolicy`, `accessControllerFactory`, `followupCanceller` (via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)`).
    4. Public method `revise(authUser: IUserDto, id: string): Promise<IEstimateDto>`.
    5. Private helpers: `buildNewRevisionDto(source: IEstimateDto)`, `cloneLineItemsForRevision(sourceLineItems, newEstimateId)` (handles the two-pass insert), `compensatingRollback(sourceId, newIdOrNull)`, `wrapFailureForThrow(error)`.

    **The `revise` method body in full:**

    ```typescript
    public async revise(authUser: IUserDto, id: string): Promise<IEstimateDto> {
      // Step 0: load + authorize
      const source = await this.estimateRetriever.findByIdOrFail(authUser, id);
      const accessController = this.accessControllerFactory.create(this.estimatePolicy);
      accessController.canUpdate(authUser, source);

      // Step 1: atomic downgrade-old (filter gated on isCurrent + allowed statuses)
      const downgraded = await this.estimateRepository.downgradeCurrent(source.id, REVISABLE_STATUSES);
      if (!downgraded) {
        throw new ConflictError(
          ErrorCodes.ESTIMATE_REVISION_CONFLICT,
          "Estimate has already been revised or is no longer revisable",
        );
      }

      // Step 2 + 3: insert new row and clone line items with compensating rollback on failure
      let insertedId: string | null = null;
      try {
        const newDto = this.buildNewRevisionDto(source);
        const inserted = await this.estimateRepository.insertRevision(newDto);
        insertedId = inserted.id;

        const sourceLineItems = await this.estimateLineItemRepository.findNonDeletedByEstimateId(source.id);
        await this.cloneLineItemsForRevision(sourceLineItems, inserted.id);

        // Step 5: return fresh DTO so totals + priceRange are recalculated by the retriever
        return await this.estimateRetriever.findByIdOrFail(authUser, inserted.id);
      } catch (error) {
        await this.compensatingRollback(source.id, insertedId);
        throw this.wrapFailureForThrow(error);
      }
    }

    private buildNewRevisionDto(source: IEstimateDto): IEstimateDto {
      return {
        ...source,
        id: "", // repository will assign on insert
        status: EstimateStatus.DRAFT,
        isCurrent: true,
        revisionNumber: source.revisionNumber + 1,
        parentEstimateId: source.id,
        rootEstimateId: source.rootEstimateId,
        sentAt: null,
        firstViewedAt: null,
        respondedAt: null,
        convertedAt: null,
        convertedToQuoteId: null,
        declinedAt: null,
        lostAt: null,
        expiresAt: null,
        deletedAt: null,
        lastResponseType: null,
        lastResponseAt: null,
        lastResponseMessage: null,
        declineReason: null,
        siteVisitAvailability: null,
        lineItems: [],
      };
    }

    private async cloneLineItemsForRevision(
      sourceItems: IEstimateLineItemDto[],
      newEstimateId: string,
    ): Promise<void> {
      const sourceRoots = sourceItems.filter((item) => !item.parentLineItemId);
      const sourceChildren = sourceItems.filter((item) => item.parentLineItemId);

      const clonedRoots: IEstimateLineItemDto[] = sourceRoots.map((root) => ({
        ...root,
        id: "",
        estimateId: newEstimateId,
        parentLineItemId: null,
        status: EstimateLineItemStatus.PENDING,
      }));

      const insertedRoots = sourceRoots.length > 0
        ? await this.estimateLineItemRepository.bulkInsertForRevision(clonedRoots)
        : [];

      const parentRemap = new Map<string, string>();
      sourceRoots.forEach((sourceRoot, index) => {
        parentRemap.set(sourceRoot.id, insertedRoots[index].id);
      });

      const clonedChildren: IEstimateLineItemDto[] = sourceChildren.map((child) => {
        const parentKey = child.parentLineItemId ?? "";
        const newParentId = parentRemap.get(parentKey);
        if (!newParentId) {
          throw new InternalServerError(
            ErrorCodes.UNKNOWN_SERVER_ERROR,
            `Orphaned child line item ${child.id}: parent ${parentKey} not found in source roots`,
          );
        }
        return {
          ...child,
          id: "",
          estimateId: newEstimateId,
          parentLineItemId: newParentId,
          status: EstimateLineItemStatus.PENDING,
        };
      });

      if (clonedChildren.length > 0) {
        await this.estimateLineItemRepository.bulkInsertForRevision(clonedChildren);
      }
    }

    private async compensatingRollback(sourceId: string, newIdOrNull: string | null): Promise<void> {
      // Runbook note: if this rollback fails, the chain may be left with zero current rows or two.
      // The partial unique index still prevents two-current state. Manual recovery:
      //   1. db.estimates.updateOne({_id: ObjectId(newId)}, {$set: {status: "deleted", isCurrent: false, deletedAt: new Date()}})
      //   2. db.estimates.updateOne({_id: ObjectId(sourceId)}, {$set: {isCurrent: true}})
      // The trader can retry the POST after manual recovery.
      try {
        // If step 2 already created a new row, soft-delete it first so the partial unique index is freed.
        if (newIdOrNull) {
          const newRow = await this.estimateRepository.findByIdOrFail(newIdOrNull);
          await this.estimateRepository.update({
            ...newRow,
            status: EstimateStatus.DELETED,
            isCurrent: false,
            deletedAt: new Date() as unknown as never,
          });
        }
        await this.estimateRepository.restoreCurrent(sourceId);
      } catch (rollbackError) {
        this.logger.error(
          `Compensating rollback failed - manual intervention required. sourceId=${sourceId}, newId=${newIdOrNull ?? "null"}`,
          rollbackError instanceof Error ? rollbackError.stack : String(rollbackError),
        );
      }
    }

    private wrapFailureForThrow(error: unknown): Error {
      if (error instanceof ConflictError) {
        return error;
      }
      if (error instanceof InternalServerError) {
        return error;
      }
      const message = error instanceof Error ? error.message : "Unknown revise failure";
      return new InternalServerError(ErrorCodes.UNKNOWN_SERVER_ERROR, message);
    }
    ```

    **Critical notes:**

    - The `deletedAt: new Date() as unknown as never` cast is a Date/DateTime mismatch: `IEstimateDto.deletedAt` is typed `DateTime | null` per CLAUDE.md DTO standard but the entity stores `Date`. The repository's `update` method converts. The `as unknown as never` pattern is ugly and should be replaced — prefer passing through the repository's existing `softDelete(id)` method if one exists. **Check `estimate.repository.ts` for a `softDelete` or equivalent method and call THAT instead.** The goal is to avoid any `as` in production code.
    - If no `softDelete` exists, the correct path is to import `DateTime` from luxon and set `deletedAt: DateTime.now()` — that matches the DTO contract and the repository handles the conversion to JS Date.
    - The DI injection for `followupCanceller` uses `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` because the token is a string, not a class reference.
    - ESLint may flag `followupCanceller` as unused because it's injected but never called (by design — D-HOOK-03). Options:
      - (a) Prefix the parameter name: `_followupCanceller` — but then the test can't spy on it via `reviser['_followupCanceller']`.
      - (b) Add a reference-only use: `void this.followupCanceller` in a private comment-explained method, so ESLint sees it as used.
      - (c) Use the class field syntax and rely on ESLint's different treatment of class fields vs function parameters.
      - **Recommendation:** store it on `this.followupCanceller` (as in the skeleton) — ESLint `@typescript-eslint/no-unused-vars` does NOT flag class fields as unused by default, only unused function parameters and local variables. The constructor-parameter-property syntax (`private readonly followupCanceller: ...`) is a class field and should not be flagged. Verify this by running `npm run lint:check` after writing the file.
    - The reviser does NOT have its own try/catch-to-HTTP wrapping — that's the controller's job (plan 42-06 wraps `createHttpError`).

    **Step 2 — Create `trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts`.**

    Full spec file with the 10 test cases from the `<behavior>` block. Use `Test.createTestingModule` with mocked dependencies:

    ```typescript
    import { Test, TestingModule } from "@nestjs/testing";
    import { EstimateReviser } from "@estimate/services/estimate-reviser.service";
    import { EstimateRepository } from "@estimate/repositories/estimate.repository";
    import { EstimateLineItemRepository } from "@estimate/repositories/estimate-line-item.repository";
    import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
    import { EstimatePolicy } from "@estimate/policies/estimate.policy";
    import { AccessControllerFactory } from "@core/factories/access-controller.factory";
    import { ConflictError } from "@core/errors/conflict.error";
    import { InternalServerError } from "@core/errors/internal-server-error.error";
    import { ErrorCodes } from "@core/errors/error-codes.enum";
    import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
    import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";
    import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";
    import { ESTIMATE_FOLLOWUP_CANCELLER } from "@estimate/services/estimate-followup-canceller.interface";
    import { EstimateRevisionMockGenerator } from "@estimate/test/mocks/estimate-revision-mock-generator";
    import { UserMockGenerator } from "@user/test/mocks/user-mock-generator"; // or wherever

    describe("EstimateReviser", () => {
      let reviser: EstimateReviser;
      let mockRepo: jest.Mocked<EstimateRepository>;
      let mockLineItemRepo: jest.Mocked<EstimateLineItemRepository>;
      let mockRetriever: jest.Mocked<EstimateRetriever>;
      let mockPolicy: jest.Mocked<EstimatePolicy>;
      let mockAccessControllerFactory: jest.Mocked<AccessControllerFactory>;
      let noopCanceller: NoopEstimateFollowupCanceller;
      const authUser = UserMockGenerator.createUserDto();

      beforeEach(async () => {
        const accessController = { canUpdate: jest.fn(), canRead: jest.fn() };
        mockAccessControllerFactory = {
          create: jest.fn().mockReturnValue(accessController),
        } as unknown as jest.Mocked<AccessControllerFactory>;
        mockRepo = {
          downgradeCurrent: jest.fn(),
          insertRevision: jest.fn(),
          restoreCurrent: jest.fn(),
          findByIdOrFail: jest.fn(),
          update: jest.fn(),
        } as unknown as jest.Mocked<EstimateRepository>;
        mockLineItemRepo = {
          findNonDeletedByEstimateId: jest.fn(),
          bulkInsertForRevision: jest.fn(),
        } as unknown as jest.Mocked<EstimateLineItemRepository>;
        mockRetriever = {
          findByIdOrFail: jest.fn(),
        } as unknown as jest.Mocked<EstimateRetriever>;
        mockPolicy = {} as unknown as jest.Mocked<EstimatePolicy>;

        const module: TestingModule = await Test.createTestingModule({
          providers: [
            EstimateReviser,
            NoopEstimateFollowupCanceller,
            { provide: EstimateRepository, useValue: mockRepo },
            { provide: EstimateLineItemRepository, useValue: mockLineItemRepo },
            { provide: EstimateRetriever, useValue: mockRetriever },
            { provide: EstimatePolicy, useValue: mockPolicy },
            { provide: AccessControllerFactory, useValue: mockAccessControllerFactory },
            { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },
          ],
        }).compile();

        reviser = module.get<EstimateReviser>(EstimateReviser);
        noopCanceller = module.get<NoopEstimateFollowupCanceller>(ESTIMATE_FOLLOWUP_CANCELLER);
      });

      afterEach(() => jest.clearAllMocks());

      describe("happy path", () => {
        it("revises a SENT estimate into a new DRAFT with revisionNumber + 1, isCurrent true", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(source); // step 0
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          const inserted = EstimateRevisionMockGenerator.createChildRevision(source);
          mockRepo.insertRevision.mockResolvedValue(inserted);
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([]);
          mockLineItemRepo.bulkInsertForRevision.mockResolvedValue([]);
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(inserted); // step 5

          const result = await reviser.revise(authUser, source.id);

          expect(result.revisionNumber).toBe(source.revisionNumber + 1);
          expect(result.isCurrent).toBe(true);
          expect(result.parentEstimateId).toBe(source.id);
          expect(result.status).toBe(EstimateStatus.DRAFT);
        });
      });

      describe("allowed source statuses", () => {
        it.each([
          EstimateStatus.SENT,
          EstimateStatus.VIEWED,
          EstimateStatus.RESPONDED,
          EstimateStatus.SITE_VISIT_REQUESTED,
        ])("allows revising a source in status %s", async (status) => {
          const source = EstimateRevisionMockGenerator.createRoot({ status });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          const inserted = EstimateRevisionMockGenerator.createChildRevision(source);
          mockRepo.insertRevision.mockResolvedValue(inserted);
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([]);
          mockLineItemRepo.bulkInsertForRevision.mockResolvedValue([]);
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(source).mockResolvedValueOnce(inserted);

          await expect(reviser.revise(authUser, source.id)).resolves.toBeDefined();
        });
      });

      describe("disallowed source statuses", () => {
        it.each([
          EstimateStatus.DRAFT,
          EstimateStatus.CONVERTED,
          EstimateStatus.DECLINED,
          EstimateStatus.EXPIRED,
          EstimateStatus.LOST,
          EstimateStatus.DELETED,
        ])("throws ConflictError when source status is %s (filter miss)", async (status) => {
          const source = EstimateRevisionMockGenerator.createRoot({ status });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue(null); // filter miss

          await expect(reviser.revise(authUser, source.id)).rejects.toBeInstanceOf(ConflictError);
          await expect(reviser.revise(authUser, source.id)).rejects.toMatchObject({
            name: "ConflictError",
          });
        });
      });

      describe("D-HOOK-03: followupCanceller is injected but not called", () => {
        it("does NOT call cancelAllFollowups during a successful revise", async () => {
          const cancelSpy = jest.spyOn(noopCanceller, "cancelAllFollowups");
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          const inserted = EstimateRevisionMockGenerator.createChildRevision(source);
          mockRepo.insertRevision.mockResolvedValue(inserted);
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([]);
          mockLineItemRepo.bulkInsertForRevision.mockResolvedValue([]);
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(inserted);

          await reviser.revise(authUser, source.id);

          expect(cancelSpy).not.toHaveBeenCalled();
        });
      });

      describe("compensating rollback", () => {
        it("restores source isCurrent on insertRevision failure", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          mockRepo.insertRevision.mockRejectedValue(new Error("insert failed"));

          await expect(reviser.revise(authUser, source.id)).rejects.toThrow();
          expect(mockRepo.restoreCurrent).toHaveBeenCalledWith(source.id);
        });

        it("soft-deletes new row AND restores source on line-item clone failure", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          const inserted = EstimateRevisionMockGenerator.createChildRevision(source);
          mockRepo.insertRevision.mockResolvedValue(inserted);
          mockRepo.findByIdOrFail.mockResolvedValue(inserted);
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([]);
          mockLineItemRepo.bulkInsertForRevision.mockRejectedValue(new Error("clone failed"));

          await expect(reviser.revise(authUser, source.id)).rejects.toBeInstanceOf(InternalServerError);
          expect(mockRepo.update).toHaveBeenCalledWith(
            expect.objectContaining({ status: EstimateStatus.DELETED, isCurrent: false }),
          );
          expect(mockRepo.restoreCurrent).toHaveBeenCalledWith(source.id);
        });
      });

      describe("line-item clone with bundle parent/child", () => {
        it("preserves bundle structure, skips DELETED, forces status PENDING", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          const inserted = EstimateRevisionMockGenerator.createChildRevision(source);
          mockRepo.insertRevision.mockResolvedValue(inserted);

          // Fixture: li-A (bundle parent), li-B (child of li-A), li-C (child of li-A), li-D (standalone)
          // Note: DELETED items are filtered at the repository (findNonDeletedByEstimateId), so the
          // reviser never sees them. This test exercises the roots/children separation on non-deleted input.
          const liA = /* bundle parent with id='li-A', parentLineItemId: null, status: APPROVED */;
          const liB = /* child of liA with parentLineItemId: 'li-A', status: APPROVED */;
          const liC = /* child of liA with parentLineItemId: 'li-A', status: APPROVED */;
          const liD = /* standalone with parentLineItemId: null, status: APPROVED */;
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([liA, liB, liC, liD]);

          // Mock bulkInsertForRevision to return inputs with new IDs
          mockLineItemRepo.bulkInsertForRevision
            .mockResolvedValueOnce([
              { ...liA, id: "new-liA", parentLineItemId: null, estimateId: inserted.id, status: EstimateLineItemStatus.PENDING },
              { ...liD, id: "new-liD", parentLineItemId: null, estimateId: inserted.id, status: EstimateLineItemStatus.PENDING },
            ])
            .mockResolvedValueOnce([
              { ...liB, id: "new-liB", parentLineItemId: "new-liA", estimateId: inserted.id, status: EstimateLineItemStatus.PENDING },
              { ...liC, id: "new-liC", parentLineItemId: "new-liA", estimateId: inserted.id, status: EstimateLineItemStatus.PENDING },
            ]);
          mockRetriever.findByIdOrFail.mockResolvedValueOnce(source).mockResolvedValueOnce(inserted);

          await reviser.revise(authUser, source.id);

          expect(mockLineItemRepo.bulkInsertForRevision).toHaveBeenCalledTimes(2);
          const [rootsCall, childrenCall] = mockLineItemRepo.bulkInsertForRevision.mock.calls;
          expect(rootsCall[0]).toHaveLength(2);
          expect(rootsCall[0].every((it) => it.parentLineItemId === null)).toBe(true);
          expect(rootsCall[0].every((it) => it.status === EstimateLineItemStatus.PENDING)).toBe(true);
          expect(childrenCall[0]).toHaveLength(2);
          expect(childrenCall[0].every((it) => it.parentLineItemId === "new-liA")).toBe(true);
          expect(childrenCall[0].every((it) => it.status === EstimateLineItemStatus.PENDING)).toBe(true);
        });
      });

      describe("revision number monotonicity", () => {
        it("revising N creates N+1", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({
            status: EstimateStatus.SENT,
            revisionNumber: 5,
          });
          // ...same setup
          // assert inserted.revisionNumber === 6
        });
      });

      describe("atomic filter shape", () => {
        it("calls downgradeCurrent with [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED]", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          mockRetriever.findByIdOrFail.mockResolvedValue(source);
          mockRepo.downgradeCurrent.mockResolvedValue({ ...source, isCurrent: false });
          mockRepo.insertRevision.mockResolvedValue(EstimateRevisionMockGenerator.createChildRevision(source));
          mockLineItemRepo.findNonDeletedByEstimateId.mockResolvedValue([]);
          mockLineItemRepo.bulkInsertForRevision.mockResolvedValue([]);

          await reviser.revise(authUser, source.id).catch(() => undefined);

          expect(mockRepo.downgradeCurrent).toHaveBeenCalledWith(
            source.id,
            expect.arrayContaining([
              EstimateStatus.SENT,
              EstimateStatus.VIEWED,
              EstimateStatus.RESPONDED,
              EstimateStatus.SITE_VISIT_REQUESTED,
            ]),
          );
        });
      });

      describe("DI injection resolution", () => {
        it("resolves ESTIMATE_FOLLOWUP_CANCELLER to NoopEstimateFollowupCanceller", () => {
          expect(noopCanceller).toBeInstanceOf(NoopEstimateFollowupCanceller);
        });
      });
    });
    ```

    **Fill in the `/* ... */` fixture gaps** using `EstimateRevisionMockGenerator.createBundleLineItemSet(...)` from Task 1 OR use the existing `EstimateLineItemMockGenerator.createLineItemDto(overrides)` with explicit overrides. Match the actual line-item DTO shape.

    **Step 3 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser
    ```

    **Prohibitions:**
    - No `any` in production code (service file).
    - Absolutely no `as` casts in the service except the clearly-documented unavoidable `deletedAt: new Date() as unknown as never` — and prefer calling `repository.softDelete` to eliminate even that.
    - No `eslint-disable` / `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck` in either file.
    - `as unknown as jest.Mocked<X>` IS acceptable in spec files — it's the standard NestJS mocking pattern for Test.createTestingModule with useValue.
    - Path aliases only.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-reviser.service.ts`
    - `test -f trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts`
    - `grep -c "export class EstimateReviser" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 1
    - `grep -c "@Injectable()" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 1
    - `grep -c "@Inject(ESTIMATE_FOLLOWUP_CANCELLER)" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 1
    - `grep -c "downgradeCurrent" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 1
    - `grep -c "insertRevision" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 1
    - `grep -c "restoreCurrent" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 1
    - `grep -c "new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 1
    - `grep -c "REVISABLE_STATUSES\\|SITE_VISIT_REQUESTED" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 2
    - `grep -c "bulkInsertForRevision" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 2 (roots call + children call)
    - `grep -c "EstimateLineItemStatus.PENDING" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns at least 2
    - `grep -c "this.followupCanceller\\." trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 0 (injected but never called — D-HOOK-03)
    - `grep -c "cancelAllFollowups" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 0 (never called from the reviser)
    - `grep -c " any " trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 0
    - `grep -c "D-HOOK-03" trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` returns at least 1
    - `grep -c "cancelAllFollowups.*not.toHaveBeenCalled\\|expect(cancelSpy).not.toHaveBeenCalled" trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` returns at least 1
    - `grep -c "ConflictError" trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` returns at least 3
    - `grep -c "restoreCurrent" trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` returns at least 2
    - `grep -c "it.each" trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` returns at least 2 (parameterized allowed + disallowed status tests)
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` exits 0
    - `cd trade-flow-api && npm run lint:check` exits 0 (confirms `followupCanceller` class-field injection does not trigger unused-var warning)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate-reviser</automated>
  </verify>
  <done>EstimateReviser service exists with the full two-write flow, compensating rollback, D-HOOK-03 non-call compliance, bundle parent/child line-item clone, and 10+ passing test cases covering happy paths, filter-miss 409, rollback paths, DI resolution, and filter shape.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| HTTP-client → EstimateReviser | Attacker crafts a `:id` for an estimate they don't own and tries to revise it |
| service → DB atomicity | The filter shape in step 1 is the correctness guarantee against concurrent revise |
| step2/3 failure → rollback | Partial failure leaves the chain in an inconsistent state if rollback also fails |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-04-01 | IDOR / Elevation of privilege | Trader A revises Trader B's estimate | mitigate | HIGH | Step 0 `estimateRetriever.findByIdOrFail(authUser, id)` applies `EstimatePolicy.canRead`; Step 0 then `accessController.canUpdate(authUser, source)` applies `EstimatePolicy.canUpdate`. Both policies scope by `businessId === authUser.businessId`. Verified by retriever spec (plan 42-05) + manual policy review. |
| T-42-04-02 | Race / Tampering | Two concurrent revise calls both succeed and create duplicate current rows | mitigate | HIGH | Step 1 `findOneAndUpdate` is document-level atomic; only one call matches the filter `{_id, isCurrent: true, status: {$in}}`. The second call sees `isCurrent: false` and returns null → 409. Secondary defense: partial unique index `{rootEstimateId, isCurrent: true}` rejects any duplicate insert at Mongo level. Verified by unit spec filter-shape assertion + manual smoke (`docs/smoke/phase-42-concurrent-revise.md` authored in plan 42-06). |
| T-42-04-03 | Information disclosure | 409 error message leaks which of three conditions tripped (already revised / status change / isCurrent false) | mitigate | LOW | D-CONC-08 specifies an intentionally generic message: "Estimate has already been revised or is no longer revisable". Enforced by plan 42-02's ERRORS_MAP entry; the reviser throws with exactly that string. |
| T-42-04-04 | Chain corruption / Data integrity | Step-3 failure after step-2 succeeded leaves a new row with no line items AND the old row as non-current (if rollback also fails) | mitigate | HIGH | `compensatingRollback(sourceId, newId)` first soft-deletes the new row (freeing the partial unique index), THEN restores the old row's `isCurrent`. Ordering is critical (D-REV-06). Log on rollback failure at error level with runbook comment for manual recovery. Tested by compensating-rollback spec case. |
| T-42-04-05 | DoS | Trader floods `/revisions` with concurrent POSTs to exhaust the DB | accept | LOW | Rate limiting is out of Phase 42 scope. Note as "follow-up hardening". The atomicity guarantee protects data integrity; throughput throttling is a separate concern. |
| T-42-04-06 | D-HOOK-03 drift / Tampering | Future edit to EstimateReviser accidentally calls `cancelAllFollowups`, breaking the "old follow-ups keep firing until new revision is sent" semantic | mitigate | MEDIUM | Spec assertion `expect(cancelSpy).not.toHaveBeenCalled()` runs on every happy-path test; `grep -c cancelAllFollowups` in the source file is asserted = 0 in acceptance_criteria. Any accidental call breaks CI. |
| T-42-04-07 | Line-item orphan / Data integrity | Bundle child is inserted before its parent, or parent is missing from the source set, producing orphaned `parentLineItemId` | mitigate | MEDIUM | `cloneLineItemsForRevision` filters `roots` then `children`, inserts roots FIRST, captures new IDs in `parentRemap`, then inserts children with remapped IDs. If `parentRemap.get` returns undefined for a child's parent id, `InternalServerError` is thrown before any children insert — fail-loud, no orphans. Tested by bundle-parent-child spec case. |
| T-42-04-08 | Mass assignment | Attacker includes a request body trying to set `isCurrent`, `revisionNumber`, or other protected fields | mitigate | HIGH | D-REV-01: endpoint takes no body. Global `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true` (confirm in `main.ts`) strips any body. The controller in plan 42-06 does not read `@Body()`. |

**Verification commands:**
- `grep -c "cancelAllFollowups" trade-flow-api/src/estimate/services/estimate-reviser.service.ts` returns 0 (D-HOOK-03)
- `cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser` exits 0
- Manual smoke procedure in plan 42-06 exercises T-42-04-02 against a real docker-compose MongoDB.
</threat_model>

<verification>
After both tasks complete:

```bash
cd trade-flow-api && npm run test -- --testPathPattern=estimate-reviser
cd trade-flow-api && npm run typecheck
cd trade-flow-api && npm run lint:check
cd trade-flow-api && npm run format:check
```

All four commands must exit 0. Additionally, plans 42-05 and 42-06 cannot run until this plan is green (wave 3 dependency).
</verification>

<success_criteria>
- `EstimateReviser.revise(authUser, id)` implements the full step 0→1→2→3→5 flow with compensating rollback on failure paths.
- D-HOOK-03 is enforced: `followupCanceller` is injected but the service file contains zero `cancelAllFollowups` call sites.
- All 10 spec cases pass: happy path, allowed statuses parameterized, disallowed statuses parameterized, non-call of canceller, rollback on step-2 failure, rollback on step-3 failure, line-item clone with bundle preservation, revision number monotonicity, atomic filter shape, DI resolution.
- `EstimateRevisionMockGenerator` provides fixture builders for roots, child revisions, and bundle line-item sets.
- `npm run ci` exits 0.
- Single commit: `feat(42): add EstimateReviser service with two-write revise flow (Phase 42 wave 3)`.
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-04-SUMMARY.md` documenting:
- The exact revise() flow step-by-step (step 0→5).
- The compensating rollback order (soft-delete new first, then restoreCurrent old).
- Confirmation via grep that `cancelAllFollowups` does not appear in the service source (D-HOOK-03 enforcement).
- Spec test count and pass summary.
- Commit hash.
</output>
