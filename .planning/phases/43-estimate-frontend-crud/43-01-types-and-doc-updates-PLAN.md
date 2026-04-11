---
phase: 43-estimate-frontend-crud
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/types/estimate.ts
  - trade-flow-ui/src/types/index.ts
  - .planning/REQUIREMENTS.md
  - .planning/ROADMAP.md
  - .planning/milestones/v1.8-ROADMAP.md
autonomous: true
requirements:
  - CONT-03
  - CONT-04

must_haves:
  truths:
    - "CONT-04 in REQUIREMENTS.md lists the five trade-agnostic uncertainty chip values"
    - "Estimate, EstimateLineItem, EstimateStatus, CreateEstimateRequest, UpdateEstimateRequest types exist in src/types/estimate.ts and are re-exported from src/types/index.ts"
    - "Future requirement SMART-04 exists in REQUIREMENTS.md for trade-specific uncertainty chip presets"
    - "ROADMAP.md and v1.8-ROADMAP.md Phase 43 success criterion #1 reference the five new chip values"
  artifacts:
    - path: "trade-flow-ui/src/types/estimate.ts"
      provides: "Estimate / EstimateLineItem / EstimateStatus / CreateEstimateRequest / UpdateEstimateRequest"
      contains: "export type EstimateStatus"
    - path: "trade-flow-ui/src/types/index.ts"
      provides: "Barrel re-exports for estimate types"
      contains: "from \"./estimate\""
    - path: ".planning/REQUIREMENTS.md"
      provides: "Updated CONT-04 + new SMART-04 Future requirement"
      contains: "SMART-04"
  key_links:
    - from: "trade-flow-ui/src/types/index.ts"
      to: "trade-flow-ui/src/types/estimate.ts"
      via: "type re-export"
      pattern: "from \"./estimate\""
---

<objective>
Establish the independent Estimate type contracts that downstream plans consume, and ship all doc edits that Phase 43 owes the repository so the roadmap and requirements match the locked decisions in 43-CONTEXT.md.

Purpose: These are the foundation artifacts everything else depends on. Types must exist before RTK Query, components, or pages can be built. Doc edits must happen in Phase 43 per D-CHIP-01 and the phase_specifics block in the orchestrator prompt.

Output: `src/types/estimate.ts` with 5 exported type names, barrel re-exports, updated CONT-04 wording, new SMART-04 Future requirement, synced roadmap success criteria.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/REQUIREMENTS.md
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@trade-flow-ui/src/types/quote.ts
@trade-flow-ui/src/types/index.ts
@trade-flow-ui/CLAUDE.md

<interfaces>
<!-- Reference shape for how Quote types are defined - Estimate mirrors structure, not names -->

From trade-flow-ui/src/types/quote.ts:
```typescript
export type QuoteStatus = "draft" | "sent" | "accepted" | "rejected" | "expired" | "deleted";
export interface QuoteLineItem { id, itemId, quantity, unit, unitPrice, originalUnitPrice, lineTotal, originalLineTotal, discountAmount, taxRate, type, status, components? }
export interface Quote { id, businessId, customerId, jobId, number, title, notes?, status, quoteDate, ... lineItems, totals: { subTotal, taxTotal, total } }
export interface CreateQuoteRequest { customerId, jobId, quoteDate, title?, notes? }
```

Phase 41 locked estimate status values (D-TXN-01):
```typescript
"draft" | "sent" | "viewed" | "responded" | "site_visit_requested" | "converted" | "declined" | "expired" | "lost" | "deleted"
```

