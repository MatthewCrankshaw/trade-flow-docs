# Phase 50: Response Display & Convert Route Fix - Patterns

**Extracted:** 2026-04-15

## Files to Modify

| # | File | Role | Data Flow Position |
|---|------|------|--------------------|
| 1 | `trade-flow-api/src/estimate/repositories/estimate.repository.ts` | Repository | Entity -> DTO conversion (toDto method) |
| 2 | `trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` | Test | Unit tests for toDto derivation |
| 3 | `trade-flow-ui/src/types/estimate.ts` | Type definition | Frontend API contract types |
| 4 | `trade-flow-ui/src/pages/EstimateDetailPage.tsx` | Page component | Renders estimate detail with response card |
| 5 | `trade-flow-ui/src/features/estimates/components/index.ts` | Barrel export | Feature component re-exports |
| 6 | `trade-flow-ui/src/App.tsx` | Route config | Top-level route table |

## File 1: `trade-flow-api/src/estimate/repositories/estimate.repository.ts`

**Role:** Repository layer -- converts MongoDB entities to DTOs via `toDto()`.

**What changes:** Line 338 currently hardcodes `responseSummary: null`. Must derive from the `responses[]` array already mapped on lines 285-290.

**Closest analog:** The `responses` mapping on lines 285-290 of the same method, which already iterates `entity.responses` and maps entity fields to DTO fields.

### Current code (lines 283-339)

```typescript
public toDto(entity: IEstimateEntity, lineItems?: DtoCollection<IEstimateLineItemDto>): IEstimateDto {
  const zeroMoney = Money.zero(DEFAULT_CURRENCY);
  const responses: IEstimateResponseDto[] = (entity.responses ?? []).map((r) => ({
    type: r.type,
    message: r.message ?? null,
    reason: r.reason ?? null,
    respondedAt: r.respondedAt.toISOString(),
  }));

  return {
    // ...all other fields...
    responses,
    // ...
    responseSummary: null,  // <-- BUG: always null
  };
}
```

### Target DTO interface (`estimate-response-summary.dto.ts`)

```typescript
export interface IEstimateResponseSummaryDto {
  lastResponseType: string | null;
  lastResponseAt: string | null;
  lastResponseMessage: string | null;
  declineReason: string | null;
}
```

### Source entity interface (`estimate.entity.ts` lines 6-11)

```typescript
export interface IEstimateResponseEntity {
  type: string;
  message?: string | null;
  reason?: string | null;
  respondedAt: Date;
}
```

### Field mapping (entity response -> DTO summary)

| Entity field (`IEstimateResponseEntity`) | DTO field (`IEstimateResponseSummaryDto`) |
|------------------------------------------|-------------------------------------------|
| `type` | `lastResponseType` |
| `respondedAt` (Date -> ISO string) | `lastResponseAt` |
| `message` | `lastResponseMessage` |
| `reason` | `declineReason` |

### Pattern: derive from already-mapped `responses` array

The `responses` local variable (line 285) already contains the mapped `IEstimateResponseDto[]`. Use the last element to build the summary. This avoids re-mapping from entity and keeps the derivation consistent with the existing responses mapping.

```typescript
// responses is already mapped above (line 285)
const lastResponse = responses.length > 0 ? responses[responses.length - 1] : null;
const responseSummary: IEstimateResponseSummaryDto | null = lastResponse
  ? {
      lastResponseType: lastResponse.type,
      lastResponseAt: lastResponse.respondedAt,
      lastResponseMessage: lastResponse.message,
      declineReason: lastResponse.reason,
    }
  : null;
```

The `IEstimateResponseSummaryDto` import already exists in the file's imports (via `estimate.dto.ts` which imports it on line 7 of that file). Verify the import is available or add:

```typescript
import { IEstimateResponseSummaryDto } from "@estimate/data-transfer-objects/estimate-response-summary.dto";
```

## File 2: `trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts`

**Role:** Unit tests for EstimateRepository.

**Closest analog:** The existing `toEntity / toDto round-trip` describe block (line 107) which already tests response round-tripping.

### Existing test setup pattern (lines 1-55)

```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { EstimateRepository } from "@estimate/repositories/estimate.repository";
import { EstimateMockGenerator } from "@estimate-test/mocks/estimate-mock-generator";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";

// Test module setup with mocked MongoDbFetcher, MongoDbWriter, MongoConnectionService
// afterEach: jest.clearAllMocks()
```

### Existing round-trip test pattern (lines 107-144)

