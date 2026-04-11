---
phase: 41-estimate-module-crud-backend
plan: 08
type: execute
wave: 7
depends_on: [01, 02, 03, 04, 05, 06, 07]
files_modified:
  - trade-flow-api/src/estimate/controllers/estimate.controller.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/openapi.yaml
  - .planning/ROADMAP.md
  - .planning/milestones/v1.8-ROADMAP.md
  - .planning/STATE.md
autonomous: true
requirements: [EST-01, EST-04, EST-05, EST-06, EST-07, EST-08, EST-09, CONT-01, CONT-02, CONT-05]
must_haves:
  truths:
    - `EstimateController` exposes exactly these 8 authenticated routes: `POST /v1/estimates`, `GET /v1/estimates`, `GET /v1/estimates/:estimateId`, `PATCH /v1/estimates/:estimateId`, `DELETE /v1/estimates/:estimateId`, `POST /v1/estimates/:estimateId/line-item`, `PATCH /v1/estimates/:estimateId/line-item/:lineItemId`, `DELETE /v1/estimates/:estimateId/line-item/:lineItemId`
    - Every controller handler is wrapped in try/catch → `createHttpError(error)` and returns `createResponse([item])`
    - Every controller handler has `@UseGuards(JwtAuthGuard)`
    - Controller has NO endpoint that triggers a non-CRUD transition (no `@Post("estimates/:id/transition")`, no send/response/convert/mark-lost) per D-TXN-05
    - `EstimateModule` is declared with providers including every service/factory/repository/policy from Plans 05-07 and exports `EstimateRepository`, `EstimateTransitionService`, `EstimateTotalsCalculator` for downstream phases
    - `AppModule` imports `EstimateModule`
    - `openapi.yaml` has 8 new endpoint entries under `/v1/estimates*` with full request/response schemas
    - `.planning/ROADMAP.md` Phase 41 success criterion #5 is rewritten per D-TKN-07
    - `.planning/milestones/v1.8-ROADMAP.md` Phase 41 success criterion #5 is also rewritten
    - `.planning/STATE.md` Phase 41 criterion #5 reference matches the D-TKN-07 locked wording (or the absence of any such reference has been confirmed with a grep assertion)
    - `.planning/ROADMAP.md` Phase 41 prose uses the locked collection name `estimatelineitems` (not `estimate_line_items`) per D-ENT-02
    - `cd trade-flow-api && npm run ci` exits 0 (FINAL gate for the phase)
  artifacts:
    - path: trade-flow-api/src/estimate/controllers/estimate.controller.ts
      provides: 8 REST endpoints for estimate CRUD + line-item CRUD
      contains: "@Post(\"estimates\")"
    - path: trade-flow-api/src/estimate/estimate.module.ts
      provides: EstimateModule with all providers and correct exports
      contains: "EstimateTransitionService"
    - path: trade-flow-api/openapi.yaml
      provides: Canonical API contract for estimate endpoints
      contains: "/v1/estimates"
  key_links:
    - from: trade-flow-api/src/estimate/estimate.module.ts
      to: trade-flow-api/src/estimate/controllers/estimate.controller.ts
      via: controllers array
      pattern: "EstimateController"
    - from: trade-flow-api/src/app.module.ts
      to: trade-flow-api/src/estimate/estimate.module.ts
      via: imports array
      pattern: "EstimateModule"
---

