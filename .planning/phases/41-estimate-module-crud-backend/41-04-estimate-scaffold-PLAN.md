---
phase: 41-estimate-module-crud-backend
plan: 04
type: execute
wave: 3
depends_on: [01, 02, 03]
files_modified:
  - trade-flow-api/src/estimate/enums/estimate-status.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-display-mode.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-transitions.ts
  - trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts
  - trade-flow-api/src/estimate/entities/estimate.entity.ts
  - trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-totals.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/bundle-line-item-group.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/component-blueprint.dto.ts
  - trade-flow-api/src/estimate/requests/create-estimate.request.ts
  - trade-flow-api/src/estimate/requests/update-estimate.request.ts
  - trade-flow-api/src/estimate/requests/list-estimates.request.ts
  - trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts
  - trade-flow-api/src/estimate/requests/update-estimate-line-item.request.ts
  - trade-flow-api/src/estimate/responses/estimate.responses.ts
  - trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts
  - trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts
autonomous: true
requirements: [EST-01, EST-03, EST-04, EST-06, EST-07, EST-08, CONT-01, CONT-02, CONT-05, RESP-08]
must_haves:
  truths:
    - All five estimate enums exist at exactly the values locked in CONTEXT.md §specifics lines 487-532
    - `ALLOWED_TRANSITIONS` map in estimate-transitions.ts matches the locked map in CONTEXT.md §specifics lines 540-574 verbatim, including the `SENT → SENT` re-send no-op
    - `estimate-transitions.spec.ts` has positive assertions for every allowed transition and negative assertions for representative disallowed transitions, plus terminal-state tests
    - `IEstimateEntity` interface contains every field listed in CONTEXT.md §specifics lines 277-324, including the reserved nullable fields for Phase 42/43/44/45/46/47
    - `IEstimateDto` contains `totals: IEstimateTotalsDto`, `priceRange: IEstimatePriceRangeDto`, and `responseSummary: IEstimateResponseSummaryDto | null` per D-CONT-06 and D-RESP-01
    - `CreateEstimateRequest.contingencyPercent` is decorated with `@Min(0) @Max(30) @IsIn([0, 5, 10, 15, 20, 25, 30])` per CONT-01 and discretion
    - `ListEstimatesRequest` exposes `status?`, `limit?` (1-100), `offset?` (>=0) with class-validator + class-transformer decorators for pagination
    - `EstimateMockGenerator.createEstimateDto(overrides?)` produces a valid `IEstimateDto` ready for use by every downstream spec
    - `cd trade-flow-api && npm run typecheck && npm run test -- --testPathPattern=estimate/enums/estimate-transitions` exits 0
    - `cd trade-flow-api && npm run ci` exits 0
  artifacts:
    - path: trade-flow-api/src/estimate/enums/estimate-status.enum.ts
      provides: EstimateStatus enum with 10 members
      contains: "DRAFT = \"draft\""
    - path: trade-flow-api/src/estimate/enums/estimate-transitions.ts
      provides: ALLOWED_TRANSITIONS map + isValidTransition + getValidTransitions helpers
      contains: "ALLOWED_TRANSITIONS"
    - path: trade-flow-api/src/estimate/entities/estimate.entity.ts
      provides: IEstimateEntity with all Phase 41 and reserved fields
      contains: "contingencyPercent"
    - path: trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts
      provides: IEstimatePriceRangeDto with low/high bounds
      exports: ["IEstimatePriceRangeDto", "IEstimatePriceRangeBoundDto"]
    - path: trade-flow-api/src/estimate/requests/create-estimate.request.ts
      provides: CreateEstimateRequest with class-validator decorators
      contains: "IsIn"
    - path: trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts
      provides: EstimateMockGenerator static factory
      exports: ["EstimateMockGenerator"]
  key_links:
    - from: trade-flow-api/src/estimate/entities/estimate.entity.ts
      to: trade-flow-api/src/estimate/enums/estimate-status.enum.ts
      via: import
      pattern: "from \"@estimate/enums/estimate-status.enum\""
    - from: trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
      to: trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts
      via: import + field
      pattern: "priceRange"
    - from: trade-flow-api/src/estimate/requests/create-estimate.request.ts
      to: trade-flow-api/src/estimate/enums/estimate-display-mode.enum.ts
      via: import
      pattern: "EstimateDisplayMode"
---

<objective>
Scaffold every type-level artifact of `src/estimate/` — enums, entities, DTOs, requests, responses, mock generators — as pure interface/type/enum definitions with zero runtime behavior. This is an interface-first plan: downstream plans (05-08) implement repositories, services, policies, and controllers against these contracts. The only runtime test that lands in this plan is `estimate-transitions.spec.ts` because the transition map is a pure data structure and has no dependencies beyond the `EstimateStatus` enum.

Purpose: Establishing the contract surface in one plan lets Plans 05-08 execute without needing to discover type shapes. Every downstream plan reads from this scaffold. The locked type shapes in CONTEXT.md §specifics (lines 230-610) and the locked transition map (lines 534-584) are copied here verbatim to prevent interpretation drift.

Output: Complete `src/estimate/{enums,entities,data-transfer-objects,requests,responses,test/mocks}/` tree plus one executable spec file (`estimate-transitions.spec.ts`) proving the map is correct.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files the executor MUST read before creating the mirror copies. -->