```typescript
describe("toEntity / toDto round-trip", () => {
  it("round-trips all Phase 41 and reserved fields through toEntity / toDto", () => {
    const responseEntry = {
      type: EstimateResponseType.DECLINE,
      message: "too expensive",
      reason: EstimateDeclineReason.TOO_EXPENSIVE,
      respondedAt: new Date().toISOString(),
    };
    const dto = EstimateMockGenerator.createEstimateDto({
      uncertaintyReasons: ["site_inspection", "materials"],
      uncertaintyNotes: "loft insulation unknown",
      responses: [responseEntry],
      sentAt: DateTime.now(),
      respondedAt: DateTime.now(),
      convertedToQuoteId: new ObjectId().toString(),
    });
    const entity = repo.toEntity(dto);
    const roundTripped = repo.toDto(entity);

    expect(roundTripped.uncertaintyReasons).toEqual(["site_inspection", "materials"]);
    expect(roundTripped.responses).toHaveLength(1);
    expect(roundTripped.responses?.[0].type).toBe(EstimateResponseType.DECLINE);
  });
});
```

### Mock generator (`estimate-mock-generator.ts`)

```typescript
// Entity with responses:
EstimateMockGenerator.createEstimateEntity({
  responses: [
    {
      type: EstimateResponseType.PROCEED,
      message: null,
      reason: null,
      respondedAt: new Date(),
    },
  ],
});

// Individual response:
EstimateMockGenerator.generateEstimateResponse({
  type: EstimateResponseType.DECLINE,
  message: "too expensive",
  reason: EstimateDeclineReason.TOO_EXPENSIVE,
});
```

### New test cases to add (inside existing `toEntity / toDto round-trip` describe)

Test cases needed:
1. `toDto` derives `responseSummary` from last response when responses exist
2. `toDto` returns `responseSummary: null` when responses array is empty
3. `toDto` uses the LAST response when multiple responses exist
4. `responseSummary` fields map correctly (type -> lastResponseType, etc.)

Pattern for assertions:

```typescript
it("derives responseSummary from last response in responses array", () => {
  const entity = EstimateMockGenerator.createEstimateEntity({
    responses: [
      EstimateMockGenerator.generateEstimateResponse({
        type: EstimateResponseType.DECLINE,
        message: "too expensive",
        reason: EstimateDeclineReason.TOO_EXPENSIVE,
      }),
    ],
  });
  const dto = repo.toDto(entity);
  expect(dto.responseSummary).not.toBeNull();
  expect(dto.responseSummary!.lastResponseType).toBe(EstimateResponseType.DECLINE);
  expect(dto.responseSummary!.lastResponseMessage).toBe("too expensive");
  expect(dto.responseSummary!.declineReason).toBe(EstimateDeclineReason.TOO_EXPENSIVE);
  expect(dto.responseSummary!.lastResponseAt).toBeDefined();
});

it("returns null responseSummary when responses array is empty", () => {
  const entity = EstimateMockGenerator.createEstimateEntity({ responses: [] });
  const dto = repo.toDto(entity);
  expect(dto.responseSummary).toBeNull();
});

it("uses last response when multiple responses exist", () => {
  const entity = EstimateMockGenerator.createEstimateEntity({
    responses: [
      EstimateMockGenerator.generateEstimateResponse({ type: EstimateResponseType.MESSAGE }),
      EstimateMockGenerator.generateEstimateResponse({ type: EstimateResponseType.DECLINE }),
    ],
  });
  const dto = repo.toDto(entity);
  expect(dto.responseSummary!.lastResponseType).toBe(EstimateResponseType.DECLINE);
});
```

## File 3: `trade-flow-ui/src/types/estimate.ts`

**Role:** Frontend TypeScript types matching API response shapes.

**What changes:** `EstimateResponseSummary` interface (lines 51-56) has mismatched field names.

### Current (WRONG)

```typescript
export interface EstimateResponseSummary {
  responseType: string;
  respondedAt: string;
  message?: string;
  declineReason?: string;
}
```

### Target (matches `IEstimateResponseSummaryResponse` from API)

```typescript
export interface EstimateResponseSummary {
  lastResponseType: string | null;
  lastResponseAt: string | null;
  lastResponseMessage: string | null;
  declineReason: string | null;
}
```

### Additional cleanup: Remove dead `site_visit_requested` status (line 8)

The backend `EstimateStatus` enum (at `trade-flow-api/src/estimate/enums/estimate-status.enum.ts`) does NOT include `site_visit_requested`. The frontend type should match:

```typescript
// REMOVE "site_visit_requested" from this union:
export type EstimateStatus =
  | "draft"
  | "sent"
  | "viewed"
  | "responded"
  // | "site_visit_requested"  <-- remove this line
  | "converted"
  | "declined"
  | "expired"
  | "lost"
  | "deleted";
```

