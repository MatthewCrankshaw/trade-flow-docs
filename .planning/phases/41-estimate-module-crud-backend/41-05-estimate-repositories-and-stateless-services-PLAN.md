---
phase: 41-estimate-module-crud-backend
plan: 05
type: execute
wave: 4
depends_on: [01, 04]
files_modified:
  - trade-flow-api/src/estimate/repositories/estimate.repository.ts
  - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
  - trade-flow-api/src/estimate/services/estimate-number-generator.service.ts
  - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
  - trade-flow-api/src/estimate/services/estimate-transition.service.ts
  - trade-flow-api/src/estimate/policies/estimate.policy.ts
  - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
  - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
  - trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-number-generator.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-totals-calculator.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts
  - trade-flow-api/src/estimate/test/policies/estimate.policy.spec.ts
  - trade-flow-api/src/estimate/test/policies/estimate-line-item.policy.spec.ts
autonomous: true
requirements: [EST-02, EST-03, EST-04, EST-05, EST-07, EST-08, CONT-02, CONT-05, RESP-08]
must_haves:
  truths:
    - `EstimateRepository.COLLECTION === "estimates"` and `EstimateLineItemRepository.COLLECTION === "estimatelineitems"` (no underscore, per D-ENT-02)
    - `EstimateRepository` creates required indexes: `{ businessId: 1, createdAt: -1 }`, `{ jobId: 1, createdAt: -1 }`, and partial-unique `{ businessId: 1, number: 1 }` on `deletedAt: null` (per D-ENT-03)
    - `EstimateRepository.toEntity` / `toDto` correctly round-trip all Phase 41 and reserved fields from CONTEXT.md §specifics (including `lastResponseType`, `uncertaintyReasons`, `parentEstimateId`)
    - `EstimateLineItemRepository.toEntity` / `toDto` correctly round-trip all line-item fields including `parentLineItemId` and Money ↔ number minor-unit conversion
    - `EstimateNumberGenerator.generateNumber(businessId)` returns a string matching `/^E-\d{4}-\d{3}$/` and uses atomic `findOneAndUpdate` + `$inc` + `upsert: true` against `estimate_counters`
    - Parameterised tests prove `EstimateTotalsCalculator` produces penny-exact `high = low × (1 + c/100)` for every allowed contingency and a varied subtotal set (including £0.01, £99.99, £1234.56)
    - `EstimateTransitionService.transition()` rejects invalid transitions with `ErrorCodes.ESTIMATE_INVALID_TRANSITION`, sets the correct timestamp per target status (`SENT → sentAt`, `DELETED → deletedAt`, etc.), and refuses to overwrite `firstViewedAt` once set (Pitfall 9)
    - `EstimatePolicy.canWrite` returns true only when `authUser.businessIds` includes the resource's `businessId`; status-based gating is NOT checked in the policy (D-DRAFT-02)
    - `cd trade-flow-api && npm run ci` exits 0
  artifacts:
    - path: trade-flow-api/src/estimate/repositories/estimate.repository.ts
      provides: EstimateRepository with CRUD + paginated list + createIndexes
      contains: "estimates"
    - path: trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
      provides: EstimateLineItemRepository with CRUD by estimateId
      contains: "estimatelineitems"
    - path: trade-flow-api/src/estimate/services/estimate-number-generator.service.ts
      provides: EstimateNumberGenerator atomic counter
      contains: "estimate_counters"
    - path: trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
      provides: EstimateTotalsCalculator with priceRange computation
      contains: "priceRange"
    - path: trade-flow-api/src/estimate/services/estimate-transition.service.ts
      provides: EstimateTransitionService state machine enforcement
      contains: "ESTIMATE_INVALID_TRANSITION"
    - path: trade-flow-api/src/estimate/policies/estimate.policy.ts
      provides: EstimatePolicy business-ownership guard
      contains: "businessIds"
  key_links:
    - from: trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
      to: trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts
      via: import + construction
      pattern: "IEstimatePriceRangeDto"
    - from: trade-flow-api/src/estimate/services/estimate-transition.service.ts
      to: trade-flow-api/src/estimate/enums/estimate-transitions.ts
      via: "isValidTransition"
      pattern: "isValidTransition"
---

<objective>
Implement the stateless / data-layer services of the estimate module: both repositories (with index creation and paginated list support), the `EstimateNumberGenerator`, the `EstimateTotalsCalculator` with range math, the `EstimateTransitionService` state-machine enforcer, and the two `EstimatePolicy` / `EstimateLineItemPolicy` classes. Every source file lands with its companion spec in the same commit so CI stays green wave-by-wave.

Purpose: These components have no dependencies on creator/retriever/updater services — they are the primitives Plans 06 and 07 compose. Building them first with full test coverage lets Plan 06/07 rely on them without mocking complex state-machine or math logic. This plan explicitly handles the pagination pattern gap flagged in RESEARCH.md Pitfall 6 — pagination is built from scratch in `EstimateRepository`, not copied from `QuoteRepository.findAllByBusinessId` (which is a flat-array method).

Output: Two repositories, three stateless services, two policies, and seven spec files proving correctness of counter math, range math, transition rules, policy scoping, and repository round-trip mapping.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files the executor MUST read before implementing. -->

Repository pattern:
- trade-flow-api/src/quote/repositories/quote.repository.ts (COLLECTION="quotes", findByIdOrFail, create, update, findAllByBusinessId)
- trade-flow-api/src/quote/repositories/quote-line-item.repository.ts (COLLECTION="quotelineitems", per-quote queries)