Quote module equivalents to mirror file-for-file:
- trade-flow-api/src/quote/enums/quote-status.enum.ts → estimate-status.enum.ts
- trade-flow-api/src/quote/enums/quote-line-item-status.enum.ts → estimate-line-item-status.enum.ts
- trade-flow-api/src/quote/enums/quote-transitions.ts → estimate-transitions.ts
- trade-flow-api/src/quote/enums/quote-transitions.spec.ts → estimate-transitions.spec.ts
- trade-flow-api/src/quote/entities/quote.entity.ts → estimate.entity.ts (with new contingency/displayMode/uncertainty/response/revision fields)
- trade-flow-api/src/quote/entities/quote-line-item.entity.ts → estimate-line-item.entity.ts (quoteId → estimateId)
- trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts → estimate.dto.ts
- trade-flow-api/src/quote/data-transfer-objects/quote-line-item.dto.ts → estimate-line-item.dto.ts
- trade-flow-api/src/quote/data-transfer-objects/quote-totals.dto.ts → estimate-totals.dto.ts
- trade-flow-api/src/quote/data-transfer-objects/bundle-line-item-group.dto.ts → bundle-line-item-group.dto.ts
- trade-flow-api/src/quote/data-transfer-objects/component-blueprint.dto.ts → component-blueprint.dto.ts
- trade-flow-api/src/quote/requests/create-quote.request.ts → create-estimate.request.ts
- trade-flow-api/src/quote/requests/update-quote.request.ts → update-estimate.request.ts
- trade-flow-api/src/quote/requests/create-quote-line-item.request.ts → create-estimate-line-item.request.ts
- trade-flow-api/src/quote/requests/update-quote-line-item.request.ts → update-estimate-line-item.request.ts
- trade-flow-api/src/quote/responses/quote.responses.ts → estimate.responses.ts
- trade-flow-api/src/quote/test/mocks/quote-mock-generator.ts → estimate-mock-generator.ts
- trade-flow-api/src/quote/test/mocks/quote-line-item-mock-generator.ts → estimate-line-item-mock-generator.ts

Phase 41 introduces these net-new DTOs (no quote equivalent):
- estimate-price-range.dto.ts (per D-CONT-05, D-CONT-06)
- estimate-response-summary.dto.ts (per D-RESP-01)
- estimate-display-mode.enum.ts (per D-CONT-07)
- estimate-response-type.enum.ts (per D-RESP-03)
- estimate-decline-reason.enum.ts (per D-RESP-03)
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Create estimate enums and the executable transition map + spec</name>
  <files>
    trade-flow-api/src/estimate/enums/estimate-status.enum.ts,
    trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts,
    trade-flow-api/src/estimate/enums/estimate-display-mode.enum.ts,
    trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts,
    trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts,
    trade-flow-api/src/estimate/enums/estimate-transitions.ts,
    trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/enums/quote-status.enum.ts (naming convention and file layout)
    - trade-flow-api/src/quote/enums/quote-line-item-status.enum.ts
    - trade-flow-api/src/quote/enums/quote-transitions.ts (transition map pattern)
    - trade-flow-api/src/quote/enums/quote-transitions.spec.ts (spec pattern with positive/negative/terminal assertions)
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (§specifics lines 487-584 — enum values and transition map are LOCKED verbatim)
  </read_first>
  <action>
**Step 1: Create `trade-flow-api/src/estimate/enums/estimate-status.enum.ts`:**

```typescript
export enum EstimateStatus {
  DRAFT = "draft",
  SENT = "sent",
  VIEWED = "viewed",
  RESPONDED = "responded",
  SITE_VISIT_REQUESTED = "site_visit_requested",
  CONVERTED = "converted",
  DECLINED = "declined",
  EXPIRED = "expired",
  LOST = "lost",
  DELETED = "deleted",
}
```

**Step 2: Create `trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts`:**

```typescript
export enum EstimateLineItemStatus {
  PENDING = "PENDING",
  APPROVED = "APPROVED",
  REJECTED = "REJECTED",
  DELETED = "DELETED",
}
```

(Note: UPPER_SNAKE_CASE values here mirror `QuoteLineItemStatus` per CONTEXT.md D-LI-02 literal mirror mandate. Verify by reading `quote-line-item-status.enum.ts`.)

**Step 3: Create `trade-flow-api/src/estimate/enums/estimate-display-mode.enum.ts`:**

```typescript
export enum EstimateDisplayMode {
  RANGE = "range",
  FROM = "from",
}
```

**Step 4: Create `trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts`:**

```typescript
export enum EstimateResponseType {
  SITE_VISIT_REQUEST = "site_visit_request",
  SEND_ME_QUOTE = "send_me_quote",
  QUESTION = "question",
  DECLINE = "decline",
}
```

**Step 5: Create `trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts`:**

```typescript
export enum EstimateDeclineReason {
  TOO_EXPENSIVE = "too_expensive",
  DECIDED_NOT_TO_DO_WORK = "decided_not_to_do_work",
  GOING_WITH_ANOTHER_TRADESPERSON = "going_with_another_tradesperson",
  JUST_GETTING_IDEA = "just_getting_idea",
  TIMING_NOT_RIGHT = "timing_not_right",
  OTHER = "other",
}
```

**Step 6: Create `trade-flow-api/src/estimate/enums/estimate-transitions.ts`** — copy the locked map verbatim from CONTEXT.md §specifics lines 540-584:

```typescript
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";

export const ALLOWED_TRANSITIONS: ReadonlyMap<EstimateStatus, readonly EstimateStatus[]> = new Map([
  [EstimateStatus.DRAFT, [EstimateStatus.SENT, EstimateStatus.DELETED]],
  [
    EstimateStatus.SENT,
    [
      EstimateStatus.VIEWED,
      EstimateStatus.RESPONDED,
      EstimateStatus.EXPIRED,
      EstimateStatus.LOST,
      EstimateStatus.CONVERTED,
      EstimateStatus.SENT,
    ],
  ],
  [
    EstimateStatus.VIEWED,
    [
      EstimateStatus.RESPONDED,
      EstimateStatus.EXPIRED,
      EstimateStatus.LOST,
      EstimateStatus.CONVERTED,
    ],
  ],
  [
    EstimateStatus.RESPONDED,
    [
      EstimateStatus.SITE_VISIT_REQUESTED,
      EstimateStatus.CONVERTED,
      EstimateStatus.DECLINED,
      EstimateStatus.LOST,
      EstimateStatus.EXPIRED,
    ],
  ],
  [
    EstimateStatus.SITE_VISIT_REQUESTED,
    [
      EstimateStatus.CONVERTED,
      EstimateStatus.DECLINED,
      EstimateStatus.LOST,
      EstimateStatus.EXPIRED,
    ],
  ],
  [EstimateStatus.CONVERTED, []],
  [EstimateStatus.DECLINED, []],
  [EstimateStatus.EXPIRED, []],
  [EstimateStatus.LOST, []],
  [EstimateStatus.DELETED, []],
]);

export const isValidTransition = (from: EstimateStatus, to: EstimateStatus): boolean => {
  const allowed = ALLOWED_TRANSITIONS.get(from);
  return allowed !== undefined && allowed.includes(to);
};

export const getValidTransitions = (from: EstimateStatus): readonly EstimateStatus[] => {
  return ALLOWED_TRANSITIONS.get(from) ?? [];
};
```

**Step 7: Create `trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts`** — mirror the shape of `quote-transitions.spec.ts`. D-TXN-06 requires: every allowed transition has a positive assertion; every disallowed transition has a negative assertion; terminal states reject further transitions; the `SENT → SENT` no-op is explicitly asserted. Minimum structure:

```typescript
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import {
  ALLOWED_TRANSITIONS,
  getValidTransitions,
  isValidTransition,
} from "@estimate/enums/estimate-transitions";

describe("estimate-transitions", () => {
  describe("isValidTransition - allowed", () => {
    it("DRAFT -> SENT", () => expect(isValidTransition(EstimateStatus.DRAFT, EstimateStatus.SENT)).toBe(true));
    it("DRAFT -> DELETED", () => expect(isValidTransition(EstimateStatus.DRAFT, EstimateStatus.DELETED)).toBe(true));
    it("SENT -> VIEWED", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.VIEWED)).toBe(true));
    it("SENT -> RESPONDED", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.RESPONDED)).toBe(true));
    it("SENT -> EXPIRED", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.EXPIRED)).toBe(true));
    it("SENT -> LOST", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.LOST)).toBe(true));
    it("SENT -> CONVERTED", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.CONVERTED)).toBe(true));
    it("SENT -> SENT (re-send no-op)", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.SENT)).toBe(true));
    it("VIEWED -> RESPONDED", () => expect(isValidTransition(EstimateStatus.VIEWED, EstimateStatus.RESPONDED)).toBe(true));
    it("VIEWED -> EXPIRED", () => expect(isValidTransition(EstimateStatus.VIEWED, EstimateStatus.EXPIRED)).toBe(true));
    it("VIEWED -> LOST", () => expect(isValidTransition(EstimateStatus.VIEWED, EstimateStatus.LOST)).toBe(true));
    it("VIEWED -> CONVERTED", () => expect(isValidTransition(EstimateStatus.VIEWED, EstimateStatus.CONVERTED)).toBe(true));
    it("RESPONDED -> SITE_VISIT_REQUESTED", () => expect(isValidTransition(EstimateStatus.RESPONDED, EstimateStatus.SITE_VISIT_REQUESTED)).toBe(true));
    it("RESPONDED -> CONVERTED", () => expect(isValidTransition(EstimateStatus.RESPONDED, EstimateStatus.CONVERTED)).toBe(true));
    it("RESPONDED -> DECLINED", () => expect(isValidTransition(EstimateStatus.RESPONDED, EstimateStatus.DECLINED)).toBe(true));
    it("RESPONDED -> LOST", () => expect(isValidTransition(EstimateStatus.RESPONDED, EstimateStatus.LOST)).toBe(true));
    it("RESPONDED -> EXPIRED", () => expect(isValidTransition(EstimateStatus.RESPONDED, EstimateStatus.EXPIRED)).toBe(true));
    it("SITE_VISIT_REQUESTED -> CONVERTED", () => expect(isValidTransition(EstimateStatus.SITE_VISIT_REQUESTED, EstimateStatus.CONVERTED)).toBe(true));
    it("SITE_VISIT_REQUESTED -> DECLINED", () => expect(isValidTransition(EstimateStatus.SITE_VISIT_REQUESTED, EstimateStatus.DECLINED)).toBe(true));
    it("SITE_VISIT_REQUESTED -> LOST", () => expect(isValidTransition(EstimateStatus.SITE_VISIT_REQUESTED, EstimateStatus.LOST)).toBe(true));
    it("SITE_VISIT_REQUESTED -> EXPIRED", () => expect(isValidTransition(EstimateStatus.SITE_VISIT_REQUESTED, EstimateStatus.EXPIRED)).toBe(true));
  });

  describe("isValidTransition - disallowed", () => {
    it("SENT -> DELETED (cannot delete sent estimate)", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.DELETED)).toBe(false));
    it("SENT -> DRAFT (cannot revert)", () => expect(isValidTransition(EstimateStatus.SENT, EstimateStatus.DRAFT)).toBe(false));
    it("DRAFT -> VIEWED (must go via SENT)", () => expect(isValidTransition(EstimateStatus.DRAFT, EstimateStatus.VIEWED)).toBe(false));
    it("DELETED -> DRAFT (terminal)", () => expect(isValidTransition(EstimateStatus.DELETED, EstimateStatus.DRAFT)).toBe(false));
    it("CONVERTED -> any (terminal)", () => {
      expect(isValidTransition(EstimateStatus.CONVERTED, EstimateStatus.SENT)).toBe(false);
      expect(isValidTransition(EstimateStatus.CONVERTED, EstimateStatus.DELETED)).toBe(false);
    });
    it("DECLINED -> any (terminal)", () => expect(isValidTransition(EstimateStatus.DECLINED, EstimateStatus.SENT)).toBe(false));
    it("EXPIRED -> any (terminal)", () => expect(isValidTransition(EstimateStatus.EXPIRED, EstimateStatus.SENT)).toBe(false));
    it("LOST -> any (terminal)", () => expect(isValidTransition(EstimateStatus.LOST, EstimateStatus.SENT)).toBe(false));
  });

  describe("getValidTransitions", () => {
    it("returns [SENT, DELETED] for DRAFT", () => {
      const transitions = getValidTransitions(EstimateStatus.DRAFT);
      expect(transitions).toContain(EstimateStatus.SENT);
      expect(transitions).toContain(EstimateStatus.DELETED);
    });

    it("returns empty array for all terminal states", () => {
      expect(getValidTransitions(EstimateStatus.CONVERTED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.DECLINED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.EXPIRED)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.LOST)).toEqual([]);
      expect(getValidTransitions(EstimateStatus.DELETED)).toEqual([]);
    });

    it("ALLOWED_TRANSITIONS has exactly 10 keys (one per status)", () => {
      expect(ALLOWED_TRANSITIONS.size).toBe(10);
    });
  });
});
```

