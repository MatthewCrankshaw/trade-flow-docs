---
phase: 42
plan: 06
type: execute
wave: 4
depends_on: ["42-04", "42-05"]
files_modified:
  - trade-flow-api/src/estimate/controllers/estimate.controller.ts
  - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/openapi.yaml
  - trade-flow-api/docs/smoke/phase-42-concurrent-revise.md
autonomous: true
requirements: [REV-02, REV-03, REV-04, REV-05]
estimate: 55min
tags: [controller, module, openapi, smoke, phase-42]

must_haves:
  truths:
    - "`POST /v1/estimates/:id/revisions` exists on `EstimateController` with `@UseGuards(JwtAuthGuard)`, takes no body, returns 201 with the new revision DTO wrapped via `createResponse([...])`, and maps errors via `createHttpError` (including 409 ConflictError → 409 Conflict and 404 ResourceNotFoundError → 404)."
    - "`GET /v1/estimates/:id/revisions` exists on `EstimateController` with `@UseGuards(JwtAuthGuard)`, returns 200 with the full chain as `IEstimateResponse[]`, resolves both root id and revision id to the same chain."
    - "`EstimateModule` registers `EstimateReviser`, `NoopEstimateFollowupCanceller`, and `{provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller}`; exports `ESTIMATE_FOLLOWUP_CANCELLER` for Phase 44/46 re-import."
    - "`openapi.yaml` documents both new endpoints with 201/200 happy paths, 409 Conflict for revise, 422 Unprocessable Entity for non-revisable state (where applicable), 404 for missing id, 403 for forbidden."
    - "`trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` exists with step-by-step docker-compose + parallel-curl instructions exercising SC #4."
    - "`cd trade-flow-api && npm run ci` exits 0."
  artifacts:
    - path: "trade-flow-api/src/estimate/controllers/estimate.controller.ts"
      provides: "Two new endpoints: POST and GET /v1/estimates/:id/revisions"
    - path: "trade-flow-api/src/estimate/estimate.module.ts"
      provides: "Registered reviser + noop canceller + ESTIMATE_FOLLOWUP_CANCELLER provider + export"
    - path: "trade-flow-api/openapi.yaml"
      provides: "API contract documentation for the two new endpoints"
    - path: "trade-flow-api/docs/smoke/phase-42-concurrent-revise.md"
      provides: "Manual smoke procedure for SC #4 concurrency verification"
  key_links:
    - from: "trade-flow-api/src/estimate/controllers/estimate.controller.ts"
      to: "trade-flow-api/src/estimate/services/estimate-reviser.service.ts"
      via: "`@Post('estimates/:id/revisions')` calls `estimateReviser.revise(authUser, id)`"
      pattern: "estimateReviser\\.revise"
    - from: "trade-flow-api/src/estimate/controllers/estimate.controller.ts"
      to: "trade-flow-api/src/estimate/services/estimate-retriever.service.ts"
      via: "`@Get('estimates/:id/revisions')` calls `estimateRetriever.findRevisionsByIdOrFail(authUser, id)`"
      pattern: "findRevisionsByIdOrFail"
    - from: "trade-flow-api/src/estimate/estimate.module.ts"
      to: "trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts"
      via: "Provider registration under ESTIMATE_FOLLOWUP_CANCELLER token"
      pattern: "useClass: NoopEstimateFollowupCanceller"
---

<objective>
Land the last wave of Phase 42: two new HTTP endpoints on `EstimateController`, module wiring for the reviser + noop canceller + DI token, OpenAPI documentation for both endpoints, and the manual smoke procedure for SC #4 (user-approved in lieu of `mongodb-memory-server` — Q1). After this plan, all 35 Phase 42 decisions have been implemented and the phase is ready for `/gsd-verify-work`.

This plan depends on plans 42-04 (reviser) and 42-05 (retriever/deleter extensions) because it imports `EstimateReviser` and calls `EstimateRetriever.findRevisionsByIdOrFail`. It touches existing Phase 41 files (`estimate.controller.ts`, `estimate.module.ts`, `openapi.yaml`) plus one new smoke document.

Output: five files modified (three source files amended, one yaml amended, one new markdown), one commit, CI green.
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
@trade-flow-api/src/estimate/controllers/estimate.controller.ts
@trade-flow-api/src/estimate/estimate.module.ts
@trade-flow-api/src/quote/controllers/quote.controller.ts
@trade-flow-api/src/quote/quote.module.ts
@trade-flow-api/openapi.yaml
@trade-flow-api/docker-compose.yaml

<interfaces>
<!-- Controller additions (Phase 42) -->

```typescript
@UseGuards(JwtAuthGuard)
@Post("estimates/:id/revisions")
public async revise(
  @Req() request: { user: IUserDto; params: { id: string } },
): Promise<IResponse<IEstimateResponse>> {
  try {
    const revision = await this.estimateReviser.revise(request.user, request.params.id);
    const response = await this.enrichAndMapToResponse(request.user, revision);
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
    const responses = await Promise.all(chain.map((revision) => this.enrichAndMapToResponse(request.user, revision)));
    return createResponse(responses);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

<!-- Module additions -->

```typescript
import { ESTIMATE_FOLLOWUP_CANCELLER } from "@estimate/services/estimate-followup-canceller.interface";
import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";
import { EstimateReviser } from "@estimate/services/estimate-reviser.service";

