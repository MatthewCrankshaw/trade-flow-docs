---
phase: 41-estimate-module-crud-backend
plan: 03
type: execute
wave: 2
depends_on: [01]
files_modified:
  - trade-flow-api/src/document-token/document-token.module.ts
  - trade-flow-api/src/document-token/entities/document-token.entity.ts
  - trade-flow-api/src/document-token/data-transfer-objects/document-token.dto.ts
  - trade-flow-api/src/document-token/repositories/document-token.repository.ts
  - trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
  - trade-flow-api/src/document-token/services/document-token-creator.service.ts
  - trade-flow-api/src/document-token/services/document-token-retriever.service.ts
  - trade-flow-api/src/document-token/services/document-token-revoker.service.ts
  - trade-flow-api/src/document-token/services/public-quote-retriever.service.ts
  - trade-flow-api/src/document-token/services/quote-response-handler.service.ts
  - trade-flow-api/src/document-token/controllers/public-quote.controller.ts
  - trade-flow-api/src/document-token/requests/decline-quote.request.ts
  - trade-flow-api/src/document-token/responses/public-quote.response.ts
  - trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts
  - trade-flow-api/src/document-token/test/guards/document-session-auth.guard.spec.ts
  - trade-flow-api/src/document-token/test/controllers/public-quote.controller.spec.ts
  - trade-flow-api/src/document-token/test/services/public-quote-retriever.service.spec.ts
  - trade-flow-api/src/document-token/test/services/document-token-creator.service.spec.ts
  - trade-flow-api/src/document-token/test/services/document-token-retriever.service.spec.ts
  - trade-flow-api/src/document-token/test/services/document-token-revoker.service.spec.ts
  - trade-flow-api/src/document-token/test/services/quote-response-handler.service.spec.ts
  - trade-flow-api/src/document-token/test/mocks/document-token-mock-generator.ts
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/quote/controllers/quote.controller.ts
  - trade-flow-api/src/quote/services/quote-transition.service.ts
autonomous: true
requirements: [EST-09]
must_haves:
  truths:
    - The `src/quote-token/` directory no longer exists anywhere in trade-flow-api
    - A new `src/document-token/` directory exists with the full rename applied per D-TKN-01..07
    - `DocumentTokenRepository.COLLECTION` equals `"document_tokens"`
    - `IDocumentTokenEntity` has `documentId: ObjectId` and `documentType: "quote" | "estimate"` fields (not `quoteId`)
    - `DocumentSessionAuthGuard` attaches `request.documentToken: IDocumentTokenDto` (not `request.quoteToken`)
    - `PublicQuoteController` asserts `request.documentToken.documentType === "quote"` at the top of every handler and responds 404 on mismatch (token-type-confusion mitigation per Pitfall 3)
    - `GET /v1/public/quote/:token` still resolves correctly — the public quote page regression test is still green
    - `AppModule` imports `DocumentTokenModule` and no longer imports `QuoteTokenModule`
    - `QuoteController` and `QuoteTransitionService` consume document-token services via the new import path
    - `cd trade-flow-api && npm run ci` exits 0
  artifacts:
    - path: trade-flow-api/src/document-token/entities/document-token.entity.ts
      provides: IDocumentTokenEntity with documentType discriminator
      contains: "documentType"
    - path: trade-flow-api/src/document-token/repositories/document-token.repository.ts
      provides: DocumentTokenRepository with `document_tokens` collection
      contains: "document_tokens"
    - path: trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
      provides: DocumentSessionAuthGuard replacing QuoteSessionAuthGuard
      exports: ["DocumentSessionAuthGuard"]
    - path: trade-flow-api/src/document-token/controllers/public-quote.controller.ts
      provides: Public quote controller with documentType === "quote" assertion
      contains: "documentType"
  key_links:
    - from: trade-flow-api/src/document-token/controllers/public-quote.controller.ts
      to: trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
      via: "@UseGuards(DocumentSessionAuthGuard)"
      pattern: "UseGuards\\(DocumentSessionAuthGuard\\)"
    - from: trade-flow-api/src/app.module.ts
      to: trade-flow-api/src/document-token/document-token.module.ts
      via: imports array entry
      pattern: "DocumentTokenModule"
---

<objective>
Pure end-to-end code rename of `src/quote-token/` to `src/document-token/` per D-TKN-01..07 in CONTEXT.md and §User Constraints in RESEARCH.md. Delete the old directory, create the new module with every file renamed (class names, types, collection constant, guard name, controller path, request field) and the `documentId` / `documentType` discriminator added to the entity/DTO. Preserve the existing `/v1/public/quote/:token` URL unchanged. Refactor `PublicQuoteController` to read `req.documentToken` and assert `documentType === "quote"` on every handler (token-type-confusion mitigation per RESEARCH.md Pitfall 3). Update `AppModule`, `QuoteController`, and `QuoteTransitionService` to import from the new path. This plan runs in parallel with Plan 02 — the file sets are disjoint.