Phase 41 locked priceRange shape (D-CONT-05, D-CONT-06):
```typescript
priceRange: {
  low:  { subTotal: number; taxTotal: number; total: number };
  high: { subTotal: number; taxTotal: number; total: number };
}
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Create src/types/estimate.ts with all estimate type exports</name>
  <files>trade-flow-ui/src/types/estimate.ts, trade-flow-ui/src/types/index.ts</files>
  <read_first>
    - trade-flow-ui/src/types/quote.ts (exact shape to mirror — do NOT import from it)
    - trade-flow-ui/src/types/index.ts (barrel pattern)
    - trade-flow-ui/src/types/api.types.ts (for `ItemType` import if needed)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-MIR-06, D-MIR-07, D-MIR-08
  </read_first>
  <behavior>
    - `EstimateStatus` exported as a string literal union matching Phase 41 D-TXN-01 exactly
    - `EstimateLineItem` interface mirrors `QuoteLineItem` structurally (independent definition, not a re-export)
    - `Estimate` interface matches Phase 41 `IEstimateDto` with `totals`, `priceRange`, `contingencyPercent`, `displayMode`, `uncertaintyReasons?`, `uncertaintyNotes?`, `responseSummary: IEstimateResponseSummaryDto | null`
    - `CreateEstimateRequest` includes `customerId`, `jobId`, `estimateDate`, `notes?`, `contingencyPercent`, `displayMode`, `uncertaintyReasons?`, `uncertaintyNotes?`
    - `UpdateEstimateRequest` covers the same mutable fields as Partial
    - Barrel `src/types/index.ts` re-exports all five estimate types
  </behavior>
  <action>
Create NEW file `trade-flow-ui/src/types/estimate.ts` with the exact content below. This file MUST NOT import from `./quote` — Estimate types are fully independent per D-MIR-01 and D-MIR-06.

```typescript
import type { ItemType } from "./api.types";

export type EstimateStatus =
  | "draft"
  | "sent"
  | "viewed"
  | "responded"
  | "site_visit_requested"
  | "converted"
  | "declined"
  | "expired"
  | "lost"
  | "deleted";

export type EstimateDisplayMode = "range" | "from";

export type UncertaintyReason =
  | "site_inspection"
  | "hidden_conditions"
  | "materials_supply"
  | "access_working_space"
  | "scope_unclear";

export interface EstimateLineItem {
  id: string;
  itemId: string;
  quantity: number;
  unit: string;
  unitPrice: number;
  originalUnitPrice: number;
  lineTotal: number;
  originalLineTotal: number;
  discountAmount: number;
  taxRate: number;
  type: ItemType;
  status: string;
  components?: EstimateLineItem[];
}

export interface EstimateMoneyBreakdown {
  subTotal: number;
  taxTotal: number;
  total: number;
}

export interface EstimatePriceRange {
  low: EstimateMoneyBreakdown;
  high: EstimateMoneyBreakdown;
}

export interface EstimateResponseSummary {
  responseType: string;
  respondedAt: string;
  message?: string;
  declineReason?: string;
}

export interface Estimate {
  id: string;
  businessId: string;
  customerId: string;
  jobId: string;
  number: string;
  title: string;
  notes?: string;
  status: EstimateStatus;
  estimateDate: string;
  sentAt?: string;
  viewedAt?: string;
  respondedAt?: string;
  deletedAt?: string;
  updatedAt: string;
  customerName: string;
  jobTitle: string;
  contingencyPercent: number;
  displayMode: EstimateDisplayMode;
  uncertaintyReasons?: UncertaintyReason[];
  uncertaintyNotes?: string;
  lineItems: EstimateLineItem[];
  totals: EstimateMoneyBreakdown;
  priceRange: EstimatePriceRange;
  responseSummary: EstimateResponseSummary | null;
}

export interface CreateEstimateRequest {
  customerId: string;
  jobId: string;
  estimateDate: string;
  title?: string;
  notes?: string;
  contingencyPercent: number;
  displayMode: EstimateDisplayMode;
  uncertaintyReasons?: UncertaintyReason[];
  uncertaintyNotes?: string;
}