This also requires removing `site_visit_requested` from `statusColors` in `EstimateDetailPage.tsx` (line 48) and from `canResend` in `EstimateActionStrip.tsx` (line 50).

## File 4: `trade-flow-ui/src/pages/EstimateDetailPage.tsx`

**Role:** Page component rendering estimate detail view.

**What changes:** Replace placeholder card (lines 277-285) with actual response summary display.

### Closest analog: `EstimateLostReasonCard` component

Location: `trade-flow-ui/src/features/estimates/components/EstimateLostReasonCard.tsx`

This is the best analog because:
- It is a status-conditional info card rendered inside EstimateDetailPage
- It uses a `Record<string, string>` for label mapping (`LOST_REASON_LABELS`)
- It conditionally renders fields
- It uses `formatDate` from `@/lib/date-helpers`
- It is exported from the features barrel and imported in the page

```typescript
import { AlertTriangle } from "lucide-react";
import { formatDate } from "@/lib/date-helpers";

const LOST_REASON_LABELS: Record<string, string> = {
  too_expensive: "Too expensive",
  going_with_someone_else: "Going with someone else",
  decided_not_to_do_work: "Decided not to do the work",
  just_getting_idea: "Just getting an idea of costs",
  timing_not_right: "Timing isn't right",
};

interface EstimateLostReasonCardProps {
  reason?: string;
  notes?: string;
  lostAt?: string;
}

export function EstimateLostReasonCard({ reason, notes, lostAt }: EstimateLostReasonCardProps) {
  return (
    <div className="border border-warning-border bg-warning rounded-lg p-4">
      <div className="flex items-start gap-3">
        <AlertTriangle className="h-4 w-4 text-warning-icon shrink-0 mt-0.5" />
        <div className="flex flex-col gap-1">
          <span className="text-sm font-semibold text-warning-foreground">Lost</span>
          {reason && <span className="text-sm text-warning-foreground">Reason: {LOST_REASON_LABELS[reason] ?? reason}</span>}
          {notes && <span className="text-sm text-warning-foreground italic">&ldquo;{notes}&rdquo;</span>}
          {lostAt && <span className="text-xs text-muted-foreground">Marked as lost on {formatDate(lostAt)}</span>}
        </div>
      </div>
    </div>
  );
}
```

### Pattern to follow for new `EstimateResponseCard` component

Create at: `trade-flow-ui/src/features/estimates/components/EstimateResponseCard.tsx`

Follow `EstimateLostReasonCard` patterns:
- Named export function component
- Props interface with fields from `EstimateResponseSummary`
- `Record<string, string>` for response type labels
- `Record<string, string>` for decline reason labels (reuse same values as `LOST_REASON_LABELS` in `EstimateLostReasonCard`)
- Conditional field rendering
- `formatDateTime` from `@/lib/date-helpers` for timestamp (use DateTime variant since responses have time precision)
- Icon from `lucide-react`
- Tailwind styling with semantic tokens

### Response type labels (from Phase 45 CTA text)

```typescript
const RESPONSE_TYPE_LABELS: Record<string, string> = {
  proceed: "Happy to Proceed",
  message: "Customer Message",
  decline: "Declined",
};
```

### Decline reason labels (same taxonomy as MarkAsLostDialog)

```typescript
const DECLINE_REASON_LABELS: Record<string, string> = {
  too_expensive: "Too expensive",
  going_with_someone_else: "Going with someone else",
  decided_not_to_do_work: "Decided not to do the work",
  just_getting_idea: "Just getting an idea of costs",
  timing_not_right: "Timing isn't right",
};
```

### Card rendering pattern in EstimateDetailPage

The placeholder card at lines 277-285 should be replaced. Follow the `EstimateLostReasonCard` integration pattern from line 253:

```tsx
{/* Lost reason card -- conditional on status */}
{estimate.status === "lost" && (
  <EstimateLostReasonCard reason={estimate.lostReason} notes={estimate.lostNotes} lostAt={estimate.lostAt} />
)}
```

The response card should use a Card wrapper (like the placeholder does) but render the `EstimateResponseCard` inside:

```tsx
<Card>
  <CardHeader>
    <CardTitle className="text-base">Customer Response</CardTitle>
  </CardHeader>
  <CardContent>
    {estimate.responseSummary ? (
      <EstimateResponseCard responseSummary={estimate.responseSummary} />
    ) : (
      <p className="text-sm text-muted-foreground">No customer response yet.</p>
    )}
  </CardContent>
</Card>
```

### Status color cleanup

