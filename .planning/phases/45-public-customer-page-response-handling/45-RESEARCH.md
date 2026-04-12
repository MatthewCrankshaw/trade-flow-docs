# Phase 45: Public Customer Page & Response Handling - Research

**Researched:** 2026-04-12
**Domain:** Public-facing estimate page (NestJS API + React UI), customer response handling, notification emails
**Confidence:** HIGH

## Summary

Phase 45 builds the customer-facing estimate page and response handling flow, mirroring the existing public quote infrastructure from v1.3 with estimate-specific adaptations. The codebase already has a complete, proven pattern: `PublicQuoteController` + `PublicQuoteRetriever` + `QuoteResponseHandler` on the API side, and `PublicQuotePage` + `PublicQuoteCard` + `PublicQuoteResponseButtons` on the UI side. Phase 45 adapts this pattern for estimates with three key differences: (1) the 3-action conversational CTA model replacing the binary accept/decline, (2) latest-revision resolution via `rootEstimateId` + `isCurrent: true` instead of direct document lookup, and (3) price range display with uncertainty notes instead of fixed totals with line items.

The estimate entity (Phase 41) and its enums already exist in the codebase. Critically, the existing `EstimateStatus` enum and `EstimateResponseType` enum still carry Phase 41's original 4-button values (`SITE_VISIT_REQUESTED`, `SITE_VISIT_REQUEST`, `SEND_ME_QUOTE`, `QUESTION`) which must be updated to the 3-action model per the CONTEXT.md decisions (D-API-05, D-CTA-01). The `ALLOWED_TRANSITIONS` map also includes `SITE_VISIT_REQUESTED` state which must be removed.

**Primary recommendation:** Mirror the public quote stack file-for-file, adapting for estimate-specific fields (price range, uncertainty notes, response array, revision resolution). Update the Phase 41 enums and transition map first, then build the new public estimate stack.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-PAGE-01:** 1:1 mirror of PublicQuotePage layout with estimate-specific additions.
**D-PAGE-02:** Non-binding disclaimer as bordered card with pale warning/info background, below business header, before estimate content. Verbatim copy: "This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit."
**D-PAGE-03:** Price displayed following displayMode from API. Range: "X - Y". From: "From X". Uses formatRange helper from Phase 43.
**D-PAGE-04:** Contingency percentage NEVER shown to customer. "Why is this a range?" section renders uncertainty notes as human-readable sentence. Hidden if no notes.
**D-PAGE-05:** Customer sees scope/description, price range, validity, trader info, disclaimer. NO line item breakdown.
**D-PAGE-06:** Zero non-essential cookies (PECR).

**D-CTA-01:** Three conversational actions replace original four buttons.
**D-CTA-02:** Primary: "Happy to Proceed" -- full-width primary button. Status -> RESPONDED.
**D-CTA-03:** Secondary: "Message [First Name]" -- outline button, inline textarea. Status -> RESPONDED.
**D-CTA-04:** Tertiary: "Not right for me" -- text link, inline expand with structured reasons + freeform. Status -> DECLINED.
**D-CTA-05:** Inline expansions push other actions down. No modals.
**D-CTA-06:** After response: inline success state replaces action area. No modal/redirect.
**D-CTA-07:** One response only. Revisit shows terminal view.

**D-TERM-01:** Customer-triggered terminal: content visible, response summary shown.
**D-TERM-02:** Trader-triggered terminal (Converted/Expired/Lost): generic "no longer available" message with contact details. Content NOT shown.
**D-TERM-03:** Terminal detection based on estimate status and response array.

**D-API-01:** Files in src/estimate/ module: PublicEstimateController, PublicEstimateRetriever, EstimateResponseHandler.
**D-API-02:** GET /v1/public/estimate/:token -- DocumentSessionAuthGuard, resolve rootEstimateId -> isCurrent:true, set firstViewedAt, trigger Sent->Viewed.
**D-API-03:** POST /v1/public/estimate/:token/respond -- type-discriminated body: proceed | message | decline.
**D-API-04:** Response stored as array on entity: responses: IEstimateResponse[]. Array for future-proofing; max one entry with current UX.
**D-API-05:** Drop SITE_VISIT_REQUESTED from EstimateStatus. Response statuses: RESPONDED and DECLINED only.
**D-API-06:** Customer-safe response filtering via dedicated public response class.