<objective>
Complete the phase: wire the `EstimateController` with all 8 REST endpoints (5 CRUD + 3 line-item sub-resources per RESEARCH.md §Open Question #3), assemble `EstimateModule` with every provider from Plans 05-07, register it in `AppModule`, update `openapi.yaml` with the new endpoints, rewrite ROADMAP.md Phase 41 success criterion #5 per D-TKN-07, and run the final `npm run ci` gate that signs off the entire phase.

Purpose: This is the ship-it plan. Every piece of backend behavior from earlier plans is now exposed to HTTP callers. After this plan, Phase 42 (revisions) and Phase 43 (frontend CRUD) can build on top of a functioning estimate backend.

Output: One controller, one module, one module registration, one OpenAPI update, one ROADMAP rewrite, and a fully green CI gate.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files. Controller + module wiring mirrors the quote module. -->

- trade-flow-api/src/quote/controllers/quote.controller.ts
- trade-flow-api/src/quote/quote.module.ts
- trade-flow-api/src/quote/test/controllers/quote.controller.spec.ts
- trade-flow-api/src/app.module.ts (existing AppModule imports array)
- trade-flow-api/openapi.yaml (existing spec — search for the `/v1/quotes` section to see the shape to mirror)
- .planning/ROADMAP.md Phase 41 section
- .planning/milestones/v1.8-ROADMAP.md Phase 41 section

Phase 41 services + repositories to wire:
- EstimateController (this plan)
- EstimateCreator, EstimateRetriever, EstimateUpdater, EstimateDeleter (Plan 07)
- EstimateNumberGenerator, EstimateTotalsCalculator, EstimateTransitionService (Plan 05)
- EstimateStandardLineItemFactory, EstimateBundleLineItemFactory, EstimateLineItemCreator, EstimateLineItemRetriever (Plan 06)
- EstimateRepository, EstimateLineItemRepository (Plan 05)
- EstimatePolicy, EstimateLineItemPolicy (Plan 05)

Per D-TKN-07, the new ROADMAP success criterion #5 wording (LOCKED):
"The `quote-token` module is fully replaced by the new `document-token` module at the code level. The existing `/v1/public/quote/:token` endpoint continues to resolve tokens with `documentType === 'quote'` via the new guard. No data migration is required because no production data exists."
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Implement EstimateController with 8 endpoints + spec</name>
  <files>
    trade-flow-api/src/estimate/controllers/estimate.controller.ts,
    trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/controllers/quote.controller.ts (source — MUST read; every endpoint pattern, guards, try/catch wrapping, response mapping)
    - trade-flow-api/src/quote/test/controllers/quote.controller.spec.ts (spec pattern)
    - trade-flow-api/src/estimate/services/estimate-creator.service.ts (Plan 07 — inject)
    - trade-flow-api/src/estimate/services/estimate-retriever.service.ts (Plan 07 — inject)
    - trade-flow-api/src/estimate/services/estimate-updater.service.ts (Plan 07 — inject; exposes addLineItem / updateLineItem / deleteLineItem)
    - trade-flow-api/src/estimate/services/estimate-deleter.service.ts (Plan 07 — inject)
    - trade-flow-api/src/estimate/requests/create-estimate.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/update-estimate.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/list-estimates.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts (Plan 04)
    - trade-flow-api/src/estimate/requests/update-estimate-line-item.request.ts (Plan 04)
    - trade-flow-api/src/estimate/responses/estimate.responses.ts (Plan 04)
    - trade-flow-api/src/auth/auth.guard.ts (JwtAuthGuard)
    - trade-flow-api/src/core/response/create-response.utility.ts
    - trade-flow-api/src/core/errors/handle-error.utility.ts (createHttpError)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Controller Handler)
  </read_first>
  <behavior>
    - Controller class `EstimateController` with `@Controller("v1")` prefix (NestJS convention).
    - Constructor injects all four CRUD services from Plan 07: `EstimateCreator`, `EstimateRetriever`, `EstimateUpdater`, `EstimateDeleter`.
    - 8 handler methods, each with `@UseGuards(JwtAuthGuard)`:
      1. `POST /v1/estimates` — creates, returns `IResponse<IEstimateResponse>`
      2. `GET /v1/estimates` — lists with pagination + status filter, returns `{ data, pagination }`
      3. `GET /v1/estimates/:estimateId` — detail, returns `IResponse<IEstimateResponse>`
      4. `PATCH /v1/estimates/:estimateId` — scalar update, returns `IResponse<IEstimateResponse>`
      5. `DELETE /v1/estimates/:estimateId` — soft delete, returns `IResponse<IEstimateResponse>`
      6. `POST /v1/estimates/:estimateId/line-item` — add line item, returns parent estimate
      7. `PATCH /v1/estimates/:estimateId/line-item/:lineItemId` — update line item, returns parent
      8. `DELETE /v1/estimates/:estimateId/line-item/:lineItemId` — soft-delete line item, returns parent
    - Every handler wrapped in try/catch with `createHttpError(error)`.
    - Every single-item response uses `createResponse([item])`; the list response uses `{ data: items.map(mapToResponse), pagination }` per the RESEARCH.md controller example.
    - Request body → DTO mapping happens in private `mapCreateToDto(body, authUser)`, `mapUpdateToDto(body)` etc. — mirror quote controller exactly. The DTO construction injects `businessId` from `authUser.businessIds[0]` (or however the quote controller resolves it).
    - Response DTO mapping happens in private `mapToResponse(estimate: IEstimateDto): IEstimateResponse`.
    - NO endpoint for transition triggers (per D-TXN-05): verified by spec assertion that no route matches `@Post("estimates/:id/transition")` or any send/respond/convert/mark-lost pattern.
  </behavior>
  <action>
**Step 1: `estimate.controller.ts`** — read `quote.controller.ts` first; mirror the whole file with the renames. Recommended skeleton based on RESEARCH.md §Code Examples → Controller Handler (lines 811-911). Apply these estimate-specific points:

```typescript
import { JwtAuthGuard } from "@auth/auth.guard";
import { createHttpError } from "@core/errors/handle-error.utility";
import { createResponse } from "@core/response/create-response.utility";
import { IResponse } from "@core/response/response.interface";
import {
  Body, Controller, Delete, Get, Param, Patch, Post, Query, Req, UseGuards,
} from "@nestjs/common";
import { IEstimateDto } from "@estimate/data-transfer-objects/estimate.dto";
import { CreateEstimateRequest } from "@estimate/requests/create-estimate.request";
import { UpdateEstimateRequest } from "@estimate/requests/update-estimate.request";
import { ListEstimatesRequest } from "@estimate/requests/list-estimates.request";
import { CreateEstimateLineItemRequest } from "@estimate/requests/create-estimate-line-item.request";
import { UpdateEstimateLineItemRequest } from "@estimate/requests/update-estimate-line-item.request";
import { IEstimateResponse } from "@estimate/responses/estimate.responses";
import { EstimateCreator } from "@estimate/services/estimate-creator.service";
import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
import { EstimateUpdater } from "@estimate/services/estimate-updater.service";
import { EstimateDeleter } from "@estimate/services/estimate-deleter.service";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Controller("v1")
export class EstimateController {
  constructor(
    private readonly estimateCreator: EstimateCreator,
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateUpdater: EstimateUpdater,
    private readonly estimateDeleter: EstimateDeleter,
  ) {}

  @UseGuards(JwtAuthGuard)
  @Post("estimates")
  public async create(
    @Req() request: { user: IUserDto },
    @Body() body: CreateEstimateRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const dto = this.mapCreateToDto(body, request.user);
      const created = await this.estimateCreator.create(request.user, dto);
      return createResponse([this.mapToResponse(created)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Get("estimates")
  public async findAll(
    @Req() request: { user: IUserDto },
    @Query() query: ListEstimatesRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const { items, pagination } = await this.estimateRetriever.findPaginated(request.user, query);
      return {
        data: items.map((e) => this.mapToResponse(e)),
        pagination,
      };
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Get("estimates/:estimateId")
  public async findOne(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const estimate = await this.estimateRetriever.findByIdOrFail(request.user, estimateId);
      return createResponse([this.mapToResponse(estimate)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Patch("estimates/:estimateId")
  public async update(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
    @Body() body: UpdateEstimateRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const updated = await this.estimateUpdater.update(request.user, estimateId, body);
      return createResponse([this.mapToResponse(updated)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Delete("estimates/:estimateId")
  public async softDelete(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const deleted = await this.estimateDeleter.softDelete(request.user, estimateId);
      return createResponse([this.mapToResponse(deleted)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Post("estimates/:estimateId/line-item")
  public async addLineItem(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
    @Body() body: CreateEstimateLineItemRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const updated = await this.estimateUpdater.addLineItem(request.user, estimateId, body);
      return createResponse([this.mapToResponse(updated)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Patch("estimates/:estimateId/line-item/:lineItemId")
  public async updateLineItem(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
    @Param("lineItemId") lineItemId: string,
    @Body() body: UpdateEstimateLineItemRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const updated = await this.estimateUpdater.updateLineItem(request.user, estimateId, lineItemId, body);
      return createResponse([this.mapToResponse(updated)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  @UseGuards(JwtAuthGuard)
  @Delete("estimates/:estimateId/line-item/:lineItemId")
  public async deleteLineItem(
    @Req() request: { user: IUserDto },
    @Param("estimateId") estimateId: string,
    @Param("lineItemId") lineItemId: string,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const updated = await this.estimateUpdater.deleteLineItem(request.user, estimateId, lineItemId);
      return createResponse([this.mapToResponse(updated)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }

  private mapCreateToDto(body: CreateEstimateRequest, authUser: IUserDto): IEstimateDto {
    // mirror quote controller's mapCreateToDto
    // resolve businessId from authUser
    // build an IEstimateDto with defaults filled in by the creator service
    throw new Error("implement per quote controller pattern");
  }

  private mapToResponse(estimate: IEstimateDto): IEstimateResponse {
    // mirror quote controller's mapToResponse
    throw new Error("implement per quote controller pattern");
  }
}
```

