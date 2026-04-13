# Phase 47: Convert to Quote & Mark as Lost - Research

**Researched:** 2026-04-13
**Domain:** Estimate lifecycle terminal transitions (Convert to Quote, Mark as Lost) -- backend services + frontend UI
**Confidence:** HIGH

## Summary

Phase 47 adds two terminal flows to the estimate lifecycle: converting an estimate into a fully independent draft quote, and manually marking an estimate as lost. Both flows share common infrastructure: they transition the estimate to a terminal status via the existing `EstimateTransitionService`, cancel pending follow-ups via `IEstimateFollowupCanceller`, and lock the estimate from further revisions.

The backend work is straightforward -- two new service classes (`EstimateToQuoteConverter`, `EstimateLostMarker`) and two new POST endpoints on the existing `EstimateController`. The convert flow requires cross-module interaction with the quote module (creating a new quote with line items copied from `estimate_line_items` to `quote_line_items`), plus new fields on the quote entity/DTO/response for the back-link. The frontend work extends `EstimateActionStrip` with two new buttons, adds a `MarkAsLostDialog`, and adds read-only display components for the locked states.

**Primary recommendation:** Implement backend services first (converter + lost marker), then frontend mutations and UI components. The quote entity schema change (`sourceEstimateId`, `sourceEstimateNumber`) is the key cross-module concern that must be handled carefully.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-CONV-01:** Convert triggered by "Convert to Quote" button in `EstimateActionStrip`, visible only on `SENT` or `RESPONDED` status
- **D-CONV-02:** Click flow: generate UUID v4 idempotency key in component state, disable button with spinner, POST, navigate to `/quotes/{quoteId}/edit`
- **D-CONV-03:** Existing `QuoteEditPage` serves as mandatory review step -- no dedicated convert-review route
- **D-CONV-04:** Idempotent double-submit within 24h returns same quoteId; UI treats identically to first-time create
- **D-CONV-05:** Idempotency key UUID generated at click time in component state, no localStorage persistence
- **D-LOCK-01:** Converted state removes all action buttons, shows "Converted to Quote" chip/link with `convertedToQuoteId`
- **D-LOCK-02:** Lost state removes all action buttons, shows reason card with structured reason/freeform text/timestamp
- **D-LOCK-03:** Both locked states render no Edit, no Revise, no Send buttons
- **D-LOST-01:** Mark as Lost triggered by button in `EstimateActionStrip`, visible on `SENT`, `VIEWED`, or `RESPONDED`
- **D-LOST-02:** Dialog with structured reason picker (same taxonomy as customer decline) and optional freeform textarea
- **D-LOST-03:** Both structured reason and freeform text are optional -- API accepts `reason: null` and `notes: null`
- **D-LOST-04:** Lost reason taxonomy uses same labels verbatim as customer decline (Phase 45 D-CTA-04)
- **D-API-01:** Two new endpoints on existing `EstimateController`: `POST /v1/estimates/:id/convert` and `POST /v1/estimates/:id/mark-lost`
- **D-API-02:** `EstimateToQuoteConverter` service: find latest revision, read estimate line items, create quote via QuoteCreator pattern, transition to Converted, write `convertedToQuoteId`, cancel follow-ups
- **D-API-03:** Idempotency via `convertedToQuoteId` field check + `convertIdempotencyKey` field on estimate entity
- **D-API-04:** `EstimateLostMarker` service: validate transition, persist reason/notes/timestamp, transition to Lost, cancel follow-ups
- **D-API-05:** Both services in `src/estimate/services/`, registered in existing `EstimateModule`
- **D-BACKLINK-01:** New nullable `sourceEstimateId` and `sourceEstimateNumber` fields on quote entity
- **D-BACKLINK-02:** "Converted from E-YYYY-NNN" back-link on quote detail page links to `/estimates/{sourceEstimateId}`