@Module({
  imports: [/* Phase 41 imports */],
  controllers: [EstimateController],
  providers: [
    // Phase 41 providers:
    EstimateCreator,
    EstimateRetriever,
    EstimateUpdater,
    EstimateDeleter,
    EstimateTotalsCalculator,
    EstimateNumberGenerator,
    EstimateTransitionService,
    EstimatePolicy,
    EstimateRepository,
    EstimateLineItemRepository,
    // ...line-item factories...

    // Phase 42 additions:
    EstimateReviser,
    NoopEstimateFollowupCanceller,
    { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },
  ],
  exports: [
    // Phase 41 exports...
    // Phase 42 addition:
    ESTIMATE_FOLLOWUP_CANCELLER,
  ],
})
export class EstimateModule {}
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Add POST + GET /v1/estimates/:id/revisions handlers to EstimateController with spec coverage</name>
  <files>trade-flow-api/src/estimate/controllers/estimate.controller.ts, trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts (entire Phase 41 file — confirm `@Controller("v1")` class decorator, constructor dependencies, existing handler shapes, the private `enrichAndMapToResponse` method, and how errors are wrapped)
    - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts (Phase 41 spec — confirm mock pattern for the existing services)
    - trade-flow-api/src/quote/controllers/quote.controller.ts (the blessed mirror for controller shape, `createHttpError` try/catch, `createResponse([...])` wrapping)
    - trade-flow-api/src/estimate/services/estimate-reviser.service.ts (produced by plan 42-04 — confirm `revise(authUser, id)` signature)
    - trade-flow-api/src/estimate/services/estimate-retriever.service.ts (produced by plan 42-05 — confirm `findRevisionsByIdOrFail(authUser, id)` signature)
    - trade-flow-api/src/estimate/responses/estimate.response.ts (confirm `IEstimateResponse` shape for the mapper output)
  </read_first>
  <behavior>
    - Test 1: `POST /v1/estimates/:id/revisions` calls `estimateReviser.revise(authUser, id)` and returns `createResponse([mappedResponse])` wrapped in IResponse.
    - Test 2: POST → 409 when the reviser throws `ConflictError` (verified via `createHttpError` mapping).
    - Test 3: POST → 403 when the reviser throws `ForbiddenError`.
    - Test 4: POST → 404 when the reviser throws `ResourceNotFoundError` (missing source id).
    - Test 5: POST does not read `@Body()` — no body parameter declared on the method.
    - Test 6: `GET /v1/estimates/:id/revisions` calls `estimateRetriever.findRevisionsByIdOrFail(authUser, id)` and wraps the resulting array in `createResponse([...])`.
    - Test 7: GET returns chain ordered by revisionNumber ascending (delegation — asserted by mocking the retriever to return `[root, rev2, rev3]` and checking the response data order).
    - Test 8: GET → 404 when retriever throws `ResourceNotFoundError`.
    - Test 9: GET → 403 when retriever throws `ForbiddenError`.
    - Test 10: GET with a revision id resolves to the same chain as GET with the root id (delegation behavior — asserted by mocking `findRevisionsByIdOrFail` to return the chain for both call shapes).
  </behavior>
  <action>
    **Step 0 — Read the current `estimate.controller.ts`.** Confirm:
    - The class decorator `@Controller("v1")` is present.
    - The constructor lists `EstimateCreator`, `EstimateRetriever`, `EstimateUpdater`, `EstimateDeleter`, plus any others from Phase 41.
    - The private helper `enrichAndMapToResponse(authUser, estimate)` exists — this is where responses are mapped with totals + customer/job info.
    - Existing handlers use the `try { ... } catch (error) { throw createHttpError(error); }` pattern.

    **Step 1 — Add the two new handlers.**

    Add `EstimateReviser` to the constructor:

    ```typescript
    constructor(
      private readonly estimateCreator: EstimateCreator,
      private readonly estimateRetriever: EstimateRetriever,
      private readonly estimateUpdater: EstimateUpdater,
      private readonly estimateDeleter: EstimateDeleter,
      private readonly estimateReviser: EstimateReviser, // Phase 42
    ) {}
    ```

    Add the two handlers. Place them adjacent to existing GET `/estimates/:id` and POST `/estimates` handlers for readability:

    ```typescript
    @UseGuards(JwtAuthGuard)
    @Post("estimates/:id/revisions")
    public async revise(
      @Req() request: { user: IUserDto; params: { id: string } },
    ): Promise<IResponse<IEstimateResponse>> {
      try {
        const revision = await this.estimateReviser.revise(request.user, request.params.id);
        const response = await this.enrichAndMapToResponse(request.user, revision);
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
        const chain = await this.estimateRetriever.findRevisionsByIdOrFail(
          request.user,
          request.params.id,
        );
        const responses = await Promise.all(
          chain.map((revision) => this.enrichAndMapToResponse(request.user, revision)),
        );
        return createResponse(responses);
      } catch (error) {
        throw createHttpError(error);
      }
    }
    ```

    **Critical notes:**
    - No `@Body()` on the POST handler — D-REV-01 says the endpoint takes no body. The global `ValidationPipe` with `whitelist: true` will strip any body the client happens to send.
    - Both handlers use the existing `enrichAndMapToResponse` pattern — no new mapper code.
    - The GET handler uses `Promise.all` for mapping — parallel, not sequential, because mapping is CPU-bound per row.
    - Import `EstimateReviser` from `@estimate/services/estimate-reviser.service`.
    - No new imports beyond the reviser.

    **Step 2 — Amend the spec file.**

    Add a new `describe("Phase 42 revision endpoints")` block in `estimate.controller.spec.ts`:

    ```typescript
    describe("Phase 42 revision endpoints", () => {
      let mockReviser: jest.Mocked<EstimateReviser>;

      beforeEach(async () => {
        mockReviser = {
          revise: jest.fn(),
        } as unknown as jest.Mocked<EstimateReviser>;

        const module: TestingModule = await Test.createTestingModule({
          controllers: [EstimateController],
          providers: [
            { provide: EstimateCreator, useValue: mockCreator },
            { provide: EstimateRetriever, useValue: mockRetriever },
            { provide: EstimateUpdater, useValue: mockUpdater },
            { provide: EstimateDeleter, useValue: mockDeleter },
            { provide: EstimateReviser, useValue: mockReviser },
            // ...other DI overrides as per the Phase 41 spec...
          ],
        }).compile();

        controller = module.get<EstimateController>(EstimateController);
      });

      describe("POST /v1/estimates/:id/revisions", () => {
        it("calls EstimateReviser.revise and returns createResponse with the mapped new revision", async () => {
          const source = EstimateRevisionMockGenerator.createRoot({ status: EstimateStatus.SENT });
          const revision = EstimateRevisionMockGenerator.createChildRevision(source);
          mockReviser.revise.mockResolvedValue(revision);

          const request = { user: authUser, params: { id: source.id } };
          const result = await controller.revise(request);

          expect(mockReviser.revise).toHaveBeenCalledWith(authUser, source.id);
          expect(result.data).toHaveLength(1);
          expect(result.data?.[0].id).toBe(revision.id);
        });

        it("maps ConflictError → HttpException 409 via createHttpError", async () => {
          mockReviser.revise.mockRejectedValue(
            new ConflictError(
              ErrorCodes.ESTIMATE_REVISION_CONFLICT,
              "Estimate has already been revised or is no longer revisable",
            ),
          );
          const request = { user: authUser, params: { id: "some-id" } };
          await expect(controller.revise(request)).rejects.toThrow();
          // createHttpError wraps ConflictError as HttpException with status 409
        });

        it("maps ForbiddenError → HttpException 403", async () => {
          mockReviser.revise.mockRejectedValue(new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "d"));
          await expect(controller.revise({ user: authUser, params: { id: "some-id" } })).rejects.toThrow();
        });

        it("maps ResourceNotFoundError → HttpException 404", async () => {
          mockReviser.revise.mockRejectedValue(
            new ResourceNotFoundError(ErrorCodes.ESTIMATE_NOT_FOUND, "d"),
          );
          await expect(controller.revise({ user: authUser, params: { id: "missing" } })).rejects.toThrow();
        });
      });

      describe("GET /v1/estimates/:id/revisions", () => {
        it("calls findRevisionsByIdOrFail and wraps the chain in createResponse", async () => {
          const root = EstimateRevisionMockGenerator.createRoot();
          const rev2 = EstimateRevisionMockGenerator.createChildRevision(root);
          const rev3 = EstimateRevisionMockGenerator.createChildRevision(rev2);
          mockRetriever.findRevisionsByIdOrFail.mockResolvedValue([root, rev2, rev3]);

          const result = await controller.findRevisions({ user: authUser, params: { id: root.id } });

          expect(mockRetriever.findRevisionsByIdOrFail).toHaveBeenCalledWith(authUser, root.id);
          expect(result.data).toHaveLength(3);
          expect(result.data?.[0].id).toBe(root.id);
          expect(result.data?.[2].id).toBe(rev3.id);
        });

        it("resolves revision id to the same chain as root id (delegation)", async () => {
          const root = EstimateRevisionMockGenerator.createRoot();
          const rev2 = EstimateRevisionMockGenerator.createChildRevision(root);
          mockRetriever.findRevisionsByIdOrFail.mockResolvedValue([root, rev2]);

          await controller.findRevisions({ user: authUser, params: { id: rev2.id } });
          expect(mockRetriever.findRevisionsByIdOrFail).toHaveBeenCalledWith(authUser, rev2.id);
        });

        it("maps ResourceNotFoundError → 404", async () => {
          mockRetriever.findRevisionsByIdOrFail.mockRejectedValue(
            new ResourceNotFoundError(ErrorCodes.ESTIMATE_NOT_FOUND, "d"),
          );
          await expect(
            controller.findRevisions({ user: authUser, params: { id: "missing" } }),
          ).rejects.toThrow();
        });

        it("maps ForbiddenError → 403", async () => {
          mockRetriever.findRevisionsByIdOrFail.mockRejectedValue(
            new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "d"),
          );
          await expect(
            controller.findRevisions({ user: authUser, params: { id: "other-business-id" } }),
          ).rejects.toThrow();
        });
      });
    });
    ```

    Match the existing spec file's DI mock setup — the block above may need to merge into a shared `beforeEach` that Phase 41 already defined.

    **Step 3 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller
    ```

    **Prohibitions:**
    - No `any`, no `as` in production code.
    - No new imports beyond `EstimateReviser`.
    - No `@Body()` on the POST handler.
    - No `eslint-disable`, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`.
  </action>
  <acceptance_criteria>
    - `grep -c "@Post(\"estimates/:id/revisions\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Get(\"estimates/:id/revisions\")" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "estimateReviser.revise" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "findRevisionsByIdOrFail" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 1
    - `grep -c "@Body" trade-flow-api/src/estimate/controllers/estimate.controller.ts | head -n 1` (count of `@Body` on revise handler) — verify the POST revise method has zero `@Body` annotations by manual inspection; if the controller has `@Body` elsewhere the count may be >0 but the revise handler specifically has none
    - `awk '/public async revise/,/^  \\}/' trade-flow-api/src/estimate/controllers/estimate.controller.ts | grep -c "@Body"` returns 0
    - `grep -c "EstimateReviser" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns at least 2 (import + constructor)
    - `grep -c "createHttpError" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns at least 2 (both new handlers wrap with createHttpError)
    - `grep -c "Phase 42 revision endpoints" trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts` returns 1
    - `grep -c "ConflictError" trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts` returns at least 1
    - `grep -c " any\\| as " trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/controllers/estimate.controller.ts` returns 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate.controller` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate.controller</automated>
  </verify>
  <done>Both endpoints exist on EstimateController, tested for happy path + 409/403/404 error mapping + delegation of GET to findRevisionsByIdOrFail; slice green.</done>