Pagination infrastructure to read:
- trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts (confirms whether findMany supports { skip, limit, sort })
- trade-flow-api/src/core/services/mongo/mongo-connection.service.ts (bypass path if MongoDbFetcher doesn't natively paginate)
- trade-flow-api/src/core/collections/dto.collection.ts (DtoCollection.create(data, queryResults, queryOptions))
- trade-flow-api/src/core/data-transfer-objects/query-results.dto.ts (IQueryResultsDto shape for pagination metadata)

Counter pattern (literal mirror):
- trade-flow-api/src/quote/services/quote-number-generator.service.ts (findOneAndUpdate + $inc + upsert + collection "quote_counters")

Totals calculator pattern (extended):
- trade-flow-api/src/quote/services/quote-totals-calculator.service.ts (iterates non-parent non-deleted line items, applies per-line taxRate)

Transition service pattern:
- trade-flow-api/src/quote/services/quote-transition.service.ts (validates, sets timestamps, calls repository.update)
- trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts (TestingModule pattern)

Policy pattern:
- trade-flow-api/src/quote/policies/quote.policy.ts (extends BasePolicy<IQuoteDto>, canRead/canCreate/canUpdate=true for same-business, canDelete=false)
- trade-flow-api/src/quote/policies/quote-line-item.policy.ts
- trade-flow-api/src/core/policies/base.policy.ts (abstract base)

Error throwing:
- trade-flow-api/src/core/errors/invalid-request.error.ts (for ESTIMATE_INVALID_TRANSITION throws)
- trade-flow-api/src/core/errors/resource-not-found.error.ts
- trade-flow-api/src/core/errors/error-codes.enum.ts (new ESTIMATE_* codes added in Plan 01 Task 3)
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Implement EstimateRepository and EstimateLineItemRepository with specs</name>
  <files>
    trade-flow-api/src/estimate/repositories/estimate.repository.ts,
    trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts,
    trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts,
    trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/repositories/quote.repository.ts (MUST read — pattern to mirror including toDto/toEntity, index creation approach, findAllByBusinessId filter)
    - trade-flow-api/src/quote/repositories/quote-line-item.repository.ts (per-parent query filter pattern)
    - trade-flow-api/src/quote/test/repositories/quote.repository.spec.ts (repository spec pattern with mocked MongoDbFetcher/Writer)
    - trade-flow-api/src/quote/test/repositories/quote-line-item.repository.spec.ts
    - trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts (verify findMany signature for pagination — see RESEARCH.md Pattern 6)
    - trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts
    - trade-flow-api/src/core/services/mongo/mongo-connection.service.ts
    - trade-flow-api/src/core/collections/dto.collection.ts
    - trade-flow-api/src/estimate/entities/estimate.entity.ts (from Plan 04 — field list for toEntity mapping)
    - trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts (from Plan 04 — test fixtures)
    - trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts
  </read_first>
  <behavior>
    - `EstimateRepository` is a single `@Injectable()` class extending the existing repository base (or none — mirror `QuoteRepository`).
    - `COLLECTION = "estimates"` (static readonly).
    - Methods: `create(dto): Promise<IEstimateDto>`, `findByIdOrFail(id): Promise<IEstimateDto>`, `update(dto): Promise<IEstimateDto>`, `findPaginatedByBusinessId(businessId, status?, limit, offset): Promise<{ items: IEstimateDto[]; pagination: IQueryResultsDto }>`, `findAllByBusinessId(businessId)` (unpaginated convenience for internal use — mirrors `QuoteRepository.findAllByBusinessId` for consumers that don't need pagination).
    - `createIndexes()` method (called from `onModuleInit()` — mirror the `SubscriptionRepository` pattern, NOT the quote pattern which has no index bootstrap) creates the **five** indexes: Phase 41's `{businessId, createdAt: -1}`, `{jobId, createdAt: -1}`, and `{businessId, number}` partial unique (now filtered on `{deletedAt: null, isCurrent: true}` per Phase 42 D-CHAIN-06), plus Phase 42's new `{rootEstimateId, isCurrent}` partial unique and `{rootEstimateId, revisionNumber}` sort index. `EstimateRepository` MUST declare `implements OnModuleInit` and inject `MongoConnectionService` via constructor. Rationale: folded forward from Phase 42 D-CHAIN-03/05/06 because Phase 41 has not executed — this avoids a runtime drop-and-recreate.
    - `toEntity(dto)` converts `DateTime` → `Date` via `@core/utilities/to-date-time.utility`, converts `string` id → `ObjectId`, and does NOT write `firstViewedAt: null` when undefined (Pitfall 8).
    - `toDto(entity)` converts `Date` → `DateTime`, `ObjectId` → hex string, and attaches `lineItems: DtoCollection.empty()`, `totals: { zero Money }`, `priceRange: { zero bounds }`, `responseSummary: null` placeholders (the creator / retriever / totals-calculator populate these on the service layer).
    - `EstimateLineItemRepository.COLLECTION = "estimatelineitems"` (no underscore — D-ENT-02).
    - `EstimateLineItemRepository` provides: `create(dto)`, `findByIdOrFail(id)`, `findByEstimateId(estimateId)`, `update(dto)`, `updateMany(dtos)` (bulk for line-item rewrites).
    - Spec files use `@nestjs/testing` TestingModule; mock `MongoDbFetcher`, `MongoDbWriter`, `MongoConnectionService` as needed. Mirror `quote.repository.spec.ts` pattern exactly.
  </behavior>
  <action>
**Step 1: `estimate.repository.ts`** — mirror `quote.repository.ts` with the following concrete rewrites:

Collection constant:
```typescript
private static readonly COLLECTION = "estimates";
```

Field mapping in `toEntity`:
- `businessId: new ObjectId(dto.businessId)`
- `customerId: new ObjectId(dto.customerId)`
- `jobId: new ObjectId(dto.jobId)`
- `parentEstimateId: dto.parentEstimateId ? new ObjectId(dto.parentEstimateId) : null`
- `rootEstimateId: dto.rootEstimateId ? new ObjectId(dto.rootEstimateId) : null`
- `convertedToQuoteId: dto.convertedToQuoteId ? new ObjectId(dto.convertedToQuoteId) : undefined`
- `estimateDate: dto.estimateDate.toJSDate()`
- All optional DateTime fields: use `toOptionalDateTime` from `@core/utilities/to-date-time.utility` (or the inverse `fromOptionalDateTime` if that's the convention — check quote.repository.ts)
- `firstViewedAt`: ONLY include in the entity if `dto.firstViewedAt !== undefined` (Pitfall 8 — do not set to `null` or `undefined` explicitly on create)
- `contingencyPercent`, `displayMode`, `revisionNumber`, `isCurrent`, `uncertaintyReasons`, `uncertaintyNotes`, `lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason`, `siteVisitAvailability`, `number`, `title`, `notes`, `status` — pass through directly.

Field mapping in `toDto`:
- String conversion for all ObjectId fields: `entity._id.toHexString()`, etc.
- DateTime conversion for all Date fields using `toDateTime` / `toOptionalDateTime`
- `lineItems: DtoCollection.empty()` (service layer replaces this)
- `totals: { subTotal: Money.zero(DEFAULT_CURRENCY), taxTotal: Money.zero(DEFAULT_CURRENCY), total: Money.zero(DEFAULT_CURRENCY) }` (service layer replaces)
- `priceRange: { displayMode: entity.displayMode, subtotal: Money.zero(DEFAULT_CURRENCY), contingencyPercent: entity.contingencyPercent, low: { exclTax: Money.zero(...), tax: Money.zero(...), inclTax: Money.zero(...) }, high: same }` (service layer replaces)
- `responseSummary: null`

Index creation — mirror how `QuoteRepository` handles it. Typical pattern (confirm by reading `quote.repository.ts`):

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

> NOTE (Phase 42 D-CHAIN-07): `QuoteRepository` does NOT create indexes. The blessed precedent is `SubscriptionRepository` which implements `OnModuleInit` and calls `ensureIndexes()` from the lifecycle hook. `EstimateRepository` MUST follow the same pattern: declare `implements OnModuleInit`, inject `MongoConnectionService` in the constructor, and call `await this.createIndexes()` from `onModuleInit()`. Do NOT create indexes in the constructor — the `MongoConnectionService` may not be ready yet.

Pagination method (`findPaginatedByBusinessId`) — per RESEARCH.md Pattern 6:
```typescript
public async findPaginatedByBusinessId(
  businessId: string,
  status: EstimateStatus | undefined,
  limit: number,
  offset: number,
): Promise<{ items: IEstimateDto[]; pagination: IQueryResultsDto }> {
  const db = await this.connection.getDb();
  const collection = db.collection<IEstimateEntity>(EstimateRepository.COLLECTION);

  const filter: Filter<IEstimateEntity> = {
    businessId: new ObjectId(businessId),
    deletedAt: null,
    ...(status ? { status } : {}),
  };

  const total = await collection.countDocuments(filter);
  const cursor = collection
    .find(filter)
    .sort({ createdAt: -1 })
    .skip(offset)
    .limit(limit);
  const entities = await cursor.toArray();

  return {
    items: entities.map((e) => this.toDto(e)),
    pagination: { total, limit, offset },
  };
}
```

If `MongoDbFetcher.findMany` natively supports `{ skip, limit, sort }`, prefer that over direct `db.collection(...)` access. If it does not, bypass via `MongoConnectionService` as shown above — this is the same pattern used by `QuoteNumberGenerator` for its atomic `findOneAndUpdate`.

**Step 2: `estimate-line-item.repository.ts`** — mirror `quote-line-item.repository.ts`:

```typescript
private static readonly COLLECTION = "estimatelineitems";
```

Methods:
- `create(dto: IEstimateLineItemDto): Promise<IEstimateLineItemDto>`
- `findByIdOrFail(id: string): Promise<IEstimateLineItemDto>`
- `findByEstimateId(estimateId: string): Promise<IEstimateLineItemDto[]>` — filter `{ estimateId: new ObjectId(estimateId) }`, sort by `createdAt: 1` (insertion order)
- `update(dto: IEstimateLineItemDto): Promise<IEstimateLineItemDto>`

`toEntity` / `toDto`: identical shape to `QuoteLineItemRepository` with `quoteId → estimateId` rename. Money fields stored as minor units (integer) in MongoDB and converted via `Money.fromMinorUnits(n, DEFAULT_CURRENCY)` in `toDto`.

**Step 3: `estimate.repository.spec.ts`** — mirror `quote.repository.spec.ts` adapting all class / type / collection-constant assertions. Must-have assertions:

```typescript
it("uses 'estimates' as the collection name", () => {
  // verify by either accessing the static or verifying the mock call
  expect((EstimateRepository as unknown as { COLLECTION: string }).COLLECTION ?? "estimates").toBe("estimates");
});
// PREFERRED (no cast): expose via a public readonly getter or verify via the mock fetcher arg.

it("round-trips all Phase 41 and reserved fields through toEntity / toDto", () => {
  const dto = EstimateMockGenerator.createEstimateDto({
    uncertaintyReasons: ["site_inspection", "materials"],
    uncertaintyNotes: "loft insulation unknown",
    lastResponseType: EstimateResponseType.DECLINE,
    lastResponseMessage: "too expensive",
    declineReason: EstimateDeclineReason.TOO_EXPENSIVE,
  });
  const entity = repo.toEntity(dto);
  const roundTripped = repo.toDto(entity);
  expect(roundTripped.uncertaintyReasons).toEqual(["site_inspection", "materials"]);
  expect(roundTripped.lastResponseType).toBe(EstimateResponseType.DECLINE);
  // ... other fields
});

it("never writes firstViewedAt on create when not provided (Pitfall 8)", () => {
  const dto = EstimateMockGenerator.createEstimateDto();
  expect(dto.firstViewedAt).toBeUndefined();
  const entity = repo.toEntity(dto);
  expect(entity.firstViewedAt).toBeUndefined();
  expect("firstViewedAt" in entity && entity.firstViewedAt === null).toBe(false);
});
```

The `as unknown as` pattern in the first assertion is forbidden — use a public getter or verify via constructor-injected MongoDbFetcher mock call args. Prefer the mock-call-args path:

```typescript
expect(mockFetcher.findById).toHaveBeenCalledWith("estimates", expect.anything());
```

**Step 4: `estimate-line-item.repository.spec.ts`** — mirror `quote-line-item.repository.spec.ts` with:

```typescript
it("uses 'estimatelineitems' (no underscore) as the collection name", () => {
  expect(mockFetcher.findMany).toHaveBeenCalledWith("estimatelineitems", expect.anything());
});
```

**Step 5: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/repositories
```

Commit message: `feat(41): add EstimateRepository and EstimateLineItemRepository with specs`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/repositories/estimate.repository.ts`
    - `test -f trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts`
    - `grep -c "\"estimates\"" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "\"estimatelineitems\"" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns at least 1
    - `grep -c "\"estimate_line_items\"" trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 0 (NOT with underscore)
    - `grep -c "\"quotes\"\\|\"quotelineitems\"" trade-flow-api/src/estimate/repositories/estimate.repository.ts trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts` returns 0
    - `grep -c "findPaginatedByBusinessId" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 2 (definition + implementation)
    - `grep -c "createIndex" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 5
    - `grep -c "partialFilterExpression" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 2
    - `grep -c "rootEstimateId: 1, isCurrent: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "rootEstimateId: 1, revisionNumber: 1" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "deletedAt: null, isCurrent: true" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "implements OnModuleInit" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns 1
    - `grep -c "onModuleInit" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "{ businessId: 1, number: 1 }" trade-flow-api/src/estimate/repositories/estimate.repository.ts` returns at least 1
    - `grep -c "never writes firstViewedAt on create when not provided" trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` returns 1
    - `grep -c "estimatelineitems" trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/repositories` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/repositories/` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/repositories</automated>
  </verify>
  <done>Both repositories exist, specs pass, indexes are declared, no type suppressions</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Implement EstimateNumberGenerator and EstimateTotalsCalculator with specs</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-number-generator.service.ts,
    trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-number-generator.service.spec.ts,
    trade-flow-api/src/estimate/test/services/estimate-totals-calculator.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-number-generator.service.ts (exact pattern — MUST be literally mirrored)
    - trade-flow-api/src/quote/services/quote-totals-calculator.service.ts (base pattern to extend with range math)
    - trade-flow-api/src/quote/test/services/quote-number-generator.service.spec.ts
    - trade-flow-api/src/quote/test/services/quote-totals-calculator.service.spec.ts
    - trade-flow-api/src/core/value-objects/money.value-object.ts (confirm `multiply(n: number)` signature and minor-unit precision)
    - trade-flow-api/src/core/services/mongo/mongo-connection.service.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts (from Plan 04)
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-totals.dto.ts (from Plan 04)
    - trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts (from Plan 04 — DELETED value for filter)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Pattern 1 EstimateNumberGenerator; Pattern 2 EstimateTotalsCalculator with verbatim code)
  </read_first>
  <behavior>
    - `EstimateNumberGenerator.generateNumber(businessId: string): Promise<string>` returns a string matching `/^E-\d{4}-\d{3}$/`.
    - The counter collection is `"estimate_counters"` (with underscore, per D-ENT-02).
    - Uses `findOneAndUpdate` + `$inc: { lastNumber: 1 }` + `upsert: true` + `returnDocument: "after"`.
    - Spec mocks `MongoConnectionService.getDb()` and verifies the exact collection name, filter, update operator, and options.
    - Spec parameterised test: 10 sequential calls for the same `{ businessId, year }` produce `E-YYYY-001` through `E-YYYY-010`.
    - `EstimateTotalsCalculator.calculateTotals(estimate: IEstimateDto): IEstimateDto` iterates `estimate.lineItems`, skips parent-scoped (where `parentLineItemId` is set) and DELETED status items, sums `unitPrice × quantity` into `subTotal` and `unitPrice × quantity × taxRate%` into `taxTotal`, then builds `totals` and `priceRange` per D-CONT-01.
    - Price range low = `{ exclTax: subTotal, tax: taxTotal, inclTax: subTotal + taxTotal }`.
    - Price range high = `{ exclTax: subTotal × (1 + c/100), tax: taxTotal × (1 + c/100), inclTax: sum }`.
    - Parameterised spec covers `contingencyPercent ∈ [0, 5, 10, 15, 20, 25, 30]` across varied subtotals (£0.01, £99.99, £1234.56, £10000.00) asserting `high = low × multiplier` penny-exact.
    - Spec asserts that when `contingencyPercent === 0`, `priceRange.low` equals `priceRange.high` value-for-value.
    - Spec asserts that `displayMode` from the input estimate is carried through to `priceRange.displayMode` regardless of mode.
    - Spec asserts zero line items produces `totals = { zero, zero, zero }` and `priceRange.low = priceRange.high = zero bounds`.
  </behavior>
  <action>
**Step 1: `estimate-number-generator.service.ts`** — verbatim from RESEARCH.md §Code Examples → Pattern 1 (lines 373-407):

```typescript
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { MongoConnectionService } from "@core/services/mongo/mongo-connection.service";
import { ObjectId } from "mongodb";

interface IEstimateCounter {
  _id: ObjectId;
  businessId: ObjectId;
  year: number;
  lastNumber: number;
}

@Injectable()
export class EstimateNumberGenerator {
  private static readonly COLLECTION = "estimate_counters";

  constructor(private readonly connection: MongoConnectionService) {}

  public async generateNumber(businessId: string): Promise<string> {
    const year = DateTime.now().year;
    const db = await this.connection.getDb();

    const result = await db
      .collection<IEstimateCounter>(EstimateNumberGenerator.COLLECTION)
      .findOneAndUpdate(
        { businessId: new ObjectId(businessId), year },
        { $inc: { lastNumber: 1 } },
        { upsert: true, returnDocument: "after" },
      );

    const lastNumber = result?.lastNumber ?? 1;
    return `E-${year}-${String(lastNumber).padStart(3, "0")}`;
  }
}
```

**Step 2: `estimate-number-generator.service.spec.ts`** — mirror `quote-number-generator.service.spec.ts`. Mock `MongoConnectionService.getDb()` to return a chainable object with `collection().findOneAndUpdate()` stub. Assertions:

```typescript
it("returns a number matching /^E-\\d{4}-\\d{3}$/", async () => {
  mockCollection.findOneAndUpdate.mockResolvedValue({ lastNumber: 42 });
  const result = await generator.generateNumber("507f1f77bcf86cd799439011");
  expect(result).toMatch(/^E-\d{4}-\d{3}$/);
  expect(result).toBe(`E-${new Date().getFullYear()}-042`);
});

it("uses estimate_counters collection and atomic $inc upsert", async () => {
  mockCollection.findOneAndUpdate.mockResolvedValue({ lastNumber: 1 });
  await generator.generateNumber("507f1f77bcf86cd799439011");
  expect(mockDb.collection).toHaveBeenCalledWith("estimate_counters");
  expect(mockCollection.findOneAndUpdate).toHaveBeenCalledWith(
    expect.objectContaining({ year: expect.any(Number) }),
    { $inc: { lastNumber: 1 } },
    { upsert: true, returnDocument: "after" },
  );
});

it("sequential calls produce sequential numbers", async () => {
  let counter = 0;
  mockCollection.findOneAndUpdate.mockImplementation(async () => ({ lastNumber: ++counter }));
  const results = [];
  for (let i = 0; i < 10; i++) {
    results.push(await generator.generateNumber("507f1f77bcf86cd799439011"));
  }
  expect(results).toEqual([
    "E-2026-001", "E-2026-002", "E-2026-003", "E-2026-004", "E-2026-005",
    "E-2026-006", "E-2026-007", "E-2026-008", "E-2026-009", "E-2026-010",
  ].map((s) => s.replace("2026", String(new Date().getFullYear()))));
});
```

**Step 3: `estimate-totals-calculator.service.ts`** — verbatim from RESEARCH.md §Code Examples → Pattern 2 (lines 418-478):

```typescript
import { DEFAULT_CURRENCY } from "@core/constants/currency.constant";
import { Money } from "@core/value-objects/money.value-object";
import { Injectable } from "@nestjs/common";
import { IEstimateTotalsDto } from "@estimate/data-transfer-objects/estimate-totals.dto";
import { IEstimatePriceRangeDto } from "@estimate/data-transfer-objects/estimate-price-range.dto";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";

@Injectable()
export class EstimateTotalsCalculator {
  public calculateTotals(estimate: IEstimateDto): IEstimateDto {
    let subTotal = Money.zero(DEFAULT_CURRENCY);
    let taxTotal = Money.zero(DEFAULT_CURRENCY);

    for (const lineItem of estimate.lineItems) {
      if (lineItem.parentLineItemId) continue;
      if (lineItem.status === EstimateLineItemStatus.DELETED) continue;

      const lineItemTotal = lineItem.unitPrice.multiply(lineItem.quantity);
      const lineItemTax = lineItemTotal.percentage(lineItem.taxRate);

      subTotal = subTotal.add(lineItemTotal);
      taxTotal = taxTotal.add(lineItemTax);
    }

    const totals: IEstimateTotalsDto = {
      subTotal,
      taxTotal,
      total: subTotal.add(taxTotal),
    };

    const priceRange = this.buildPriceRange(estimate, totals);

    return { ...estimate, totals, priceRange };
  }

  private buildPriceRange(estimate: IEstimateDto, totals: IEstimateTotalsDto): IEstimatePriceRangeDto {
    const multiplier = 1 + estimate.contingencyPercent / 100;
    const highSubTotal = totals.subTotal.multiply(multiplier);
    const highTaxTotal = totals.taxTotal.multiply(multiplier);

    return {
      displayMode: estimate.displayMode,
      subtotal: totals.subTotal,
      contingencyPercent: estimate.contingencyPercent,
      low: {
        exclTax: totals.subTotal,
        tax: totals.taxTotal,
        inclTax: totals.subTotal.add(totals.taxTotal),
      },
      high: {
        exclTax: highSubTotal,
        tax: highTaxTotal,
        inclTax: highSubTotal.add(highTaxTotal),
      },
    };
  }
}
```

Note: `estimate.lineItems` is a `DtoCollection<IEstimateLineItemDto>` — ensure iteration works by checking if `DtoCollection` is iterable (it likely exposes `.data` or is `Iterable` — verify by reading `dto.collection.ts`). If iteration requires `.data`, update the for-loop to `for (const lineItem of estimate.lineItems.data)` or similar.

**Step 4: `estimate-totals-calculator.service.spec.ts`** — parameterised test per RESEARCH.md Pitfall 2:

```typescript
describe("EstimateTotalsCalculator", () => {
  let calculator: EstimateTotalsCalculator;
  beforeEach(() => { calculator = new EstimateTotalsCalculator(); });

  const multiplierCases = [
    { c: 0, multiplier: 1.0 },
    { c: 5, multiplier: 1.05 },
    { c: 10, multiplier: 1.10 },
    { c: 15, multiplier: 1.15 },
    { c: 20, multiplier: 1.20 },
    { c: 25, multiplier: 1.25 },
    { c: 30, multiplier: 1.30 },
  ];

  const subtotalCases = [
    { label: "£0.01", minorUnits: 1 },
    { label: "£99.99", minorUnits: 9999 },
    { label: "£1234.56", minorUnits: 123456 },
    { label: "£10000.00", minorUnits: 1000000 },
  ];

  for (const { c, multiplier } of multiplierCases) {
    for (const { label, minorUnits } of subtotalCases) {
      it(`high = low × ${multiplier} penny-exact at ${label} with ${c}% contingency`, () => {
        const estimate = EstimateMockGenerator.createEstimateDto({
          contingencyPercent: c,
          lineItems: DtoCollection.create(
            [EstimateLineItemMockGenerator.createEstimateLineItemDto({
              unitPrice: Money.fromMinorUnits(minorUnits, DEFAULT_CURRENCY),
              quantity: 1,
              taxRate: 0,
              status: EstimateLineItemStatus.APPROVED,
            })],
            { total: 1, limit: 10, offset: 0 },
          ),
        });
        const result = calculator.calculateTotals(estimate);
        expect(result.priceRange.low.exclTax.toMinorUnits()).toBe(minorUnits);
        expect(result.priceRange.high.exclTax.toMinorUnits()).toBe(Math.round(minorUnits * multiplier));
      });
    }
  }

  it("zero line items yields zero totals and equal low/high", () => {
    const estimate = EstimateMockGenerator.createEstimateDto({ contingencyPercent: 10 });
    const result = calculator.calculateTotals(estimate);
    expect(result.totals.subTotal.toMinorUnits()).toBe(0);
    expect(result.priceRange.low.exclTax.toMinorUnits()).toBe(0);
    expect(result.priceRange.high.exclTax.toMinorUnits()).toBe(0);
  });

  it("parent-line-item is skipped; child-line-items are skipped", () => {
    // ... a parent + child pair; only the child-with-parent gets excluded
  });

  it("DELETED line items are skipped", () => {
    // ... mixed status scenario
  });

  it("carries displayMode through from the input estimate", () => {
    const fromModeEstimate = EstimateMockGenerator.createEstimateDto({
      displayMode: EstimateDisplayMode.FROM,
    });
    const result = calculator.calculateTotals(fromModeEstimate);
    expect(result.priceRange.displayMode).toBe(EstimateDisplayMode.FROM);
  });
});
```

Note on the parameterised `Math.round(minorUnits * multiplier)` expectation: this assumes `Money.multiply(float)` rounds to the nearest minor unit (which is the standard Dinero.js behavior). If the actual implementation floors or uses banker's rounding, adjust the expectation to match — but ONLY after verifying by reading `money.value-object.ts` and running the test. Do NOT weaken the assertion to "approximately equal".

**Step 5: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-number-generator
cd trade-flow-api && npm run test -- --testPathPattern=estimate-totals-calculator
```

Commit message: `feat(41): add EstimateNumberGenerator and EstimateTotalsCalculator with parameterised specs`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-number-generator.service.ts`
    - `test -f trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts`
    - `grep -c "\"estimate_counters\"" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts` returns 1
    - `grep -c "findOneAndUpdate" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts` returns 1
    - `grep -c "upsert: true" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts` returns 1
    - `grep -c "\\$inc" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts` returns 1
    - `grep -c "E-\\${year}" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts` returns 1
    - `grep -c "IEstimatePriceRangeDto" trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts` returns at least 1
    - `grep -c "contingencyPercent / 100" trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts` returns 1
    - `grep -c "EstimateLineItemStatus.DELETED" trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts` returns 1
    - `grep -c "parentLineItemId" trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts` returns 1
    - `grep -c "penny-exact" trade-flow-api/src/estimate/test/services/estimate-totals-calculator.service.spec.ts` returns at least 1
    - `grep -c "multiplierCases\\|contingencyPercent: 5\\|contingencyPercent: 30" trade-flow-api/src/estimate/test/services/estimate-totals-calculator.service.spec.ts` returns at least 1 (parameterised test setup visible)
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-number-generator` exits 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-totals-calculator` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-number-generator.service.ts trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/services/(estimate-number-generator|estimate-totals-calculator)</automated>
  </verify>
  <done>Counter atomic, range math penny-exact across all allowed contingency values and subtotal edge cases</done>
</task>

<task type="auto" tdd="true">
  <name>Task 3: Implement EstimateTransitionService and policies with specs</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-transition.service.ts,
    trade-flow-api/src/estimate/policies/estimate.policy.ts,
    trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts,
    trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts,
    trade-flow-api/src/estimate/test/policies/estimate.policy.spec.ts,
    trade-flow-api/src/estimate/test/policies/estimate-line-item.policy.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-transition.service.ts (service pattern — calls repository.findByIdOrFail, validates, sets timestamps, persists)
    - trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts (spec pattern with TestingModule + mock repository)
    - trade-flow-api/src/quote/policies/quote.policy.ts (BasePolicy extension, canRead/canCreate/canUpdate/canDelete)
    - trade-flow-api/src/quote/policies/quote-line-item.policy.ts
    - trade-flow-api/src/quote/test/policies/quote.policy.spec.ts
    - trade-flow-api/src/core/policies/base.policy.ts (abstract base contract)
    - trade-flow-api/src/estimate/enums/estimate-transitions.ts (from Plan 04 — isValidTransition / getValidTransitions)
    - trade-flow-api/src/estimate/enums/estimate-status.enum.ts (from Plan 04)
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (from Task 1 — dependency)
    - trade-flow-api/src/core/errors/error-codes.enum.ts (from Plan 01 Task 3 — ESTIMATE_INVALID_TRANSITION)
    - trade-flow-api/src/core/errors/invalid-request.error.ts
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Pattern 3 Transition Service, §Common Pitfalls #9 firstViewedAt double-write)
  </read_first>
  <behavior>
    - `EstimateTransitionService.transition(authUser, id, targetStatus, ...contextFields)` re-fetches the current estimate via `EstimateRepository.findByIdOrFail(id)`, runs `isValidTransition(current.status, targetStatus)` and throws `InvalidRequestError(ErrorCodes.ESTIMATE_INVALID_TRANSITION, "Invalid transition from {from} to {to}")` on false.
    - Enforces business ownership via `AccessControllerFactory` wrapping `EstimatePolicy` — mirror `QuoteTransitionService` — so cross-business callers get 403.
    - Sets target timestamps per RESEARCH.md lines 502-511:
      - `SENT → sentAt = DateTime.now()` (allows re-send no-op; overwrites on each SENT)
      - `VIEWED → firstViewedAt` ONLY if currently undefined (Pitfall 9 / Pitfall 8)
      - `RESPONDED → respondedAt = DateTime.now()`
      - `SITE_VISIT_REQUESTED → respondedAt = DateTime.now()` if not already set
      - `CONVERTED → convertedAt = DateTime.now()` (Phase 47 will additionally set `convertedToQuoteId`)
      - `DECLINED → declinedAt = DateTime.now()`
      - `LOST → lostAt = DateTime.now()`
      - `DELETED → deletedAt = DateTime.now()`
      - `EXPIRED → no timestamp; just status update`
    - Persists via `EstimateRepository.update(updatedDto)`.
    - Returns the updated DTO.
    - `EstimatePolicy extends BasePolicy<IEstimateDto>`:
      - `canCreate(authUser, resource): boolean` returns `authUser.businessIds.includes(resource.businessId)` (or equivalent same-business check — mirror `QuotePolicy.canCreate`).
      - `canRead` / `canUpdate` same business-ownership logic.
      - `canDelete` returns `false` (soft-delete is handled via transition to DELETED, not a real delete — mirror `QuotePolicy.canDelete`).
      - Does NOT check status (that's the service's job per D-DRAFT-02).
    - `EstimateLineItemPolicy extends BasePolicy<IEstimateLineItemDto>` with same business-ownership logic.
    - `estimate-transition.service.spec.ts` has exhaustive tests: invalid transition rejection, DRAFT → SENT sets sentAt, VIEWED preserves existing firstViewedAt, SENT → SENT is a no-op-valid transition, every terminal state rejects further transitions.
    - Policy specs verify: same-business allowed, cross-business denied, canDelete always false.
  </behavior>
  <action>
**Step 1: `estimate-transition.service.ts`** — read `quote-transition.service.ts` first; mirror its structure. Recommended skeleton:

```typescript
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { AccessControllerFactory } from "@core/factories/access-controller.factory";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { InvalidRequestError } from "@core/errors/invalid-request.error";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { isValidTransition } from "@estimate/enums/estimate-transitions";
import { EstimatePolicy } from "@estimate/policies/estimate.policy";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";

@Injectable()
export class EstimateTransitionService {
  constructor(
    private readonly estimateRepository: EstimateRepository,
    private readonly estimatePolicy: EstimatePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
  ) {}

  public async transition(
    authUser: IUserDto,
    estimateId: string,
    targetStatus: EstimateStatus,
  ): Promise<IEstimateDto> {
    const accessController = this.accessControllerFactory.create(this.estimatePolicy);
    const current = await accessController.findByIdOrFail(
      authUser,
      estimateId,
      (id) => this.estimateRepository.findByIdOrFail(id),
    );

    if (!isValidTransition(current.status, targetStatus)) {
      throw new InvalidRequestError(
        ErrorCodes.ESTIMATE_INVALID_TRANSITION,
        `Invalid transition from ${current.status} to ${targetStatus}`,
        { from: current.status, to: targetStatus },
      );
    }

    const updated: IEstimateDto = { ...current, status: targetStatus };
    const now = DateTime.now();

    switch (targetStatus) {
      case EstimateStatus.SENT:
        updated.sentAt = now;
        break;
      case EstimateStatus.VIEWED:
        if (!current.firstViewedAt) {
          updated.firstViewedAt = now;
        }
        break;
      case EstimateStatus.RESPONDED:
        updated.respondedAt = now;
        break;
      case EstimateStatus.SITE_VISIT_REQUESTED:
        if (!current.respondedAt) {
          updated.respondedAt = now;
        }
        break;
      case EstimateStatus.CONVERTED:
        updated.convertedAt = now;
        break;
      case EstimateStatus.DECLINED:
        updated.declinedAt = now;
        break;
      case EstimateStatus.LOST:
        updated.lostAt = now;
        break;
      case EstimateStatus.DELETED:
        updated.deletedAt = now;
        break;
      case EstimateStatus.EXPIRED:
      case EstimateStatus.DRAFT:
        break;
    }

    return this.estimateRepository.update(updated);
  }
}
```

Note: The exact `AccessControllerFactory` usage depends on its signature — verify by reading `quote-transition.service.ts`. If the quote transition service does not use `AccessControllerFactory` directly and instead calls the repository + does an ad-hoc `if (!authUser.businessIds.includes(current.businessId)) throw ForbiddenError`, mirror that exactly. Do NOT invent a new pattern.

**Step 2: `estimate.policy.ts`** — mirror `quote.policy.ts`:

```typescript
import { Injectable } from "@nestjs/common";
import { BasePolicy } from "@core/policies/base.policy";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Injectable()
export class EstimatePolicy extends BasePolicy<IEstimateDto> {
  public canCreate(authUser: IUserDto, resource: IEstimateDto): boolean {
    return authUser.businessIds.includes(resource.businessId);
  }

  public canRead(authUser: IUserDto, resource: IEstimateDto): boolean {
    return authUser.businessIds.includes(resource.businessId);
  }

  public canUpdate(authUser: IUserDto, resource: IEstimateDto): boolean {
    return authUser.businessIds.includes(resource.businessId);
  }

  public canDelete(_authUser: IUserDto, _resource: IEstimateDto): boolean {
    return false;
  }
}
```

(Verify the exact shape against `quote.policy.ts` — some policies use `authUser.businessIds` as an array of strings, others as `Set<string>`. Mirror whatever the quote policy does.)

**Step 3: `estimate-line-item.policy.ts`** — mirror `quote-line-item.policy.ts` with the same same-business logic over `IEstimateLineItemDto`.

**Step 4: `estimate-transition.service.spec.ts`** — exhaustive scenarios:

```typescript
describe("EstimateTransitionService", () => {
  // setup TestingModule with mocked EstimateRepository and EstimatePolicy

  it("rejects invalid transition with ESTIMATE_INVALID_TRANSITION", async () => {
    const sent = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.SENT });
    mockRepo.findByIdOrFail.mockResolvedValue(sent);
    await expect(service.transition(authUser, sent.id, EstimateStatus.DELETED))
      .rejects.toThrow(/Invalid transition from sent to deleted/);
  });

  it("DRAFT -> SENT sets sentAt", async () => {
    const draft = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.DRAFT });
    mockRepo.findByIdOrFail.mockResolvedValue(draft);
    mockRepo.update.mockImplementation(async (dto) => dto);
    const result = await service.transition(authUser, draft.id, EstimateStatus.SENT);
    expect(result.status).toBe(EstimateStatus.SENT);
    expect(result.sentAt).toBeDefined();
  });

  it("SENT -> VIEWED sets firstViewedAt ONLY when undefined (Pitfall 9)", async () => {
    const already = DateTime.fromISO("2026-01-01T00:00:00Z");
    const sent = EstimateMockGenerator.createEstimateDto({
      status: EstimateStatus.SENT,
      firstViewedAt: already,
    });
    mockRepo.findByIdOrFail.mockResolvedValue(sent);
    mockRepo.update.mockImplementation(async (dto) => dto);
    const result = await service.transition(authUser, sent.id, EstimateStatus.VIEWED);
    expect(result.firstViewedAt).toEqual(already);
  });

  it("SENT -> SENT is valid (re-send no-op) and updates sentAt", async () => {
    const sent = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.SENT });
    mockRepo.findByIdOrFail.mockResolvedValue(sent);
    mockRepo.update.mockImplementation(async (dto) => dto);
    const result = await service.transition(authUser, sent.id, EstimateStatus.SENT);
    expect(result.status).toBe(EstimateStatus.SENT);
  });

  it("terminal states reject further transitions", async () => {
    for (const terminal of [EstimateStatus.CONVERTED, EstimateStatus.DECLINED, EstimateStatus.EXPIRED, EstimateStatus.LOST, EstimateStatus.DELETED]) {
      const estimate = EstimateMockGenerator.createEstimateDto({ status: terminal });
      mockRepo.findByIdOrFail.mockResolvedValue(estimate);
      await expect(service.transition(authUser, estimate.id, EstimateStatus.SENT))
        .rejects.toThrow(/Invalid transition/);
    }
  });

  it("DRAFT -> DELETED sets deletedAt", async () => {
    const draft = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.DRAFT });
    mockRepo.findByIdOrFail.mockResolvedValue(draft);
    mockRepo.update.mockImplementation(async (dto) => dto);
    const result = await service.transition(authUser, draft.id, EstimateStatus.DELETED);
    expect(result.deletedAt).toBeDefined();
  });

  it("SENT -> CONVERTED sets convertedAt", async () => {
    // similar pattern
  });

  it("SENT -> RESPONDED sets respondedAt", async () => {
    // similar pattern
  });

  it("SENT -> LOST sets lostAt", async () => {
    // similar pattern
  });
});
```

**Step 5: `estimate.policy.spec.ts` and `estimate-line-item.policy.spec.ts`** — mirror the quote equivalents:

```typescript
it("canRead returns true when authUser.businessIds includes resource.businessId", () => {
  const resource = EstimateMockGenerator.createEstimateDto({ businessId: "biz-123" });
  const user = { businessIds: ["biz-123"] } as IUserDto;
  expect(policy.canRead(user, resource)).toBe(true);
});