Purpose: Phase 45 and future public-estimate endpoints need a unified token model with a discriminator. D-TKN-01 mandates a pure code rename (no data migration) because production has no quote-token data (verified in Plan 01 Task 1). The refactor MUST preserve every existing behavior of the quote token flow — including the specific endpoint URL `/v1/public/quote/:token` — while renaming everything in the module. Plan 01 has already added `@document-token/*` path aliases and the new `DOCUMENT_TOKEN_*` error codes.

Output: New `src/document-token/` module, deleted `src/quote-token/` directory, updated imports across `AppModule` / `QuoteModule` consumers, and a green full CI gate with the existing public-quote controller spec still passing.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files the executor MUST read before editing. -->
<!-- These are the files being renamed/relocated — executors must see the current shape. -->

Source tree (to be renamed): trade-flow-api/src/quote-token/
Target tree (new):            trade-flow-api/src/document-token/

Required field-level renames per D-TKN-04:
- `QuoteTokenEntity.quoteId: ObjectId` → `DocumentTokenEntity.documentId: ObjectId`
- NEW field on entity + DTO: `documentType: "quote" | "estimate"`
- Preserved fields: `token: string`, `expiresAt: Date`, `revokedAt?: Date`, `sentAt: Date`, `recipientEmail: string`, `firstViewedAt?: Date`

Required class renames per D-TKN-02:
- `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard`
- `QuoteTokenRepository` → `DocumentTokenRepository` (collection constant `"quote_tokens"` → `"document_tokens"`)
- `QuoteTokenCreator` / `QuoteTokenRetriever` / `QuoteTokenRevoker` → `DocumentTokenCreator` / `DocumentTokenRetriever` / `DocumentTokenRevoker`
- `IQuoteTokenEntity` / `IQuoteTokenDto` → `IDocumentTokenEntity` / `IDocumentTokenDto`
- `QuoteTokenModule` → `DocumentTokenModule`

Preserved names (because they are quote-specific by purpose, not token-specific):
- `PublicQuoteController` — stays named PublicQuoteController (it handles /v1/public/quote/:token)
- `PublicQuoteRetriever` — stays named
- `QuoteResponseHandler` — stays named
- `IPublicQuoteResponse` — stays named
- `DeclineQuoteRequest` — stays named
- Controller route prefix `@Controller("v1/public")` + `@Get("quote/:token")` MUST NOT CHANGE — URL `/v1/public/quote/:token` is customer-facing.

Request-field rename per D-TKN-02:
- `req.quoteToken` → `req.documentToken`

Token-type-confusion mitigation per D-TKN-03 and Pitfall 3:
- Every handler in PublicQuoteController MUST start with:
  ```typescript
  if (request.documentToken.documentType !== "quote") {
    throw createHttpError(new ResourceNotFoundError(ErrorCodes.DOCUMENT_TOKEN_TYPE_MISMATCH, "Quote not found"));
  }
  ```

Create flow: when `DocumentTokenCreator` creates a token for a quote, it MUST set `documentType: "quote"` on the entity. (Estimate token creation comes in Phase 44.)
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create src/document-token/ with renamed entities, DTOs, repository, guard, and module</name>
  <files>
    trade-flow-api/src/document-token/document-token.module.ts,
    trade-flow-api/src/document-token/entities/document-token.entity.ts,
    trade-flow-api/src/document-token/data-transfer-objects/document-token.dto.ts,
    trade-flow-api/src/document-token/repositories/document-token.repository.ts,
    trade-flow-api/src/document-token/guards/document-session-auth.guard.ts,
    trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts,
    trade-flow-api/src/document-token/test/guards/document-session-auth.guard.spec.ts,
    trade-flow-api/src/document-token/test/mocks/document-token-mock-generator.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote-token/quote-token.module.ts (source module wiring to replicate)
    - trade-flow-api/src/quote-token/entities/quote-token.entity.ts (source entity with quoteId field)
    - trade-flow-api/src/quote-token/data-transfer-objects/quote-token.dto.ts (source DTO)
    - trade-flow-api/src/quote-token/repositories/quote-token.repository.ts (source repository with COLLECTION = "quote_tokens")
    - trade-flow-api/src/quote-token/guards/quote-session-auth.guard.ts (source guard)
    - trade-flow-api/src/quote-token/test/repositories/quote-token.repository.spec.ts (source spec if present)
    - trade-flow-api/src/quote-token/test/guards/quote-session-auth.guard.spec.ts (source spec if present)
    - trade-flow-api/src/quote-token/test/mocks/quote-token-mock-generator.ts (source mock generator if present)
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (§specifics — locked IDocumentTokenEntity shape around lines 249-266)
  </read_first>
  <behavior>
    - `IDocumentTokenEntity` exports a TypeScript interface with exactly these fields: `token: string; documentType: "quote" | "estimate"; documentId: ObjectId; expiresAt: Date; revokedAt?: Date; sentAt: Date; recipientEmail: string; firstViewedAt?: Date;` (plus the fields inherited from `IBaseEntity`).
    - `IDocumentTokenDto` mirrors the entity with `documentId: string` instead of `ObjectId` and `DateTime` for the date fields per the project DTO/entity convention.
    - `DocumentTokenRepository.COLLECTION` is `"document_tokens"`.
    - `DocumentSessionAuthGuard.canActivate` attaches `request.documentToken: IDocumentTokenDto` (NOT `request.quoteToken`) and rejects missing token (404 via `DOCUMENT_TOKEN_NOT_FOUND`), expired token (410 via `DOCUMENT_TOKEN_EXPIRED`), revoked token (410 via `DOCUMENT_TOKEN_REVOKED`).
    - The repository never writes `firstViewedAt: null` on create — it omits the field entirely per Pitfall 8.
    - The relocated specs still pass and assert the new class/type/collection names.
    - Zero narrative comments; zero `eslint-disable`; zero `as` casts.
  </behavior>
  <action>