**D-NOTIFY-01:** Single Maizzle template estimate-response.html with conditional sections.
**D-NOTIFY-02:** Subject line includes response type. Body includes customer name, reference, type, message/reason, deep link.
**D-NOTIFY-03:** Failure-tolerant: email failure does not block response persistence.

### Claude's Discretion

- Visual hierarchy and spacing of customer page sections
- Exact icon choices for response buttons (Lucide)
- Exact wording of "Why is this a range?" heading
- Loading skeleton/spinner pattern
- Error state presentation
- Exact responsive breakpoints

### Deferred Ideas (OUT OF SCOPE)

- Quote CTA Overhaul (future phase)
- Optional E-Signature Toggle (future phase)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CUST-01 | Customer views estimate via secure token link without login | Existing DocumentSessionAuthGuard pattern; new /estimate/:token route |
| CUST-02 | Customer page shows scope, price range, contingency explanation, validity, uncertainty notes, trader info | PublicEstimateRetriever returns computed fields; formatRange helper from Phase 43 |
| CUST-03 | Non-binding legal language prominently at top | Disclaimer card component, same copy as Phase 44 D-LGL-03 |
| CUST-04 | NO Accept button, NO signature, NO single fixed total | 3-action CTA model; price range display; no accept semantics |
| CUST-05 | firstViewedAt recorded, Sent->Viewed transition | DocumentSessionAuthGuard already sets firstViewedAt; PublicEstimateRetriever triggers transition |
| CUST-06 | Always resolves to latest revision via rootEstimateId | PublicEstimateRetriever queries {rootEstimateId, isCurrent: true} |
| CUST-07 | Zero non-essential cookies (PECR) | Separate publicEstimateApi RTK Query instance with no auth headers |
| RESP-01 | "Book a site visit" -> now "Message [Name]" per D-CTA-03 (requirements update needed) | Message CTA with freeform textarea |
| RESP-02 | "Send me a quote" -> now "Happy to Proceed" per D-CTA-02 (requirements update needed) | Proceed CTA, single button click |
| RESP-03 | "I have a question" -> now covered by "Message [Name]" per D-CTA-03 (requirements update needed) | Message CTA, same channel |
| RESP-04 | "Not right now" -> now "Not right for me" per D-CTA-04 (requirements update needed) | Decline CTA with structured reasons |
| RESP-05 | Trader receives notification email with response type and message | EstimateNotificationEmailRenderer + EstimateResponseHandler sends email |
| RESP-06 | Status transitions on response (RESPONDED / DECLINED) | EstimateTransitionService updated; SITE_VISIT_REQUESTED removed |
| RESP-07 | Terminal-state estimates show read-only message | Terminal state detection in PublicEstimateRetriever response shape |
</phase_requirements>

## Standard Stack

### Core (existing -- no new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Project standard [VERIFIED: codebase] |
| React | 19.x | UI framework | Project standard [VERIFIED: codebase] |
| RTK Query | 2.x | API data fetching | Project standard for public API slices [VERIFIED: publicQuoteApi.ts] |
| Maizzle | (existing) | Email template rendering | Proven in v1.3 notification emails [VERIFIED: notification-email-renderer.service.ts] |
| Luxon | 3.x | Date handling | Project standard for all date/time [VERIFIED: codebase] |
| class-validator | 0.14.x | Request DTO validation | Project standard [VERIFIED: codebase] |
| Lucide React | 0.563.x | Icons | Project standard [VERIFIED: package.json] |
| Sonner | 2.x | Toast notifications | Used in existing PublicQuoteResponseButtons [VERIFIED: codebase] |

### No new npm packages needed

Phase 45 uses only existing project dependencies. No `npm install` required.

## Architecture Patterns

### Recommended Project Structure