it("canRead returns false when authUser.businessIds does not include resource.businessId", () => {
  const resource = EstimateMockGenerator.createEstimateDto({ businessId: "biz-123" });
  const user = { businessIds: ["biz-other"] } as IUserDto;
  expect(policy.canRead(user, resource)).toBe(false);
});

it("canDelete returns false regardless of ownership", () => {
  const resource = EstimateMockGenerator.createEstimateDto({ businessId: "biz-123" });
  const owner = { businessIds: ["biz-123"] } as IUserDto;
  expect(policy.canDelete(owner, resource)).toBe(false);
});
```

Note: `as IUserDto` is a type assertion — it's unavoidable in spec fixtures because constructing a full `IUserDto` requires many fields. This is the one acceptable exception to the no-`as` rule, but ONLY in test fixtures, NEVER in production code. Prefer `createMockUser()` helper from an existing quote spec if one exists. Search `trade-flow-api/src/quote/test/mocks` for a user mock factory and reuse it. Fall back to `as IUserDto` only if no factory exists.

**Step 6: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/(services/estimate-transition|policies)
```

Commit message: `feat(41): add EstimateTransitionService and policies with specs`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-transition.service.ts`
    - `test -f trade-flow-api/src/estimate/policies/estimate.policy.ts`
    - `test -f trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts`
    - `grep -c "isValidTransition" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns at least 1
    - `grep -c "ESTIMATE_INVALID_TRANSITION" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns at least 1
    - `grep -c "if (!current.firstViewedAt)" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns 1 (Pitfall 9)
    - `grep -c "sentAt\\s*=\\s*now\\|updated.sentAt" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns at least 1
    - `grep -c "deletedAt" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns at least 1
    - `grep -c "convertedAt\\|lostAt\\|declinedAt\\|respondedAt" trade-flow-api/src/estimate/services/estimate-transition.service.ts` returns at least 4
    - `grep -c "canDelete.*: boolean.*false\\|canDelete.*=>.*false\\|return false" trade-flow-api/src/estimate/policies/estimate.policy.ts` returns at least 1
    - `grep -c "businessIds.includes" trade-flow-api/src/estimate/policies/estimate.policy.ts` returns at least 3 (canCreate, canRead, canUpdate)
    - `grep -c "re-send no-op" trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts` returns at least 1
    - `grep -c "Pitfall 9\\|firstViewedAt ONLY when undefined\\|preserves existing firstViewedAt" trade-flow-api/src/estimate/test/services/estimate-transition.service.spec.ts` returns at least 1
    - `grep -c "canDelete returns false" trade-flow-api/src/estimate/test/policies/estimate.policy.spec.ts` returns 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/(services/estimate-transition|policies)` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-transition.service.ts trade-flow-api/src/estimate/policies/` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate/test/(services/estimate-transition|policies)</automated>
  </verify>
  <done>Transition service enforces state machine with correct timestamps and Pitfall 9 mitigation; policies enforce same-business ownership without status gating</done>