IMPORTANT: The `throw new Error("implement ...")` placeholders MUST be replaced with the real mappers before commit. The mapper logic is a line-by-line mirror of `quote.controller.ts`. Read the quote controller and copy the patterns.

**Step 2: `estimate.controller.spec.ts`** — mirror `quote.controller.spec.ts` with additional assertions per RESEARCH.md §Validation Architecture Test Map. Must-have tests:

```typescript
describe("EstimateController", () => {
  // TestingModule setup

  describe("POST /v1/estimates", () => {
    it("creates a draft estimate and returns createResponse([...])", async () => {
      mockCreator.create.mockResolvedValue(draftEstimate);
      const response = await controller.create({ user: authUser }, validCreateRequest);
      expect(response.data).toHaveLength(1);
      expect(response.data?.[0].id).toBe(draftEstimate.id);
    });
  });

  describe("GET /v1/estimates", () => {
    it("returns paginated list with pagination metadata", async () => {
      mockRetriever.findPaginated.mockResolvedValue({
        items: [draftEstimate, sentEstimate],
        pagination: { total: 2, limit: 20, offset: 0 },
      });
      const response = await controller.findAll({ user: authUser }, {});
      expect(response.data).toHaveLength(2);
      expect(response.pagination).toEqual({ total: 2, limit: 20, offset: 0 });
    });

    it("passes status filter through to retriever", async () => {
      mockRetriever.findPaginated.mockResolvedValue({ items: [], pagination: { total: 0, limit: 20, offset: 0 } });
      await controller.findAll({ user: authUser }, { status: EstimateStatus.DRAFT });
      expect(mockRetriever.findPaginated).toHaveBeenCalledWith(authUser, { status: EstimateStatus.DRAFT });
    });
  });

  describe("GET /v1/estimates/:estimateId", () => {
    it("returns estimate detail with totals, priceRange, responseSummary: null", async () => {
      mockRetriever.findByIdOrFail.mockResolvedValue(draftEstimateWithTotals);
      const response = await controller.findOne({ user: authUser }, draftEstimate.id);
      expect(response.data?.[0].totals).toBeDefined();
      expect(response.data?.[0].priceRange).toBeDefined();
      expect(response.data?.[0].responseSummary).toBeNull();
    });
  });

  describe("PATCH /v1/estimates/:estimateId", () => {
    it("delegates to EstimateUpdater.update", async () => {
      mockUpdater.update.mockResolvedValue(draftEstimate);
      await controller.update({ user: authUser }, draftEstimate.id, { title: "new" });
      expect(mockUpdater.update).toHaveBeenCalledWith(authUser, draftEstimate.id, { title: "new" });
    });
  });

  describe("DELETE /v1/estimates/:estimateId", () => {
    it("delegates to EstimateDeleter.softDelete", async () => {
      mockDeleter.softDelete.mockResolvedValue({ ...draftEstimate, status: EstimateStatus.DELETED });
      await controller.softDelete({ user: authUser }, draftEstimate.id);
      expect(mockDeleter.softDelete).toHaveBeenCalledWith(authUser, draftEstimate.id);
    });
  });

  describe("POST /v1/estimates/:estimateId/line-item", () => {
    it("delegates to EstimateUpdater.addLineItem and returns parent estimate", async () => { /* ... */ });
  });

  describe("PATCH /v1/estimates/:estimateId/line-item/:lineItemId", () => {
    it("delegates to EstimateUpdater.updateLineItem", async () => { /* ... */ });
  });

  describe("DELETE /v1/estimates/:estimateId/line-item/:lineItemId", () => {
    it("delegates to EstimateUpdater.deleteLineItem", async () => { /* ... */ });
  });

  describe("D-TXN-05: Phase 41 exposes no non-CRUD transition endpoints", () => {
    it("does not define a route for POST /estimates/:id/transition", () => {
      // Reflection-based check over the controller's methods
      const controllerProto = EstimateController.prototype;
      const methodNames = Object.getOwnPropertyNames(controllerProto).filter((name) => typeof controllerProto[name as keyof typeof controllerProto] === "function" && name !== "constructor");
      const suspects = methodNames.filter((name) => /send|respond|convert|markLost|transition/i.test(name));
      expect(suspects).toEqual([]);
    });
  });
});
```