### Claude's Discretion
- Exact visual styling of "Converted to Quote" chip/link in action strip
- Exact visual design of Lost reason card (color, border, icon)
- RTK Query cache invalidation strategy after convert/mark-lost
- Whether `EstimateToQuoteConverter` calls existing `QuoteCreator` service or writes directly to repository
- Unit test scope for `EstimateToQuoteConverter` and `EstimateLostMarker`

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CONV-01 | User can convert a Sent or Responded estimate to a new quote from the estimate detail page | Backend: `EstimateToQuoteConverter` + `POST /v1/estimates/:id/convert`; Frontend: Convert to Quote button in `EstimateActionStrip` |
| CONV-02 | Convert pulls from latest revision, copies line items with literal tax rate percentages (snapshot), drops contingency | `EstimateToQuoteConverter` reads `isCurrent: true` revision, maps `IEstimateLineItemDto` fields to `IQuoteLineItemDto` with `taxRate` preserved as-is, no contingency on quote |
| CONV-03 | Convert opens the new quote in edit mode for trader review before saving (mandatory review) | Frontend navigates to `/quotes/{quoteId}/edit` after successful POST; existing `QuoteEditPage` serves as review step |
| CONV-04 | Convert endpoint accepts `Idempotency-Key` header and returns same quote on double-submit within 24h | `convertedToQuoteId` field check + `convertIdempotencyKey` field on estimate entity |
| CONV-05 | Converted estimate transitions to Converted status with `convertedToQuoteId` back-link, locked from further revisions | `EstimateTransitionService.transition()` to `CONVERTED` + write `convertedToQuoteId`; transition map already blocks all exits from `CONVERTED` |
| CONV-06 | Converted quote displays "Converted from E-YYYY-NNN" back-link on its detail page | New `sourceEstimateId` + `sourceEstimateNumber` fields on `IQuoteEntity`/`IQuoteDto`/`IQuoteResponse`; `QuoteSourceEstimateLink` component on quote detail page |
| LOST-01 | User can manually mark an estimate as Lost with structured reason or freeform text | `EstimateLostMarker` + `POST /v1/estimates/:id/mark-lost`; `MarkAsLostDialog` component |
| LOST-02 | Marking an estimate as Lost cancels all pending follow-ups | `cancelAllFollowups` called via `IEstimateFollowupCanceller` DI token |
</phase_requirements>

## Standard Stack

No new libraries are needed for Phase 47. All work uses the existing technology stack.

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Backend framework | Existing project framework [VERIFIED: codebase] |
| mongodb | 7.0 | Native MongoDB driver | Existing database layer [VERIFIED: codebase] |
| class-validator | 0.14.1 | Request DTO validation | Existing validation pattern [VERIFIED: codebase] |
| uuid | (npm) | UUID v4 generation for idempotency key | Frontend needs `crypto.randomUUID()` (Web API, no library needed) [VERIFIED: MDN docs] |

### Frontend (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19.x | UI framework | Existing [VERIFIED: codebase] |
| @reduxjs/toolkit | 2.x | RTK Query mutations | Existing API layer [VERIFIED: codebase] |
| lucide-react | 0.563.0 | Icons (ArrowRightLeft, XCircle, AlertTriangle, Loader2, ArrowLeft) | Existing icon library [VERIFIED: codebase] |
| shadcn/ui Dialog | installed | MarkAsLostDialog | Already installed [VERIFIED: UI-SPEC] |

**No new npm packages to install.**

## Architecture Patterns

### Backend: New Services in Existing Module

Both new services register in `EstimateModule` (already has 20+ providers). No new module needed. [VERIFIED: `estimate.module.ts`]

```
src/estimate/
  services/
    estimate-to-quote-converter.service.ts    # NEW
    estimate-lost-marker.service.ts           # NEW
  requests/
    convert-estimate.request.ts               # NEW (empty body, headers only)
    mark-estimate-lost.request.ts             # NEW
  controllers/
    estimate.controller.ts                    # MODIFIED (add 2 endpoints)
```

### Backend: Cross-Module Quote Schema Change

The quote module needs new fields for the back-link. This is the key cross-module concern.