Remove `site_visit_requested` from `statusColors` (line 48):

```typescript
// Current (line 48):
site_visit_requested: "default",
// DELETE this line
```

### Import additions needed

```typescript
import { EstimateResponseCard } from "@/features/estimates/components";
```

## File 5: `trade-flow-ui/src/features/estimates/components/index.ts`

**Role:** Barrel export for estimate feature components.

**What changes:** Add export for new `EstimateResponseCard` component.

### Current pattern (full file)

```typescript
export * from "./ContingencySlider";
export * from "./UncertaintyChipGroup";
// ...18 total exports...
export * from "./EstimateConvertedLink";
```

### Addition

```typescript
export * from "./EstimateResponseCard";
```

## File 6: `trade-flow-ui/src/App.tsx`

**Role:** Top-level React Router route configuration.

**What changes:** Add `/quotes/:quoteId/edit` route pointing to `QuoteDetailPage`.

### Closest analog: existing quote routes (lines 106-107)

```tsx
<Route path="/quotes" element={<QuotesPage />} />
<Route path="/quotes/:quoteId" element={<QuoteDetailPage />} />
```

### Route addition pattern

Add as sibling immediately after the existing `/quotes/:quoteId` route:

```tsx
<Route path="/quotes" element={<QuotesPage />} />
<Route path="/quotes/:quoteId" element={<QuoteDetailPage />} />
<Route path="/quotes/:quoteId/edit" element={<QuoteDetailPage />} />
```

No new imports needed -- `QuoteDetailPage` is already imported (line 23).

### Route nesting context

The new route must be inside the same guard hierarchy as the existing quote routes:

```tsx
<Route element={<AuthenticatedLayout />}>     {/* auth guard */}
  <Route element={<OnboardingGuard />}>        {/* onboarding guard */}
    <Route element={<PaywallGuard />}>         {/* paywall guard */}
      {/* ...existing routes... */}
      <Route path="/quotes/:quoteId/edit" element={<QuoteDetailPage />} />
    </Route>
  </Route>
</Route>
```

## Additional File: `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx`

**What changes:** Remove `site_visit_requested` from `canResend` array (line 50).

### Current (line 50)

```typescript
const canResend = ["sent", "viewed", "responded", "site_visit_requested"].includes(estimate.status);
```

### Target

```typescript
const canResend = ["sent", "viewed", "responded"].includes(estimate.status);
```

## Enums Reference

### Backend `EstimateResponseType` (`trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts`)

```typescript
export enum EstimateResponseType {
  PROCEED = "proceed",
  MESSAGE = "message",
  DECLINE = "decline",
}
```

### Backend `EstimateDeclineReason` (`trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts`)

```typescript
export enum EstimateDeclineReason {
  TOO_EXPENSIVE = "too_expensive",
  GOING_WITH_SOMEONE_ELSE = "going_with_someone_else",
  DECIDED_NOT_TO_DO_WORK = "decided_not_to_do_work",
  JUST_GETTING_IDEA = "just_getting_idea",
  TIMING_NOT_RIGHT = "timing_not_right",
}
```

### Backend `EstimateStatus` (`trade-flow-api/src/estimate/enums/estimate-status.enum.ts`)

```typescript
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

Note: No `SITE_VISIT_REQUESTED` value exists in the backend enum.

## Date Formatting Reference

Location: `trade-flow-ui/src/lib/date-helpers.ts`

```typescript
export function formatDate(isoString: string): string {
  return DateTime.fromISO(isoString).toLocaleString(DateTime.DATE_MED);
}

export function formatDateTime(isoString: string): string {
  return DateTime.fromISO(isoString).toLocaleString(DateTime.DATETIME_MED);
}
```

Use `formatDateTime` for the response timestamp since it includes time precision.

## Controller Response Mapping Reference

The controller's `mapResponseSummary` method (at `trade-flow-api/src/estimate/controllers/estimate.controller.ts` lines 347-357) already maps `IEstimateResponseSummaryDto` to `IEstimateResponseSummaryResponse` correctly. It passes through `null` when `estimate.responseSummary` is null. Once the repository `toDto()` derives the summary, this controller method works without changes.

```typescript
private mapResponseSummary(estimate: IEstimateDto): IEstimateResponseSummaryResponse | null {
  if (!estimate.responseSummary) {
    return null;
  }
  return {
    lastResponseType: estimate.responseSummary.lastResponseType,
    lastResponseAt: estimate.responseSummary.lastResponseAt,
    lastResponseMessage: estimate.responseSummary.lastResponseMessage,
    declineReason: estimate.responseSummary.declineReason,
  };
}
```