The reflection-based "no transition endpoints" test is a structural guarantee: if Phase 44/45/47 accidentally adds a method to EstimateController, this test will fail until those endpoints move to their own phase-specific controllers. Keep the test simple — no `as` casts.

**Step 3: Run the slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller
```

Commit message: `feat(41): add EstimateController with 8 authenticated CRUD + line-item endpoints`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/controllers/estimate.controller.ts`
    - `grep -c "@Controller(\"v1\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Post(\"estimates\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Get(\"estimates\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Get(\"estimates/:estimateId\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Patch(\"estimates/:estimateId\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Delete(\"estimates/:estimateId\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Post(\"estimates/:estimateId/line-item\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Patch(\"estimates/:estimateId/line-item/:lineItemId\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Delete(\"estimates/:estimateId/line-item/:lineItemId\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@UseGuards(JwtAuthGuard)" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 8 (every handler authenticated)
    - `grep -c "createHttpError" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 8 (every handler wraps errors)
    - `grep -c "createResponse" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns at least 7 (single-item responses)
    - `grep -c "@Post(\"estimates/:\\(estimateId\\|id\\)/transition\")\\|@Post(\"estimates/:.*id/send\")\\|markLost\\|convert\\|respond" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 0 (D-TXN-05)
    - `grep -c "D-TXN-05: Phase 41 exposes no non-CRUD transition endpoints\\|does not define a route for POST /estimates/:id/transition" trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts` returns at least 1
    - `grep -c "throw new Error(\"implement per quote" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 0 (placeholders replaced)
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller</automated>
  </verify>
  <done>EstimateController exposes 8 authenticated endpoints with full spec coverage and no transition routes</done>
</task>

<task type="auto">
  <name>Task 2: Wire EstimateModule and register in AppModule</name>
  <files>
    trade-flow-api/src/estimate/estimate.module.ts,
    trade-flow-api/src/app.module.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/quote.module.ts (source — full providers + exports set for quote)
    - trade-flow-api/src/app.module.ts (current imports array — must see the existing shape)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Module Wiring — EstimateModule skeleton on lines 747-807)
    - All Plan 05/06/07 service files (read_first verified those files exist; module must list them all)
  </read_first>
  <action>
**Step 1: `estimate.module.ts`** — use the skeleton from RESEARCH.md §Code Examples → Module Wiring (lines 747-807). Final form:

```typescript
import { BusinessModule } from "@business/business.module";
import { CoreModule } from "@core/core.module";
import { CustomerModule } from "@customer/customer.module";
import { DocumentTokenModule } from "@document-token/document-token.module";
import { ItemModule } from "@item/item.module";
import { JobModule } from "@job/job.module";
import { forwardRef, Module } from "@nestjs/common";
import { EstimateController } from "@estimate/controllers/estimate.controller";
import { EstimatePolicy } from "@estimate/policies/estimate.policy";
import { EstimateLineItemPolicy } from "@estimate/policies/estimate-line-item.policy";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateLineItemRepository } from "@estimate/repositories/estimate-line-item.repository";
import { EstimateCreator } from "@estimate/services/estimate-creator.service";
import { EstimateRetriever } from "@estimate/services/estimate-retriever.service";
import { EstimateUpdater } from "@estimate/services/estimate-updater.service";
import { EstimateDeleter } from "@estimate/services/estimate-deleter.service";
import { EstimateNumberGenerator } from "@estimate/services/estimate-number-generator.service";
import { EstimateTotalsCalculator } from "@estimate/services/estimate-totals-calculator.service";
import { EstimateTransitionService } from "@estimate/services/estimate-transition.service";
import { EstimateStandardLineItemFactory } from "@estimate/services/estimate-standard-line-item-factory.service";
import { EstimateBundleLineItemFactory } from "@estimate/services/estimate-bundle-line-item-factory.service";
import { EstimateLineItemCreator } from "@estimate/services/estimate-line-item-creator.service";
import { EstimateLineItemRetriever } from "@estimate/services/estimate-line-item-retriever.service";
import { TaxRateModule } from "@tax-rate/tax-rate.module";
import { UserModule } from "@user/user.module";

