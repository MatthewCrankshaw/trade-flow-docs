---
phase: 41-estimate-module-crud-backend
plan: 07
type: execute
wave: 6
depends_on: [04, 05, 06]
files_modified:
  - trade-flow-api/src/estimate/services/estimate-creator.service.ts
  - trade-flow-api/src/estimate/services/estimate-retriever.service.ts
  - trade-flow-api/src/estimate/services/estimate-updater.service.ts
  - trade-flow-api/src/estimate/services/estimate-deleter.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-creator.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-updater.service.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts
autonomous: true
requirements: [EST-01, EST-04, EST-05, EST-06, EST-07, CONT-02, CONT-05, RESP-08]
must_haves:
  truths:
    - `EstimateCreator.create(authUser, dto)` validates customerId + jobId resolve to accessible resources, generates `E-YYYY-NNN` via `EstimateNumberGenerator`, persists via `EstimateRepository` through `AuthorizedCreatorFactory` + `EstimatePolicy`, attaches totals + priceRange via `EstimateTotalsCalculator`, and sets defaults: `contingencyPercent = 10`, `displayMode = "range"`, `revisionNumber = 1`, `isCurrent = true`, `parentEstimateId = null`, `rootEstimateId = null`
    - `EstimateRetriever.findByIdOrFail(authUser, id)` loads the estimate + line items, attaches totals + priceRange, returns `responseSummary: null` in Phase 41
    - `EstimateRetriever.findPaginated(authUser, listRequest)` returns `{ items: IEstimateDto[], pagination: IQueryResultsDto }` with status tab filtering and limit clamping (defaults: limit=20, offset=0)
    - `EstimateUpdater.update(authUser, id, body)` RE-FETCHES current estimate from repository (Pitfall 5), rejects if `status !== DRAFT` with `ESTIMATE_NOT_EDITABLE`, applies patch to scalar fields (title/notes/customer/job/contingencyPercent/displayMode/uncertaintyReasons/uncertaintyNotes/estimateDate), persists, re-attaches totals, returns updated DTO
    - `EstimateUpdater.addLineItem(authUser, estimateId, request)`, `updateLineItem`, and `deleteLineItem` methods mirror the quote equivalents — each re-fetches the estimate, asserts DRAFT status, delegates to `EstimateLineItemCreator` / `EstimateLineItemRepository`, returns updated estimate DTO
    - `EstimateDeleter.softDelete(authUser, id)` re-fetches, asserts DRAFT, sets `status = DELETED` + `deletedAt = now` via `EstimateTransitionService.transition(..., DELETED)`, does NOT touch line items (D-DRAFT-03, EST-05), returns the updated (deleted) DTO
    - `cd trade-flow-api && npm run ci` exits 0
  artifacts:
    - path: trade-flow-api/src/estimate/services/estimate-creator.service.ts
      provides: Create with number generation + totals attachment + policy-wrapped persist
      contains: "EstimateNumberGenerator"
    - path: trade-flow-api/src/estimate/services/estimate-retriever.service.ts
      provides: findByIdOrFail + findPaginated with status tab filtering
      contains: "findPaginated"
    - path: trade-flow-api/src/estimate/services/estimate-updater.service.ts
      provides: Draft-only update + line-item add/update/delete
      contains: "ESTIMATE_NOT_EDITABLE"
    - path: trade-flow-api/src/estimate/services/estimate-deleter.service.ts
      provides: Draft-only soft delete via transition
      contains: "EstimateStatus.DELETED"
  key_links:
    - from: trade-flow-api/src/estimate/services/estimate-creator.service.ts
      to: trade-flow-api/src/estimate/services/estimate-number-generator.service.ts
      via: constructor injection
      pattern: "EstimateNumberGenerator"
    - from: trade-flow-api/src/estimate/services/estimate-retriever.service.ts
      to: trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
      via: calculateTotals call
      pattern: "calculateTotals"
    - from: trade-flow-api/src/estimate/services/estimate-updater.service.ts
      to: trade-flow-api/src/core/errors/error-codes.enum.ts
      via: ESTIMATE_NOT_EDITABLE reference
      pattern: "ESTIMATE_NOT_EDITABLE"
---

<objective>
Implement the four CRUD service classes that the estimate controller calls: `EstimateCreator`, `EstimateRetriever`, `EstimateUpdater`, `EstimateDeleter`. Each is a literal mirror of its quote-module counterpart (D-LI-03 side-by-side review) with these estimate-specific behaviors: create attaches `totals` and `priceRange` via `EstimateTotalsCalculator`; list uses the paginated repository method from Plan 05; update and delete re-fetch current status and reject non-DRAFT with `ESTIMATE_NOT_EDITABLE` (D-DRAFT-01, Pitfall 5); delete calls `EstimateTransitionService.transition(..., DELETED)` instead of a hard repository delete.

Purpose: These services bundle the creation, reading, editing, and deletion use cases with their authorization wrapping, totals attachment, and state-machine enforcement. Plan 08's controller becomes a thin mapping layer because all business logic lives here.

Output: Four service files + four specs. Every service re-fetches state before writing. Every service attaches totals on read paths. The updater rejects non-Draft at the service layer (not the policy).
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files. Every estimate service mirrors its quote counterpart. -->