**Step 1: Create the new directory structure:**

```
mkdir -p trade-flow-api/src/document-token/{controllers,data-transfer-objects,entities,guards,repositories,requests,responses,services}
mkdir -p trade-flow-api/src/document-token/test/{controllers,guards,mocks,repositories,services}
```

**Step 2: Move and rename files** using `git mv` so history is preserved. Do NOT delete originals manually — `git mv` handles it. For each file below, move to the new location AND apply the in-file rename:

| FROM | TO |
|------|-----|
| `src/quote-token/quote-token.module.ts` | `src/document-token/document-token.module.ts` |
| `src/quote-token/entities/quote-token.entity.ts` | `src/document-token/entities/document-token.entity.ts` |
| `src/quote-token/data-transfer-objects/quote-token.dto.ts` | `src/document-token/data-transfer-objects/document-token.dto.ts` |
| `src/quote-token/repositories/quote-token.repository.ts` | `src/document-token/repositories/document-token.repository.ts` |
| `src/quote-token/guards/quote-session-auth.guard.ts` | `src/document-token/guards/document-session-auth.guard.ts` |
| `src/quote-token/test/repositories/quote-token.repository.spec.ts` | `src/document-token/test/repositories/document-token.repository.spec.ts` |
| `src/quote-token/test/guards/quote-session-auth.guard.spec.ts` | `src/document-token/test/guards/document-session-auth.guard.spec.ts` |
| `src/quote-token/test/mocks/quote-token-mock-generator.ts` | `src/document-token/test/mocks/document-token-mock-generator.ts` |

If any source file does not exist (e.g., a spec was never written), skip the move and create a new file at the destination using the same shape as the existing patterns in `src/quote/test/`.

**Step 3: Apply in-file renames** (for every moved file). These are exact string-for-string substitutions across each file:

Class / type / interface renames:
- `QuoteTokenModule` → `DocumentTokenModule`
- `QuoteTokenEntity` → `DocumentTokenEntity`
- `IQuoteTokenEntity` → `IDocumentTokenEntity`
- `QuoteTokenDto` → `DocumentTokenDto`
- `IQuoteTokenDto` → `IDocumentTokenDto`
- `QuoteTokenRepository` → `DocumentTokenRepository`
- `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard`
- `QuoteTokenMockGenerator` → `DocumentTokenMockGenerator`

Field renames (only on the entity/DTO/repository/guard — see Step 4 for controllers/services):
- `quoteId: ObjectId` → `documentId: ObjectId` (entity)
- `quoteId: string` → `documentId: string` (DTO)
- Add NEW field: `documentType: "quote" | "estimate"` (both entity and DTO; place after `token` per the locked shape in CONTEXT.md §specifics)
- `request.quoteToken` → `request.documentToken` (guard)

Path alias renames:
- Every `@quote-token/*` import → `@document-token/*`
- Every `"src/quote-token/*"` tsconfig reference → `"src/document-token/*"` (but that was handled in Plan 01)

Collection constant rename in `DocumentTokenRepository`:
- `static readonly COLLECTION = "quote_tokens"` → `static readonly COLLECTION = "document_tokens"`

**Step 4: Update the guard's error handling** to use new ErrorCodes from Plan 01:
- 404 path → `ErrorCodes.DOCUMENT_TOKEN_NOT_FOUND` (was something like `QUOTE_TOKEN_NOT_FOUND`)
- 410 Expired path → `ErrorCodes.DOCUMENT_TOKEN_EXPIRED`
- 410 Revoked path → `ErrorCodes.DOCUMENT_TOKEN_REVOKED`

**Step 5: Update the repository's `toEntity` mapping** to:
- Write `documentType` from the incoming DTO (required field — rejects create without it).
- NEVER write `firstViewedAt: null` or `firstViewedAt: undefined` explicitly on create — OMIT the field when the DTO doesn't provide it (Pitfall 8).
- Write `documentId: new ObjectId(dto.documentId)` (was `quoteId`).