@Module({
  imports: [
    CoreModule,
    forwardRef(() => UserModule),
    CustomerModule,
    BusinessModule,
    ItemModule,
    TaxRateModule,
    JobModule,
    forwardRef(() => DocumentTokenModule),
  ],
  controllers: [EstimateController],
  providers: [
    EstimateRepository,
    EstimateLineItemRepository,
    EstimatePolicy,
    EstimateLineItemPolicy,
    EstimateCreator,
    EstimateRetriever,
    EstimateUpdater,
    EstimateDeleter,
    EstimateNumberGenerator,
    EstimateTotalsCalculator,
    EstimateTransitionService,
    EstimateStandardLineItemFactory,
    EstimateBundleLineItemFactory,
    EstimateLineItemCreator,
    EstimateLineItemRetriever,
  ],
  exports: [
    EstimateRepository,
    EstimateLineItemRepository,
    EstimateTransitionService,
    EstimateTotalsCalculator,
    EstimateCreator,
    EstimateRetriever,
  ],
})
export class EstimateModule {}
```

Note: `forwardRef(() => DocumentTokenModule)` — check if QuoteModule uses the same forwardRef pattern for the document-token module. Mirror exactly. If no cross-module dep exists yet (Phase 44 wires the estimate send flow), you can drop `DocumentTokenModule` from imports until then; but declaring it now as a forwardRef is safe and future-proof.

Do not export repositories and services the rest of the app does not need. The minimum export set is what Phases 42-47 will consume:
- Phase 42 needs `EstimateRepository` (for revision chains)
- Phase 44 needs `EstimateRepository`, `EstimateTransitionService` (for send flow)
- Phase 45 needs `EstimateRepository`, `EstimateTransitionService`, `EstimateTotalsCalculator` (for public controller and response handling)
- Phase 47 needs `EstimateRetriever` (for convert source read) and `EstimateTransitionService`

Export at least those five.

**Step 2: `app.module.ts`** — add `EstimateModule` to the imports array alongside `QuoteModule` and `DocumentTokenModule` (which was added in Plan 03 Task 3).

```typescript
import { EstimateModule } from "@estimate/estimate.module";
// ...
@Module({
  imports: [
    // ...existing modules...
    QuoteModule,
    DocumentTokenModule,
    EstimateModule,  // <-- ADD
    // ...
  ],
})
export class AppModule {}
```

**Step 3: Run typecheck + full test suite to confirm DI wiring:**

```
cd trade-flow-api && npm run typecheck
cd trade-flow-api && npm run test
```

A common DI failure at this step: "Nest can't resolve dependencies of the EstimateCreator" — means a provider is missing from the module's providers array. Double-check every service from Plans 05/06/07 is listed.

Commit message: `feat(41): wire EstimateModule and register in AppModule`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/estimate.module.ts`
    - `grep -c "export class EstimateModule" trade-flow-api/src/estimate/estimate.module.ts` returns 1
    - `grep -c "EstimateController" trade-flow-api/src/estimate/estimate.module.ts` returns at least 2 (import + controllers array)
    - `grep -c "EstimateRepository" trade-flow-api/src/estimate/estimate.module.ts` returns at least 3 (import + providers + exports)
    - `grep -c "EstimateLineItemRepository" trade-flow-api/src/estimate/estimate.module.ts` returns at least 2 (import + providers; optionally exports)
    - `grep -c "EstimatePolicy\\|EstimateLineItemPolicy" trade-flow-api/src/estimate/estimate.module.ts` returns at least 4 (2 imports + 2 providers)
    - `grep -c "EstimateCreator\\|EstimateRetriever\\|EstimateUpdater\\|EstimateDeleter" trade-flow-api/src/estimate/estimate.module.ts` returns at least 8 (4 imports + 4 providers)
    - `grep -c "EstimateNumberGenerator" trade-flow-api/src/estimate/estimate.module.ts` returns at least 2
    - `grep -c "EstimateTotalsCalculator" trade-flow-api/src/estimate/estimate.module.ts` returns at least 3 (import + providers + exports)
    - `grep -c "EstimateTransitionService" trade-flow-api/src/estimate/estimate.module.ts` returns at least 3 (import + providers + exports)
    - `grep -c "EstimateStandardLineItemFactory\\|EstimateBundleLineItemFactory\\|EstimateLineItemCreator\\|EstimateLineItemRetriever" trade-flow-api/src/estimate/estimate.module.ts` returns at least 8
    - `grep -c "ItemModule" trade-flow-api/src/estimate/estimate.module.ts` returns at least 2 (imports the module that provides bundle helpers from Plan 02)
    - `grep -c "EstimateModule" trade-flow-api/src/app.module.ts` returns at least 2 (import + imports array entry)
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `cd trade-flow-api && npm run test` exits 0 (full suite — catches any DI miss at spec boot time)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck && npm run test</automated>
  </verify>
  <done>EstimateModule wired with all providers and exports; AppModule registers it; full test suite passes</done>
</task>

<task type="auto">
  <name>Task 3: Update openapi.yaml with 8 new estimate endpoints</name>
  <files>trade-flow-api/openapi.yaml</files>
  <read_first>
    - trade-flow-api/openapi.yaml (full file — executor must locate the `paths:` section, the `components.schemas:` section, and the existing `/v1/quotes` entries to use as a template)
    - trade-flow-api/src/estimate/requests/create-estimate.request.ts (validator decorators → OpenAPI schema constraints)
    - trade-flow-api/src/estimate/requests/update-estimate.request.ts
    - trade-flow-api/src/estimate/requests/list-estimates.request.ts
    - trade-flow-api/src/estimate/responses/estimate.responses.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts (full DTO shape for schema)
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts (new schema)
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts (new schema)
  </read_first>
  <action>
**Step 1: Locate the existing `/v1/quotes` section** in `openapi.yaml` — it serves as the template for the new estimate paths. The structure will look like:

```yaml
paths:
  /v1/quotes:
    get:
      summary: List quotes
      ...
    post:
      summary: Create quote
      ...
  /v1/quotes/{quoteId}:
    get:
      ...
    patch:
      ...
    delete:
      ...
```

**Step 2: Add a parallel `/v1/estimates*` section** with 8 endpoints:

```yaml
  /v1/estimates:
    post:
      summary: Create an estimate
      operationId: createEstimate
      tags: [estimates]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateEstimateRequest"
      responses:
        "201":
          description: Estimate created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EstimateResponseEnvelope"
        "401": { $ref: "#/components/responses/Unauthorized" }
        "422": { $ref: "#/components/responses/InvalidRequest" }

    get:
      summary: List estimates with optional status filter and pagination
      operationId: listEstimates
      tags: [estimates]
      security:
        - bearerAuth: []
      parameters:
        - in: query
          name: status
          schema:
            type: string
            enum: [draft, sent, viewed, responded, site_visit_requested, converted, declined, expired, lost, deleted]
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
        - in: query
          name: offset
          schema: { type: integer, minimum: 0, default: 0 }
      responses:
        "200":
          description: Paginated estimate list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EstimateListResponseEnvelope"
        "401": { $ref: "#/components/responses/Unauthorized" }

  /v1/estimates/{estimateId}:
    parameters:
      - in: path
        name: estimateId
        required: true
        schema: { type: string, format: objectid }
    get:
      summary: Get estimate detail
      operationId: getEstimate
      tags: [estimates]
      security:
        - bearerAuth: []
      responses:
        "200":
          description: Estimate detail with line items, totals, price range
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
        "404": { $ref: "#/components/responses/NotFound" }
    patch:
      summary: Edit a Draft estimate
      operationId: updateEstimate
      tags: [estimates]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/UpdateEstimateRequest" }
      responses:
        "200":
          description: Estimate updated
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
        "404": { $ref: "#/components/responses/NotFound" }
        "422":
          description: Estimate not in Draft status or invalid payload
    delete:
      summary: Soft-delete a Draft estimate
      operationId: deleteEstimate
      tags: [estimates]
      security:
        - bearerAuth: []
      responses:
        "200":
          description: Estimate soft-deleted
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
        "404": { $ref: "#/components/responses/NotFound" }
        "422":
          description: Estimate not in Draft status

  /v1/estimates/{estimateId}/line-item:
    parameters:
      - in: path
        name: estimateId
        required: true
        schema: { type: string, format: objectid }
    post:
      summary: Add a line item to a Draft estimate
      operationId: addEstimateLineItem
      tags: [estimates]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/CreateEstimateLineItemRequest" }
      responses:
        "200":
          description: Estimate with new line item attached
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
        "404": { $ref: "#/components/responses/NotFound" }
        "422": description: Invalid payload or non-Draft status

  /v1/estimates/{estimateId}/line-item/{lineItemId}:
    parameters:
      - in: path
        name: estimateId
        required: true
        schema: { type: string, format: objectid }
      - in: path
        name: lineItemId
        required: true
        schema: { type: string, format: objectid }
    patch:
      summary: Update a line item on a Draft estimate
      operationId: updateEstimateLineItem
      tags: [estimates]
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/UpdateEstimateLineItemRequest" }
      responses:
        "200":
          description: Estimate with updated line item
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
    delete:
      summary: Soft-delete a line item on a Draft estimate
      operationId: deleteEstimateLineItem
      tags: [estimates]
      security:
        - bearerAuth: []
      responses:
        "200":
          description: Estimate with line item removed
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EstimateResponseEnvelope" }
```

**Step 3: Add schemas under `components.schemas:`**

Add these schema definitions (content modeled on the `QuoteResponse` schema and on the DTO shapes from Plan 04):

- `EstimateResponse` — mirrors the IEstimateResponse shape including `totals`, `priceRange`, `responseSummary: null`, and all reserved fields
- `EstimateResponseEnvelope` — `{ data: [EstimateResponse] }` style single-item envelope
- `EstimateListResponseEnvelope` — `{ data: [EstimateResponse], pagination: QueryResults }`
- `CreateEstimateRequest` — matches the class-validator decorators (contingencyPercent enum, displayMode enum, required customerId/jobId/title)
- `UpdateEstimateRequest` — same shape, all fields optional
- `CreateEstimateLineItemRequest` — mirror the existing `CreateQuoteLineItemRequest` schema
- `UpdateEstimateLineItemRequest` — mirror `UpdateQuoteLineItemRequest`
- `EstimateTotals` — `{ subTotal, taxTotal, total }` (Money values as { amount: integer, currency: string })
- `EstimatePriceRange` — `{ displayMode, subtotal, contingencyPercent, low, high }` (bounds contain `{ exclTax, tax, inclTax }`)
- `EstimateResponseSummary` — nullable object

IMPORTANT: If the existing `openapi.yaml` uses a different schema layout convention (flat vs nested, `$ref` vs inline), mirror it exactly. Do NOT invent a new convention.

**Step 4: Remove any lingering `quote-token` internal type references** — search for any internal types from the deleted `quote-token` module:

```
grep -n "quote-token\\|QuoteToken\\|QuoteSessionAuth" trade-flow-api/openapi.yaml
```

If any are found (typical: `IQuoteTokenEntity` referenced in a schema), replace with the document-token equivalents. The public URL `/v1/public/quote/:token` MUST remain unchanged in the OpenAPI paths.

**Step 5: Verify the YAML parses cleanly.** If a YAML validator is installed, run it. Otherwise rely on the next task's `npm run ci` which may include an OpenAPI lint step (check the `package.json` scripts; if there's no explicit validator, the manual inspection + final `npm run ci` must suffice).

Commit message: `docs(41): add estimate endpoints and DTOs to openapi.yaml`.
  </action>
  <acceptance_criteria>
    - `grep -c "/v1/estimates" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "/v1/estimates/{estimateId}" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "/v1/estimates/{estimateId}/line-item" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "/v1/estimates/{estimateId}/line-item/{lineItemId}" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "operationId: createEstimate" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: listEstimates" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: getEstimate" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: updateEstimate" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: deleteEstimate" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: addEstimateLineItem" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: updateEstimateLineItem" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "operationId: deleteEstimateLineItem" trade-flow-api/openapi.yaml` returns 1
    - `grep -c "CreateEstimateRequest\\|UpdateEstimateRequest\\|EstimateResponse\\|EstimatePriceRange" trade-flow-api/openapi.yaml` returns at least 4
    - `grep -c "quote-token\\|QuoteToken\\|QuoteSessionAuth" trade-flow-api/openapi.yaml` returns 0
    - `grep -c "/v1/public/quote/{token}" trade-flow-api/openapi.yaml` returns at least 1 (public quote URL preserved — regression check)
  </acceptance_criteria>
  <verify>
    <automated>grep -c "operationId: createEstimate" trade-flow-api/openapi.yaml && grep -c "operationId: deleteEstimateLineItem" trade-flow-api/openapi.yaml && grep -c "/v1/public/quote/{token}" trade-flow-api/openapi.yaml</automated>
  </verify>
  <done>openapi.yaml contains all 8 new endpoints with schemas; no stale quote-token internal types; public quote URL preserved</done>
</task>