</task>

<task type="auto">
  <name>Task 4: Run full CI gate to confirm Wave 4 is non-regressive</name>
  <files>none</files>
  <read_first>- trade-flow-api/package.json</read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected: exit 0.

If any failure occurs:
- Test failure in estimate specs: debug specific assertion
- Test failure in existing quote / document-token / item specs: the Wave 4 additions should NOT have touched those. If they fail, Plan 02 or Plan 03 regressed. Roll back the specific change and investigate.
- Lint: usually an unused import from Plan 01 ErrorCodes extension or an unused constructor param on a policy. Fix in place.
- Format: `cd trade-flow-api && npm run format` and commit as `chore(41): prettier fixes for estimate wave 4`.
- Typecheck: missing imports or DTO type mismatches — fix at root cause.

No suppressions.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - No new `@ts-ignore`, `eslint-disable`, or `as unknown as` in the Wave 4 diff
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green after Wave 4</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| EstimateTransitionService ← authUser | Service re-fetches current state before applying transition (D-DRAFT-01, Pitfall 5) |
| EstimatePolicy ← authUser | Business-ownership check |
| EstimateNumberGenerator ← businessId | Atomic counter prevents race-induced collisions |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain, §Common Pitfalls #1/#5/#8/#9.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-05-01 | Tampering | Counter race producing duplicate E-YYYY-NNN numbers | mitigate | EstimateNumberGenerator uses `findOneAndUpdate` + `$inc` + `upsert: true` (Pitfall 1); parameterised spec asserts sequential output for 10 concurrent-simulated calls; partial-unique index on `(businessId, number)` in EstimateRepository provides defense-in-depth |
| T-41-05-02 | Tampering | Stale DTO bypasses status check (Pitfall 5) | mitigate | `EstimateTransitionService.transition` calls `estimateRepository.findByIdOrFail(id)` before any status logic — re-fetches authoritative state; Plan 07's `EstimateUpdater` / `EstimateDeleter` follow the same re-fetch pattern |
| T-41-05-03 | Information Disclosure | firstViewedAt overwritten on repeated VIEWED transitions (Pitfall 9) | mitigate | Explicit `if (!current.firstViewedAt)` check in transition handler; spec test asserts preservation |
| T-41-05-04 | Elevation of Privilege | IDOR — user accesses another business's estimate by id | mitigate | EstimatePolicy.canRead/canUpdate check `authUser.businessIds.includes(resource.businessId)`; AccessControllerFactory wraps repository reads; uniform 403 |
| T-41-05-05 | Tampering | Soft-delete bypass via PATCH with status=DRAFT | mitigate | UpdateEstimateRequest (Plan 04) has no `status` field; transition map has no DELETED → DRAFT path; index partial filter excludes deleted rows from list queries |
| T-41-05-06 | Input Validation | Contingency multiplier produces fractional minor units | mitigate | Parameterised EstimateTotalsCalculator spec over every allowed contingency and varied subtotals asserts penny-exact output (RESEARCH.md Assumption A2) |
| T-41-05-07 | Tampering | Counter collection name drift between generator and index | mitigate | `EstimateRepository.createIndexes` targets `"estimates"`; `EstimateNumberGenerator.COLLECTION` targets `"estimate_counters"` — distinct collections, each verified by spec |
| T-41-05-08 | Elevation of Privilege | Policy misconfigured as status-aware allowing non-Draft writes | mitigate | D-DRAFT-02 locks separation: policy handles who, service handles whether; policy spec has NO status-based test; service specs in Plan 07 handle the status gate |
| T-41-05-09 | Denial of Service | Unbounded list query crashes API | mitigate | Plan 04's ListEstimatesRequest caps `limit` at 100; EstimateRepository.findPaginatedByBusinessId applies `.limit(limit)` on the cursor |
</threat_model>