Update the `toDto` mapping to the mirror: `documentId: entity.documentId.toHexString()` and `documentType: entity.documentType`.

**Step 6: Update the spec files** to:
- Import from `@document-token/*`
- Assert the new class names, field names, and collection constant `"document_tokens"`
- For `DocumentTokenMockGenerator`, default `documentType: "quote"` in the factory method so existing callers get the same semantics they had before the rename.
- Add a new test case in the repository spec: `expect(DocumentTokenRepository.COLLECTION).toBe("document_tokens")`
- Add a new test case in the repository spec: `create({ ...dtoWithoutFirstViewedAt }).then(entity => expect(entity.firstViewedAt).toBeUndefined())` per Pitfall 8

**Step 7: Module file** — `document-token.module.ts` must:
- Export `DocumentTokenModule` class with the same `@Module({...})` shape as the old `QuoteTokenModule`
- Providers include (will be filled in by Task 2 and Task 3): the three services, the guard, the repository, the controller
- Exports include: `DocumentTokenRepository`, `DocumentSessionAuthGuard`, the three service classes
- `imports` arrays stay the same as `QuoteTokenModule` had (probably `CoreModule`, `BusinessModule`, and `forwardRef(() => QuoteModule)`)

**Step 8: Run the slice** `cd trade-flow-api && npm run test -- --testPathPattern=document-token/test/(repositories|guards)` and confirm the two relocated specs pass.

Commit message: `refactor(41): rename quote-token → document-token entities, repository, guard, and module shell`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/document-token/document-token.module.ts` (file exists)
    - `test -f trade-flow-api/src/document-token/entities/document-token.entity.ts`
    - `test -f trade-flow-api/src/document-token/data-transfer-objects/document-token.dto.ts`
    - `test -f trade-flow-api/src/document-token/repositories/document-token.repository.ts`
    - `test -f trade-flow-api/src/document-token/guards/document-session-auth.guard.ts`
    - `grep -c "export class DocumentTokenRepository" trade-flow-api/src/document-token/repositories/document-token.repository.ts` returns 1
    - `grep -c "\"document_tokens\"" trade-flow-api/src/document-token/repositories/document-token.repository.ts` returns 1
    - `grep -c "\"quote_tokens\"" trade-flow-api/src/document-token/repositories/document-token.repository.ts` returns 0
    - `grep -c "documentType" trade-flow-api/src/document-token/entities/document-token.entity.ts` returns at least 1
    - `grep -c "documentId" trade-flow-api/src/document-token/entities/document-token.entity.ts` returns at least 1
    - `grep -c "quoteId" trade-flow-api/src/document-token/entities/document-token.entity.ts` returns 0
    - `grep -c "export class DocumentSessionAuthGuard" trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` returns 1
    - `grep -c "request.documentToken" trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` returns at least 1
    - `grep -c "request.quoteToken" trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` returns 0
    - `grep -c "DOCUMENT_TOKEN_NOT_FOUND\\|DOCUMENT_TOKEN_EXPIRED\\|DOCUMENT_TOKEN_REVOKED" trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` returns at least 3
    - `cd trade-flow-api && npm run test -- --testPathPattern=document-token/test/repositories` exits 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=document-token/test/guards` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|as unknown as" trade-flow-api/src/document-token` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=document-token/test/(repositories|guards)</automated>
  </verify>
  <done>Core document-token files exist with all renames applied and relocated specs pass</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Rename and relocate document-token services and the PublicQuoteController with type assertion</name>
  <files>
    trade-flow-api/src/document-token/services/document-token-creator.service.ts,
    trade-flow-api/src/document-token/services/document-token-retriever.service.ts,
    trade-flow-api/src/document-token/services/document-token-revoker.service.ts,
    trade-flow-api/src/document-token/services/public-quote-retriever.service.ts,
    trade-flow-api/src/document-token/services/quote-response-handler.service.ts,
    trade-flow-api/src/document-token/controllers/public-quote.controller.ts,
    trade-flow-api/src/document-token/requests/decline-quote.request.ts,
    trade-flow-api/src/document-token/responses/public-quote.response.ts,
    trade-flow-api/src/document-token/test/services/document-token-creator.service.spec.ts,
    trade-flow-api/src/document-token/test/services/document-token-retriever.service.spec.ts,
    trade-flow-api/src/document-token/test/services/document-token-revoker.service.spec.ts,
    trade-flow-api/src/document-token/test/services/public-quote-retriever.service.spec.ts,
    trade-flow-api/src/document-token/test/services/quote-response-handler.service.spec.ts,
    trade-flow-api/src/document-token/test/controllers/public-quote.controller.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote-token/services/quote-token-creator.service.ts (source)
    - trade-flow-api/src/quote-token/services/quote-token-retriever.service.ts (source)
    - trade-flow-api/src/quote-token/services/quote-token-revoker.service.ts (source)
    - trade-flow-api/src/quote-token/services/public-quote-retriever.service.ts (source)
    - trade-flow-api/src/quote-token/services/quote-response-handler.service.ts (source)
    - trade-flow-api/src/quote-token/controllers/public-quote.controller.ts (source — CRITICAL: executor must see the existing `@Controller("v1/public")` and `@Get("quote/:token")` decorators — they MUST NOT change)
    - trade-flow-api/src/quote-token/requests/decline-quote.request.ts (source)
    - trade-flow-api/src/quote-token/responses/public-quote.response.ts (source)
    - trade-flow-api/src/quote-token/test/services/quote-token-creator.service.spec.ts (source)
    - trade-flow-api/src/quote-token/test/services/quote-token-retriever.service.spec.ts (source)
    - trade-flow-api/src/quote-token/test/services/quote-token-revoker.service.spec.ts (source)
    - trade-flow-api/src/quote-token/test/services/public-quote-retriever.service.spec.ts (source)
    - trade-flow-api/src/quote-token/test/services/quote-response-handler.service.spec.ts (source)
    - trade-flow-api/src/quote-token/test/controllers/public-quote.controller.spec.ts (source)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Pattern 7 Session-Auth Guard with documentType Discriminator; §Common Pitfalls → Pitfall 3)
  </read_first>
  <behavior>
    - `DocumentTokenCreator.create(...)` accepts `documentType` as an input (defaults to `"quote"` for backward compatibility with quote flow) and writes it to the repository.
    - `DocumentTokenRetriever` / `DocumentTokenRevoker` are pure renames of the quote-token equivalents — no new logic.
    - `PublicQuoteController` keeps `@Controller("v1/public")` + `@Get("quote/:token")` decorators unchanged so the URL `/v1/public/quote/:token` still resolves.
    - Every `PublicQuoteController` handler starts with a `documentType === "quote"` assertion that throws `DOCUMENT_TOKEN_TYPE_MISMATCH` as a 404 on mismatch.
    - The existing `public-quote.controller.spec.ts` tests continue to pass with only import-path and class-name updates (no behavioral changes to asserted outputs).
    - A NEW spec test is added asserting that a document token with `documentType: "estimate"` causes the public quote controller to return 404 (not 200, not 500).
  </behavior>
  <action>