Step 8: Run the spec:

```
cd trade-flow-api && npm run test -- --testPathPattern=estimate-transitions
```

All tests must pass.

Commit message: `feat(41): add estimate enums and transition map with spec`.
  </action>
  <acceptance_criteria>
    - All 7 enum/transition files exist
    - `grep -c "DRAFT = \"draft\"" trade-flow-api/src/estimate/enums/estimate-status.enum.ts` returns 1
    - `grep -c "SITE_VISIT_REQUESTED = \"site_visit_requested\"" trade-flow-api/src/estimate/enums/estimate-status.enum.ts` returns 1
    - `grep -c "ALLOWED_TRANSITIONS" trade-flow-api/src/estimate/enums/estimate-transitions.ts` returns at least 2 (definition + export)
    - `grep -c "EstimateStatus.SENT, \\[" trade-flow-api/src/estimate/enums/estimate-transitions.ts` returns 1 (the SENT row exists)
    - `grep -c "EstimateStatus.SENT,$\\|EstimateStatus.SENT," trade-flow-api/src/estimate/enums/estimate-transitions.ts | head -1` — verify SENT appears in its own list (re-send no-op)
    - `grep -c "isValidTransition" trade-flow-api/src/estimate/enums/estimate-transitions.ts` returns at least 2 (definition + export)
    - `grep -c "getValidTransitions" trade-flow-api/src/estimate/enums/estimate-transitions.ts` returns at least 2
    - `grep -c "SENT -> SENT (re-send no-op)" trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts` returns 1
    - `grep -c "ALLOWED_TRANSITIONS.size).toBe(10)" trade-flow-api/src/estimate/enums/estimate-transitions.spec.ts` returns 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=estimate-transitions` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as " trade-flow-api/src/estimate/enums/` returns 0 (no casts, no suppressions — `as ` pattern catches `as const`, `as any`, etc.; `as const` is acceptable ONLY on the ALLOWED_TRANSITIONS map but the project pattern uses `ReadonlyMap<...>` which is the preferred approach — confirm zero `as` in this file)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=estimate-transitions</automated>
  </verify>
  <done>All 5 enums, the transition map, and the transition spec exist and the spec passes</done>
</task>

<task type="auto">
  <name>Task 2: Create estimate entities, DTOs, and response summary + price range DTOs</name>
  <files>
    trade-flow-api/src/estimate/entities/estimate.entity.ts,
    trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts,
    trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/estimate-totals.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/bundle-line-item-group.dto.ts,
    trade-flow-api/src/estimate/data-transfer-objects/component-blueprint.dto.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/entities/quote.entity.ts (field layout and IBaseEntity extension pattern)
    - trade-flow-api/src/quote/entities/quote-line-item.entity.ts (for the quoteId → estimateId rename)
    - trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts (DateTime handling pattern)
    - trade-flow-api/src/quote/data-transfer-objects/quote-line-item.dto.ts (Money field types)
    - trade-flow-api/src/quote/data-transfer-objects/quote-totals.dto.ts (subTotal/taxTotal/total shape)
    - trade-flow-api/src/quote/data-transfer-objects/bundle-line-item-group.dto.ts
    - trade-flow-api/src/quote/data-transfer-objects/component-blueprint.dto.ts
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (§specifics lines 268-483 — every type shape is LOCKED verbatim)
  </read_first>
  <action>