</task>

<task type="auto">
  <name>Task 2: Register EstimateReviser + NoopEstimateFollowupCanceller + ESTIMATE_FOLLOWUP_CANCELLER in EstimateModule</name>
  <files>trade-flow-api/src/estimate/estimate.module.ts</files>
  <read_first>
    - trade-flow-api/src/estimate/estimate.module.ts (entire file — confirm current providers list + exports)
    - trade-flow-api/src/quote/quote.module.ts (the blessed mirror for module structure)
    - trade-flow-api/src/estimate/services/estimate-reviser.service.ts (confirm class is @Injectable())
    - trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts (confirm class is @Injectable())
    - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts (confirm ESTIMATE_FOLLOWUP_CANCELLER is exported as a const string)
  </read_first>
  <action>
    Amend `trade-flow-api/src/estimate/estimate.module.ts`.

    **Step 1 — Add imports at the top:**

    ```typescript
    import { EstimateReviser } from "@estimate/services/estimate-reviser.service";
    import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";
    import { ESTIMATE_FOLLOWUP_CANCELLER } from "@estimate/services/estimate-followup-canceller.interface";
    ```

    **Step 2 — Add to the `providers` array.** Find the existing providers list (after Phase 41 services, likely near the end of the array). Append:

    ```typescript
    providers: [
      // ...existing Phase 41 providers...
      EstimateReviser,
      NoopEstimateFollowupCanceller,
      { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },
    ],
    ```

    **Step 3 — Add to the `exports` array.** `ESTIMATE_FOLLOWUP_CANCELLER` must be exported so Phase 44's `EstimateEmailSender` (or Phase 46's `EstimateFollowupsModule`) can import it via `EstimateModule`:

    ```typescript
    exports: [
      // ...existing Phase 41 exports...
      ESTIMATE_FOLLOWUP_CANCELLER,
    ],
    ```

    `EstimateReviser` does NOT need to be exported because no other module calls it — only the controller inside this module. Same for `NoopEstimateFollowupCanceller` (it's an implementation detail; consumers inject the token, not the class).

    **Step 4 — Verify by running the test suite.** NestJS will fail fast at module compile time if the providers array is malformed or a circular dependency exists. Run:

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=estimate
    ```

    Expected: all estimate specs green (controller, reviser, retriever, deleter, repository, line-item repository, noop canceller, conflict error). If any spec fails with "Nest can't resolve dependencies of X", the providers array is missing a transitive dependency — debug and fix.

    **Prohibitions:**
    - No `any`, no `as`, no suppression comments.
    - Path aliases only.
    - No deleting or reordering existing providers — only add to the end of the array.
  </action>
  <acceptance_criteria>
    - `grep -c "import { EstimateReviser }" trade-flow-api/src/estimate/estimate.module.ts` returns 1
    - `grep -c "import { NoopEstimateFollowupCanceller }" trade-flow-api/src/estimate/estimate.module.ts` returns 1
    - `grep -c "import { ESTIMATE_FOLLOWUP_CANCELLER }" trade-flow-api/src/estimate/estimate.module.ts` returns 1
    - `grep -c "EstimateReviser," trade-flow-api/src/estimate/estimate.module.ts` returns at least 1 (in providers)
    - `grep -c "NoopEstimateFollowupCanceller," trade-flow-api/src/estimate/estimate.module.ts` returns at least 1 (in providers)
    - `grep -c "provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller" trade-flow-api/src/estimate/estimate.module.ts` returns 1
    - `grep -c "ESTIMATE_FOLLOWUP_CANCELLER," trade-flow-api/src/estimate/estimate.module.ts` returns at least 2 (once in providers, once in exports)
    - `grep -c " any\\| as " trade-flow-api/src/estimate/estimate.module.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/estimate.module.ts` returns 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate` exits 0 (all estimate specs resolve their DI dependencies)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=estimate</automated>
  </verify>
  <done>EstimateModule registers reviser, noop canceller, and the token provider; exports ESTIMATE_FOLLOWUP_CANCELLER; all estimate specs resolve their DI graph cleanly.</done>
</task>

<task type="auto">
  <name>Task 3: Document POST + GET /v1/estimates/:id/revisions in openapi.yaml</name>
  <files>trade-flow-api/openapi.yaml</files>
  <read_first>
    - trade-flow-api/openapi.yaml (at minimum, grep for `/v1/quotes/{id}` to see the existing quote endpoint shape, and grep for existing `EstimateResponse` / `IEstimateResponse` schemas; also grep for `ErrorResponse` / existing `409` entries if any — 409 is new in this codebase)
    - trade-flow-api/src/estimate/responses/estimate.response.ts (Phase 41 — confirm response shape for schema mapping)
    - .planning/phases/42-revisions/42-CONTEXT.md §Specifics "409 Conflict response shape" (the exact JSON body shape)
  </read_first>
  <action>
    Amend `trade-flow-api/openapi.yaml` to document both new endpoints.

    **Step 1 — Add a `components.schemas.ErrorResponse409` entry** (if an equivalent doesn't already exist) describing the 409 Conflict body shape:

    ```yaml
    components:
      schemas:
        ErrorResponse409:
          type: object
          properties:
            data:
              type: array
              items: {}
              example: []
            errors:
              type: array
              items:
                type: object
                required: [code, message]
                properties:
                  code:
                    type: string
                    example: "ESTIMATE_REVISION_CONFLICT"
                  message:
                    type: string
                    example: "Estimate has already been revised or is no longer revisable"
                  details:
                    type: string
                    nullable: true
    ```

    If `ErrorResponse` already exists as a shared schema, just reference it — do not duplicate. Inspect `openapi.yaml` first.

    **Step 2 — Add the POST endpoint** under the `paths` section. Find the existing `/v1/estimates` or `/v1/estimates/{id}` entries and add adjacent:

    ```yaml
    /v1/estimates/{id}/revisions:
      post:
        tags: [Estimates]
        summary: Create a new revision of an existing estimate
        description: |
          Clones a Sent/Viewed/Responded/SiteVisitRequested estimate into a new Draft revision under the
          same E-YYYY-NNN number. The previous revision keeps its status and has `isCurrent` flipped to
          false. The new revision is created as Draft with `isCurrent: true`. Line items are deep-copied
          with status forced to PENDING. The endpoint takes NO body beyond authentication.

          Follow-ups on the previous revision are NOT cancelled at POST time — they are cancelled when
          the new revision is actually Sent (Phase 44).
        operationId: reviseEstimate
        security:
          - bearerAuth: []
        parameters:
          - name: id
            in: path
            required: true
            schema:
              type: string
            description: Estimate id (accepts either root id or current revision id)
        responses:
          "201":
            description: Revision created — returns the new Draft revision DTO
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/EstimateResponse"
          "403":
            description: Forbidden — the authenticated user does not own the estimate's business
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse"
          "404":
            description: Not found — the estimate does not exist
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse"
          "409":
            description: |
              Conflict — the estimate has already been revised by another request, or its current status
              is not one of the revisable statuses (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED).
              The error code is `ESTIMATE_REVISION_CONFLICT` and the message is intentionally generic
              so that the three underlying causes (concurrent revise, status changed, already
              non-current) are not distinguished.
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse409"
          "422":
            description: |
              Unprocessable entity — the estimate is in a non-revisable state (e.g., Draft, Converted,
              Declined, Expired, Lost). This is returned only when the state check can be made upfront
              before the atomic filter runs; in practice, Phase 42 collapses most 422-shaped failures
              into 409 via the atomic filter miss. The error code is `ESTIMATE_NOT_REVISABLE`.
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse"
      get:
        tags: [Estimates]
        summary: Get the full revision chain for an estimate
        description: |
          Returns the full revision chain for an estimate, ordered by revisionNumber ascending (oldest
          first). Accepts either the root id or any revision id — both resolve to the same chain via
          the target row's `rootEstimateId`. Soft-deleted revisions are excluded.

          Each entry in the returned array is a full IEstimateResponse with totals and priceRange
          attached, so the trader's History panel can render expandable line-item details without
          additional round-trips.
        operationId: getEstimateRevisions
        security:
          - bearerAuth: []
        parameters:
          - name: id
            in: path
            required: true
            schema:
              type: string
            description: Estimate id (root or any revision — both resolve to the same chain)
        responses:
          "200":
            description: Chain returned — array ordered by revisionNumber ascending
            content:
              application/json:
                schema:
                  type: object
                  properties:
                    data:
                      type: array
                      items:
                        $ref: "#/components/schemas/EstimateResponse"
          "403":
            description: Forbidden
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse"
          "404":
            description: Not found
            content:
              application/json:
                schema:
                  $ref: "#/components/schemas/ErrorResponse"
    ```

    **Adapt the schema references** (`EstimateResponse`, `ErrorResponse`) to match the actual schema names in `openapi.yaml`. If Phase 41 named them differently, use the actual names.

    **Step 3 — Validate the yaml.** Run:

    ```bash
    cd trade-flow-api && npx --yes @redocly/cli lint openapi.yaml 2>/dev/null || npx --yes swagger-cli validate openapi.yaml 2>/dev/null || cat openapi.yaml | head -5
    ```

    Preferred: any existing npm script for OpenAPI validation. If none exists, at minimum confirm the yaml parses cleanly — `node -e "const yaml = require('js-yaml'); yaml.load(require('fs').readFileSync('openapi.yaml','utf8'))"` should exit 0.

    **Prohibitions:**
    - Do not add new top-level sections or reorganize existing ones.
    - Do not delete existing endpoints.
    - Do not use `oneOf` or `anyOf` — keep the schemas flat and match existing Phase 41 conventions.
  </action>
  <acceptance_criteria>
    - `grep -c "/v1/estimates/{id}/revisions" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "reviseEstimate\\|operationId: reviseEstimate" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "getEstimateRevisions\\|operationId: getEstimateRevisions" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "ESTIMATE_REVISION_CONFLICT" trade-flow-api/openapi.yaml` returns at least 1
    - `grep -c "\"409\"\\|'409'" trade-flow-api/openapi.yaml` returns at least 1 (the 409 response documented)
    - `node -e "const yaml = require('js-yaml'); yaml.load(require('fs').readFileSync('trade-flow-api/openapi.yaml','utf8')); console.log('ok')"` prints "ok" (yaml parses)
  </acceptance_criteria>
  <verify>
    <automated>node -e "const yaml = require('js-yaml'); yaml.load(require('fs').readFileSync('trade-flow-api/openapi.yaml','utf8')); console.log('ok')" | grep -q ok</automated>
  </verify>
  <done>openapi.yaml documents both endpoints with correct status codes (201/200/403/404/409/422 as applicable) and the generic 409 message; yaml parses cleanly.</done>
</task>

<task type="auto">
  <name>Task 4: Author the manual smoke procedure for SC #4 concurrent revise</name>
  <files>trade-flow-api/docs/smoke/phase-42-concurrent-revise.md</files>
  <read_first>
    - trade-flow-api/docker-compose.yaml (confirm the Mongo service name and port — typically `mongo:27017`)
    - trade-flow-api/.env.example (confirm API port — typically 3000)
    - .planning/phases/42-revisions/42-VALIDATION.md §Manual-Only Verifications (defines what the smoke doc must cover)
    - .planning/phases/42-revisions/42-CONTEXT.md §User Decisions Q1 (the user-approved substitution of smoke test for mongodb-memory-server integration test)
  </read_first>
  <action>
    Create `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md`:

    ```markdown
    # Phase 42 Concurrent Revise — Manual Smoke Procedure

    **Purpose:** Verify Phase 42 Success Criterion #4 end-to-end against a real MongoDB instance. The partial unique index on `{rootEstimateId: 1, isCurrent: 1}` with filter `{isCurrent: true}` cannot be exercised by unit tests (mocked `MongoDbWriter` does not enforce indexes). This procedure is REQUIRED before `/gsd-verify-work` sign-off on Phase 42.

    **Why manual:** The existing test infrastructure is pure unit tests with mocked primitives. Per user decision Q1 (2026-04-11), Phase 42 did NOT add `mongodb-memory-server` as a dev dependency. This smoke test substitutes.

    ## Prerequisites

    1. Docker + Docker Compose installed.
    2. `trade-flow-api` cloned and dependencies installed (`npm install`).
    3. A Firebase test user with a valid Bearer token (easiest: sign in to `trade-flow-ui` locally, copy the token from DevTools → Application → IndexedDB → firebaseLocalStorageDb → `firebaseLocalStorageDb` → the `access_token` field of the signed-in user).
    4. An existing estimate in the local test database in status `SENT` with `isCurrent: true`. Create via the API if needed (POST to `/v1/estimates`, then transition via the PATCH/status endpoint or by manually running the transition service).

    ## Procedure

    ### Step 1: Start the local stack

    ```bash
    cd trade-flow-api
    docker compose up -d mongo
    npm run start:dev
    ```

    Wait for the API to log `Nest application successfully started` on port 3000.

    ### Step 2: Capture baseline state

    Open a second terminal and run:

    ```bash
    docker compose exec mongo mongosh trade_flow --quiet --eval '
      const chain = db.estimates.find({ number: "E-2026-001" }).sort({ revisionNumber: 1 }).toArray();
      chain.forEach((e, i) => print(JSON.stringify({
        rev: i + 1, _id: e._id.toString(), status: e.status, isCurrent: e.isCurrent,
        revisionNumber: e.revisionNumber, parentEstimateId: e.parentEstimateId?.toString() ?? null,
        rootEstimateId: e.rootEstimateId?.toString() ?? null,
      })));
      print("COUNT_CURRENT: " + db.estimates.countDocuments({ rootEstimateId: chain[0].rootEstimateId ?? chain[0]._id, isCurrent: true }));
    '
    ```

    Expected: one row with `isCurrent: true`, `revisionNumber: 1`, `parentEstimateId: null`, `rootEstimateId === _id` (per Phase 42 D-CHAIN-03). `COUNT_CURRENT: 1`.

    Replace `E-2026-001` with your actual test estimate number.

    ### Step 3: Fire two concurrent POST requests

    In the second terminal, replace `${TOKEN}` with your Firebase Bearer token and `${ESTIMATE_ID}` with the source estimate's `_id` string:

    ```bash
    TOKEN="your-firebase-bearer-token-here"
    ESTIMATE_ID="your-estimate-id-here"
    BASE_URL="http://localhost:3000"

    # Fire two concurrent POST calls with deterministic assertion
    (
      curl -sS -o /tmp/resp-A.json -w "A_STATUS:%{http_code}\n" \
        -X POST "${BASE_URL}/v1/estimates/${ESTIMATE_ID}/revisions" \
        -H "Authorization: Bearer ${TOKEN}" &
      curl -sS -o /tmp/resp-B.json -w "B_STATUS:%{http_code}\n" \
        -X POST "${BASE_URL}/v1/estimates/${ESTIMATE_ID}/revisions" \
        -H "Authorization: Bearer ${TOKEN}" &
      wait
    )
    cat /tmp/resp-A.json; echo
    cat /tmp/resp-B.json; echo
    ```

    **Expected output exactly:**

    - One of `A_STATUS:201` / `B_STATUS:201` (the winner).
    - One of `A_STATUS:409` / `B_STATUS:409` (the loser).
    - The 201 response body contains a new estimate DTO with `revisionNumber: 2`, `isCurrent: true`, `status: "draft"`, `parentEstimateId` matching `ESTIMATE_ID`.
    - The 409 response body matches:
      ```json
      {
        "errors": [
          {
            "code": "ESTIMATE_REVISION_CONFLICT",
            "message": "Estimate has already been revised or is no longer revisable",
            "details": null
          }
        ]
      }
      ```

    If BOTH return 201 or BOTH return 409, the test FAILS — investigate immediately. This would indicate the atomic filter is broken or the partial unique index is not in place.

    ### Step 4: Verify post-state

    Back in mongosh (from Step 2), rerun the baseline query:

    ```bash
    docker compose exec mongo mongosh trade_flow --quiet --eval '
      const chain = db.estimates.find({ number: "E-2026-001" }).sort({ revisionNumber: 1 }).toArray();
      chain.forEach((e, i) => print(JSON.stringify({
        rev: i + 1, _id: e._id.toString(), status: e.status, isCurrent: e.isCurrent,
        revisionNumber: e.revisionNumber, parentEstimateId: e.parentEstimateId?.toString() ?? null,
      })));
      print("COUNT_CURRENT: " + db.estimates.countDocuments({ rootEstimateId: chain[0].rootEstimateId ?? chain[0]._id, isCurrent: true }));
    '
    ```

    **Expected:**
    - **Exactly 2 rows** in the chain (one pre-existing root + one new Draft revision).
    - Row 1: `revisionNumber: 1`, `isCurrent: false`, `status: "sent"` (or whatever the pre-existing status was).
    - Row 2: `revisionNumber: 2`, `isCurrent: true`, `status: "draft"`, `parentEstimateId` referencing row 1's `_id`.
    - `COUNT_CURRENT: 1` — exactly one row with `isCurrent: true` per chain.

    If `COUNT_CURRENT` is 0 or 2, the partial unique index is not enforcing. Investigate:
    - Run `db.estimates.getIndexes()` in mongosh and confirm the index `rootEstimateId_1_isCurrent_1` exists with `partialFilterExpression: { isCurrent: true }` and `unique: true`.
    - If the index is missing, check `EstimateRepository.onModuleInit()` — did it run? Check app logs for "Estimate indexes ensured" or equivalent.

    ### Step 5: Verify the index is present

    ```bash
    docker compose exec mongo mongosh trade_flow --quiet --eval '
      db.estimates.getIndexes().filter(i => i.name.includes("rootEstimateId")).forEach(i => print(JSON.stringify(i)));
    '
    ```

    **Expected:** at least two index entries:
    - `{rootEstimateId: 1, isCurrent: 1}` with `unique: true`, `partialFilterExpression: {isCurrent: true}`.
    - `{rootEstimateId: 1, revisionNumber: 1}` without `unique`.

    ### Step 6: Tear down

    ```bash
    docker compose down
    ```

    ## Pass/Fail Criteria

    - **PASS:** Step 3 shows one 201 and one 409. Step 4 shows `COUNT_CURRENT: 1`. Step 5 shows both indexes.
    - **FAIL:** Any of the above conditions is not met. Do NOT proceed to `/gsd-verify-work` — debug and re-run.

    ## Known Limitations

    - This procedure is manual and not in CI. It must be run by the developer before each deploy of Phase 42 changes.
    - The procedure does not cover the step-3 rollback (line-item clone failure) — that's covered by unit tests.
    - The procedure assumes a single-instance Railway deployment. Multi-replica deployments would need a different rollback strategy (see research §8 "Index drop-and-recreate race in multi-replica bootstrap").
    ```

    **Prohibitions:**
    - No `any`, no code that could compile/break the build.
    - No sensitive data committed (the `TOKEN` placeholder is obviously a placeholder — the reader substitutes at runtime).
    - The document is pure markdown; no yaml or executable code.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/docs/smoke/phase-42-concurrent-revise.md`
    - `grep -c "docker compose" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 2
    - `grep -c "curl" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 2
    - `grep -c "ESTIMATE_REVISION_CONFLICT" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 1
    - `grep -c "COUNT_CURRENT" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 2
    - `grep -c "rootEstimateId_1_isCurrent_1\\|rootEstimateId: 1, isCurrent: 1" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 1
    - `grep -c "201\\|409" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 2
    - `grep -c "PASS\\|FAIL" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` returns at least 2
  </acceptance_criteria>
  <verify>
    <automated>test -f trade-flow-api/docs/smoke/phase-42-concurrent-revise.md &amp;&amp; grep -q "COUNT_CURRENT" trade-flow-api/docs/smoke/phase-42-concurrent-revise.md</automated>
  </verify>
  <done>Manual smoke procedure exists with step-by-step docker-compose + parallel curl + post-state assertion + index verification + pass/fail criteria.</done>
</task>

<task type="auto">
  <name>Task 5: Run full CI gate and confirm Phase 42 is ready for verification</name>
  <files></files>
  <read_first>
    - All files modified in Phase 42 plans 01-06 (no new reads — this is the final gate task)
  </read_first>
  <action>
    Run the full CI gate for `trade-flow-api` and confirm every check passes:

    ```bash
    cd trade-flow-api && npm run ci
    ```

    If `npm run ci` fails on any check (tests, lint, format, typecheck), debug the specific failure and fix. Do NOT suppress errors with `eslint-disable`, `@ts-ignore`, etc. — fix the root cause.

    Then run a targeted Phase 42 test slice to confirm no regressions:

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern="estimate|conflict|handle-error|noop-estimate-followup-canceller"
    ```

    Expected: all specs green.

    Finally, confirm the Phase 42 frontmatter on `42-VALIDATION.md` can be flipped to `nyquist_compliant: true` by checking all the plan items marked pending have moved to complete. Do NOT edit VALIDATION.md in this task — that's the `/gsd-verify-work` agent's job. But the plan executor should note any items still pending in the SUMMARY.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - `cd trade-flow-api && npm run test -- --testPathPattern="estimate|conflict|handle-error|noop-estimate-followup-canceller"` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run ci</automated>
  </verify>
  <done>Full CI gate green; Phase 42 is ready for `/gsd-verify-work`.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| HTTP request body → controller | D-REV-01 says the endpoint takes no body; ValidationPipe strips anything sent |
| JWT token → controller | JwtAuthGuard validates before the handler runs |
| OpenAPI doc → API contract drift | Wrong status codes in openapi.yaml mislead client generators |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-06-01 | Mass assignment | Attacker POSTs a body with `isCurrent`, `revisionNumber`, `parentEstimateId` fields trying to set them directly | mitigate | HIGH | The POST handler does NOT declare `@Body()`. The global `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true` (confirm in `main.ts`) strips any body. The reviser constructs the new row from the source snapshot, not from the request. Grep asserts `@Body` does not appear in the revise method. |
| T-42-06-02 | IDOR | Attacker passes another business's estimate id in `:id` | mitigate | HIGH | JwtAuthGuard attaches `request.user`. `EstimateReviser` applies `EstimatePolicy.canUpdate` at step 0. For GET, `EstimateRetriever.findRevisionsByIdOrFail` applies `EstimatePolicy.canRead`. Both policies scope by `businessId`. Tested at plan 42-04 and plan 42-05. |
| T-42-06-03 | Information disclosure | OpenAPI 409 schema leaks internal field names or PII | accept | LOW | The 409 schema example contains only the generic message and the error code — no PII, no internal structure beyond what's already exposed in 422/404 responses. |
| T-42-06-04 | Tampering | Missing `@UseGuards(JwtAuthGuard)` on the new handlers allows anonymous access | mitigate | HIGH | Grep acceptance criteria in Task 1 asserts `@UseGuards(JwtAuthGuard)` appears in front of both new handler definitions. Controller spec additionally uses `Test.createTestingModule` with the guard applied. |
| T-42-06-05 | DoS | Attacker spams POST `/revisions` to exhaust DB writes | accept | LOW | Rate limiting out of Phase 42 scope. The atomic filter + partial unique index protects data integrity; throughput is a separate concern. Flag as follow-up hardening. |
| T-42-06-06 | Data integrity | DI registration drift — `ESTIMATE_FOLLOWUP_CANCELLER` resolves to `undefined` at runtime because the module forgot the `useClass` binding | mitigate | MEDIUM | Task 2 acceptance criteria grep asserts `provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller` appears literally. DI resolution is exercised by the reviser spec at plan 42-04 (which includes a "DI injection resolution" test case). |
| T-42-06-07 | Smoke procedure drift | Manual smoke doc references wrong index name / wrong filter shape | mitigate | MEDIUM | Task 4 acceptance criteria grep for the exact index name `rootEstimateId_1_isCurrent_1` and the exact `COUNT_CURRENT` assertion strings |

**Verification commands:**
- `grep -c "@UseGuards(JwtAuthGuard)" trade-flow-api/src/estimate/controllers/estimate.controller.ts` — multiple matches (all authenticated endpoints).
- `cd trade-flow-api && npm run ci` — full green gate.
- Manual smoke run against a real docker-compose Mongo: one 201, one 409, COUNT_CURRENT: 1.
</threat_model>

<verification>
After all five tasks complete:

```bash
cd trade-flow-api && npm run ci
cd trade-flow-api && npm run test -- --testPathPattern=estimate
test -f trade-flow-api/docs/smoke/phase-42-concurrent-revise.md
node -e "require('js-yaml').load(require('fs').readFileSync('trade-flow-api/openapi.yaml','utf8'))" && echo "yaml-ok"
```

All four must exit 0 / print expected output.

**Before calling Phase 42 "done":** run the manual smoke procedure at `trade-flow-api/docs/smoke/phase-42-concurrent-revise.md` against a local docker-compose Mongo. This is the only verification for SC #4 per user decision Q1.
</verification>

<success_criteria>
- Both new HTTP endpoints exist on `EstimateController` with `@UseGuards(JwtAuthGuard)`, no `@Body` on POST, `createHttpError` wrapping both.
- `EstimateModule` registers reviser + noop canceller + token provider, exports the token.
- `openapi.yaml` documents both endpoints with correct status codes (including 409).
- Manual smoke doc exists with docker-compose + parallel curl recipe + post-state assertion + pass/fail criteria.
- `npm run ci` exits 0 — full Phase 42 code suite green.
- Single commit: `feat(42): add revision endpoints, module wiring, openapi, and smoke procedure (Phase 42 wave 4)`.
- Manual smoke procedure executed successfully once locally (not in CI — logged in SUMMARY).
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-06-SUMMARY.md` documenting:
- Controller endpoint line numbers + handler signatures.
- Module provider registration — the exact `{provide: ..., useClass: ...}` entry.
- OpenAPI section headers added (list the path keys).
- Manual smoke procedure outcome (pass/fail from local run).
- Full `npm run ci` output summary.
- Commit hash.

After plan 42-06 ships, Phase 42 is ready for `/gsd-verify-work` and subsequently `/gsd-document-phase`.
</output>