**Step 1: `git mv` the service, controller, request, response, and spec files** from `src/quote-token/...` to `src/document-token/...`. For each file, apply the in-file renames:

Class renames:
- `QuoteTokenCreator` → `DocumentTokenCreator`
- `QuoteTokenRetriever` → `DocumentTokenRetriever`
- `QuoteTokenRevoker` → `DocumentTokenRevoker`
- (Keep `PublicQuoteRetriever`, `QuoteResponseHandler`, `PublicQuoteController`, `IPublicQuoteResponse`, `DeclineQuoteRequest` names unchanged — they are quote-specific by purpose, not token-specific.)

Constructor-parameter / type-import renames:
- `IQuoteTokenDto` → `IDocumentTokenDto`
- `IQuoteTokenEntity` → `IDocumentTokenEntity`
- `QuoteTokenRepository` → `DocumentTokenRepository`
- `QuoteSessionAuthGuard` → `DocumentSessionAuthGuard`
- `QuoteTokenCreator` → `DocumentTokenCreator` (at callsites)
- `QuoteTokenRetriever` → `DocumentTokenRetriever`
- `QuoteTokenRevoker` → `DocumentTokenRevoker`

Path alias renames: every `from "@quote-token/..."` → `from "@document-token/..."`

Request-field rename inside the controller:
- `request.quoteToken` → `request.documentToken`

**Step 2: `DocumentTokenCreator.create`** — update signature to accept `documentType` parameter. The quote flow always passes `"quote"`. Recommended signature:
```typescript
public async create(
  documentId: string,
  documentType: "quote" | "estimate",
  recipientEmail: string,
  // ... any existing params unchanged
): Promise<IDocumentTokenDto>
```
Internally: pass `documentType` through to the repository entity. If the existing quote-token-creator has a different signature, preserve its shape and just add `documentType` as a required parameter positioned consistently.

Update the one existing caller in `QuoteTransitionService` (Task 4 below) to pass `"quote"`.

**Step 3: Rewrite `PublicQuoteController`** — CRITICAL for token-type-confusion mitigation (Pitfall 3 / D-TKN-03):

The decorators `@Controller("v1/public")` and `@Get("quote/:token")` MUST NOT CHANGE. The URL `/v1/public/quote/:token` is customer-facing and was shipped in v1.3.

Every handler method (there may be 1-3: one for GET quote detail, possibly one POST for decline, etc.) MUST start with:

```typescript
@UseGuards(DocumentSessionAuthGuard)
@Get("quote/:token")
public async findByToken(
  @Req() request: { documentToken: IDocumentTokenDto; params: { token: string } },
): Promise<IResponse<IPublicQuoteResponse>> {
  if (request.documentToken.documentType !== "quote") {
    throw createHttpError(
      new ResourceNotFoundError(ErrorCodes.DOCUMENT_TOKEN_TYPE_MISMATCH, "Quote not found"),
    );
  }
  try {
    const response = await this.publicQuoteRetriever.getPublicQuote(request.documentToken.documentId);
    return createResponse([response]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

Apply the same assertion pattern to every other handler in the file (decline, any accept, view tracking, etc.). Do NOT add a narrative comment — the assertion is self-documenting via the error code name `DOCUMENT_TOKEN_TYPE_MISMATCH`.

If the existing controller injects `QuoteTokenRetriever` or similar, rename to `DocumentTokenRetriever` at the constructor.

**Step 4: Update the controller spec** `public-quote.controller.spec.ts`:
- Replace old class/type imports with new names
- Update the mock document-token fixture to include `documentType: "quote"`
- Existing tests continue to assert the same HTTP responses as before
- ADD a NEW test case: "returns 404 when document token has documentType 'estimate'"
  ```typescript
  it("returns 404 when document token has documentType 'estimate'", async () => {
    const estimateToken = DocumentTokenMockGenerator.create({ documentType: "estimate" });
    const request = { documentToken: estimateToken, params: { token: estimateToken.token } };
    await expect(controller.findByToken(request)).rejects.toThrow(/Quote not found/);
  });
  ```

**Step 5: Update the service specs** (`document-token-creator`, `document-token-retriever`, `document-token-revoker`, `public-quote-retriever`, `quote-response-handler`) — pure mechanical rename of imports, class names, and mock fixtures. Each existing assertion stays the same; only the symbols change.

For `DocumentTokenCreator.spec.ts`, ADD a NEW assertion: "sets documentType on the created entity" — verify the new parameter is plumbed through to the repository call.

**Step 6: Run the full document-token slice:**

```
cd trade-flow-api && npm run test -- --testPathPattern=document-token
```

Every spec in the slice MUST pass. Fix any import path or type annotation at the root cause.

Commit message: `refactor(41): rename document-token services and controller with documentType assertion`.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/document-token/services/document-token-creator.service.ts`
    - `test -f trade-flow-api/src/document-token/services/document-token-retriever.service.ts`
    - `test -f trade-flow-api/src/document-token/services/document-token-revoker.service.ts`
    - `test -f trade-flow-api/src/document-token/services/public-quote-retriever.service.ts`
    - `test -f trade-flow-api/src/document-token/services/quote-response-handler.service.ts`
    - `test -f trade-flow-api/src/document-token/controllers/public-quote.controller.ts`
    - `grep -c "export class DocumentTokenCreator" trade-flow-api/src/document-token/services/document-token-creator.service.ts` returns 1
    - `grep -c "QuoteTokenCreator\\|QuoteTokenRetriever\\|QuoteTokenRevoker" trade-flow-api/src/document-token/services/` returns 0 (recursively — no stale references)
    - `grep -c "@Controller(\"v1/public\")" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns 1
    - `grep -c "@Get(\"quote/:token\")" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns 1
    - `grep -c "documentType !== \"quote\"" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns at least 1 (MANDATORY token-type assertion)
    - `grep -c "DOCUMENT_TOKEN_TYPE_MISMATCH" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns at least 1
    - `grep -c "request.quoteToken" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns 0
    - `grep -c "request.documentToken" trade-flow-api/src/document-token/controllers/public-quote.controller.ts` returns at least 1
    - `grep -c "returns 404 when document token has documentType 'estimate'" trade-flow-api/src/document-token/test/controllers/public-quote.controller.spec.ts` returns 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=document-token` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|as unknown as" trade-flow-api/src/document-token` returns 0
    - `grep -rc "as QuoteStatus\\|as EstimateStatus\\|as IQuoteTokenDto\\|as IDocumentTokenDto" trade-flow-api/src/document-token` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=document-token</automated>
  </verify>
  <done>All document-token services, controller, and specs relocated with renames applied and token-type assertion in place</done>
</task>

<task type="auto">
  <name>Task 3: Delete src/quote-token/, update AppModule, QuoteController, and QuoteTransitionService imports</name>
  <files>
    trade-flow-api/src/app.module.ts,
    trade-flow-api/src/quote/controllers/quote.controller.ts,
    trade-flow-api/src/quote/services/quote-transition.service.ts
  </files>
  <read_first>
    - trade-flow-api/src/app.module.ts (current imports array — must see QuoteTokenModule entry to replace it)
    - trade-flow-api/src/quote/controllers/quote.controller.ts (any `@quote-token/*` imports and their callsites — rename to `@document-token/*` and update class names)
    - trade-flow-api/src/quote/services/quote-transition.service.ts (imports `QuoteTokenRevoker` per RESEARCH.md line 11 — must be renamed and constructor updated)
    - trade-flow-api/src/quote-token/ (verify the entire directory is still present before deleting — `find trade-flow-api/src/quote-token/ -type f`)
  </read_first>
  <action>