export interface UpdateEstimateRequest {
  title?: string;
  notes?: string;
  contingencyPercent?: number;
  displayMode?: EstimateDisplayMode;
  uncertaintyReasons?: UncertaintyReason[];
  uncertaintyNotes?: string;
  customerId?: string;
  jobId?: string;
}
```

Then EDIT `trade-flow-ui/src/types/index.ts` to add a new line at the bottom that re-exports the estimate types. Append after the existing `from "./quote"` line:

```typescript
export type {
  Estimate,
  EstimateLineItem,
  EstimateStatus,
  EstimateDisplayMode,
  EstimateMoneyBreakdown,
  EstimatePriceRange,
  EstimateResponseSummary,
  UncertaintyReason,
  CreateEstimateRequest,
  UpdateEstimateRequest,
} from "./estimate";
```

Do NOT touch any other existing re-exports. Do NOT unionise `Estimate` with `Quote`.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/types/estimate.ts` exists
    - `grep -c "export type EstimateStatus" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"site_visit_requested\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"site_inspection\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"hidden_conditions\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"materials_supply\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"access_working_space\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "\"scope_unclear\"" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "priceRange: EstimatePriceRange" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "export interface Estimate " trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "export interface CreateEstimateRequest" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "export interface UpdateEstimateRequest" trade-flow-ui/src/types/estimate.ts` returns at least 1
    - `grep -c "from \"./estimate\"" trade-flow-ui/src/types/index.ts` returns at least 1
    - `grep -c "from \"./quote\"" trade-flow-ui/src/types/estimate.ts` returns 0 (zero cross-imports)
    - `cd trade-flow-ui && npm run typecheck` exits 0
  </acceptance_criteria>
  <done>Estimate types exist as an independent module, are re-exported from the barrel, and typecheck passes.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Update CONT-04 and add SMART-04 in REQUIREMENTS.md</name>
  <files>.planning/REQUIREMENTS.md</files>
  <read_first>
    - .planning/REQUIREMENTS.md (to see current CONT-04 wording and the Future Requirements section where SMART-04 will be added)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-CHIP-01 through D-CHIP-06
  </read_first>
  <action>
EDIT `.planning/REQUIREMENTS.md`:

**Change 1 — replace CONT-04** (currently line ~28). Replace the current single line:

```
- [ ] **CONT-04**: User can add optional uncertainty notes via quick-tap preset reasons (site inspection, pipework condition, materials, access) plus freeform text
```

with:

```
- [ ] **CONT-04**: User can add optional uncertainty reasons via five quick-tap preset chips — Site inspection needed (`site_inspection`), Hidden conditions (`hidden_conditions`), Materials & supply (`materials_supply`), Access & working space (`access_working_space`), Scope unclear until investigation (`scope_unclear`) — plus a freeform notes textarea. The five chip values are trade-agnostic and cover all ten `BusinessTrade` values without requiring per-trade dispatch.
```

**Change 2 — add SMART-04** in the "Smart Contingency Defaults (v1.9+)" Future section (currently lines ~99-103). Append a new bullet after the existing SMART-03:

```
- **SMART-04**: Trade-specific uncertainty chip presets per `BusinessTrade`. Plumber surfaces pipework-specific chips, electrician surfaces wiring/consumer-unit chips, carpenter surfaces timber-rot chips, builder surfaces structural/foundations chips, etc. Phase 43 ships five trade-agnostic chips as the v1.8 baseline; SMART-04 introduces trade-aware dispatch once content design bandwidth is available.
```

Do NOT touch any other requirements. Do NOT change existing SMART-01, SMART-02, SMART-03.
  </action>
  <verify>
    <automated>grep -c "site_inspection\|hidden_conditions\|materials_supply\|access_working_space\|scope_unclear" .planning/REQUIREMENTS.md</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "SMART-04" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "Trade-specific uncertainty chip presets per" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "CONT-04.*site_inspection.*hidden_conditions" .planning/REQUIREMENTS.md` returns at least 1 (or inspect manually that all five chip values are in CONT-04 line)
    - `grep -c "pipework condition" .planning/REQUIREMENTS.md` returns 0 (old wording removed)
    - `grep -c "site_inspection" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "hidden_conditions" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "materials_supply" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "access_working_space" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "scope_unclear" .planning/REQUIREMENTS.md` returns at least 1
    - `grep -c "SMART-01" .planning/REQUIREMENTS.md` returns at least 1 (existing line preserved)
  </acceptance_criteria>
  <done>CONT-04 lists the five new chip values with their stored identifiers, SMART-04 appears in the Future section, and the old four-chip plumber-biased wording is removed.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 3: Sync ROADMAP.md and v1.8-ROADMAP.md Phase 43 success criterion #1</name>
  <files>.planning/ROADMAP.md, .planning/milestones/v1.8-ROADMAP.md</files>
  <read_first>
    - .planning/ROADMAP.md (find "### Phase 43:" block — current SC #1 mentions "pipework, materials, access")
    - .planning/milestones/v1.8-ROADMAP.md (find Phase 43 section — same wording)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decision D-CHIP-01
  </read_first>
  <action>
EDIT both roadmap files so Phase 43 Success Criterion #1 matches the new chip list exactly.

**In `.planning/ROADMAP.md`**: under `### Phase 43: Estimate Frontend CRUD`, replace the existing SC #1 line:

```
  1. The shared Create Document dialog displays a Quote/Estimate toggle; selecting Estimate reveals the ContingencySlider (0-30% in 5% steps, default 10%), the display-mode toggle (range / "from £X"), and the quick-tap uncertainty chips (site inspection, pipework, materials, access) plus a freeform notes field, and the submitted estimate appears in the list.
```

with:

```
  1. The shared Create Document dialog displays a Quote/Estimate toggle; selecting Estimate reveals the ContingencySlider (0-30% in 5% steps, default 10%), the display-mode toggle (range / "from £X"), the five trade-agnostic uncertainty chips (site inspection needed, hidden conditions, materials & supply, access & working space, scope unclear until investigation) plus a freeform notes field, and the submitted estimate appears in the list.
```

**In `.planning/milestones/v1.8-ROADMAP.md`**: under `### Phase 43: Estimate Frontend CRUD`, replace the equivalent SC #1 line with the same new wording (five trade-agnostic chip list).

Do not change any other phase, any other success criterion, or any surrounding structure. Do not touch dependency arrows or requirement mappings.
  </action>
  <verify>
    <automated>grep -c "five trade-agnostic uncertainty chips" .planning/ROADMAP.md .planning/milestones/v1.8-ROADMAP.md</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "five trade-agnostic uncertainty chips" .planning/ROADMAP.md` returns at least 1
    - `grep -c "five trade-agnostic uncertainty chips" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
    - `grep -c "site inspection, pipework, materials, access" .planning/ROADMAP.md` returns 0 (old wording removed)
    - `grep -c "site inspection, pipework, materials, access" .planning/milestones/v1.8-ROADMAP.md` returns 0 (old wording removed)
    - `grep -c "hidden conditions" .planning/ROADMAP.md` returns at least 1
    - `grep -c "scope unclear until investigation" .planning/ROADMAP.md` returns at least 1
    - `grep -c "hidden conditions" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
    - `grep -c "scope unclear until investigation" .planning/milestones/v1.8-ROADMAP.md` returns at least 1
  </acceptance_criteria>
  <done>Both roadmap files reference the five new chip values in Phase 43 SC #1 and neither contains the obsolete four-chip wording.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| none | This plan only defines TypeScript types and edits markdown docs — no untrusted input, no runtime code paths, no network boundary |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-01-01 | Information disclosure | src/types/estimate.ts drifts from Phase 41 IEstimateDto | mitigate | Field-for-field mirror of Phase 41 D-CONT-05/06/07 locked in decisions; plan 05 fixture golden-file test catches runtime drift |
| T-43-01-02 | Tampering | Doc edits leave obsolete four-chip wording behind and produce confusing contract | mitigate | Explicit grep acceptance criteria on both roadmap files assert old wording returns 0 matches |
</threat_model>

<verification>
Run `cd trade-flow-ui && npm run typecheck` — must exit 0.

Run `cd trade-flow-ui && npm run lint && npm run format:check` — must exit 0 for any file this plan touches.

Grep sanity: none of the old four-chip wording survives anywhere except in version history.
</verification>

<success_criteria>
- `trade-flow-ui/src/types/estimate.ts` exists and exports all 10 listed type names
- `trade-flow-ui/src/types/index.ts` barrel re-exports estimate types
- `npm run typecheck` passes in trade-flow-ui
- REQUIREMENTS.md CONT-04 references all five chip stored values
- REQUIREMENTS.md Future section has SMART-04 entry
- ROADMAP.md and v1.8-ROADMAP.md Phase 43 SC #1 match the new chip list
- No `as` casts, no `@ts-ignore` / `@ts-expect-error` / `@ts-nocheck`, no `eslint-disable`
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-01-SUMMARY.md`
</output>