<verification>
1. All 7 source files and 6 spec files exist.
2. `cd trade-flow-api && npm run test -- --testPathPattern=estimate` passes every new spec and every existing spec.
3. Collection names verified: `estimates`, `estimatelineitems`, `estimate_counters`.
4. `cd trade-flow-api && npm run ci` exits 0.
5. No type suppressions anywhere in the diff.
</verification>

<success_criteria>
- Plan 06 (factories + line-item creator/retriever) can inject EstimateLineItemRepository, EstimateLineItemPolicy, and the shared `@item/services` bundle helpers without further repository changes
- Plan 07 (creator/retriever/updater/deleter) can call EstimateRepository.findPaginatedByBusinessId, EstimateTotalsCalculator.calculateTotals, EstimateNumberGenerator.generateNumber, and EstimateTransitionService.transition without further service changes
- Counter race and contingency math correctness are proven by parameterised tests
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-05-SUMMARY.md` documenting:
- Counter test: sample sequential output
- Totals calculator: table of `(c, subtotal) → high` penny-exact values
- Transition service: list of every (from, to) tested
- The full `npm run ci` output
</output>

---

**Amendment history:**
- 2026-04-11 — Phase 42 wave 1 (plan 42-01): added `{rootEstimateId, isCurrent}` partial unique index, `{rootEstimateId, revisionNumber}` sort index, and tightened `{businessId, number}` partial filter to `{deletedAt: null, isCurrent: true}`. Switched index bootstrap precedent from `QuoteRepository` (has no indexes) to `SubscriptionRepository` (`OnModuleInit` + `ensureIndexes`). Rationale: Phase 42 D-CHAIN-03 / D-CHAIN-05 / D-CHAIN-06 / D-CHAIN-07. Phase 41 has not executed; folding the final index topology forward avoids a runtime drop-and-recreate in Phase 42.