**API side (new files in existing estimate module):**
```
src/estimate/
  controllers/
    public-estimate.controller.ts     # NEW
  services/
    public-estimate-retriever.service.ts  # NEW
    estimate-response-handler.service.ts  # NEW
  responses/
    public-estimate.response.ts       # NEW
  requests/
    respond-to-estimate.request.ts    # NEW
  data-transfer-objects/
    estimate-response.dto.ts          # NEW (IEstimateResponseDto for array entries)
  test/
    controllers/
      public-estimate.controller.spec.ts  # NEW
    services/
      public-estimate-retriever.service.spec.ts  # NEW
      estimate-response-handler.service.spec.ts  # NEW
    mocks/
      (extend existing mock generators)

src/email/
  services/
    estimate-notification-email-renderer.service.ts  # NEW
  templates/
    estimate-response.html            # NEW Maizzle template
```

**API side (modified files):**
```
src/estimate/
  enums/
    estimate-status.enum.ts           # MODIFY: remove SITE_VISIT_REQUESTED
    estimate-response-type.enum.ts    # MODIFY: replace with proceed/message/decline
    estimate-transitions.ts           # MODIFY: remove SITE_VISIT_REQUESTED, simplify
    estimate-transitions.spec.ts      # MODIFY: update tests
  entities/
    estimate.entity.ts                # MODIFY: add responses array, remove siteVisitAvailability
  data-transfer-objects/
    estimate.dto.ts                   # MODIFY: add responses array
    estimate-response-summary.dto.ts  # MODIFY: remove siteVisitAvailability
  repositories/
    estimate.repository.ts            # MODIFY: add pushResponse method, update toDto

src/document-token/
  guards/
    document-session-auth.guard.ts    # MODIFY: handle estimate tokens in expired/revoked path
  document-token.module.ts            # MODIFY: register new controller/services
```

**UI side (new files):**
```
src/features/public-estimate/
  api/
    publicEstimateApi.ts              # NEW (separate RTK Query instance, no auth)
  components/
    PublicEstimateCard.tsx             # NEW
    PublicEstimateDisclaimer.tsx       # NEW
    PublicEstimatePriceRange.tsx       # NEW
    PublicEstimateUncertainty.tsx      # NEW
    PublicEstimateResponseButtons.tsx  # NEW (3-action model)
    PublicEstimateDeclineForm.tsx      # NEW
    PublicEstimateMessageForm.tsx      # NEW
    PublicEstimateTerminalState.tsx    # NEW
    PublicEstimateSkeleton.tsx         # NEW
    PublicEstimateError.tsx            # NEW
  types/
    public-estimate.types.ts          # NEW
  index.ts                            # NEW barrel export

src/pages/
  PublicEstimatePage.tsx              # NEW

src/App.tsx                          # MODIFY: add /estimate/:token route
src/store/store.ts                   # MODIFY: register publicEstimateApi reducer
```

### Pattern 1: Public Controller with Type-Discriminated Guard

**What:** Controller under `/v1/public/estimate/:token` guarded by `DocumentSessionAuthGuard`, asserting `documentType === "estimate"`.
**When to use:** All public estimate endpoints.
**Example:**
```typescript
// Source: existing PublicQuoteController pattern [VERIFIED: codebase]
@Controller("v1/public")
@UseGuards(ThrottlerGuard)
export class PublicEstimateController {
  @Throttle({ default: { limit: 60, ttl: 60000 } })
  @UseGuards(DocumentSessionAuthGuard)
  @Get("estimate/:token")
  public async findByToken(
    @Req() request: { documentToken: IDocumentTokenDto },
  ): Promise<IResponse<IPublicEstimateResponse>> {
    if (request.documentToken.documentType !== "estimate") {
      throw createHttpError(
        new ResourceNotFoundError(ErrorCodes.DOCUMENT_TOKEN_TYPE_MISMATCH, "Estimate not found"),
      );
    }
    try {
      const response = await this.publicEstimateRetriever.getPublicEstimate(
        request.documentToken,
      );
      return createResponse([response]);
    } catch (error) {
      throw createHttpError(error);
    }
  }
}
```