**Step 1: Update `trade-flow-api/src/app.module.ts`:**

- Replace `import { QuoteTokenModule } from "@quote-token/quote-token.module";` with `import { DocumentTokenModule } from "@document-token/document-token.module";`
- In the `imports: [...]` array, replace `QuoteTokenModule` with `DocumentTokenModule`
- If `AppModule` has any `forwardRef(() => QuoteTokenModule)` wrappers, update them to `forwardRef(() => DocumentTokenModule)`
- Do NOT touch any other module entries

**Step 2: Update `trade-flow-api/src/quote/controllers/quote.controller.ts`:**

- Any `import { QuoteTokenRetriever } from "@quote-token/..."` → `import { DocumentTokenRetriever } from "@document-token/services/document-token-retriever.service"`
- Constructor parameter `private readonly quoteTokenRetriever: QuoteTokenRetriever` → `private readonly documentTokenRetriever: DocumentTokenRetriever`
- Every callsite `this.quoteTokenRetriever.xxx` → `this.documentTokenRetriever.xxx`
- If the retriever is called with just a quote id and needs to filter by documentType, the caller must pass `"quote"` explicitly to maintain backward compatibility. Check the method signature after Task 2 and update callsites accordingly.

**Step 3: Update `trade-flow-api/src/quote/services/quote-transition.service.ts`:**