**Step 1: Create `estimate.entity.ts`** — copy verbatim from CONTEXT.md §specifics lines 277-324. Every field must appear exactly as locked:

```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";
import { ObjectId } from "mongodb";

export interface IEstimateEntity extends IBaseEntity {
  businessId: ObjectId;
  customerId: ObjectId;
  jobId: ObjectId;

  number: string;
  title: string;
  estimateDate: Date;
  notes?: string;

  status: EstimateStatus;
  sentAt?: Date;
  firstViewedAt?: Date;
  respondedAt?: Date;
  convertedAt?: Date;
  convertedToQuoteId?: ObjectId;
  declinedAt?: Date;
  lostAt?: Date;
  expiresAt?: Date;
  deletedAt?: Date;

  contingencyPercent: number;
  displayMode: EstimateDisplayMode;

  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;

  lastResponseType?: EstimateResponseType;
  lastResponseAt?: Date;
  lastResponseMessage?: string;
  declineReason?: EstimateDeclineReason;
  siteVisitAvailability?: string;

  parentEstimateId?: ObjectId | null;
  rootEstimateId?: ObjectId | null;
  revisionNumber: number;
  isCurrent: boolean;

  lineItems?: ObjectId[];
}
```

**Step 2: Create `estimate-line-item.entity.ts`** — verbatim from CONTEXT.md §specifics lines 431-453:

```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ItemType } from "@item/enums/item-type.enum";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";
import { ObjectId } from "mongodb";

export interface IEstimateLineItemEntity extends IBaseEntity {
  estimateId: ObjectId;
  businessId: ObjectId;
  itemId: ObjectId;
  quantity: number;
  unit: string;
  unitPrice: number;
  lineTotal: number;
  originalUnitPrice: number;
  originalLineTotal: number;
  discountAmount: number;
  taxRate: number;
  type: ItemType;
  status: EstimateLineItemStatus;
  parentLineItemId?: ObjectId | null;
}
```

**Step 3: Create `estimate-totals.dto.ts`** — verbatim from CONTEXT.md §specifics lines 383-392:

```typescript
import { Money } from "@core/value-objects/money.value-object";

export interface IEstimateTotalsDto {
  subTotal: Money;
  taxTotal: Money;
  total: Money;
}
```

**Step 4: Create `estimate-price-range.dto.ts`** — verbatim from CONTEXT.md §specifics lines 394-412:

```typescript
import { Money } from "@core/value-objects/money.value-object";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export interface IEstimatePriceRangeBoundDto {
  exclTax: Money;
  tax: Money;
  inclTax: Money;
}

export interface IEstimatePriceRangeDto {
  displayMode: EstimateDisplayMode;
  subtotal: Money;
  contingencyPercent: number;
  low: IEstimatePriceRangeBoundDto;
  high: IEstimatePriceRangeBoundDto;
}
```

**Step 5: Create `estimate-response-summary.dto.ts`** — verbatim from CONTEXT.md §specifics lines 416-429:

```typescript
import { DateTime } from "luxon";
import { EstimateResponseType } from "@estimate/enums/estimate-response-type.enum";
import { EstimateDeclineReason } from "@estimate/enums/estimate-decline-reason.enum";

export interface IEstimateResponseSummaryDto {
  type: EstimateResponseType;
  respondedAt: DateTime;
  message?: string;
  declineReason?: EstimateDeclineReason;
  siteVisitAvailability?: string;
}
```

**Step 6: Create `estimate-line-item.dto.ts`** — verbatim from CONTEXT.md §specifics lines 458-482:

```typescript
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { Money } from "@core/value-objects/money.value-object";
import { ItemType } from "@item/enums/item-type.enum";
import { EstimateLineItemStatus } from "@estimate/enums/estimate-line-item-status.enum";

export interface IEstimateLineItemDto extends IBaseResourceDto {
  id: string;
  estimateId: string;
  itemId: string;
  businessId: string;
  quantity: number;
  unit: string;
  unitPrice: Money;
  lineTotal: Money;
  originalUnitPrice: Money;
  originalLineTotal: Money;
  discountAmount: Money;
  taxRate: number;
  type: ItemType;
  status: EstimateLineItemStatus;
  parentLineItemId?: string | null;
}
```

**Step 7: Create `estimate.dto.ts`** — verbatim from CONTEXT.md §specifics lines 328-381:

```typescript
import { DateTime } from "luxon";
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { DtoCollection } from "@core/collections/dto.collection";
import { IEstimateLineItemDto } from "@estimate/data-transfer-objects/estimate-line-item.dto";
import { IEstimateTotalsDto } from "@estimate/data-transfer-objects/estimate-totals.dto";
import { IEstimatePriceRangeDto } from "@estimate/data-transfer-objects/estimate-price-range.dto";
import { IEstimateResponseSummaryDto } from "@estimate/data-transfer-objects/estimate-response-summary.dto";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export interface IEstimateDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  customerId: string;
  jobId: string;

  number: string;
  title: string;
  estimateDate: DateTime;
  notes?: string;

  status: EstimateStatus;
  sentAt?: DateTime;
  firstViewedAt?: DateTime;
  respondedAt?: DateTime;
  convertedAt?: DateTime;
  convertedToQuoteId?: string;
  declinedAt?: DateTime;
  lostAt?: DateTime;
  expiresAt?: DateTime;
  deletedAt?: DateTime;
  updatedAt?: DateTime;

  contingencyPercent: number;
  displayMode: EstimateDisplayMode;

  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;

  parentEstimateId?: string | null;
  rootEstimateId?: string | null;
  revisionNumber: number;
  isCurrent: boolean;

  lineItems: DtoCollection<IEstimateLineItemDto>;
  totals: IEstimateTotalsDto;
  priceRange: IEstimatePriceRangeDto;

  responseSummary: IEstimateResponseSummaryDto | null;
}
```

**Step 8: Create `bundle-line-item-group.dto.ts` and `component-blueprint.dto.ts`** — mirror the quote-module equivalents verbatim with `Quote → Estimate` / `quoteId → estimateId` renames. Read the source files first (listed in <read_first>) to get the exact shapes. These DTOs exist to support the bundle factory stack in Plan 06.

**Step 9: Verify everything compiles:**

```
cd trade-flow-api && npm run typecheck
```

Expected: exit 0. If errors appear, they will almost certainly be missing imports of `IBaseEntity`, `IBaseResourceDto`, `Money`, `DtoCollection`, `ItemType`, or `ObjectId`. Check the read_first quote sources for the correct import paths.

Commit message: `feat(41): add estimate entities, DTOs, and price range + response summary types`.
  </action>
  <acceptance_criteria>
    - `grep -c "export interface IEstimateEntity" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "contingencyPercent: number" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "revisionNumber: number" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "isCurrent: boolean" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "lastResponseType" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "uncertaintyReasons" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 1
    - `grep -c "quoteId" trade-flow-api/src/estimate/entities/estimate.entity.ts` returns 0 (no leftover field name from copy-paste)
    - `grep -c "export interface IEstimateLineItemEntity" trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts` returns 1
    - `grep -c "estimateId: ObjectId" trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts` returns 1
    - `grep -c "quoteId" trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts` returns 0
    - `grep -c "export interface IEstimateDto" trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts` returns 1
    - `grep -c "priceRange: IEstimatePriceRangeDto" trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts` returns 1
    - `grep -c "totals: IEstimateTotalsDto" trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts` returns 1
    - `grep -c "responseSummary: IEstimateResponseSummaryDto | null" trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts` returns 1
    - `grep -c "export interface IEstimatePriceRangeDto" trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts` returns 1
    - `grep -c "export interface IEstimatePriceRangeBoundDto" trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts` returns 1
    - `grep -c "displayMode: EstimateDisplayMode" trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts` returns 1
    - `grep -c "vat\\|VAT" trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts` returns 0 (country-agnostic naming enforced)
    - `grep -c "export interface IEstimateResponseSummaryDto" trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts` returns 1
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any" trade-flow-api/src/estimate/entities/ trade-flow-api/src/estimate/data-transfer-objects/` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck</automated>
  </verify>
  <done>All entity and DTO files exist with locked shapes and compile cleanly</done>
</task>

<task type="auto">
  <name>Task 3: Create estimate request/response DTOs and mock generators</name>
  <files>
    trade-flow-api/src/estimate/requests/create-estimate.request.ts,
    trade-flow-api/src/estimate/requests/update-estimate.request.ts,
    trade-flow-api/src/estimate/requests/list-estimates.request.ts,
    trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts,
    trade-flow-api/src/estimate/requests/update-estimate-line-item.request.ts,
    trade-flow-api/src/estimate/responses/estimate.responses.ts,
    trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts,
    trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/requests/create-quote.request.ts (class-validator decorator conventions, field ordering)
    - trade-flow-api/src/quote/requests/update-quote.request.ts
    - trade-flow-api/src/quote/requests/create-quote-line-item.request.ts
    - trade-flow-api/src/quote/requests/update-quote-line-item.request.ts
    - trade-flow-api/src/quote/responses/quote.responses.ts (response shape including totals)
    - trade-flow-api/src/quote/test/mocks/quote-mock-generator.ts (static factory method pattern)
    - trade-flow-api/src/quote/test/mocks/quote-line-item-mock-generator.ts
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Code Examples → Pattern 6 ListEstimatesRequest; §Code Examples → Mock Generator)
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (D-UNC-01 — CreateEstimateRequest and UpdateEstimateRequest MUST accept `uncertaintyReasons?: string[]` and `uncertaintyNotes?: string`)
  </read_first>
  <action>
**Step 1: `create-estimate.request.ts`** — mirror `create-quote.request.ts` with these added/changed fields:

```typescript
import {
  IsArray,
  IsEnum,
  IsIn,
  IsInt,
  IsMongoId,
  IsOptional,
  IsString,
  Max,
  MaxLength,
  Min,
  MinLength,
} from "class-validator";
import { Type } from "class-transformer";
import { EstimateDisplayMode } from "@estimate/enums/estimate-display-mode.enum";

export class CreateEstimateRequest {
  @IsMongoId()
  public customerId!: string;

  @IsMongoId()
  public jobId!: string;

  @IsString()
  @MinLength(1)
  @MaxLength(255)
  public title!: string;

  @IsOptional()
  @IsString()
  public notes?: string;

  @IsOptional()
  @IsString()
  public estimateDate?: string;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  @Max(30)
  @IsIn([0, 5, 10, 15, 20, 25, 30])
  public contingencyPercent?: number;

  @IsOptional()
  @IsEnum(EstimateDisplayMode)
  public displayMode?: EstimateDisplayMode;

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  public uncertaintyReasons?: string[];

  @IsOptional()
  @IsString()
  public uncertaintyNotes?: string;
}
```

Notes:
- All fields are optional on create EXCEPT `customerId`, `jobId`, `title` (required for estimate identity).
- `contingencyPercent` validation stack is the discretion decision from CONTEXT.md.
- `displayMode` defaults to `"range"` in the creator service (Plan 07), not in the request.
- `estimateDate` is a string on the wire; the service converts via Luxon. Match the quote module pattern if it uses `@IsString()` or `@IsDateString()`.
- No line items on create request — line items are added via separate `POST /estimates/:id/line-item` endpoints per RESEARCH.md §Open Question #3.

**Step 2: `update-estimate.request.ts`** — similar shape but all fields optional (PATCH semantics). No `customerId`/`jobId` required. Include `contingencyPercent`, `displayMode`, `title`, `notes`, `estimateDate`, `uncertaintyReasons`, `uncertaintyNotes`, and the customer/job fields as optional. NO `status` field — the only status transition writable via PATCH in Phase 41 is the Draft→Draft no-op (status not changed).

**Step 3: `list-estimates.request.ts`** — per RESEARCH.md §Pattern 6:

```typescript
import { IsEnum, IsInt, IsOptional, Max, Min } from "class-validator";
import { Type } from "class-transformer";
import { EstimateStatus } from "@estimate/enums/estimate-status.enum";

export class ListEstimatesRequest {
  @IsOptional()
  @IsEnum(EstimateStatus)
  public status?: EstimateStatus;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  public limit?: number;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  public offset?: number;
}
```

**Step 4: `create-estimate-line-item.request.ts`** — mirror `create-quote-line-item.request.ts` with class/field names rewritten. Read the source file first to get the exact decorator stack for item type, quantity, discount, price, tax rate, parent line item id, bundle fields.

**Step 5: `update-estimate-line-item.request.ts`** — mirror the quote equivalent. All fields optional (PATCH).

**Step 6: `estimate.responses.ts`** — mirror `quote.responses.ts`:

```typescript
// Skeleton — executor expands to match the full quote response shape including:
// - id, businessId, customerId, jobId, number, title, estimateDate, notes
// - status, contingencyPercent, displayMode
// - all reserved nullable timestamps (sentAt, firstViewedAt, etc.)
// - totals (IEstimateTotalsDto)
// - priceRange (IEstimatePriceRangeDto)
// - lineItems (IEstimateLineItemResponse[])
// - responseSummary (IEstimateResponseSummaryDto | null)
// - revisionNumber, isCurrent, parentEstimateId, rootEstimateId (read-only in Phase 41)
```

Response field names mirror the DTO. Dates serialized as ISO strings (not Luxon DateTime) per the quote-module convention — verify by reading `quote.responses.ts`.

**Step 7: `estimate-mock-generator.ts`** — use the pattern from RESEARCH.md §Code Examples → Mock Generator, already transcribed on lines 965-1013. Copy it as the starting point. Add helper methods:
- `createEstimateDto(overrides?: Partial<IEstimateDto>): IEstimateDto` — returns a Draft estimate with zero line items, zero totals, and a valid priceRange (zero-valued bounds).
- `createEstimateEntity(overrides?: Partial<IEstimateEntity>): IEstimateEntity` — native Date equivalent.

**Step 8: `estimate-line-item-mock-generator.ts`** — mirror `quote-line-item-mock-generator.ts`:
- `createEstimateLineItemDto(overrides?): IEstimateLineItemDto` returns a standard (non-bundle) line item with sensible Money defaults.
- `createEstimateLineItemEntity(overrides?): IEstimateLineItemEntity`.

**Step 9: Verify typecheck:**

```
cd trade-flow-api && npm run typecheck
```

Expected: exit 0. Likely early failures: missing decorator import from `class-validator`, missing `Type` from `class-transformer`, wrong enum path.