### Pattern 2: Latest-Revision Resolution

**What:** Public retriever resolves the token's `documentId` (which is `rootEstimateId`) to the current revision.
**When to use:** Every public estimate read path.
**Example:**
```typescript
// Source: Phase 42 CONTEXT.md revision chain pattern [VERIFIED: 42-CONTEXT.md]
// The token.documentId carries rootEstimateId (set by Phase 44 send flow)
const currentRevision = await this.estimateRepository.findOne({
  rootEstimateId: new ObjectId(token.documentId),
  isCurrent: true,
});
// If no rootEstimateId match, fall back to direct ID lookup
// (handles pre-revision estimates where rootEstimateId === self._id)
```

### Pattern 3: Type-Discriminated Response Endpoint

**What:** Single POST endpoint with `{ type: "proceed" | "message" | "decline" }` discriminator.
**When to use:** Customer response submission.
**Example:**
```typescript
// Source: CONTEXT.md D-API-03 [VERIFIED: 45-CONTEXT.md]
@Throttle({ default: { limit: 10, ttl: 60000 } })
@UseGuards(DocumentSessionAuthGuard)
@Post("estimate/:token/respond")
public async respondToEstimate(
  @Req() request: { documentToken: IDocumentTokenDto },
  @Body() body: RespondToEstimateRequest,
): Promise<IResponse<IPublicEstimateResponse>> {
  // Assert documentType === "estimate"
  // Delegate to EstimateResponseHandler
}
```

### Pattern 4: Failure-Tolerant Notification Email

**What:** Email send wrapped in try/catch; failure logged but never blocks response persistence.
**When to use:** All notification emails to trader after customer response.
**Example:**
```typescript
// Source: existing QuoteResponseHandler pattern [VERIFIED: codebase]
try {
  await this.sendNotificationEmail(estimate, responseType, message);
} catch (emailError) {
  this.logger.error("Failed to send response notification email", {
    estimateId: estimate.id,
    error: String(emailError),
  });
}
```

### Pattern 5: Separate Public RTK Query Instance

**What:** Dedicated `createApi` instance for public endpoints with no auth headers.
**When to use:** All public estimate API calls from the UI.
**Example:**
```typescript
// Source: existing publicQuoteApi.ts pattern [VERIFIED: codebase]
const publicBaseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:3000",
});

export const publicEstimateApi = createApi({
  reducerPath: "publicEstimateApi",
  baseQuery: publicBaseQuery,
  tagTypes: ["PublicEstimate"],
  endpoints: (builder) => ({ /* ... */ }),
});
```

### Pattern 6: Inline Response Expansion (Optimistic UI)

**What:** Response buttons use `useState` to track view state. After mutation success, `onSuccess` callback replaces buttons with confirmation.
**When to use:** All customer response interactions.
**Example:**
```typescript
// Source: existing PublicQuoteResponseButtons pattern [VERIFIED: codebase]
const [view, setView] = useState<ResponseView>("buttons");
// After successful mutation:
onSuccess(result);  // Parent replaces entire action area
```

### Anti-Patterns to Avoid

- **Reusing authenticated estimate API for public page:** The public page MUST use a separate RTK Query instance without auth headers. Leaking Firebase auth tokens to public endpoints is a security risk and breaks PECR compliance. [VERIFIED: publicQuoteApi.ts uses separate fetchBaseQuery]
- **Client-side contingency math:** The API returns pre-computed `low` and `high` values. The UI MUST NOT perform any contingency multiplication. [VERIFIED: D-CONT-05, D-PAGE-03]
- **Exposing line items on customer page:** The customer sees the total range only, not per-item breakdown. [VERIFIED: D-PAGE-05]
- **Showing contingency percentage:** Never expose the contingency number to customers. [VERIFIED: D-PAGE-04]
- **Multiple response endpoints:** Use a single POST with type discriminator, not separate accept/decline/message endpoints. [VERIFIED: D-API-03]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Price range formatting | Custom formatter | `formatRange` from Phase 43 `src/lib/currency.ts` | Already handles both display modes, currency formatting [VERIFIED: D-FMT-01 in 43-CONTEXT.md] |
| Email HTML rendering | String concatenation | Maizzle template + `NotificationEmailRenderer` pattern | Proven pattern, handles Tailwind CSS inlining [VERIFIED: codebase] |
| Token validation | Custom middleware | `DocumentSessionAuthGuard` | Already handles expiry, revocation, first-view tracking [VERIFIED: codebase] |
| Status transition validation | Manual if/else chains | `EstimateTransitionService` + `ALLOWED_TRANSITIONS` map | Type-safe, centralized, testable [VERIFIED: codebase] |
| Toast notifications | Custom alert system | Sonner `toast.error()` | Already used in quote response buttons [VERIFIED: codebase] |