- `import { QuoteTokenRevoker } from "@quote-token/..."` → `import { DocumentTokenRevoker } from "@document-token/services/document-token-revoker.service"`
- Constructor parameter rename: `quoteTokenRevoker: QuoteTokenRevoker` → `documentTokenRevoker: DocumentTokenRevoker`
- `this.quoteTokenRevoker.revokeAllForQuote(existing.id)` → `this.documentTokenRevoker.revokeAllForDocument(existing.id, "quote")` (or equivalent — match the new method signature from Task 2's `DocumentTokenRevoker`).

NOTE: If the revoker method was `revokeAllForQuote(quoteId)`, it should have been renamed in Task 2 to `revokeAllForDocument(documentId, documentType)` to support both document types. Confirm this at the revoker file; if Task 2 did not rename the method (because the plan was mechanical-only), add the rename as part of this task and update the spec accordingly.

**Step 4: Grep for any surviving `@quote-token/*` imports anywhere in the tree:**

```
grep -rn "@quote-token/" trade-flow-api/src
```

This MUST return zero matches. If any file still has a `@quote-token/*` import, rename it in place before proceeding.

**Step 5: Delete the old `src/quote-token/` directory entirely:**

```
git rm -r trade-flow-api/src/quote-token/
```

Verify removal: `test ! -d trade-flow-api/src/quote-token` must succeed.

**Step 6: Run typecheck + full quote suite + full document-token suite:**

```
cd trade-flow-api && npm run typecheck
cd trade-flow-api && npm run test -- --testPathPattern=quote
cd trade-flow-api && npm run test -- --testPathPattern=document-token
```

All three must exit 0.

Commit message: `refactor(41): delete src/quote-token/ and rewire AppModule + QuoteModule consumers to @document-token`.
  </action>
  <acceptance_criteria>
    - `test ! -d trade-flow-api/src/quote-token` (directory does not exist)
    - `grep -rn "@quote-token/" trade-flow-api/src` returns zero matches
    - `grep -rn "from \"@quote-token\"" trade-flow-api/src` returns zero matches
    - `grep -c "DocumentTokenModule" trade-flow-api/src/app.module.ts` returns at least 2 (import + imports array entry)
    - `grep -c "QuoteTokenModule" trade-flow-api/src/app.module.ts` returns 0
    - `grep -c "DocumentTokenRetriever\\|DocumentTokenRevoker" trade-flow-api/src/quote/controllers/quote.controller.ts` returns at least 1 (if the controller uses token services at all)
    - `grep -c "QuoteTokenRetriever\\|QuoteTokenRevoker" trade-flow-api/src/quote/controllers/quote.controller.ts` returns 0
    - `grep -c "DocumentTokenRevoker" trade-flow-api/src/quote/services/quote-transition.service.ts` returns at least 1
    - `grep -c "QuoteTokenRevoker" trade-flow-api/src/quote/services/quote-transition.service.ts` returns 0
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=quote` exits 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=document-token` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck && npm run test -- --testPathPattern=quote && npm run test -- --testPathPattern=document-token</automated>
  </verify>
  <done>src/quote-token/ deleted; AppModule, QuoteController, and QuoteTransitionService all reference document-token; all affected test suites pass</done>
</task>

<task type="auto">
  <name>Task 4: Run full CI gate to confirm token rename is non-regressive</name>
  <files>none</files>
  <read_first>
    - trade-flow-api/package.json
  </read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected exit code: 0.

This runs the full suite (test + lint:check + format:check + typecheck). Pitfall 7 + Pitfall 3 scenarios would surface here if the rename or the token-type assertion is broken.

If format:check fails, run `cd trade-flow-api && npm run format` and commit the prettier fixes as `chore(41): prettier fixes for document-token rename`.

If lint:check fails, fix the reported issues at their root cause — never disable rules. Typical failures: unused import (from a stale rename), missing return type (copy-paste from old file that used a different signature).

If test fails, re-read the failing test output against the changes from Task 3 — the most likely culprit is a TestingModule in a spec file that imports from the old `@quote-token/*` path. Fix and re-commit.

No `@ts-ignore`, no `eslint-disable`, no test skipping.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - `grep -rn "@quote-token/" trade-flow-api/src trade-flow-api/openapi.yaml trade-flow-api/tsconfig.json` returns zero matches (final sweep)
    - `find trade-flow-api/src/quote-token -type f` returns empty
    - No suppressions introduced anywhere in the diff (`grep -rc "@ts-ignore\\|@ts-expect-error\\|eslint-disable" trade-flow-api/src/document-token trade-flow-api/src/app.module.ts trade-flow-api/src/quote/controllers/quote.controller.ts trade-flow-api/src/quote/services/quote-transition.service.ts` returns 0)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green; quote-token → document-token rename landed with zero regressions and token-type-confusion mitigation in place</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| customer browser → /v1/public/quote/:token | Untrusted token; validated by DocumentSessionAuthGuard (existence, expiry, revocation) |
| DocumentSessionAuthGuard → PublicQuoteController | Guard loads token; controller asserts documentType discriminator |
| QuoteTransitionService → DocumentTokenRevoker | Internal service call; no trust boundary crossed |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain and §Common Pitfalls #3.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-03-01 | Spoofing | Token-type confusion — estimate token accepted at /v1/public/quote/:token OR vice versa | mitigate | PublicQuoteController asserts `documentType === "quote"` on every handler entry (Task 2); returns 404 via `DOCUMENT_TOKEN_TYPE_MISMATCH` on mismatch; new spec test asserts this path |
| T-41-03-02 | Tampering | Silent customer-link breakage during rename due to production data loss | mitigate | Plan 01 Task 1 BLOCKING pre-check verifies `quote_tokens` is empty before this plan can run |
| T-41-03-03 | Information Disclosure | 410 Revoked vs 404 Not-Found leaks token existence | accept | Existing quote-token flow already uses this distinction; preserving the same behavior is not a regression; rate limiting at ThrottlerGuard layer limits enumeration |
| T-41-03-04 | Elevation of Privilege | `documentType: "quote"` set at token creation can be tampered with on update | mitigate | Repository `updateOne` calls in `DocumentTokenRepository` MUST NOT include `documentType` in the `$set` block — it is immutable after create; verify in Task 1 Step 5 |
| T-41-03-05 | Denial of Service | Pattern 7 (Session-Auth Guard) rate limit preserved from quote-token flow | mitigate | `@Throttle` decorator on PublicQuoteController is preserved during the rename; Task 2 confirms decorator block is moved intact |
| T-41-03-06 | Tampering | MongoDB injection via user-supplied token string | mitigate | `DocumentTokenRepository.findByToken` takes a `string` parameter that is passed directly to `findOne({ token })` — class-validator's global `ValidationPipe` + `forbidNonWhitelisted` strips any object payload; `:token` route param is extracted by NestJS as a plain string |
| T-41-03-07 | Information Disclosure | `firstViewedAt` written on create rather than lazily | mitigate | Pitfall 8 mitigation: `DocumentTokenRepository.toEntity` omits `firstViewedAt` entirely when the DTO does not provide it; new spec test asserts `entity.firstViewedAt === undefined` after create |
</threat_model>

<verification>
1. `src/quote-token/` deleted; `src/document-token/` exists with full file set.
2. `grep -rn "@quote-token/" trade-flow-api/src trade-flow-api/tsconfig.json` returns zero.
3. `grep -rn "quote_tokens" trade-flow-api/src` returns zero (collection constant renamed).
4. `grep -rn "documentType !== \"quote\"" trade-flow-api/src/document-token/controllers` returns at least 1.
5. `cd trade-flow-api && npm run ci` exits 0.
6. Existing PublicQuoteController spec still passes against the new imports (regression check).
7. New spec test for `documentType: "estimate"` → 404 passes.
</verification>

<success_criteria>
- `/v1/public/quote/:token` URL continues to resolve for existing quote flows
- `DocumentTokenModule` is registered in `AppModule` and exports the guard + services for Phase 45 reuse
- Token-type-confusion mitigation is structurally enforced (controller assertion + spec test)
- `DocumentTokenCreator` accepts `documentType` parameter so Phase 44 (email/send) can create estimate tokens
- Plan 04+ (estimate scaffold) can assume the new path aliases and error codes are available
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-03-SUMMARY.md` documenting:
- List of files renamed (FROM → TO)
- Class/type/field-level renames applied
- New tests added (token-type mismatch 404 case, firstViewedAt undefined case)
- The full `npm run ci` output
</output>