<task type="auto">
  <name>Task 4: Rewrite ROADMAP.md / v1.8-ROADMAP.md / STATE.md per D-TKN-07 and fix estimatelineitems prose drift per D-ENT-02</name>
  <files>
    .planning/ROADMAP.md,
    .planning/milestones/v1.8-ROADMAP.md,
    .planning/STATE.md
  </files>
  <read_first>
    - .planning/ROADMAP.md (current Phase 41 section — success criterion #5 AND all prose using `estimate_line_items` vs `estimatelineitems` per D-ENT-02)
    - .planning/milestones/v1.8-ROADMAP.md (if it exists — the canonical milestone roadmap that mirrors the top-level ROADMAP.md)
    - .planning/STATE.md (check whether Phase 41 criterion #5 is referenced anywhere; D-TKN-07 requires the planner to update STATE.md as part of this phase)
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (D-TKN-07 LOCKED replacement text AND D-ENT-02 locked collection name)
  </read_first>
  <action>
**Step 1: Locate the current success criterion #5 in `.planning/ROADMAP.md`** (Phase 41 section, which the executor already saw in Task 1 context). The current wording is:

> 5. `quote-token` module is renamed to `document-token` end-to-end: entity field `quoteId` -> `documentId`, `documentType: "quote" | "estimate"` discriminator added, collection `quote_tokens` -> `document_tokens` via one-shot reversible migration, guard renamed to `DocumentSessionAuthGuard`, existing quote tokens continue to validate, and the public quote page (`/v1/public/quote/:token`) still resolves its token unchanged.

**Step 2: Replace it with the LOCKED wording from D-TKN-07:**

> 5. The `quote-token` module is fully replaced by the new `document-token` module at the code level. The existing `/v1/public/quote/:token` endpoint continues to resolve tokens with `documentType === "quote"` via the new guard. No data migration is required because no production data exists.

**Step 3: Apply the same rewrite to `.planning/milestones/v1.8-ROADMAP.md`** if that file exists. Use the same locked wording. If the file does not exist, skip this step (the top-level ROADMAP.md is the only canonical source).

**Step 4: Update `.planning/STATE.md` per D-TKN-07.** D-TKN-07 explicitly directs the planner to update STATE.md as part of Phase 41 execution. Two paths:

- If `grep -nE "one-shot reversible migration|quote-token.*rename.*document-token" .planning/STATE.md` returns any match, rewrite that text to align with the D-TKN-07 locked wording (use the same replacement text as Steps 1-2).
- If no match exists (STATE.md has no Phase 41 criterion #5 reference), add a one-line note under the current Phase 41 section confirming the locked wording is authoritative, OR record the absence explicitly in the commit body. Either way, the acceptance criteria below must pass.

**Step 5: Fix the `estimate_line_items` / `estimatelineitems` drift in `.planning/ROADMAP.md` per D-ENT-02.** The locked collection name (D-ENT-02) is `estimatelineitems` with no underscore. ROADMAP.md Phase 41 prose currently uses `estimate_line_items`. Replace every occurrence of the underscore form in the Phase 41 prose with the locked form. Scope the edit to the Phase 41 section only (between `### Phase 41` and `### Phase 42` headings) — do not alter the phase slug `estimate-module-crud-backend` or any hyphenated usage such as `estimate-line-item.repository`. Apply the same fix to `.planning/milestones/v1.8-ROADMAP.md` if that file references `estimate_line_items` in the Phase 41 section.

**Step 6: Commit:**

```
git add .planning/ROADMAP.md .planning/milestones/v1.8-ROADMAP.md .planning/STATE.md
git commit -m "docs(41): rewrite phase 41 success criterion #5 per D-TKN-07 and align estimatelineitems prose per D-ENT-02"
```
  </action>
  <acceptance_criteria>
    - `grep -c "one-shot reversible migration" .planning/ROADMAP.md` returns 0 (old wording gone)
    - `grep -c "fully replaced by the new \`document-token\` module at the code level\\|No data migration is required because no production data exists" .planning/ROADMAP.md` returns at least 1 (new wording present)
    - If `.planning/milestones/v1.8-ROADMAP.md` exists:
      - `grep -c "one-shot reversible migration" .planning/milestones/v1.8-ROADMAP.md` returns 0
      - `grep -c "No data migration is required" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
    - `grep -c "one-shot reversible migration" .planning/STATE.md` returns 0 (STATE.md contains no stale D-TKN wording)
    - Either `grep -c "No data migration is required" .planning/STATE.md` returns at least 1 (locked wording present) OR `grep -c "Phase 41.*criterion #5\\|quote-token.*document-token" .planning/STATE.md` returns 0 (locked absence confirmed); the executor records which branch applied in the commit body
    - `awk '/^### Phase 41/,/^### Phase 42/' .planning/ROADMAP.md | grep -c "estimate_line_items"` returns 0 (D-ENT-02 drift fixed in Phase 41 prose)
    - `awk '/^### Phase 41/,/^### Phase 42/' .planning/ROADMAP.md | grep -c "estimatelineitems"` returns at least 1 (locked collection name present)
    - The rest of Phase 41 success criteria 1-4 and 6 are UNCHANGED (sanity check with `grep -c "atomically generated\\|paginated list with tab filtering" .planning/ROADMAP.md` returns at least 2)
  </acceptance_criteria>
  <verify>
    <automated>grep -c "No data migration is required" .planning/ROADMAP.md</automated>
  </verify>
  <done>ROADMAP.md, v1.8-ROADMAP.md, and STATE.md success criterion #5 all match the D-TKN-07 locked wording; ROADMAP.md Phase 41 prose uses `estimatelineitems` per D-ENT-02</done>
</task>

<task type="auto">
  <name>Task 5: Run the FINAL full CI gate for the phase</name>
  <files>none</files>
  <read_first>- trade-flow-api/package.json</read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected: exit code 0.

This is the FINAL gate for Phase 41. It must exit 0 with:
- Every test from Plans 01-08 passing
- Every existing test (quote, document-token, item, customer, business, job, auth, etc.) still passing
- No lint errors, no format errors, no type errors
- No `@ts-ignore`, `eslint-disable`, `as any`, `as unknown as` introduced anywhere in the Phase 41 diff (except the one documented test-fixture exception in Plan 07 Task 1)

If any failure:
1. **Test failure in estimate/**: fix the spec or source at the root cause
2. **Test failure in quote/ or document-token/**: something regressed — investigate which plan introduced it, fix before continuing
3. **Lint failure**: fix the rule violation in place (unused import, missing type, etc.)
4. **Format failure**: `cd trade-flow-api && npm run format` and commit as `chore(41): prettier fixes for phase 41 final gate`
5. **Typecheck failure**: most likely a missing export from EstimateModule or a DI wiring miss — fix at the module level

Do NOT suppress. Do NOT skip tests. Fix root causes per CLAUDE.md CI gate policy.

Post-fix commit message: `chore(41): phase 41 final CI gate green`.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - `grep -rc "@ts-ignore\\|@ts-expect-error\\|eslint-disable" trade-flow-api/src/estimate trade-flow-api/src/document-token trade-flow-api/src/item/services/bundle-* trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts` returns 0
    - `grep -rn "@quote-token/" trade-flow-api/src` returns 0 matches
    - `grep -rn "@quote/services/bundle-" trade-flow-api/src` returns 0 matches
    - `find trade-flow-api/src/quote-token -type f 2>/dev/null` returns empty
    - `test -d trade-flow-api/src/document-token`
    - `test -d trade-flow-api/src/estimate`
    - Every spec file listed in 41-VALIDATION.md §Wave 0 Requirements exists
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Phase 41 complete: full CI gate green, 28 spec files exist, no suppressions, ROADMAP updated, OpenAPI updated, estimate module fully wired</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| HTTP client → EstimateController | Untrusted input; JwtAuthGuard authenticates; ValidationPipe validates request shapes |
| EstimateModule → AppModule | Internal DI graph; no trust boundary |
| OpenAPI spec → external consumers | Contract surface; incorrect schemas leak or hide behavior |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-08-01 | Spoofing | Unauthenticated access to estimate endpoints | mitigate | Every handler has `@UseGuards(JwtAuthGuard)`; Task 1 acceptance criterion verifies count is 8 |
| T-41-08-02 | Elevation of Privilege | Phase 41 controller accidentally exposes non-CRUD transition endpoint (send/respond/convert/markLost) | mitigate | D-TXN-05 lockdown; Task 1 spec has reflection-based assertion that no suspect method names exist on the controller |
| T-41-08-03 | Tampering | Mass-assignment via PATCH body bypasses Draft guard | mitigate | UpdateEstimateRequest whitelists fields; global ValidationPipe `forbidNonWhitelisted: true`; Plan 07 EstimateUpdater re-fetches state and checks Draft at service layer |
| T-41-08-04 | Information Disclosure | OpenAPI schema exposes internal field `lastResponseMessage` before Phase 45 is ready | accept | Phase 41 reserves these fields at the DTO level (RESP-08); OpenAPI documents them as nullable/read-only; no behavioral impact until Phase 45 populates them |
| T-41-08-05 | Denial of Service | Public list endpoint accepts unbounded pagination | mitigate | ListEstimatesRequest @Max(100) on limit; EstimateRetriever defaults to 20; EstimateRepository cursor applies `.limit(n)` |
| T-41-08-06 | Tampering | Incorrect DI wiring lets an unauthorized path reach the repository directly | mitigate | Task 2 runs full `npm run test` to boot every TestingModule; any DI failure fails a spec; module exports are explicit so unexported services cannot be injected elsewhere |
| T-41-08-07 | Information Disclosure | Error messages in openapi.yaml expose internal state | mitigate | OpenAPI response schemas reference `createErrorResponse` format which sanitizes; no raw stack traces; no internal IDs in error responses |
| T-41-08-08 | Tampering | ROADMAP rewrite drops an unrelated line by accident | mitigate | Task 4 acceptance criterion grep for "atomically generated" and "paginated list with tab filtering" confirms other success criteria are untouched |
</threat_model>

<verification>
1. `EstimateController` has 8 authenticated routes, no transition routes (D-TXN-05 enforced structurally).
2. `EstimateModule` providers list contains every class from Plans 05/06/07.
3. `AppModule.imports` contains `EstimateModule`, `DocumentTokenModule`, `QuoteModule` (no `QuoteTokenModule`).
4. `openapi.yaml` documents 8 new endpoints and removes any `quote-token` internal types.
5. `.planning/ROADMAP.md` Phase 41 success criterion #5 matches D-TKN-07 wording.
6. `cd trade-flow-api && npm run ci` exits 0 (phase-final gate).
7. All 28 spec files from 41-VALIDATION.md §Wave 0 Requirements exist.
</verification>

<success_criteria>
- An authenticated trader can hit `POST /v1/estimates`, `GET /v1/estimates`, `GET /v1/estimates/:id`, `PATCH /v1/estimates/:id`, `DELETE /v1/estimates/:id`, `POST /v1/estimates/:id/line-item`, `PATCH /v1/estimates/:id/line-item/:lineItemId`, `DELETE /v1/estimates/:id/line-item/:lineItemId` against the running API and receive correct responses (verified via unit tests at the controller + service layers)
- Phase 42 (revisions), Phase 43 (frontend), Phase 44 (send flow) can build on Phase 41's exported services without further backend refactor
- Every phase requirement ID (EST-01..09, CONT-01/02/05, RESP-08) has at least one spec asserting its behavior
- No type suppressions or lint disables anywhere in the Phase 41 diff
- `quote-token` is fully retired; `/v1/public/quote/:token` still resolves via DocumentSessionAuthGuard
- ROADMAP success criterion #5 and the accompanying OpenAPI spec both reflect the pure-code-rename reality
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-08-SUMMARY.md` documenting:
- Final CI gate output
- The 8 endpoints and their operationIds
- The module providers/exports summary
- The ROADMAP rewrite diff
- Confirmation that every requirement ID is covered by at least one spec
</output>