```
src/quote/
  entities/quote.entity.ts           # MODIFIED (add sourceEstimateId, sourceEstimateNumber)
  data-transfer-objects/quote.dto.ts  # MODIFIED (add sourceEstimateId, sourceEstimateNumber)
  responses/quote.responses.ts        # MODIFIED (add sourceEstimateId, sourceEstimateNumber)
  repositories/quote.repository.ts    # MODIFIED (toDto mapping for new fields)
  controllers/quote.controller.ts     # MODIFIED (mapToResponse for new fields)
```

### Backend: Convert Flow Sequence

1. Load current estimate (via `EstimateRetriever`, `isCurrent: true`)
2. Check idempotency: if `convertedToQuoteId` is already set AND `convertIdempotencyKey` matches header, return existing quoteId
3. Validate status allows transition to CONVERTED (SENT or RESPONDED per D-CONV-01; transition map also allows VIEWED)
4. Load estimate line items (via `EstimateLineItemRetriever`)
5. Build quote DTO from estimate data (map fields, drop contingency, snapshot tax rates)
6. Create quote via `QuoteCreator.create()` (generates Q-YYYY-NNN number, writes to `quotes` + `quotelineitems`)
7. Update quote with `sourceEstimateId` and `sourceEstimateNumber` (if QuoteCreator doesn't support these fields natively)
8. Transition estimate to CONVERTED via `EstimateTransitionService.transition()`
9. Write `convertedToQuoteId` and `convertIdempotencyKey` on estimate entity via `EstimateRepository.update()`
10. Cancel follow-ups via `IEstimateFollowupCanceller.cancelAllFollowups()`
11. Return quoteId in response

### Backend: Mark as Lost Flow Sequence

1. Load current estimate
2. Validate status allows transition to LOST
3. Write `lostReason`, `lostNotes`, `markedLostAt` on estimate entity
4. Transition to LOST via `EstimateTransitionService.transition()`
5. Cancel follow-ups via `IEstimateFollowupCanceller.cancelAllFollowups()`
6. Return updated estimate

### Frontend: RTK Query Mutations

```
trade-flow-ui/src/features/estimates/api/estimateApi.ts  # MODIFIED (add 2 mutations)
```

Two new mutations needed:
- `useConvertEstimateMutation` -- POST with Idempotency-Key header
- `useMarkEstimateLostMutation` -- POST with reason/notes body

### Frontend: Component Modifications

```
trade-flow-ui/
  src/features/estimates/components/
    EstimateActionStrip.tsx          # MODIFIED (add Convert + Mark as Lost buttons, locked states)
    MarkAsLostDialog.tsx             # NEW
    EstimateLostReasonCard.tsx       # NEW
    EstimateConvertedLink.tsx        # NEW
  src/features/quotes/components/
    QuoteSourceEstimateLink.tsx      # NEW
    QuoteDetailPage.tsx              # MODIFIED (render QuoteSourceEstimateLink)
  src/types/
    estimate.ts                      # MODIFIED (add lostReason, lostNotes, lostAt, convertedToQuoteId fields)
    quote.ts                         # MODIFIED (add sourceEstimateId, sourceEstimateNumber fields)
```

### Anti-Patterns to Avoid
- **Do NOT create a new module for the converter service** -- it lives in `EstimateModule` per D-API-05 [VERIFIED: CONTEXT.md]
- **Do NOT use a separate idempotency table** -- the `convertedToQuoteId` field on the estimate IS the idempotency check [VERIFIED: CONTEXT.md D-API-03]
- **Do NOT call `QuoteCreator.create()` without understanding its validation** -- it validates customer is ACTIVE and job exists, both of which are already true on an estimate [VERIFIED: `quote-creator.service.ts` lines 29-57]
- **Do NOT persist the idempotency key to localStorage** -- it lives in component state only [VERIFIED: CONTEXT.md D-CONV-05]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Quote creation from estimate | Custom direct MongoDB insert for quotes | `QuoteCreator.create()` | Handles number generation, customer/job validation, policy enforcement, totals calculation [VERIFIED: codebase] |
| Status transition validation | Manual status check + update | `EstimateTransitionService.transition()` | Already validates against `ALLOWED_TRANSITIONS` map, sets timestamp fields automatically [VERIFIED: codebase] |
| Follow-up cancellation | Direct BullMQ job removal | `IEstimateFollowupCanceller` DI token injection | Abstracted behind interface; Phase 46 provides real implementation, current noop is safe [VERIFIED: codebase] |
| UUID v4 for idempotency | npm uuid package | `crypto.randomUUID()` | Native Web API, supported in all modern browsers, zero bundle cost [ASSUMED] |

## Common Pitfalls

### Pitfall 1: QuoteCreator Doesn't Accept sourceEstimateId
**What goes wrong:** `QuoteCreator.create()` builds the quote DTO from the request, which doesn't include `sourceEstimateId` or `sourceEstimateNumber`. The back-link fields would be lost.
**Why it happens:** `QuoteCreator` was built for the standard quote creation flow, not for conversion from estimates.
**How to avoid:** Either (a) modify `QuoteCreator.create()` to accept optional `sourceEstimateId`/`sourceEstimateNumber` fields on the DTO, or (b) use `QuoteRepository` directly to update the back-link fields after `QuoteCreator.create()` returns. Option (a) is cleaner if the fields are added to `IQuoteDto` and the repository maps them.
**Warning signs:** Quote created without back-link fields; CONV-06 broken.

### Pitfall 2: Idempotency Race Condition
**What goes wrong:** Two concurrent convert requests with the same idempotency key could both pass the "not yet converted" check and create two quotes.
**Why it happens:** Read-then-write without atomicity.
**How to avoid:** Use `findOneAndUpdate` with filter `{ _id: estimateId, convertedToQuoteId: null }` to atomically set `convertedToQuoteId`. If the update matches 0 documents, the estimate was already converted -- return the existing `convertedToQuoteId`.
**Warning signs:** Two quotes created for the same estimate conversion.

### Pitfall 3: Estimate Line Item to Quote Line Item Field Mapping
**What goes wrong:** Estimate line items have `estimateId` as the foreign key; quote line items have `quoteId`. Missing this remapping causes orphaned line items.
**Why it happens:** The two DTOs are structurally identical except for the parent document reference field name.
**How to avoid:** Explicitly map each field. Key differences: `estimateId` -> `quoteId`, `IEstimateLineItemDto` -> `IQuoteLineItemDto`, `EstimateLineItemStatus` -> `QuoteLineItemStatus`. Tax rate is copied verbatim (literal percentage snapshot per CONV-02).
**Warning signs:** Quote has no line items, or line items reference wrong parent.

### Pitfall 4: cancelAllFollowups vs cancelAllFollowupsForChain
**What goes wrong:** Using `cancelAllFollowups(estimateId, revisionNumber)` only cancels follow-ups for the specific revision, not the entire chain. If the estimate was revised, older revision follow-ups may still fire.
**Why it happens:** The interface offers single-revision cancellation; chain-wide cancellation is a separate concept referenced in CONTEXT.md but NOT present in the current interface file.
**How to avoid:** The current `IEstimateFollowupCanceller` interface only has `cancelAllFollowups(estimateId, revisionNumber)`. CONTEXT.md references `cancelAllFollowupsForChain(rootEstimateId)` but this method does NOT exist in the interface yet [VERIFIED: `estimate-followup-canceller.interface.ts`]. The planner must decide: (a) add `cancelAllFollowupsForChain` to the interface and noop implementation, or (b) call `cancelAllFollowups` for only the current revision (sufficient if Phase 46's send flow already cancelled old revision follow-ups when the new revision was sent).
**Warning signs:** Follow-up emails still firing after conversion/mark-as-lost.

### Pitfall 5: Missing Estimate Type Fields in Frontend
**What goes wrong:** The frontend `Estimate` type in `types/estimate.ts` lacks `convertedToQuoteId`, `lostAt`, `lostReason`, `lostNotes`, and `convertedAt` fields. Components can't access these values.
**Why it happens:** These fields were reserved on the backend entity but never added to the frontend type because no Phase needed them until now.
**How to avoid:** Add all missing fields to the `Estimate` interface in `types/estimate.ts` and `Quote` interface in `types/quote.ts`.
**Warning signs:** TypeScript errors in components trying to access these fields.

### Pitfall 6: QuoteCreator Validates Customer Status
**What goes wrong:** `QuoteCreator.create()` validates that the customer is ACTIVE before creating a quote. If a customer was deactivated between estimate send and conversion, the conversion fails.
**Why it happens:** `QuoteCreator` has a `validateQuote` method that checks `CustomerStatus.ACTIVE`.
**How to avoid:** This is actually correct behavior -- a deactivated customer should block quote creation. However, if the planner decides to bypass this check for conversions, the converter would need to call the repository directly instead of `QuoteCreator`. Recommendation: keep the validation; if the customer is inactive, the convert fails with a clear error.
**Warning signs:** Convert fails with `CUSTOMER_INACTIVE` error on a valid estimate.

## Code Examples

### Backend: EstimateToQuoteConverter Service Pattern

```typescript
// Source: Mirrors existing service patterns in trade-flow-api [VERIFIED: codebase]
@Injectable()
export class EstimateToQuoteConverter {
  constructor(
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateRepository: EstimateRepository,
    private readonly estimateLineItemRetriever: EstimateLineItemRetriever,
    private readonly estimateTransitionService: EstimateTransitionService,
    private readonly quoteCreator: QuoteCreator,
    @Inject(ESTIMATE_FOLLOWUP_CANCELLER)
    private readonly followupCanceller: IEstimateFollowupCanceller,
  ) {}

  public async convert(authUser: IUserDto, estimateId: string, idempotencyKey: string): Promise<string> {
    const estimate = await this.estimateRetriever.findByIdOrFail(authUser, estimateId);

    // Idempotency check
    if (estimate.convertedToQuoteId) {
      return estimate.convertedToQuoteId;
    }

    // Build quote DTO from estimate, create quote, transition estimate, cancel follow-ups
    // ... (planner fills in the detail)
  }
}
```

### Backend: Mark as Lost Request DTO

```typescript
// Source: Mirrors existing request patterns [VERIFIED: codebase]
import { IsOptional, IsString, MaxLength, IsIn } from "class-validator";

const LOST_REASONS = [
  "too_expensive",
  "going_with_someone_else",
  "decided_not_to_do_work",
  "just_getting_idea",
  "timing_not_right",
] as const;

export class MarkEstimateLostRequest {
  @IsOptional()
  @IsString()
  @IsIn(LOST_REASONS)
  reason?: string;

  @IsOptional()
  @IsString()
  @MaxLength(500)
  notes?: string;
}
```

### Backend: Controller Endpoint Pattern

```typescript
// Source: Mirrors existing estimate controller [VERIFIED: estimate.controller.ts]
@UseGuards(JwtAuthGuard)
@Post("estimates/:id/convert")
public async convert(
  @Req() request: { user: IUserDto; params: { id: string }; headers: { "idempotency-key"?: string } },
): Promise<IResponse<{ quoteId: string }>> {
  try {
    const idempotencyKey = request.headers["idempotency-key"];
    const quoteId = await this.estimateToQuoteConverter.convert(
      request.user, request.params.id, idempotencyKey,
    );
    return createResponse([{ quoteId }]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

### Frontend: RTK Query Mutation Pattern

```typescript
// Source: Mirrors existing estimateApi.ts mutations [VERIFIED: estimateApi.ts]
convertEstimate: builder.mutation<{ quoteId: string }, { estimateId: string; idempotencyKey: string }>({
  query: ({ estimateId, idempotencyKey }) => ({
    url: `/v1/estimates/${estimateId}/convert`,
    method: "POST",
    headers: { "Idempotency-Key": idempotencyKey },
  }),
  transformResponse: (response: StandardResponse<{ quoteId: string }>) => response.data[0],
  invalidatesTags: (_result, _error, { estimateId }) => [
    { type: "Estimate", id: estimateId },
    { type: "Estimate", id: "LIST" },
    { type: "Quote", id: "LIST" },
  ],
}),

markEstimateLost: builder.mutation<Estimate, { estimateId: string; reason?: string; notes?: string }>({
  query: ({ estimateId, ...body }) => ({
    url: `/v1/estimates/${estimateId}/mark-lost`,
    method: "POST",
    body,
  }),
  transformResponse: unwrapSingle,
  invalidatesTags: (_result, _error, { estimateId }) => [
    { type: "Estimate", id: estimateId },
    { type: "Estimate", id: "LIST" },
  ],
}),
```

### Frontend: Idempotency Key Generation

```typescript
// Source: Web Crypto API [ASSUMED: standard Web API]
const idempotencyKey = crypto.randomUUID(); // generates UUID v4
```

## Key Entity Schema Findings

### Estimate Entity: Fields Already Reserved [VERIFIED: `estimate.entity.ts`]

The following fields exist on `IEstimateEntity` and are ready for Phase 47 to write:

| Field | Type | Current Value | Phase 47 Action |
|-------|------|--------------|-----------------|
| `convertedAt` | `Date?` | undefined | Set by `EstimateTransitionService` on CONVERTED transition |
| `convertedToQuoteId` | `ObjectId?` | undefined | Written by `EstimateToQuoteConverter` |
| `lostAt` | `Date?` | undefined | Set by `EstimateTransitionService` on LOST transition |

### Estimate Entity: Fields NOT Yet Reserved [VERIFIED: `estimate.entity.ts`]

These fields are referenced in CONTEXT.md but do NOT exist yet:

| Field | Type | Needed For |
|-------|------|-----------|
| `convertIdempotencyKey` | `string?` | D-API-03: 24h idempotency window check |
| `lostReason` | `string?` | D-API-04: structured reason for lost |
| `lostNotes` | `string?` | D-API-04: freeform text for lost |
| `markedLostAt` | `Date?` | D-API-04: timestamp (note: `lostAt` exists but is set by transition service; `markedLostAt` may be redundant -- planner decides whether to reuse `lostAt` or add a separate field) |

### Quote Entity: Fields NOT Yet Present [VERIFIED: `quote.entity.ts`]

These fields need to be added per D-BACKLINK-01:

| Field | Type | Purpose |
|-------|------|---------|
| `sourceEstimateId` | `ObjectId?` | Back-link to source estimate |
| `sourceEstimateNumber` | `string?` | Snapshot of E-YYYY-NNN at convert time |

### Quote DTO and Response: Fields NOT Yet Present [VERIFIED: `quote.dto.ts`, `quote.responses.ts`]

Both need corresponding nullable fields added.

### Frontend Estimate Type: Missing Fields [VERIFIED: `types/estimate.ts`]

The `Estimate` interface needs:

| Field | Type | Purpose |
|-------|------|---------|
| `convertedAt` | `string?` | Conversion timestamp |
| `convertedToQuoteId` | `string?` | Link to converted quote |
| `lostAt` | `string?` | Lost timestamp |
| `lostReason` | `string?` | Structured reason |
| `lostNotes` | `string?` | Freeform notes |

### Frontend Quote Type: Missing Fields [VERIFIED: `types/quote.ts`]

The `Quote` interface needs:

| Field | Type | Purpose |
|-------|------|---------|
| `sourceEstimateId` | `string?` | Back-link to source estimate |
| `sourceEstimateNumber` | `string?` | Snapshot of E-YYYY-NNN |

## Transition Map Analysis [VERIFIED: `estimate-transitions.ts`]

| From Status | To CONVERTED | To LOST | Notes |
|-------------|-------------|---------|-------|
| SENT | Allowed | Allowed | Primary conversion path |
| VIEWED | Allowed | Allowed | D-CONV-01 says only SENT/RESPONDED, but transition map allows VIEWED too |
| RESPONDED | Allowed | Allowed | Primary conversion path |
| DECLINED | Not allowed | Allowed | LOST from declined is allowed in transition map |
| Others | Not allowed | Not allowed | Terminal states block all transitions |

**Important:** D-CONV-01 says convert is available from SENT or RESPONDED only. The transition map also allows CONVERTED from VIEWED. The controller/service should enforce D-CONV-01's restriction (SENT or RESPONDED only) even though the transition map is more permissive. The UI button visibility already enforces this per D-CONV-01.

D-LOST-01 says mark-as-lost is available from SENT, VIEWED, or RESPONDED. The transition map also allows LOST from DECLINED. The controller/service should enforce D-LOST-01's restriction.

## IEstimateFollowupCanceller Interface Status [VERIFIED: source code]

Current interface has ONLY:
```typescript
export interface IEstimateFollowupCanceller {
  cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
}
```

CONTEXT.md D-API-02 references `cancelAllFollowupsForChain(rootEstimateId)` but this method does NOT exist. Two options:
1. Add `cancelAllFollowupsForChain` to the interface (and noop impl) -- cleaner but modifies Phase 42/46 code
2. Call `cancelAllFollowups` for the current revision only -- sufficient if prior revision follow-ups were already cancelled when the new revision was sent

**Recommendation:** Use `cancelAllFollowups(currentEstimate.id, currentEstimate.revisionNumber)` for the current revision. When converting or marking lost, the estimate is always the current revision (`isCurrent: true`), and any prior revision's follow-ups would have been cancelled when this revision was sent (Phase 44/46 behavior). This is safe.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate idempotency table | Entity field as idempotency check | Phase 47 design | Simpler, no new collection needed |
| Custom UUID library | `crypto.randomUUID()` | Browser native API | Zero bundle cost, no import needed |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `crypto.randomUUID()` is available in all target browsers | Don't Hand-Roll | Low -- supported in Chrome 92+, Firefox 95+, Safari 15.4+; project targets modern browsers |
| A2 | `markedLostAt` is redundant with `lostAt` (set by transition service) | Entity Schema Findings | Low -- if they differ, add a separate field; planner decides |
| A3 | Converting from VIEWED status (allowed by transition map but not by D-CONV-01) should be blocked at the service level | Transition Map Analysis | Medium -- if VIEWED estimates should be convertible, the service restriction is wrong |

## Open Questions

1. **Should `EstimateToQuoteConverter` call `QuoteCreator.create()` or write directly?**
   - What we know: `QuoteCreator.create()` validates customer status and job existence, generates Q-YYYY-NNN number, uses `AuthorizedCreatorFactory` for policy enforcement, and attaches totals. [VERIFIED: `quote-creator.service.ts`]
   - What's unclear: Whether the `sourceEstimateId`/`sourceEstimateNumber` fields can be passed through `QuoteCreator.create()` or need a separate repository update after creation.
   - Recommendation: Call `QuoteCreator.create()` for the standard creation flow (number generation, validation, policy), then do a follow-up `QuoteRepository.update()` to set the back-link fields. This avoids modifying `QuoteCreator`'s interface.

2. **Should `lostAt` (set by transition service) also serve as `markedLostAt`?**
   - What we know: `EstimateTransitionService.transition()` already sets `lostAt` when transitioning to LOST status. D-API-04 references `markedLostAt` as a separate field.
   - What's unclear: Whether they should be the same timestamp or separate.
   - Recommendation: Reuse `lostAt` from the transition service. Adding `markedLostAt` as a separate field is redundant -- the only way an estimate becomes Lost is through the mark-as-lost action. The UI can display `lostAt` as "Marked as lost on {date}".

3. **How should the convert response be shaped?**
   - What we know: The controller needs to return the new `quoteId` so the UI can navigate to `/quotes/{quoteId}/edit`.
   - What's unclear: Whether to return the full estimate response (with convertedToQuoteId) or just `{ quoteId }`.
   - Recommendation: Return `{ quoteId: string }` in the response data array. The UI only needs the quoteId for navigation. The estimate cache will be invalidated by RTK Query, so the next fetch picks up the converted status.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 with ts-jest |
| Framework (UI) | Vitest 4.1.3 |
| Config file (API) | jest config in package.json |
| Config file (UI) | vitest.config.ts |
| Quick run command (API) | `npm run test -- --testPathPattern=estimate-to-quote-converter` |
| Quick run command (UI) | `npm run test -- --testPathPattern=MarkAsLostDialog` |
| Full suite command (API) | `npm run ci` |
| Full suite command (UI) | `npm run ci` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CONV-01 | Convert endpoint creates quote from SENT/RESPONDED estimate | unit | `npm test -- estimate-to-quote-converter` | Wave 0 |
| CONV-02 | Line items copied with literal tax rates, no contingency | unit | `npm test -- estimate-to-quote-converter` | Wave 0 |
| CONV-04 | Idempotency key returns same quoteId on double-submit | unit | `npm test -- estimate-to-quote-converter` | Wave 0 |
| CONV-05 | Estimate transitions to Converted, locked from revisions | unit | `npm test -- estimate-to-quote-converter` | Wave 0 |
| LOST-01 | Mark as lost with optional reason/notes | unit | `npm test -- estimate-lost-marker` | Wave 0 |
| LOST-02 | Follow-ups cancelled on mark-as-lost | unit | `npm test -- estimate-lost-marker` | Wave 0 |
| CONV-03 | UI navigates to quote edit page | manual | Manual verification | N/A |
| CONV-06 | Back-link rendered on quote detail | manual | Manual verification | N/A |

### Wave 0 Gaps
- [ ] `src/estimate/test/services/estimate-to-quote-converter.service.spec.ts` -- covers CONV-01, CONV-02, CONV-04, CONV-05
- [ ] `src/estimate/test/services/estimate-lost-marker.service.spec.ts` -- covers LOST-01, LOST-02
- [ ] `src/estimate/test/mocks/` -- may need new mock generators for conversion scenarios

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | `JwtAuthGuard` on both new endpoints [VERIFIED: existing controller pattern] |
| V3 Session Management | no | JWT stateless auth, no sessions |
| V4 Access Control | yes | `EstimatePolicy.canUpdate()` via `EstimateTransitionService`; `QuotePolicy` via `QuoteCreator` [VERIFIED: codebase] |
| V5 Input Validation | yes | class-validator on `MarkEstimateLostRequest`; `Idempotency-Key` header validated as string [VERIFIED: existing patterns] |
| V6 Cryptography | no | No crypto operations in this phase |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Unauthorized conversion | Elevation | `JwtAuthGuard` + `EstimatePolicy` ownership check via `EstimateTransitionService` |
| Idempotency key replay | Tampering | `convertIdempotencyKey` stored on entity; same key returns same result (safe by design) |
| Rate-limited conversion spam | Denial of Service | Existing NestJS throttler on authenticated endpoints |
| Invalid status transition | Tampering | `ALLOWED_TRANSITIONS` map enforced by `EstimateTransitionService` |

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/estimate/` -- entity, DTO, controller, transition service, followup canceller interface
- `trade-flow-api/src/quote/` -- creator service, entity, DTO, response
- `trade-flow-ui/src/features/estimates/` -- action strip, RTK Query API
- `trade-flow-ui/src/types/` -- Estimate and Quote frontend types
- `.planning/phases/47-convert-to-quote-mark-as-lost/47-CONTEXT.md` -- all implementation decisions
- `.planning/phases/47-convert-to-quote-mark-as-lost/47-UI-SPEC.md` -- visual and interaction contracts

### Secondary (MEDIUM confidence)
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` -- entity field inventory, transition map design
- `.planning/phases/42-revisions/42-CONTEXT.md` -- followup canceller interface, revision chain semantics
- `.planning/phases/45-public-customer-page-response-handling/45-CONTEXT.md` -- decline reason taxonomy
- `.planning/phases/46-follow-up-queue-automation/46-CONTEXT.md` -- cancelAllFollowups vs cancelAllFollowupsForChain

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new libraries, all existing patterns [VERIFIED: codebase]
- Architecture: HIGH -- direct extension of existing estimate module patterns [VERIFIED: codebase]
- Pitfalls: HIGH -- identified through concrete codebase analysis [VERIFIED: source files]
- Entity schema gaps: HIGH -- verified field presence/absence against actual source files

**Research date:** 2026-04-13
**Valid until:** 2026-05-13 (stable -- all patterns are established project conventions)