## Common Pitfalls

### Pitfall 1: DocumentSessionAuthGuard Quote-Specific Business Name Lookup

**What goes wrong:** The existing `DocumentSessionAuthGuard` (line 34-38) looks up business name for expired/revoked tokens using `quoteRepository.findByIdOrFail(tokenDto.documentId)`. For estimate tokens, `documentId` is the `rootEstimateId`, and the lookup needs `estimateRepository` instead.
**Why it happens:** The guard was built for quotes only in v1.3, and Phase 41's document-token rename didn't update the expired/revoked fallback path.
**How to avoid:** Modify the guard to branch on `tokenDto.documentType`: use `quoteRepository` for quotes, `estimateRepository` for estimates. The guard already has access to `documentType` on the token DTO.
**Warning signs:** Expired/revoked estimate tokens returning empty business name in error response, or throwing 500 errors if `quoteRepository.findByIdOrFail` throws ResourceNotFoundError for estimate IDs.

### Pitfall 2: Entity Schema Change -- responses Array vs Flat Fields

**What goes wrong:** Phase 41 reserved flat fields (`lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason`, `siteVisitAvailability`) on the entity. CONTEXT.md D-API-04 mandates an array `responses: IEstimateResponse[]`. These two designs conflict.
**Why it happens:** Phase 41 was designed before the D-CTA discussion simplified the model.
**How to avoid:** Phase 45 must add the `responses` array to the entity AND update the flat summary fields for backward compatibility with the detail view (Phase 41's `responseSummary` on the DTO). The flat fields (`lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason`) serve as a quick-access summary; the array is the source of truth. Remove `siteVisitAvailability` since site visit requests are now just messages.
**Warning signs:** The existing `IEstimateResponseSummaryDto` references `siteVisitAvailability` which no longer has a use case.

### Pitfall 3: Enum Updates Breaking Existing Tests

**What goes wrong:** Removing `SITE_VISIT_REQUESTED` from `EstimateStatus` and updating `EstimateResponseType` will break existing Phase 41 tests that reference these values.
**Why it happens:** Phase 41 shipped with the original 4-button model values.
**How to avoid:** Update enums first, then fix all compile errors and test failures before building new code. The transition map test file (`estimate-transitions.spec.ts`) will need updating.
**Warning signs:** TypeScript compile errors in estimate module after enum changes.

### Pitfall 4: Revision Resolution Edge Case -- rootEstimateId is Self

**What goes wrong:** Phase 42 retrofitted `rootEstimateId = self._id` on root estimates. But if Phase 41 originally set `rootEstimateId: null`, some estimates may have null `rootEstimateId`.
**Why it happens:** Phase 42's backfill may or may not have run depending on execution order.
**How to avoid:** `PublicEstimateRetriever` should handle both: first try `{rootEstimateId: documentId, isCurrent: true}`, then fallback to direct `{_id: documentId}` if no match. This handles both pre-revision and post-revision estimates.
**Warning signs:** Public page returning 404 for estimates that were sent before Phase 42 revisions were implemented.

### Pitfall 5: EstimateTransitionService Requires authUser for Public Transitions

**What goes wrong:** The existing `EstimateTransitionService.transition()` requires `authUser: IUserDto` and performs policy checks via `AccessControllerFactory`. Public endpoints have no authenticated user.
**Why it happens:** The transition service was designed for authenticated internal use.
**How to avoid:** Add a `publicTransition(estimateId, targetStatus)` method to `EstimateTransitionService` (mirroring the existing `QuoteTransitionService.publicTransition` pattern). This method skips policy checks since token authentication already validated access. [VERIFIED: QuoteResponseHandler calls quoteTransitionService.publicTransition]
**Warning signs:** 403 errors when customer views or responds to an estimate.

### Pitfall 6: Respondable State Validation

**What goes wrong:** Customer submits a response to an estimate that's already in a terminal state (CONVERTED, DECLINED, EXPIRED, LOST, DELETED).
**Why it happens:** Race condition between trader actions and customer response, or customer uses cached page.
**How to avoid:** `EstimateResponseHandler` must validate that the estimate is in a respondable state (`VIEWED` or `SENT`) before processing. Return a user-friendly error (not a 500) that the page can display gracefully.
**Warning signs:** Duplicate responses or responses on terminal estimates.

## Code Examples

### Public Estimate Response Interface

```typescript
// Source: derived from existing IPublicQuoteResponse + CONTEXT.md decisions [VERIFIED: codebase + D-API-06]
export interface IPublicEstimateResponse {
  businessName: string;
  businessPhone?: string;
  businessEmail?: string;
  customerFirstName: string;
  traderFirstName: string;
  estimateNumber: string;
  estimateDate: string;
  validUntil?: string;
  scope?: string;
  notes?: string;
  status: string;
  displayMode: string;
  priceRange: {
    low: { inclTax: number };
    high: { inclTax: number };
  };
  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;
  responses: IPublicEstimateResponseEntry[];
  respondable: boolean;
  terminalState: "customer_responded" | "trader_closed" | null;
}

export interface IPublicEstimateResponseEntry {
  type: string;
  respondedAt: string;
  message?: string;
  reason?: string;
}
```

### Request DTO for Respond Endpoint

```typescript
// Source: CONTEXT.md D-API-03 [VERIFIED: 45-CONTEXT.md]
import { IsEnum, IsOptional, IsString, MaxLength, ValidateIf } from "class-validator";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";

export class RespondToEstimateRequest {
  @IsEnum(EstimateResponseType)
  type: EstimateResponseType;

  @ValidateIf((o) => o.type === EstimateResponseType.MESSAGE)
  @IsString()
  @MaxLength(2000)
  message?: string;

  @ValidateIf((o) => o.type === EstimateResponseType.DECLINE)
  @IsOptional()
  @IsEnum(EstimateDeclineReason)
  reason?: EstimateDeclineReason;

  @ValidateIf((o) => o.type === EstimateResponseType.DECLINE)
  @IsOptional()
  @IsString()
  @MaxLength(500)
  declineMessage?: string;
}
```

### Updated EstimateResponseType Enum

```typescript
// Source: CONTEXT.md D-CTA-01 [VERIFIED: 45-CONTEXT.md]
// Replaces the Phase 41 original 4-type enum
export enum EstimateResponseType {
  PROCEED = "proceed",
  MESSAGE = "message",
  DECLINE = "decline",
}
```

### Updated EstimateStatus Enum

```typescript
// Source: CONTEXT.md D-API-05 [VERIFIED: 45-CONTEXT.md]
// SITE_VISIT_REQUESTED removed
export enum EstimateStatus {
  DRAFT = "draft",
  SENT = "sent",
  VIEWED = "viewed",
  RESPONDED = "responded",
  CONVERTED = "converted",
  DECLINED = "declined",
  EXPIRED = "expired",
  LOST = "lost",
  DELETED = "deleted",
}
```

### Response Entity Addition

```typescript
// Source: CONTEXT.md D-API-04 [VERIFIED: 45-CONTEXT.md]
// Added to IEstimateEntity
export interface IEstimateResponseEntity {
  type: string;
  message?: string;
  reason?: string;
  respondedAt: Date;
}

// On IEstimateEntity:
responses?: IEstimateResponseEntity[];
```

### UI Response View State Machine

```typescript
// Source: adapted from PublicQuoteResponseButtons [VERIFIED: codebase]
type ResponseView = "buttons" | "message-form" | "decline-form" | "success";
const [view, setView] = useState<ResponseView>("buttons");
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| 4-button CTA (site visit, send quote, question, decline) | 3-action conversational CTA (proceed, message, decline) | Phase 45 design (2026-04-12) | Simpler UX, fewer endpoints, cleaner status model |
| SITE_VISIT_REQUESTED status | Removed; site visits captured as messages | Phase 45 design | Enum/transition simplification |
| Flat response fields on entity | responses[] array on entity + flat summary fields | Phase 45 design | Future-proofs for multi-response evolution |
| Separate accept/decline endpoints | Single /respond endpoint with type discriminator | Phase 45 design | Cleaner API surface |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 42 backfill sets rootEstimateId = self._id on root estimates | Pitfall 4 | PublicEstimateRetriever fallback handles this; low risk |
| A2 | The guard's expired/revoked path can be extended with estimate repository injection without circular dependency | Pitfall 1 | May need to move lookup logic to a shared service; medium risk |
| A3 | Existing estimate entity flat fields (lastResponseType etc.) should be kept alongside the responses array for summary access | Pitfall 2 | Could simplify to array-only if flat fields are unused elsewhere; low risk |

## Open Questions

1. **DocumentSessionAuthGuard circular dependency risk**
   - What we know: The guard lives in `document-token` module and currently imports `QuoteRepository`. Adding `EstimateRepository` may create a circular dependency if `estimate.module` imports `document-token.module`.
   - What's unclear: Whether `EstimateModule` already imports `DocumentTokenModule` (for the send flow in Phase 44).
   - Recommendation: Check module imports at plan time. If circular, extract the business-name-lookup into a small service that both repositories inject into, or use `forwardRef()`.

2. **PublicEstimateController registration location**
   - What we know: D-API-01 says "within the existing EstimateModule." The existing `PublicQuoteController` is registered in `DocumentTokenModule`.
   - What's unclear: Whether the estimate module should own its public controller or follow the quote pattern of having it in the token module.
   - Recommendation: Follow D-API-01 and register in `EstimateModule`. This is a cleaner separation than the quote pattern where public and token concerns are mixed.

3. **REQUIREMENTS.md RESP-01..04 updates**
   - What we know: CONTEXT.md explicitly states these requirements need rewriting to reflect the 3-action model.
   - What's unclear: Whether the planner should include doc updates as a separate wave/task or fold them into the enum update task.
   - Recommendation: Include as the first task in the plan since subsequent tasks reference the new model.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 with ts-jest |
| Framework (UI) | Vitest 4.1.3 |
| Config file (API) | `trade-flow-api/jest.config.ts` (or `package.json` jest key) |
| Config file (UI) | `trade-flow-ui/vitest.config.ts` |
| Quick run command (API) | `cd trade-flow-api && npx jest --testPathPattern=estimate --no-coverage` |
| Full suite command (API) | `cd trade-flow-api && npm run ci` |
| Quick run command (UI) | `cd trade-flow-ui && npx vitest run --reporter=verbose src/features/public-estimate` |
| Full suite command (UI) | `cd trade-flow-ui && npm run ci` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CUST-01 | Token resolves to estimate | unit | `npx jest public-estimate-retriever.service.spec.ts -t "returns estimate"` | Wave 0 |
| CUST-02 | Response includes all display fields | unit | `npx jest public-estimate-retriever.service.spec.ts -t "includes fields"` | Wave 0 |
| CUST-05 | firstViewedAt set + Sent->Viewed | unit | `npx jest public-estimate-retriever.service.spec.ts -t "firstViewedAt"` | Wave 0 |
| CUST-06 | Latest revision resolved | unit | `npx jest public-estimate-retriever.service.spec.ts -t "latest revision"` | Wave 0 |
| RESP-01/02/03/04 | Response handling per type | unit | `npx jest estimate-response-handler.service.spec.ts` | Wave 0 |
| RESP-05 | Notification email sent | unit | `npx jest estimate-response-handler.service.spec.ts -t "notification"` | Wave 0 |
| RESP-06 | Status transitions correctly | unit | `npx jest estimate-response-handler.service.spec.ts -t "transition"` | Wave 0 |
| RESP-07 | Terminal state returns read-only | unit | `npx jest public-estimate-retriever.service.spec.ts -t "terminal"` | Wave 0 |

### Sampling Rate

- **Per task commit:** Quick run command for changed module
- **Per wave merge:** Full CI in both repos (`npm run ci`)
- **Phase gate:** Full suite green in both repos before `/gsd-verify-work`

### Wave 0 Gaps

- [ ] `src/estimate/test/services/public-estimate-retriever.service.spec.ts` -- covers CUST-01, CUST-02, CUST-05, CUST-06, RESP-07
- [ ] `src/estimate/test/services/estimate-response-handler.service.spec.ts` -- covers RESP-01..06
- [ ] `src/estimate/test/controllers/public-estimate.controller.spec.ts` -- covers controller routing + type assertion
- [ ] `src/email/test/services/estimate-notification-email-renderer.service.spec.ts` -- covers template rendering
- [ ] Update `src/estimate/enums/estimate-transitions.spec.ts` -- covers updated transition map

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | DocumentSessionAuthGuard token-based (no login required) |
| V3 Session Management | no | Token-based, no server sessions |
| V4 Access Control | yes | documentType assertion prevents token-type confusion; respondable state validation |
| V5 Input Validation | yes | class-validator on RespondToEstimateRequest; MaxLength on message/reason fields |
| V6 Cryptography | no | Tokens already generated via crypto.randomBytes in Phase 41 |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Token-type confusion (quote token used on estimate endpoint) | Spoofing | documentType assertion in controller [VERIFIED: existing pattern] |
| Response replay (submitting same response twice) | Tampering | Respondable state check (VIEWED/SENT only); idempotent array push |
| Rate limiting bypass on response endpoint | Denial of Service | ThrottlerGuard with 10/min limit [VERIFIED: existing pattern] |
| XSS via customer message in notification email | Tampering | escapeHtml() on all user-provided content before email rendering [VERIFIED: existing pattern] |
| PECR violation via third-party scripts | Information Disclosure | Zero non-essential cookies; separate RTK Query instance with no auth [VERIFIED: D-PAGE-06] |
| Expired/revoked token accessing estimate data | Spoofing | DocumentSessionAuthGuard checks expiry/revocation before controller [VERIFIED: existing pattern] |

## Sources

### Primary (HIGH confidence)

- `trade-flow-api/src/document-token/controllers/public-quote.controller.ts` -- public controller pattern
- `trade-flow-api/src/document-token/services/public-quote-retriever.service.ts` -- public retriever pattern
- `trade-flow-api/src/document-token/services/quote-response-handler.service.ts` -- response handler pattern with failure-tolerant email
- `trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` -- guard with firstViewedAt tracking
- `trade-flow-api/src/email/services/notification-email-renderer.service.ts` -- Maizzle rendering pattern
- `trade-flow-ui/src/features/public-quote/` -- complete public quote UI feature module
- `trade-flow-api/src/estimate/` -- existing entity, DTO, enum, transition, repository
- `.planning/phases/45-public-customer-page-response-handling/45-CONTEXT.md` -- all locked decisions
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` -- entity shape, enum values, token design
- `.planning/phases/42-revisions/42-CONTEXT.md` -- revision chain resolution
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` -- email rendering, token reuse, legal copy
- `.planning/RETROSPECTIVE.md` -- v1.3 lessons on token-based public access, failure-tolerant emails

### Secondary (MEDIUM confidence)

- `.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md` -- formatRange helper, uncertainty chips, features/estimates scaffolding

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in use, zero new dependencies
- Architecture: HIGH -- direct mirror of proven v1.3 public quote pattern with well-documented adaptations
- Pitfalls: HIGH -- verified against actual codebase; guard limitation and enum conflicts are concrete findings
- Security: HIGH -- mirrors existing threat model from v1.3 with same mitigations

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- no external dependency changes)