- trade-flow-api/src/quote/services/quote-creator.service.ts
- trade-flow-api/src/quote/services/quote-retriever.service.ts
- trade-flow-api/src/quote/services/quote-updater.service.ts
- trade-flow-api/src/quote/services/quote-deleter.service.ts (if it exists; if not, the delete is a method on QuoteUpdater or QuoteTransitionService)
- trade-flow-api/src/quote/test/services/quote-creator.service.spec.ts
- trade-flow-api/src/quote/test/services/quote-retriever.service.spec.ts
- trade-flow-api/src/quote/test/services/quote-updater.service.spec.ts

Phase 41 dependencies (injected):
- trade-flow-api/src/estimate/repositories/estimate.repository.ts (Plan 05)
- trade-flow-api/src/estimate/policies/estimate.policy.ts (Plan 05)
- trade-flow-api/src/estimate/services/estimate-number-generator.service.ts (Plan 05)
- trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts (Plan 05)
- trade-flow-api/src/estimate/services/estimate-transition.service.ts (Plan 05)
- trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts (Plan 06)
- trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts (Plan 06)
- trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (Plan 05)
- trade-flow-api/src/customer/services/customer-retriever.service.ts
- trade-flow-api/src/job/services/job-retriever.service.ts
- trade-flow-api/src/item/services/item-retriever.service.ts
- trade-flow-api/src/core/factories/authorized-creator.factory.ts
- trade-flow-api/src/core/factories/access-controller.factory.ts
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Implement EstimateCreator with default values, number generation, and totals attachment</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-creator.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-creator.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-creator.service.ts (source — MUST mirror; includes customer/job validation, AuthorizedCreatorFactory wrapping, totals attachment)
    - trade-flow-api/src/quote/test/services/quote-creator.service.spec.ts
    - trade-flow-api/src/estimate/services/estimate-number-generator.service.ts (Plan 05)
    - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts (Plan 05)
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (Plan 05)
    - trade-flow-api/src/estimate/policies/estimate.policy.ts (Plan 05)
    - trade-flow-api/src/customer/services/customer-retriever.service.ts
    - trade-flow-api/src/job/services/job-retriever.service.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (§decisions → Claude's Discretion defaults: contingencyPercent=10, displayMode=range, revisionNumber=1, isCurrent=true, parentEstimateId=null, rootEstimateId=null, estimateDate=now)
  </read_first>
  <behavior>
    - Method signature: `public async create(authUser: IUserDto, dto: IEstimateDto): Promise<IEstimateDto>` (mirror quote-creator signature exactly).
    - Validates `customerId` resolves via `CustomerRetriever.findByIdOrFail(customerId)` — if not found, throws `ResourceNotFoundError(ESTIMATE_CUSTOMER_NOT_FOUND)`.
    - Validates `jobId` resolves via `JobRetrieverService.findByIdOrFail(jobId)` — if not found, throws `ResourceNotFoundError(ESTIMATE_JOB_NOT_FOUND)`.
    - Generates `number` via `EstimateNumberGenerator.generateNumber(dto.businessId)`.
    - Applies defaults if not provided: `contingencyPercent ??= 10`, `displayMode ??= EstimateDisplayMode.RANGE`, `estimateDate ??= DateTime.now()`, `revisionNumber = 1`, `isCurrent = true`, `parentEstimateId = null`, `rootEstimateId = null`, `status = EstimateStatus.DRAFT`, `responseSummary = null`.
    - Persists via `AuthorizedCreatorFactory.create(estimatePolicy)` wrapping `estimateRepository.create`.
    - Attaches totals + priceRange via `estimateTotalsCalculator.calculateTotals(created)` before returning.
    - Spec coverage: happy path, customer-not-found, job-not-found, default values applied, number-generator called with correct businessId, totals attached on return.
  </behavior>
  <action>
**Step 1: `estimate-creator.service.ts`** — mirror `quote-creator.service.ts`. Read the source file first to see the exact dependency injection shape and method body. Recommended skeleton:

```typescript
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { AuthorizedCreatorFactory } from "@core/factories/authorized-creator.factory";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ResourceNotFoundError } from "@core/errors/resource-not-found.error";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { CustomerRetriever } from "@customer/services/customer-retriever.service";
import { JobRetrieverService } from "@job/services/job-retriever.service";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";
import { EstimatePolicy } from "@estimate/policies/estimate.policy";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateNumberGenerator } from "@estimate/services/estimate-number-generator.service";
import { EstimateTotalsCalculator } from "@estimate/services/estimate-totals-calculator.service";

@Injectable()
export class EstimateCreator {
  constructor(
    private readonly estimateRepository: EstimateRepository,
    private readonly estimatePolicy: EstimatePolicy,
    private readonly authorizedCreatorFactory: AuthorizedCreatorFactory,
    private readonly numberGenerator: EstimateNumberGenerator,
    private readonly totalsCalculator: EstimateTotalsCalculator,
    private readonly customerRetriever: CustomerRetriever,
    private readonly jobRetrieverService: JobRetrieverService,
  ) {}

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
  }
}
```

Adjust the `AuthorizedCreatorFactory` invocation shape to match `quote-creator.service.ts` exactly — the factory may take the policy directly or return a wrapper. Do NOT invent a new API.

If `CustomerRetriever.findByIdOrFail` already throws when not found (returning never reaches the null check), remove the `if (!customer)` clause and let it throw — mirror the quote pattern.

**Step 2: `estimate-creator.service.spec.ts`** — mirror `quote-creator.service.spec.ts`. Key assertions:

```typescript
it("generates E-YYYY-NNN number via EstimateNumberGenerator", async () => {
  mockNumberGenerator.generateNumber.mockResolvedValue("E-2026-042");
  mockRepo.create.mockImplementation(async (dto) => dto);
  mockCustomerRetriever.findByIdOrFail.mockResolvedValue(customer);
  mockJobRetriever.findByIdOrFail.mockResolvedValue(job);
  const result = await creator.create(authUser, inputDto);
  expect(result.number).toBe("E-2026-042");
  expect(mockNumberGenerator.generateNumber).toHaveBeenCalledWith(inputDto.businessId);
});

it("applies defaults when fields are not provided", async () => {
  mockNumberGenerator.generateNumber.mockResolvedValue("E-2026-001");
  mockRepo.create.mockImplementation(async (dto) => dto);
  mockCustomerRetriever.findByIdOrFail.mockResolvedValue(customer);
  mockJobRetriever.findByIdOrFail.mockResolvedValue(job);
  const inputWithoutDefaults = EstimateMockGenerator.createEstimateDto({
    contingencyPercent: undefined as unknown as number,
    displayMode: undefined as unknown as EstimateDisplayMode,
  });
  const result = await creator.create(authUser, inputWithoutDefaults);
  expect(result.contingencyPercent).toBe(10);
  expect(result.displayMode).toBe(EstimateDisplayMode.RANGE);
  expect(result.status).toBe(EstimateStatus.DRAFT);
  expect(result.revisionNumber).toBe(1);
  expect(result.isCurrent).toBe(true);
  expect(result.parentEstimateId).toBeNull();
  expect(result.rootEstimateId).toBeNull();
});

it("attaches totals and priceRange via EstimateTotalsCalculator", async () => {
  mockRepo.create.mockImplementation(async (dto) => dto);
  mockCustomerRetriever.findByIdOrFail.mockResolvedValue(customer);
  mockJobRetriever.findByIdOrFail.mockResolvedValue(job);
  mockTotalsCalculator.calculateTotals.mockImplementation((e) => ({ ...e, totals: { /* ... */ } }));
  await creator.create(authUser, inputDto);
  expect(mockTotalsCalculator.calculateTotals).toHaveBeenCalled();
});

it("throws ESTIMATE_CUSTOMER_NOT_FOUND when customer does not resolve", async () => {
  mockCustomerRetriever.findByIdOrFail.mockResolvedValue(null);
  // OR: mockCustomerRetriever.findByIdOrFail.mockRejectedValue(new ResourceNotFoundError(...))
  // depending on the quote pattern
  await expect(creator.create(authUser, inputDto)).rejects.toThrow(/Customer not found/);
});

it("throws ESTIMATE_JOB_NOT_FOUND when job does not resolve", async () => {
  // similar
});
```

Note the comment on `undefined as unknown as number` — this double-cast is used in spec fixtures to force TypeScript to allow an undefined-on-required-field scenario. Preferred alternative: use a type helper or loosely-typed fixture object constructed as `Partial<IEstimateDto>` cast to `IEstimateDto`. In this case the `as unknown as` pattern is acceptable in TEST code only — the production code never uses it. If you can avoid it with a `Partial<IEstimateDto>` intermediate variable, do so. Otherwise keep it in test code and add a one-line `// test-only: exercise default-application path` above each occurrence (the only acceptable comment exception under CLAUDE.md — "non-obvious consequence").

**Step 3: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-creator
```

Commit message: `feat(41): add EstimateCreator with number generation, defaults, and totals attachment`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-creator.service.ts`
    - `grep -c "EstimateNumberGenerator" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 2 (import + constructor)
    - `grep -c "EstimateTotalsCalculator" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 2
    - `grep -c "contingencyPercent \\?\\? 10\\|contingencyPercent: .*\\?\\? 10" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1
    - `grep -c "EstimateDisplayMode.RANGE" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1
    - `grep -c "revisionNumber: 1" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1
    - `grep -c "isCurrent: true" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1
    - `grep -c "parentEstimateId: null" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1
    - `grep -c "rootEstimateId: null" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 1
    - `grep -c "EstimateStatus.DRAFT" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1
    - `grep -c "ESTIMATE_CUSTOMER_NOT_FOUND\\|ESTIMATE_JOB_NOT_FOUND" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns at least 1 (may be 0 if CustomerRetriever already throws)
    - `grep -c "applies defaults when fields are not provided" trade-flow-api/src/estimate/test/services/estimate-creator.service.spec.ts` returns 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-creator` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 0
    - `grep -c "as unknown as\\|as any" trade-flow-api/src/estimate/services/estimate-creator.service.ts` returns 0 (production code is clean; spec may have the one documented exception)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-creator</automated>
  </verify>
  <done>EstimateCreator creates estimates with correct defaults, number, and attached totals; spec passes</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Implement EstimateRetriever with findByIdOrFail and findPaginated</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-retriever.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-retriever.service.ts (source — findByIdOrFail with totals attachment; note QuoteRetriever has `findAllByBusinessId` NOT paginated per Pitfall 6)
    - trade-flow-api/src/quote/test/services/quote-retriever.service.spec.ts
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (Plan 05 — findByIdOrFail, findPaginatedByBusinessId)
    - trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts (Plan 06)
    - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts (Plan 05)
    - trade-flow-api/src/estimate/policies/estimate.policy.ts (Plan 05)
    - trade-flow-api/src/estimate/requests/list-estimates.request.ts (Plan 04)
    - trade-flow-api/src/core/factories/access-controller.factory.ts
    - trade-flow-api/src/core/collections/dto.collection.ts
  </read_first>
  <behavior>
    - `findByIdOrFail(authUser, id): Promise<IEstimateDto>` fetches the estimate via `AccessControllerFactory` wrapping `EstimatePolicy`, loads line items via `EstimateLineItemRetriever.findByEstimateId`, attaches them into the DTO, then calls `EstimateTotalsCalculator.calculateTotals()` and returns.
    - `findPaginated(authUser, listRequest): Promise<{ items: IEstimateDto[]; pagination: IQueryResultsDto }>` resolves the authUser's businessId (mirror how quote-retriever resolves it), clamps limit (default 20, max 100), default offset 0, calls `EstimateRepository.findPaginatedByBusinessId`, attaches line items to each result (bulk lookup or per-item loop — mirror the quote pattern if it exists, otherwise per-item is acceptable), attaches totals to each, returns.
    - Spec coverage: happy path, not-found → 404, policy deny → 403, status tab filter, pagination limit clamping, totals attached.
  </behavior>
  <action>
**Step 1: `estimate-retriever.service.ts`** — mirror `quote-retriever.service.ts`. The retriever should:

```typescript
@Injectable()
export class EstimateRetriever {
  constructor(
    private readonly estimateRepository: EstimateRepository,
    private readonly estimatePolicy: EstimatePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
    private readonly estimateLineItemRetriever: EstimateLineItemRetriever,
    private readonly totalsCalculator: EstimateTotalsCalculator,
  ) {}

  public async findByIdOrFail(authUser: IUserDto, id: string): Promise<IEstimateDto> {
    const accessController = this.accessControllerFactory.create(this.estimatePolicy);
    const estimate = await accessController.findByIdOrFail(
      authUser,
      id,
      (estimateId) => this.estimateRepository.findByIdOrFail(estimateId),
    );

    const lineItems = await this.estimateLineItemRetriever.findByEstimateId(authUser, estimate.id);
    const withLineItems: IEstimateDto = {
      ...estimate,
      lineItems: DtoCollection.create(lineItems, { total: lineItems.length, limit: lineItems.length, offset: 0 }),
    };

    return this.totalsCalculator.calculateTotals(withLineItems);
  }

  public async findPaginated(
    authUser: IUserDto,
    request: ListEstimatesRequest,
  ): Promise<{ items: IEstimateDto[]; pagination: IQueryResultsDto }> {
    const businessId = authUser.businessIds[0]; // or however quote-retriever resolves it
    const limit = request.limit ?? 20;
    const offset = request.offset ?? 0;

    const { items, pagination } = await this.estimateRepository.findPaginatedByBusinessId(
      businessId,
      request.status,
      limit,
      offset,
    );

    const withTotals = await Promise.all(
      items.map(async (estimate) => {
        const lineItems = await this.estimateLineItemRetriever.findByEstimateId(authUser, estimate.id);
        const withLineItems: IEstimateDto = {
          ...estimate,
          lineItems: DtoCollection.create(lineItems, { total: lineItems.length, limit: lineItems.length, offset: 0 }),
        };
        return this.totalsCalculator.calculateTotals(withLineItems);
      }),
    );

    return { items: withTotals, pagination };
  }
}
```

Verify the `businessId` resolution against the quote pattern — the quote retriever probably reads `authUser.businessIds[0]` or uses a dedicated helper. If a helper like `getActiveBusinessId(authUser)` exists, use it.

Verify `DtoCollection.create` signature by reading `@core/collections/dto.collection.ts`.

**Step 2: `estimate-retriever.service.spec.ts`** — mirror quote equivalent + add pagination-specific tests:

```typescript
it("findByIdOrFail attaches line items and totals", async () => {
  mockAccessController.findByIdOrFail.mockResolvedValue(baseEstimate);
  mockLineItemRetriever.findByEstimateId.mockResolvedValue([lineItem1, lineItem2]);
  mockTotalsCalculator.calculateTotals.mockImplementation((e) => ({ ...e, totals: nonZeroTotals, priceRange: nonZeroPriceRange }));
  const result = await retriever.findByIdOrFail(authUser, baseEstimate.id);
  expect(result.lineItems.data).toHaveLength(2);
  expect(result.totals).toBe(nonZeroTotals);
  expect(result.priceRange).toBe(nonZeroPriceRange);
});

it("findPaginated applies status tab filter", async () => {
  mockRepo.findPaginatedByBusinessId.mockResolvedValue({ items: [], pagination: { total: 0, limit: 20, offset: 0 } });
  await retriever.findPaginated(authUser, { status: EstimateStatus.DRAFT });
  expect(mockRepo.findPaginatedByBusinessId).toHaveBeenCalledWith(
    authUser.businessIds[0],
    EstimateStatus.DRAFT,
    20,
    0,
  );
});

it("findPaginated uses default limit 20 when not provided", async () => {
  mockRepo.findPaginatedByBusinessId.mockResolvedValue({ items: [], pagination: { total: 0, limit: 20, offset: 0 } });
  await retriever.findPaginated(authUser, {});
  expect(mockRepo.findPaginatedByBusinessId).toHaveBeenCalledWith(
    expect.any(String),
    undefined,
    20,
    0,
  );
});

it("findPaginated attaches totals to every estimate in the result", async () => {
  // ... verify calculateTotals called for each item
});
```

**Step 3: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever
```

Commit message: `feat(41): add EstimateRetriever with findByIdOrFail and paginated list`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-retriever.service.ts`
    - `grep -c "findByIdOrFail" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "findPaginated" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 1
    - `grep -c "calculateTotals" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 2 (called in both find methods)
    - `grep -c "request.limit \\?\\? 20\\|limit = request.limit \\?\\? 20" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 1
    - `grep -c "request.offset \\?\\? 0\\|offset = request.offset \\?\\? 0" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 1
    - `grep -c "findByEstimateId" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns at least 2 (called in both find methods)
    - `grep -c "attaches line items and totals\\|findByIdOrFail attaches" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns at least 1
    - `grep -c "findPaginated applies status tab filter\\|status tab" trade-flow-api/src/estimate/test/services/estimate-retriever.service.spec.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-retriever.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-retriever</automated>
  </verify>
  <done>EstimateRetriever loads single + paginated estimates with line items and attached totals</done>
</task>

<task type="auto" tdd="true">
  <name>Task 3: Implement EstimateUpdater with Draft-only enforcement and line-item CRUD methods</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-updater.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-updater.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-updater.service.ts (source — MUST read; includes re-fetch pattern, addLineItem/updateLineItem/deleteLineItem methods)
    - trade-flow-api/src/quote/test/services/quote-updater.service.spec.ts
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (Plan 05)
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts (Plan 05)
    - trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts (Plan 06)
    - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts (Plan 05)
    - trade-flow-api/src/estimate/policies/estimate.policy.ts (Plan 05)
    - trade-flow-api/src/estimate/enums/estimate-status.enum.ts (Plan 04)
    - trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/update-estimate.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/update-estimate-line-item.request.ts (Plan 04)
    - trade-flow-api/src/core/errors/invalid-request.error.ts
    - trade-flow-api/src/item/services/item-retriever.service.ts (for add line item — looks up the IItemDto)
  </read_first>
  <behavior>
    - `update(authUser, id, body: UpdateEstimateRequest): Promise<IEstimateDto>`:
      1. Re-fetches current estimate via `estimateRepository.findByIdOrFail(id)` (Pitfall 5).
      2. Enforces ownership via `EstimatePolicy.canUpdate` through `AccessControllerFactory`.
      3. Asserts `current.status === EstimateStatus.DRAFT`; else throws `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_EDITABLE, "Estimate can only be edited in Draft status")`.
      4. Applies patch (title/notes/customer/job/contingencyPercent/displayMode/uncertaintyReasons/uncertaintyNotes/estimateDate) — only fields present on the request body.
      5. Persists via `estimateRepository.update(updated)`.
      6. Reloads line items via `EstimateLineItemRetriever.findByEstimateId`.
      7. Attaches totals via `EstimateTotalsCalculator.calculateTotals`.
      8. Returns the fresh DTO.
    - `addLineItem(authUser, estimateId, request: CreateEstimateLineItemRequest): Promise<IEstimateDto>`:
      1. Re-fetches estimate, asserts DRAFT.
      2. Looks up the `itemId` via `ItemRetrieverService.findByIdOrFail`.
      3. Delegates to `EstimateLineItemCreator.create(authUser, estimate, item, request)`.
      4. Returns the estimate via `EstimateRetriever.findByIdOrFail` OR by re-fetching inline (mirror quote pattern).
    - `updateLineItem(authUser, estimateId, lineItemId, request): Promise<IEstimateDto>`:
      1. Re-fetches estimate, asserts DRAFT.
      2. Fetches the existing line item via `EstimateLineItemRepository.findByIdOrFail`.
      3. Asserts line item belongs to the estimate.
      4. Applies patch and persists via `estimateLineItemRepository.update`.
      5. Returns updated estimate.
    - `deleteLineItem(authUser, estimateId, lineItemId): Promise<IEstimateDto>`:
      1. Re-fetches estimate, asserts DRAFT.
      2. Soft-deletes the line item: sets `status = EstimateLineItemStatus.DELETED` via `estimateLineItemRepository.update`.
      3. Returns updated estimate.
    - Spec coverage: Draft-only rejection for every method, happy path for each patch field, line-item CRUD happy paths, line-item-not-belonging-to-estimate rejection.
  </behavior>
  <action>
**Step 1: `estimate-updater.service.ts`** — mirror `quote-updater.service.ts`. Apply all class/type/policy renames. Critical pattern for the Draft check:

```typescript
public async update(
  authUser: IUserDto,
  id: string,
  body: UpdateEstimateRequest,
): Promise<IEstimateDto> {
  const current = await this.estimateRepository.findByIdOrFail(id);

  const accessController = this.accessControllerFactory.create(this.estimatePolicy);
  await accessController.canUpdateOrFail(authUser, current);

  if (current.status !== EstimateStatus.DRAFT) {
    throw new InvalidRequestError(
      ErrorCodes.ESTIMATE_NOT_EDITABLE,
      "Estimate can only be edited in Draft status",
      { currentStatus: current.status },
    );
  }

  const updated: IEstimateDto = {
    ...current,
    ...(body.title !== undefined && { title: body.title }),
    ...(body.notes !== undefined && { notes: body.notes }),
    ...(body.customerId !== undefined && { customerId: body.customerId }),
    ...(body.jobId !== undefined && { jobId: body.jobId }),
    ...(body.contingencyPercent !== undefined && { contingencyPercent: body.contingencyPercent }),
    ...(body.displayMode !== undefined && { displayMode: body.displayMode }),
    ...(body.uncertaintyReasons !== undefined && { uncertaintyReasons: body.uncertaintyReasons }),
    ...(body.uncertaintyNotes !== undefined && { uncertaintyNotes: body.uncertaintyNotes }),
    ...(body.estimateDate !== undefined && { estimateDate: DateTime.fromISO(body.estimateDate) }),
  };

  const persisted = await this.estimateRepository.update(updated);
  const lineItems = await this.estimateLineItemRetriever.findByEstimateId(authUser, persisted.id);
  const withLineItems: IEstimateDto = {
    ...persisted,
    lineItems: DtoCollection.create(lineItems, { total: lineItems.length, limit: lineItems.length, offset: 0 }),
  };
  return this.totalsCalculator.calculateTotals(withLineItems);
}
```

Verify `AccessControllerFactory` has a `canUpdateOrFail` method — if quote-updater uses a different access pattern, mirror that.

If the customer or job is changed via the PATCH, validate they still resolve before the persist (mirror the creator's customer/job validation — quote-updater may or may not do this; mirror exactly).

Implement `addLineItem`, `updateLineItem`, `deleteLineItem` methods as literal mirrors of `QuoteUpdater.addLineItem`/`updateLineItem`/`deleteLineItem`. Each starts with a re-fetch + DRAFT assertion.

**Step 2: `estimate-updater.service.spec.ts`** — exhaustive Draft-only tests:

```typescript
describe("EstimateUpdater.update", () => {
  it("applies title/notes/customer/job/contingencyPercent/displayMode/uncertainty fields on Draft", async () => {
    mockRepo.findByIdOrFail.mockResolvedValue(draftEstimate);
    mockRepo.update.mockImplementation(async (dto) => dto);
    const body = { title: "new title", contingencyPercent: 15, displayMode: EstimateDisplayMode.FROM };
    const result = await updater.update(authUser, draftEstimate.id, body);
    expect(result.title).toBe("new title");
    expect(result.contingencyPercent).toBe(15);
    expect(result.displayMode).toBe(EstimateDisplayMode.FROM);
  });

  it("rejects update on SENT estimate with ESTIMATE_NOT_EDITABLE", async () => {
    const sent = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.SENT });
    mockRepo.findByIdOrFail.mockResolvedValue(sent);
    await expect(updater.update(authUser, sent.id, { title: "attempt" }))
      .rejects.toThrow(/Estimate can only be edited in Draft status/);
  });

  it("rejects update on VIEWED/RESPONDED/CONVERTED/DECLINED/EXPIRED/LOST/DELETED estimates", async () => {
    for (const status of [
      EstimateStatus.VIEWED, EstimateStatus.RESPONDED, EstimateStatus.CONVERTED,
      EstimateStatus.DECLINED, EstimateStatus.EXPIRED, EstimateStatus.LOST, EstimateStatus.DELETED,
    ]) {
      const estimate = EstimateMockGenerator.createEstimateDto({ status });
      mockRepo.findByIdOrFail.mockResolvedValue(estimate);
      await expect(updater.update(authUser, estimate.id, { title: "x" }))
        .rejects.toThrow(/Estimate can only be edited in Draft status/);
    }
  });

  it("re-fetches from repository (Pitfall 5) — does not trust cached state", async () => {
    // Pass in a stale DTO with DRAFT, but mockRepo.findByIdOrFail returns SENT
    mockRepo.findByIdOrFail.mockResolvedValue(EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.SENT }));
    await expect(updater.update(authUser, "some-id", { title: "x" }))
      .rejects.toThrow(/Draft status/);
    expect(mockRepo.findByIdOrFail).toHaveBeenCalledBefore(mockRepo.update);
  });
});

describe("EstimateUpdater.addLineItem", () => {
  it("rejects on non-Draft status", async () => {
    // similar
  });

  it("looks up item and dispatches to EstimateLineItemCreator", async () => {
    // similar to quote equivalent
  });
});

describe("EstimateUpdater.updateLineItem", () => {
  it("rejects on non-Draft status", async () => { /* similar */ });
  it("rejects when lineItem does not belong to estimate", async () => { /* similar */ });
});

describe("EstimateUpdater.deleteLineItem", () => {
  it("soft-deletes line item by setting status = DELETED", async () => {
    mockRepo.findByIdOrFail.mockResolvedValue(draftEstimate);
    mockLineItemRepo.findByIdOrFail.mockResolvedValue(existingLineItem);
    await updater.deleteLineItem(authUser, draftEstimate.id, existingLineItem.id);
    expect(mockLineItemRepo.update).toHaveBeenCalledWith(
      expect.objectContaining({ status: EstimateLineItemStatus.DELETED }),
    );
  });
});
```

**Step 3: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-updater
```

Commit message: `feat(41): add EstimateUpdater with Draft-only enforcement and line-item CRUD`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-updater.service.ts`
    - `grep -c "ESTIMATE_NOT_EDITABLE" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `grep -c "EstimateStatus.DRAFT" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `grep -c "findByIdOrFail" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 2 (re-fetch in multiple methods)
    - `grep -c "addLineItem" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `grep -c "updateLineItem" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `grep -c "deleteLineItem" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `grep -c "rejects update on SENT" trade-flow-api/src/estimate/test/services/estimate-updater.service.spec.ts` returns at least 1
    - `grep -c "re-fetches from repository" trade-flow-api/src/estimate/test/services/estimate-updater.service.spec.ts` returns at least 1
    - `grep -c "EstimateLineItemStatus.DELETED" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-updater` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-updater.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-updater</automated>
  </verify>
  <done>EstimateUpdater enforces Draft-only editing via re-fetch; line-item CRUD methods work; all specs pass</done>
</task>

<task type="auto" tdd="true">
  <name>Task 4: Implement EstimateDeleter with Draft-only soft delete via transition service</name>
  <files>
    trade-flow-api/src/estimate/services/estimate-deleter.service.ts,
    trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/quote-deleter.service.ts (if exists; if not, delete lives inside quote-updater or quote-transition — mirror whichever pattern)
    - trade-flow-api/src/quote/test/services/quote-deleter.service.spec.ts (if exists)
    - trade-flow-api/src/estimate/services/estimate-transition.service.ts (Plan 05 — this is the service to call for the soft-delete transition)
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts (Plan 05)
    - trade-flow-api/src/estimate/enums/estimate-status.enum.ts (Plan 04)
    - trade-flow-api/src/core/errors/invalid-request.error.ts
  </read_first>
  <behavior>
    - `softDelete(authUser, id): Promise<IEstimateDto>`:
      1. Re-fetches current estimate via `estimateRepository.findByIdOrFail(id)`.
      2. Asserts `current.status === EstimateStatus.DRAFT`; else throws `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_EDITABLE)` (same code as updater — reflects "cannot edit/delete non-Draft" semantics per D-DRAFT-01).
      3. Calls `estimateTransitionService.transition(authUser, id, EstimateStatus.DELETED)` which performs the transition (sets status=DELETED, deletedAt=now) and persists.
      4. Does NOT touch line items (D-DRAFT-03, EST-05 — history preservation).
      5. Returns the updated DTO.
  </behavior>
  <action>
**Step 1: `estimate-deleter.service.ts`**:

```typescript
import { Injectable } from "@nestjs/common";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { InvalidRequestError } from "@core/errors/invalid-request.error";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateTransitionService } from "@estimate/services/estimate-transition.service";

@Injectable()
export class EstimateDeleter {
  constructor(
    private readonly estimateRepository: EstimateRepository,
    private readonly estimateTransitionService: EstimateTransitionService,
  ) {}

  public async softDelete(authUser: IUserDto, id: string): Promise<IEstimateDto> {
    const current = await this.estimateRepository.findByIdOrFail(id);

    if (current.status !== EstimateStatus.DRAFT) {
      throw new InvalidRequestError(
        ErrorCodes.ESTIMATE_NOT_EDITABLE,
        "Estimate can only be deleted in Draft status",
        { currentStatus: current.status },
      );
    }

    return this.estimateTransitionService.transition(authUser, id, EstimateStatus.DELETED);
  }
}
```

Note: The ownership check happens inside `EstimateTransitionService.transition` (which uses `AccessControllerFactory` + `EstimatePolicy`) — so we don't need to duplicate it here. Verify this by reading Plan 05's transition service implementation.

If the quote module has a dedicated `QuoteDeleter` that takes a different approach (e.g., directly sets fields via repository.update), mirror that pattern exactly. But per D-DRAFT-03 the transition-service path is preferred because it unifies the state machine.

**Step 2: `estimate-deleter.service.spec.ts`**:

```typescript
describe("EstimateDeleter", () => {
  let deleter: EstimateDeleter;
  let mockRepo: jest.Mocked<EstimateRepository>;
  let mockTransition: jest.Mocked<EstimateTransitionService>;

  beforeEach(async () => {
    // TestingModule setup with mocked EstimateRepository and EstimateTransitionService
  });

  it("soft-deletes a Draft estimate via transition service", async () => {
    const draft = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.DRAFT });
    const deleted = { ...draft, status: EstimateStatus.DELETED, deletedAt: DateTime.now() };
    mockRepo.findByIdOrFail.mockResolvedValue(draft);
    mockTransition.transition.mockResolvedValue(deleted);
    const result = await deleter.softDelete(authUser, draft.id);
    expect(result.status).toBe(EstimateStatus.DELETED);
    expect(result.deletedAt).toBeDefined();
    expect(mockTransition.transition).toHaveBeenCalledWith(authUser, draft.id, EstimateStatus.DELETED);
  });

  it("rejects deletion of SENT estimate with ESTIMATE_NOT_EDITABLE", async () => {
    const sent = EstimateMockGenerator.createEstimateDto({ status: EstimateStatus.SENT });
    mockRepo.findByIdOrFail.mockResolvedValue(sent);
    await expect(deleter.softDelete(authUser, sent.id))
      .rejects.toThrow(/Draft status/);
    expect(mockTransition.transition).not.toHaveBeenCalled();
  });

  it("rejects deletion of every non-Draft status", async () => {
    for (const status of [
      EstimateStatus.VIEWED, EstimateStatus.RESPONDED, EstimateStatus.CONVERTED,
      EstimateStatus.DECLINED, EstimateStatus.EXPIRED, EstimateStatus.LOST, EstimateStatus.DELETED,
    ]) {
      const estimate = EstimateMockGenerator.createEstimateDto({ status });
      mockRepo.findByIdOrFail.mockResolvedValue(estimate);
      await expect(deleter.softDelete(authUser, estimate.id))
        .rejects.toThrow(/Draft status/);
    }
  });

  it("does not touch line items (history preservation per D-DRAFT-03)", async () => {
    // Verify EstimateLineItemRepository is NOT injected and not called.
    // This is an absence assertion — the deleter should not have a line-item dep at all.
  });
});
```

**Step 3: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter
```

Commit message: `feat(41): add EstimateDeleter with Draft-only soft delete via transition service`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-deleter.service.ts`
    - `grep -c "EstimateTransitionService" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 2 (import + constructor)
    - `grep -c "EstimateStatus.DELETED" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "ESTIMATE_NOT_EDITABLE" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns at least 1
    - `grep -c "EstimateLineItemRepository\\|EstimateLineItemRetriever" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns 0 (deleter does NOT touch line items per D-DRAFT-03)
    - `grep -c "rejects deletion of every non-Draft status\\|rejects deletion of SENT" trade-flow-api/src/estimate/test/services/estimate-deleter.service.spec.ts` returns at least 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/services/estimate-deleter.service.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-deleter</automated>
  </verify>
  <done>EstimateDeleter soft-deletes via transition service, preserves line items, rejects non-Draft</done>
</task>

<task type="auto">
  <name>Task 5: Run full CI gate to confirm Wave 6 is non-regressive</name>
  <files>none</files>
  <read_first>- trade-flow-api/package.json</read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected: exit 0.

The four new service files should compile, lint, format clean, and pass all specs plus every existing spec (quote, document-token, item).

No suppressions.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - No `@ts-ignore`, `eslint-disable`, `as any`, `as unknown as` in Wave 6 diff (spec exception documented in Task 1)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green after Wave 6</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| EstimateCreator ← authUser | AuthorizedCreatorFactory + EstimatePolicy enforce business ownership |
| EstimateUpdater ← authUser | Re-fetch + canUpdateOrFail + status check (Pitfall 5) |
| EstimateDeleter ← authUser | Re-fetch + status check + delegate to transition service (policy applies there) |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-07-01 | Tampering | Stale DTO bypasses Draft check (Pitfall 5) | mitigate | EstimateUpdater and EstimateDeleter call `estimateRepository.findByIdOrFail(id)` BEFORE any write; spec "re-fetches from repository" asserts this |
| T-41-07-02 | Tampering | User patches `number`, `businessId`, `status`, `sentAt`, or other privileged fields via PATCH body | mitigate | UpdateEstimateRequest (Plan 04) only declares whitelisted fields; global ValidationPipe strips extras; EstimateUpdater only spreads validated request fields into the updated DTO (never `...body` unconditionally) |
| T-41-07-03 | Elevation of Privilege | Create path attaches an estimate to a business the user doesn't own | mitigate | AuthorizedCreatorFactory wraps the repository create with EstimatePolicy.canCreate; customer/job validation additionally restricts to accessible resources |
| T-41-07-04 | Elevation of Privilege | Line-item CRUD on non-owned estimate | mitigate | addLineItem/updateLineItem/deleteLineItem each re-fetch the estimate and apply access control + DRAFT check; line-item repository itself scoped by businessId via EstimateLineItemPolicy |
| T-41-07-05 | Denial of Service | findPaginated with abusive limit/offset consumes memory | mitigate | ListEstimatesRequest caps limit at 100 (Plan 04); EstimateRetriever applies default 20 / max 100 additionally; repository cursor uses `.skip(offset).limit(limit)` |
| T-41-07-06 | Tampering | Counter consumed but create rolls back — gaps in E-YYYY-NNN sequence | accept | Gaps in counter sequence are cosmetic; atomic counter never produces duplicates; existing quote module accepts identical semantics |
| T-41-07-07 | Information Disclosure | Error message exposes internal IDs or business state | mitigate | `createHttpError` + `createErrorResponse` sanitize error details; error messages reference fields by name ("Estimate can only be edited in Draft status") not IDs |
| T-41-07-08 | Tampering | Line-item deletion touches estimate-level status (should not, per D-DRAFT-03) | mitigate | EstimateDeleter has NO injected `EstimateLineItemRepository`/`EstimateLineItemRetriever` dependency — enforced by Task 4 acceptance criterion (grep returns 0) |
</threat_model>

<verification>
1. Four service files and four specs exist under `src/estimate/services/` and `src/estimate/test/services/`.
2. Every CRUD method re-fetches the current state before writing.
3. Draft-only rejection is service-layer, not policy-layer.
4. Totals + priceRange attached on every read path.
5. `cd trade-flow-api && npm run ci` exits 0.
</verification>

<success_criteria>
- Plan 08 can wire all four services plus the supporting line-item services into `EstimateController` without further business logic
- Every EST-01 / EST-04 / EST-05 / EST-06 / EST-07 requirement is covered by a unit test
- Pagination pattern is proven working end-to-end at the service layer
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-07-SUMMARY.md` documenting:
- The four service files created
- Draft-only enforcement test scenarios (list of statuses tested)
- The full `npm run ci` output
</output>
