---
phase: 41-estimate-module-crud-backend
plan: 06
type: execute
wave: 5
depends_on: [02, 04, 05]
files_modified:
  - trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts
  - trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts
  - trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts
  - trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-standard-line-item-factory.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-bundle-line-item-factory.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-line-item-creator.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-line-item-retriever.service.spec.ts
autonomous: true
requirements: [EST-03]
must_haves:
  truths:
    - `EstimateStandardLineItemFactory.create(...)` produces a standalone `IEstimateLineItemDto` for a single item (non-bundle) with the line-item fields from the quote-module equivalent
    - `EstimateBundleLineItemFactory.createLineItems(...)` produces the full parent + child line-item set for a bundle item by delegating bundle-config validation, pricing planning, and tax-rate calculation to the shared `@item/services/*` helpers from Plan 02
    - `EstimateLineItemCreator.create(...)` persists line items via `EstimateLineItemRepository` with ownership enforced via `EstimatePolicy` + `AccessControllerFactory`
    - `EstimateLineItemRetriever.findByEstimateId(authUser, estimateId)` returns all non-deleted line items for an estimate after policy check
    - Every factory and service spec mirrors its quote-module equivalent test coverage one-to-one (D-LI-03 side-by-side review)
    - `cd trade-flow-api && npm run ci` exits 0
  artifacts:
    - path: trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts
      provides: Standalone line-item construction from an IItemDto + create request
      contains: "IEstimateLineItemDto"
    - path: trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts
      provides: Bundle parent + children line-item construction
      contains: "BundleConfigValidator"
    - path: trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts
      provides: Creator service with policy-wrapped persistence
      contains: "EstimateLineItemPolicy"
    - path: trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts
      provides: Retriever service for per-estimate line-item lookup
      contains: "findByEstimateId"
  key_links:
    - from: trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts
      to: trade-flow-api/src/item/services/bundle-config-validator.service.ts
      via: "@item/services/bundle-config-validator.service import"
      pattern: "@item/services/bundle-config-validator.service"
    - from: trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts
      to: trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts
      via: "@item/services/bundle-tax-rate-calculator.service import"
      pattern: "@item/services/bundle-tax-rate-calculator.service"
    - from: trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts
      to: trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
      via: constructor injection
      pattern: "EstimateLineItemRepository"
---