Commit message: `feat(41): add estimate request/response DTOs and mock generators`.
  </action>
  <acceptance_criteria>
    - `grep -c "export class CreateEstimateRequest" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns 1
    - `grep -c "@IsIn(\\[0, 5, 10, 15, 20, 25, 30\\])" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns 1
    - `grep -c "@Min(0)" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns at least 1
    - `grep -c "@Max(30)" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns at least 1
    - `grep -c "uncertaintyReasons" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns at least 1
    - `grep -c "uncertaintyNotes" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns at least 1
    - `grep -c "displayMode" trade-flow-api/src/estimate/requests/create-estimate.request.ts` returns at least 1
    - `grep -c "export class UpdateEstimateRequest" trade-flow-api/src/estimate/requests/update-estimate.request.ts` returns 1
    - `grep -c "status" trade-flow-api/src/estimate/requests/update-estimate.request.ts` returns 0 (status is not writable via PATCH in Phase 41)
    - `grep -c "export class ListEstimatesRequest" trade-flow-api/src/estimate/requests/list-estimates.request.ts` returns 1
    - `grep -c "@IsEnum(EstimateStatus)" trade-flow-api/src/estimate/requests/list-estimates.request.ts` returns 1
    - `grep -c "@Max(100)" trade-flow-api/src/estimate/requests/list-estimates.request.ts` returns 1
    - `grep -c "@IsIn\\[0, 5, 10, 15, 20, 25, 30\\]" trade-flow-api/src/estimate/requests/update-estimate.request.ts || grep -c "@IsIn(\\[0, 5, 10, 15, 20, 25, 30\\])" trade-flow-api/src/estimate/requests/update-estimate.request.ts` returns 1 (same validator stack)
    - `grep -c "export class EstimateMockGenerator" trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts` returns 1
    - `grep -c "createEstimateDto" trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts` returns at least 1
    - `grep -c "EstimateDisplayMode.RANGE" trade-flow-api/src/estimate/test/mocks/estimate-mock-generator.ts` returns at least 1
    - `grep -c "export class EstimateLineItemMockGenerator" trade-flow-api/src/estimate/test/mocks/estimate-line-item-mock-generator.ts` returns 1
    - `grep -rc "vat\\|VAT" trade-flow-api/src/estimate/requests/ trade-flow-api/src/estimate/responses/` returns 0
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|as any\\|as unknown as" trade-flow-api/src/estimate/requests/ trade-flow-api/src/estimate/responses/ trade-flow-api/src/estimate/test/mocks/` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck</automated>
  </verify>
  <done>All request/response DTOs and mock generators exist, compile cleanly, and enforce the locked contingency validation stack</done>
</task>

<task type="auto">
  <name>Task 4: Run full CI gate</name>
  <files>none</files>
  <read_first>- trade-flow-api/package.json</read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected: exit 0.

The new estimate files have no runtime behavior except the `estimate-transitions.ts` helpers, which are unit tested in Task 1. Everything else is type-level — the CI gate verifies they compile clean, lint clean, and format clean.

If format:check fails, run `cd trade-flow-api && npm run format` then commit as `chore(41): prettier fixes for estimate scaffold`.

If typecheck fails, the usual culprits are: missing imports, `IBaseResourceDto` not exported from the expected path, `DtoCollection` missing generic, or a Luxon type used where a Date is expected (or vice versa).
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - All 25 scaffolded files exist in the tree
    - No suppressions in the diff
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Estimate scaffold compiles, lints, formats, and tests clean</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| HTTP client → NestJS ValidationPipe | Untrusted request body validated against CreateEstimateRequest / UpdateEstimateRequest / ListEstimatesRequest |
| Internal services → typed DTOs | Type-level safety guarantees field presence |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain; no new runtime surface introduced in this plan.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-04-01 | Tampering | Mass-assignment of privileged fields (number, status beyond DRAFT, sentAt, convertedToQuoteId) via create/update | mitigate | CreateEstimateRequest / UpdateEstimateRequest whitelist only safe fields; global ValidationPipe with `forbidNonWhitelisted: true` rejects extras; status field NOT present on either request class (Task 3 acceptance criterion verifies) |
| T-41-04-02 | Tampering | contingencyPercent outside allowed set permits unbounded markup | mitigate | class-validator stack `@Min(0) @Max(30) @IsIn([0, 5, 10, 15, 20, 25, 30])` on the request class rejects everything else with 400 |
| T-41-04-03 | Input Validation | uncertaintyReasons array can contain non-string values | mitigate | `@IsArray() @IsString({ each: true })` validates every element |
| T-41-04-04 | DoS | Unbounded pagination limit crashes API with large result sets | mitigate | ListEstimatesRequest `@Max(100)` on limit (Task 3 acceptance criterion verifies) |
| T-41-04-05 | Input Validation | customerId / jobId accept arbitrary strings permitting enumeration | mitigate | `@IsMongoId()` decorator rejects non-ObjectId strings before the service layer runs |
| T-41-04-06 | Tampering | Type-level pollution via `as` casts in DTOs | mitigate | Acceptance criteria grep for `as any` / `as unknown as` / `eslint-disable` on every file in this plan |
</threat_model>

<verification>
1. All 5 enum files and the transition map exist at locked values.
2. `ALLOWED_TRANSITIONS.size === 10` and every transition test passes.
3. All entity/DTO files exist with the exact field set from CONTEXT.md §specifics.
4. CreateEstimateRequest rejects contingencyPercent values outside `[0, 5, 10, 15, 20, 25, 30]`.
5. ListEstimatesRequest caps limit at 100.
6. No `vat*` fields anywhere.
7. `cd trade-flow-api && npm run ci` exits 0.
</verification>

<success_criteria>
- Plans 05-08 can reference `@estimate/entities/*`, `@estimate/data-transfer-objects/*`, `@estimate/enums/*`, `@estimate/requests/*`, `@estimate/responses/*` without encountering missing symbols
- The transition map is proven correct so Plan 05's `EstimateTransitionService` can rely on it without re-testing the map itself
- The mock generators are ready for use by every downstream service/repository/controller spec
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-04-SUMMARY.md` documenting:
- List of files created
- Confirmation the transition spec has 10 keys and SENT→SENT no-op assertion
- The full `npm run ci` output
</output>