<objective>
Implement the four entity-scoped line-item services that the `EstimateUpdater` (Plan 07) and `EstimateController` (Plan 08) will invoke: `EstimateStandardLineItemFactory`, `EstimateBundleLineItemFactory`, `EstimateLineItemCreator`, and `EstimateLineItemRetriever`. Per D-LI-02 / D-LI-03, these are **literal 1:1 mirrors** of the quote-module equivalents with `quote → estimate` / `Quote → Estimate` renames applied to class names, types, collection constants, and test files. The bundle factory imports bundle helpers from `@item/services/*` (Plan 02's refactor output) — never from `@quote/services/*`.

Purpose: Line-item construction is entity-scoped business logic that the user's separation-over-DRY memory mandates stay duplicated rather than abstracted. The bundle-math itself (which IS shared) lives in `@item/services/`. This plan fills the estimate-specific wrapper layer that glues item-level bundle math to the estimate-line-item persistence layer.

Output: Four service files + four spec files. All imports of bundle helpers use `@item/services/*`. The specs mirror the quote test coverage exactly so side-by-side review remains feasible.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files — executor MUST read before writing the estimate mirrors. -->

Factory mirrors:
- trade-flow-api/src/quote/services/quote-standard-line-item-factory.service.ts
- trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts (already updated in Plan 02 to import from @item/services — use that version as the template)
- trade-flow-api/src/quote/test/services/quote-standard-line-item-factory.service.spec.ts
- trade-flow-api/src/quote/test/services/quote-bundle-line-item-factory.service.spec.ts

Line-item creator / retriever mirrors:
- trade-flow-api/src/quote/services/quote-line-item-creator.service.ts
- trade-flow-api/src/quote/services/quote-line-item-retriever.service.ts
- trade-flow-api/src/quote/test/services/quote-line-item-creator.service.spec.ts
- trade-flow-api/src/quote/test/services/quote-line-item-retriever.service.spec.ts

Shared helpers (from Plan 02):
- trade-flow-api/src/item/services/bundle-config-validator.service.ts
- trade-flow-api/src/item/services/bundle-pricing-planner.service.ts
- trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts
- trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts

Phase 41 dependencies:
- trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (Plan 05 Task 1)
- trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts (Plan 05 Task 3)
- trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts (Plan 04 Task 2)
- trade-flow-api/src/estimate/data-transfer-objects/bundle-line-item-group.dto.ts (Plan 04 Task 2)
- trade-flow-api/src/estimate/data-transfer-objects/component-blueprint.dto.ts (Plan 04 Task 2)
- trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts (Plan 04 Task 1)
- trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts (Plan 04 Task 3)
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create EstimateStandardLineItemFactory and EstimateBundleLineItemFactory with specs</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts,
    trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-standard-line-item-factory.service.spec.ts,
    trade-flow-api/src/estimate/test/services/estimate-bundle-line-item-factory.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-standard-line-item-factory.service.ts (source — MUST mirror class shape, method signature, and imports)
    - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts (source — already updated in Plan 02 to import from @item/services; use as template)
    - trade-flow-api/src/quote/test/services/quote-standard-line-item-factory.service.spec.ts (test coverage to replicate)
    - trade-flow-api/src/quote/test/services/quote-bundle-line-item-factory.service.spec.ts
    - trade-flow-api/src/item/services/bundle-config-validator.service.ts (from Plan 02 — signature of validateConfig)
    - trade-flow-api/src/item/services/bundle-pricing-planner.service.ts (from Plan 02)
    - trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts (from Plan 02 — generalised signature accepting ILineItemTaxInput[])
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts (Plan 04)
    - trade-flow-api/src/estimate/data-transfer-objects/bundle-line-item-group.dto.ts (Plan 04)
    - trade-flow-api/src/estimate/data-transfer-objects/component-blueprint.dto.ts (Plan 04)
    - trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts (Plan 04)
  </read_first>
  <behavior>
    - `EstimateStandardLineItemFactory.create(authUser, request, item, estimate): IEstimateLineItemDto` produces a single standalone line item from a standard (non-bundle) item. Mirrors `QuoteStandardLineItemFactory.create` exactly except for type/class/field renames.
    - `EstimateBundleLineItemFactory.createLineItems(authUser, request, item, estimate): IEstimateLineItemDto[]` produces the full parent + child line-item array for a bundle. Mirrors `QuoteBundleLineItemFactory.createLineItems` exactly. Imports `BundleConfigValidator`, `BundlePricingPlanner`, `BundleTaxRateCalculator` from `@item/services/*` (not from `@quote/services/*` — those no longer exist after Plan 02).
    - Bundle factory uses the structural `ILineItemTaxInput` type when passing data to `BundleTaxRateCalculator`. Because `IEstimateLineItemDto` already has `{ unitPrice: Money, quantity: number, taxRate: number }`, structural typing handles the compatibility with zero casts.
    - Both factories produce line items with `status: EstimateLineItemStatus.PENDING` (or APPROVED — check which status the quote factories default to and mirror exactly).
    - Spec coverage mirrors quote factory specs one-for-one: standalone item happy path, bundle happy path with 2+ children, tax-rate propagation, discount handling if present in the quote equivalents.
  </behavior>
  <action>
**Step 1: `estimate-standard-line-item-factory.service.ts`** — open `quote-standard-line-item-factory.service.ts`, COPY the file contents, then apply these global find-and-replace operations within the copy:

- `QuoteStandardLineItemFactory` → `EstimateStandardLineItemFactory`
- `IQuoteLineItemDto` → `IEstimateLineItemDto`
- `IQuoteDto` → `IEstimateDto` (if the factory takes the parent quote as an argument)
- `QuoteLineItemStatus` → `EstimateLineItemStatus`
- `quoteId` → `estimateId` (field on the line-item DTO)
- `quote_line_items` or `quotelineitems` → `estimatelineitems` (if a collection string appears)
- `@quote/data-transfer-objects/quote-line-item.dto` → `@estimate/data-transfer-objects/estimate-line-item.dto`
- `@quote/data-transfer-objects/quote.dto` → `@estimate/data-transfer-objects/estimate.dto`
- `@quote/enums/quote-line-item-status.enum` → `@estimate/enums/estimate-line-item-status.enum`

Everything else — method bodies, Money arithmetic, audit-field application — stays byte-for-byte identical. D-LI-03 explicitly wants side-by-side diff-reviewable files.

If the factory imports `BundleTaxRateCalculator` (unlikely for the standard factory; more likely for the bundle factory), update the import path to `@item/services/bundle-tax-rate-calculator.service`.

**Step 2: `estimate-bundle-line-item-factory.service.ts`** — same process against `quote-bundle-line-item-factory.service.ts` (post-Plan-02 version, which already imports from `@item/services/*`). Apply the same find-and-replace set. The three helper imports should already be `@item/services/...` — verify and do NOT change them.

Key invariants after the rename:
- `import { BundleConfigValidator } from "@item/services/bundle-config-validator.service";`
- `import { BundlePricingPlanner } from "@item/services/bundle-pricing-planner.service";`
- `import { BundleTaxRateCalculator } from "@item/services/bundle-tax-rate-calculator.service";`
- Parent line item and child line items all carry `estimateId: estimate.id`.
- Child line items have `parentLineItemId: parent.id` set.

**Step 3: `estimate-standard-line-item-factory.service.spec.ts`** — copy `quote-standard-line-item-factory.service.spec.ts`, apply the same find-and-replace, and additionally:
- Replace `QuoteMockGenerator` with `EstimateMockGenerator`
- Replace `QuoteLineItemMockGenerator` with `EstimateLineItemMockGenerator`
- Run the spec to confirm every assertion still passes

**Step 4: `estimate-bundle-line-item-factory.service.spec.ts`** — same process against the quote bundle factory spec. The spec will import `BundleConfigValidator` / `BundlePricingPlanner` / `BundleTaxRateCalculator` mocks — verify the mock paths resolve to `@item/services/*`.

**Step 5: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-(standard|bundle)-line-item-factory
```

Fix any failures at the root cause. Common issues after mechanical mirror:
- Leftover `Quote` / `quote` references in test descriptions (mechanical, fix)
- Stale import path (`@quote/test/mocks/quote-mock-generator` instead of `@estimate/test/mocks/estimate-mock-generator`)
- Type mismatch because an assertion checks `line.quoteId` (rename to `line.estimateId`)

Commit message: `feat(41): add EstimateStandardLineItemFactory and EstimateBundleLineItemFactory mirroring quote factories`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts`
    - `test -f trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts`
    - `grep -c "export class EstimateStandardLineItemFactory" trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts` returns 1
    - `grep -c "export class EstimateBundleLineItemFactory" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 1
    - `grep -c "@item/services/bundle-config-validator" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 1
    - `grep -c "@item/services/bundle-pricing-planner" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 1
    - `grep -c "@item/services/bundle-tax-rate-calculator" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 1
    - `grep -rc "@quote/services/bundle-" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 0
    - `grep -c "estimateId" trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts` returns at least 1
    - `grep -c "quoteId" trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts` returns 0
    - `grep -c "estimateId" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns at least 1
    - `grep -c "quoteId" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 0
    - `grep -c "parentLineItemId" trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-(standard|bundle)-line-item-factory` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-(standard|bundle)-line-item-factory</automated>
  </verify>
  <done>Both factories mirror their quote equivalents, import bundle helpers from @item/services, and specs pass</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Create EstimateLineItemCreator and EstimateLineItemRetriever with specs</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts,
    trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-line-item-creator.service.spec.ts,
    trade-flow-api/src/estimate/test/services/estimate-line-item-retriever.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-line-item-creator.service.ts (source — MUST mirror)
    - trade-flow-api/src/quote/services/quote-line-item-retriever.service.ts (source — MUST mirror)
    - trade-flow-api/src/quote/test/services/quote-line-item-creator.service.spec.ts
    - trade-flow-api/src/quote/test/services/quote-line-item-retriever.service.spec.ts
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (Plan 05 — dependency)
    - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts (Plan 05 — dependency)
    - trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts (Task 1)
    - trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts (Task 1)
    - trade-flow-api/src/core/factories/access-controller.factory.ts
    - trade-flow-api/src/core/factories/authorized-creator.factory.ts
  </read_first>
  <behavior>
    - `EstimateLineItemCreator.create(authUser, estimate, item, request): Promise<IEstimateLineItemDto[]>` dispatches on item type: bundle → `EstimateBundleLineItemFactory.createLineItems()`, non-bundle → `EstimateStandardLineItemFactory.create()`. Persists via `EstimateLineItemRepository` through `AuthorizedCreatorFactory` wrapping `EstimateLineItemPolicy`. Mirrors `QuoteLineItemCreator` exactly.
    - `EstimateLineItemRetriever.findByEstimateId(authUser, estimateId): Promise<IEstimateLineItemDto[]>` looks up all line items for an estimate via `EstimateLineItemRepository.findByEstimateId`, filters out `DELETED` status, and applies `AccessControllerFactory` + `EstimateLineItemPolicy` for ownership enforcement (each line item's `businessId` must be in `authUser.businessIds`). Mirrors `QuoteLineItemRetriever` exactly.
    - Spec coverage: bundle path, standard path, ownership-failure path (ForbiddenError), not-found path (ResourceNotFoundError from the repository).
  </behavior>
  <action>
**Step 1: `estimate-line-item-creator.service.ts`** — read `quote-line-item-creator.service.ts`, copy the full file, apply the find-and-replace set from Task 1. Additional specific renames:
- `QuoteBundleLineItemFactory` → `EstimateBundleLineItemFactory`
- `QuoteStandardLineItemFactory` → `EstimateStandardLineItemFactory`
- `QuoteLineItemRepository` → `EstimateLineItemRepository`
- `QuoteLineItemPolicy` → `EstimateLineItemPolicy`
- `QuoteLineItemCreator` → `EstimateLineItemCreator`

Import updates:
- `@quote/services/quote-bundle-line-item-factory.service` → `@estimate/services/estimate-bundle-line-item-factory.service`
- `@quote/services/quote-standard-line-item-factory.service` → `@estimate/services/estimate-standard-line-item-factory.service`
- `@quote/repositories/quote-line-item.repository` → `@estimate/repositories/estimate-line-item.repository`
- `@quote/policies/quote-line-item.policy` → `@estimate/policies/estimate-line-item.policy`
- `@quote/data-transfer-objects/quote-line-item.dto` → `@estimate/data-transfer-objects/estimate-line-item.dto`
- `@quote/data-transfer-objects/quote.dto` → `@estimate/data-transfer-objects/estimate.dto`

Method bodies stay identical. If the creator uses `AuthorizedCreatorFactory.create(policy)` to wrap the repository, the same call pattern applies — just with the new policy instance.

**Step 2: `estimate-line-item-retriever.service.ts`** — same process against `quote-line-item-retriever.service.ts`. The `findByEstimateId(...)` method mirrors the quote's `findByQuoteId(...)` equivalent. Rename method if the quote version is called `findByQuoteId` → `findByEstimateId`.

**Step 3: `estimate-line-item-creator.service.spec.ts`** — copy quote equivalent, apply renames. Key assertions to preserve:

```typescript
it("dispatches to bundle factory when item.type is BUNDLE", async () => {
  const bundleItem = ItemMockGenerator.createItemDto({ type: ItemType.BUNDLE });
  const estimate = EstimateMockGenerator.createEstimateDto();
  await creator.create(authUser, estimate, bundleItem, request);
  expect(mockBundleFactory.createLineItems).toHaveBeenCalled();
  expect(mockStandardFactory.create).not.toHaveBeenCalled();
});

it("dispatches to standard factory when item.type is STANDARD/PRODUCT/SERVICE", async () => {
  // ... mirror quote equivalent
});

it("persists created line items via EstimateLineItemRepository", async () => {
  // ... mirror quote equivalent
});

it("enforces ownership via EstimateLineItemPolicy — throws on cross-business", async () => {
  // ... mirror quote equivalent (if present)
});
```

**Step 4: `estimate-line-item-retriever.service.spec.ts`** — same process. Preserve the test for DELETED status filtering:

```typescript
it("filters out DELETED line items from the result", async () => {
  mockRepo.findByEstimateId.mockResolvedValue([
    EstimateLineItemMockGenerator.createEstimateLineItemDto({ status: EstimateLineItemStatus.PENDING }),
    EstimateLineItemMockGenerator.createEstimateLineItemDto({ status: EstimateLineItemStatus.DELETED }),
    EstimateLineItemMockGenerator.createEstimateLineItemDto({ status: EstimateLineItemStatus.APPROVED }),
  ]);
  const result = await retriever.findByEstimateId(authUser, "some-id");
  expect(result).toHaveLength(2);
  expect(result.every((li) => li.status !== EstimateLineItemStatus.DELETED)).toBe(true);
});
```

(Verify the quote equivalent does this filtering — if it filters at the repository level instead, mirror that; if it filters at the service level, mirror that. Do NOT invent new behavior.)

**Step 5: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item-(creator|retriever)
```

Commit message: `feat(41): add EstimateLineItemCreator and EstimateLineItemRetriever mirroring quote services`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts`
    - `test -f trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts`
    - `grep -c "export class EstimateLineItemCreator" trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts` returns 1
    - `grep -c "export class EstimateLineItemRetriever" trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts` returns 1
    - `grep -c "EstimateBundleLineItemFactory\\|EstimateStandardLineItemFactory" trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts` returns at least 2
    - `grep -c "findByEstimateId" trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts` returns at least 1
    - `grep -rc "@quote/services/quote-line-item-\\|@quote/repositories/quote-line-item\\|@quote/policies/quote-line-item" trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts` returns 0
    - `grep -c "quoteId\\|QuoteLineItem" trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts` returns 0 (no leftover quote references)
    - `grep -c "dispatches to bundle factory" trade-flow-api/src/estimate/test/services/estimate-line-item-creator.service.spec.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item-(creator|retriever)` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-line-item-(creator|retriever)</automated>
  </verify>
  <done>Line-item creator and retriever mirror quote equivalents with specs passing</done>
</task>

<task type="auto">
  <name>Task 3: Run full CI gate to confirm Wave 5 is non-regressive</name>
  <files>none</files>
  <read_first>- trade-flow-api/package.json</read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected: exit 0.

Common failure: a stale `@quote/services/bundle-*` import — this should have been caught by Plan 02's CI gate, but double-check. Run `grep -rn "@quote/services/bundle-" trade-flow-api/src` — must return empty.

No suppressions.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - `grep -rn "@quote/services/bundle-" trade-flow-api/src` returns 0 matches (Plan 02 regression check)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green after Wave 5</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| EstimateLineItemCreator ← authUser | Creator wraps repository with AuthorizedCreatorFactory + EstimateLineItemPolicy |
| EstimateLineItemRetriever ← authUser | Retriever applies EstimateLineItemPolicy via AccessControllerFactory |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain. No new threat surface — factories and creator/retriever mirror existing quote controls.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-06-01 | Elevation of Privilege | Line items created under wrong businessId by cross-business user | mitigate | `EstimateLineItemCreator` uses `AuthorizedCreatorFactory.create(estimateLineItemPolicy)` which enforces `authUser.businessIds.includes(resource.businessId)` before persist; mirror of quote pattern |
| T-41-06-02 | Tampering | Stale bundle helper reference (Pitfall 7) causes DI failure at runtime | mitigate | Plan 02's full CI gate already proved the refactor is stable; this plan's Task 1 acceptance criteria grep for `@quote/services/bundle-` to catch any regression |
| T-41-06-03 | Information Disclosure | Retriever returns DELETED line items (historical) to caller | mitigate | Retriever filters by `status !== DELETED` before returning; spec asserts this behavior |
| T-41-06-04 | Tampering | Bundle math drift from quote equivalent | mitigate | Bundle math is centralised in `@item/services/*`; both quote and estimate factories call the same helpers; drift impossible by construction |
| T-41-06-05 | Input Validation | ILineItemTaxInput structural type accepts arbitrary object | mitigate | Factory argument is constructed INSIDE the service (not from user input); inputs come from `IEstimateLineItemDto` built by the factory from validated request fields |
</threat_model>

<verification>
1. Four service files and four spec files exist under `src/estimate/services/` and `src/estimate/test/services/`.
2. Bundle factory imports from `@item/services/*` exclusively.
3. No `@quote/services/bundle-*` references in the estimate module.
4. `cd trade-flow-api && npm run ci` exits 0.
</verification>

<success_criteria>
- Plan 07 can inject `EstimateLineItemCreator` and `EstimateLineItemRetriever` into the estimate updater for Draft-only line-item CRUD
- Plan 08 can register these services in `EstimateModule` providers
- Line-item construction semantics are byte-for-byte equivalent to the quote module modulo renames (D-LI-03 side-by-side review feasible)
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-06-SUMMARY.md` documenting:
- The four service files created
- Confirmation that `@item/services/bundle-*` imports are used throughout
- The full `npm run ci` output
</output>
