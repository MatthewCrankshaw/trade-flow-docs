---
phase: 43-estimate-frontend-crud
plan: 05
type: execute
wave: 3
depends_on:
  - 43-01
  - 43-02
  - 43-03
  - 43-04
files_modified:
  - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesCardSkeleton.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesTableSkeleton.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx
  - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx
  - trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx
  - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx
  - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
  - trade-flow-ui/src/features/estimates/components/index.ts
  - trade-flow-ui/src/pages/EstimatesPage.tsx
autonomous: true
requirements:
  - CONT-03
  - CONT-04

must_haves:
  truths:
    - "EstimatesTable renders rows with customer, job, E-YYYY-NNN number, status badge, and formatRange-formatted price"
    - "EstimatesDataView switches between EstimatesTable (desktop) and EstimatesCardList (mobile)"
    - "EstimatesPage renders 7 grouped status tabs (All / Draft / Sent / Responded / Converted / Declined / Expired-Lost), client-side filtered, deleted excluded"
    - "EstimateLineItemsCard gates edit on status === 'draft' (narrower than quotes)"
    - "EstimateActionStrip exists as a file (may be a minimal stub) so Phase 44 has a place to hang SendEstimateDialog"
  artifacts:
    - path: "trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx"
      provides: "Desktop table with formatRange totals column"
      contains: "formatRange"
    - path: "trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx"
      provides: "Responsive switcher"
      contains: "useMediaQuery"
    - path: "trade-flow-ui/src/pages/EstimatesPage.tsx"
      provides: "List page with 7 grouped tabs + prerequisite alert + CreateDocumentDialog"
      contains: "CreateDocumentDialog"
    - path: "trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx"
      provides: "Line-item container with draft-only edit gate"
      contains: "isEditable"
  key_links:
    - from: "trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx"
      to: "trade-flow-ui/src/lib/currency.ts formatRange"
      via: "totals column cell"
      pattern: "formatRange\\("
    - from: "trade-flow-ui/src/pages/EstimatesPage.tsx"
      to: "useGetEstimatesQuery + CreateDocumentDialog"
      via: "RTK Query hook + dialog component"
      pattern: "useGetEstimatesQuery"
---

<objective>
Mirror the entire quote list-and-line-items component surface into `features/estimates/` — nine components plus the `EstimatesPage` list route — so traders can view, filter, and navigate into estimates.

Purpose: This plan delivers Success Criteria #2 (list page renders estimates with tabs + formatRange price) and #4 (formatRange is actually called on the list cells and matches the API shape to the penny).

Output: Nine new components under `features/estimates/components/`, updated barrel, new `EstimatesPage` under `src/pages/`.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@trade-flow-ui/src/features/quotes/components/QuotesTable.tsx
@trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx
@trade-flow-ui/src/features/quotes/components/QuotesCardSkeleton.tsx
@trade-flow-ui/src/features/quotes/components/QuotesTableSkeleton.tsx
@trade-flow-ui/src/features/quotes/components/QuotesDataView.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
@trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
@trade-flow-ui/src/pages/QuotesPage.tsx
@trade-flow-ui/src/features/estimates/api/estimateApi.ts
@trade-flow-ui/src/lib/currency.ts
@trade-flow-ui/src/components/PrerequisiteAlert.tsx
@trade-flow-ui/src/hooks/useCurrency.ts
@trade-flow-ui/src/hooks/useCurrentBusiness.ts

<interfaces>
<!-- Locked decisions specific to this plan -->

Phase 43 D-TAB-01 — seven grouped tabs (not one per status):
```
  1. "all"           → every estimate except deleted
  2. "draft"         → draft
  3. "sent"          → sent + viewed
  4. "responded"     → responded + site_visit_requested
  5. "converted"     → converted
  6. "declined"      → declined
  7. "expired_lost"  → expired + lost
```

Phase 43 D-BADGE-01/02 — local statusColors map per page, NOT a shared helper:
```typescript
const statusColors: Record<EstimateStatus, "default" | "secondary" | "destructive" | "outline"> = {
  draft: "secondary",
  sent: "outline",
  viewed: "outline",
  responded: "default",
  site_visit_requested: "default",
  converted: "default",
  declined: "destructive",
  expired: "secondary",
  lost: "destructive",
  deleted: "destructive",
};
```

Phase 43 D-EDIT-02 — EstimateLineItemsCard edit gate is narrower than quote's:
```typescript
const isEditable = estimate.status === "draft"; // NOT "draft" || "sent"
```

Phase 43 D-FMT-01/03 — formatRange signature (from Plan 02):
```typescript
formatRange(
  low: MoneyBreakdown,
  high: MoneyBreakdown,
  mode: "range" | "from",
  currencyCode: SupportedCurrencyCode,
): string
```
Called by EstimatesTable, EstimatesCardList, and (Plan 06) EstimateDetailPage.

From trade-flow-ui/src/features/estimates/api/estimateApi.ts:
```typescript
useGetEstimatesQuery(businessId: string) → { data: Estimate[], isLoading, ... }
useDeleteEstimateMutation() → [deleteEstimate, { isLoading }]
useAddEstimateLineItemMutation() → [addLineItem, { isLoading }]
useUpdateEstimateLineItemMutation()
useDeleteEstimateLineItemMutation()
```

From trade-flow-ui/src/types/estimate.ts:
```typescript
Estimate.priceRange: { low: MoneyBreakdown; high: MoneyBreakdown }
Estimate.displayMode: "range" | "from"
Estimate.status: EstimateStatus
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Mirror QuotesTable/CardList/DataView/Skeletons into features/estimates/components</name>
  <files>trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx, trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx, trade-flow-ui/src/features/estimates/components/EstimatesCardSkeleton.tsx, trade-flow-ui/src/features/estimates/components/EstimatesTableSkeleton.tsx, trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx</files>
  <read_first>
    - trade-flow-ui/src/features/quotes/components/QuotesTable.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuotesCardSkeleton.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuotesTableSkeleton.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuotesDataView.tsx (full file)
    - trade-flow-ui/src/lib/currency.ts (for formatRange signature — call it directly, do NOT reimplement totals)
    - trade-flow-ui/src/hooks/useCurrency.ts (for useBusinessCurrency — returns `currencyCode: "GBP"` which formatRange consumes)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md D-BADGE-01/02, D-FMT-03, D-MIR-01/02
  </read_first>
  <action>
Mirror five files from `features/quotes/components/` into `features/estimates/components/` with these surgical changes:

**EstimatesTable.tsx** — based on QuotesTable.tsx. Changes:
1. Rename `Quote` → `Estimate`, `QuoteStatus` → `EstimateStatus`, `quote` → `estimate`, `quotes` → `estimates`, `onDeleteQuote` → `onDeleteEstimate`
2. Replace the local `statusColors` map with the 10-entry EstimateStatus map listed in the interfaces block above (copy verbatim)
3. Table columns: `Estimate #` | Job (sm+) | Customer (md+) | Status | Price Range (right-aligned) | Date (lg+) | menu
4. Route navigation target is `/estimates/${estimate.id}` (NOT `/quotes/...`)
5. Price column: instead of `currency.formatAmount(quote.totals.total)`, call:
   ```typescript
   import { formatRange } from "@/lib/currency";
   import { useBusinessCurrency } from "@/hooks/useCurrency";
   // inside component:
   const { currencyCode } = useBusinessCurrency();
   // in the cell:
   <TableCell className="text-right">
     {formatRange(estimate.priceRange.low, estimate.priceRange.high, estimate.displayMode, currencyCode)}
   </TableCell>
   ```
6. Date column reads `estimate.estimateDate` (not `quoteDate`), still via `DateTime.fromISO(...).toFormat("d MMM yyyy")`

**EstimatesCardList.tsx** — based on QuotesCardList.tsx. Same renames + formatRange swap in the mobile card total area. Status badge uses the local 10-entry map.

**EstimatesCardSkeleton.tsx** — based on QuotesCardSkeleton.tsx. Simple renames; structural skeletons unchanged. Export as `EstimatesCardSkeleton`.

**EstimatesTableSkeleton.tsx** — based on QuotesTableSkeleton.tsx. Same.

**EstimatesDataView.tsx** — based on QuotesDataView.tsx:
```typescript
import { useMediaQuery } from "@/hooks/useMediaQuery";
import { EstimatesTable } from "./EstimatesTable";
import { EstimatesCardList } from "./EstimatesCardList";
import { EstimatesTableSkeleton } from "./EstimatesTableSkeleton";
import { EstimatesCardSkeleton } from "./EstimatesCardSkeleton";
import type { Estimate } from "@/types";

interface EstimatesDataViewProps {
  estimates: Estimate[];
  isLoading: boolean;
  onDeleteEstimate: (estimate: Estimate) => void;
}

export function EstimatesDataView({ estimates, isLoading, onDeleteEstimate }: EstimatesDataViewProps) {
  const isDesktop = useMediaQuery("(min-width: 768px)");
  if (isLoading) {
    return isDesktop ? <EstimatesTableSkeleton /> : <EstimatesCardSkeleton />;
  }
  return isDesktop ? (
    <EstimatesTable estimates={estimates} onDeleteEstimate={onDeleteEstimate} />
  ) : (
    <EstimatesCardList estimates={estimates} onDeleteEstimate={onDeleteEstimate} />
  );
}
```

Do NOT cross-import from `@/features/quotes` anywhere. Every symbol is re-imported independently. No `as` casts.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/features/estimates</automated>
  </verify>
  <acceptance_criteria>
    - Files `EstimatesTable.tsx`, `EstimatesCardList.tsx`, `EstimatesCardSkeleton.tsx`, `EstimatesTableSkeleton.tsx`, `EstimatesDataView.tsx` all exist under `trade-flow-ui/src/features/estimates/components/`
    - `grep -c "formatRange" trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` returns at least 1
    - `grep -c "formatRange" trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx` returns at least 1
    - `grep -c "/estimates/" trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` returns at least 1
    - `grep -c "draft\|sent\|viewed\|responded\|site_visit_requested\|converted\|declined\|expired\|lost\|deleted" trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` returns at least 10
    - `grep -rc "from \"@/features/quotes\"\|from \"@/features/quotes/" trade-flow-ui/src/features/estimates/components/` returns 0 (no cross-imports)
    - `grep -rc "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/features/estimates/components/` returns 0
    - `cd trade-flow-ui && npm run typecheck` exits 0
  </acceptance_criteria>
  <done>Five list components mirrored with formatRange integration and typecheck green.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Mirror EstimateLineItemsCard/Table/CardList + EstimateActionStrip stub</name>
  <files>trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx, trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx, trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx, trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx, trade-flow-ui/src/features/estimates/components/index.ts</files>
  <read_first>
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx (full file)
    - trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx (full file — this is the 234-line action strip the stub will later replace)
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts (for useAddEstimateLineItemMutation, useUpdateEstimateLineItemMutation, useDeleteEstimateLineItemMutation)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md D-EDIT-02/03/06, D-MIR-02/03
  </read_first>
  <action>
**EstimateLineItemsCard.tsx** — based on QuoteLineItemsCard.tsx with these surgical changes:
1. Rename `Quote` → `Estimate`, `quote` → `estimate`, `useAddLineItemMutation` → `useAddEstimateLineItemMutation`
2. Change the edit gate (narrower than quotes):
   ```typescript
   const isEditable = estimate.status === "draft";
   ```
   (Quote's version is `"draft" || "sent"` — estimate narrows to draft-only per D-EDIT-02)
3. Pass `estimate` + `businessId` props, same structure
4. Render `<EstimateLineItemsTable>` on desktop and `<EstimateLineItemsCardList>` on mobile
5. Keep `SearchableItemPicker` integration unchanged
6. On add, call `addLineItem({ businessId, estimateId: estimate.id, itemId: item.id, quantity: 1 })`

**EstimateLineItemsTable.tsx** — based on QuoteLineItemsTable.tsx. Rename hooks:
- `useUpdateLineItemMutation` → `useUpdateEstimateLineItemMutation`
- `useDeleteLineItemMutation` → `useDeleteEstimateLineItemMutation`
- All `quoteId` payload fields → `estimateId`
- Type imports from `@/types` for `EstimateLineItem` / `Estimate`

**EstimateLineItemsCardList.tsx** — same treatment as above: rename mutation hooks, type imports, quoteId → estimateId.

**EstimateActionStrip.tsx** — this is a stub for Phase 43 per D-MIR-02 discretion note. Minimum viable: a file that exports a component Phase 44 can extend. Concrete shape:

```typescript
import { Button } from "@/components/ui/button";
import { Trash2 } from "lucide-react";

import type { Estimate } from "@/types";

interface EstimateActionStripProps {
  estimate: Estimate;
  businessId: string;
  customerEmail?: string;
  customerName?: string;
  onDelete: () => void;
}

export function EstimateActionStrip({ estimate, onDelete }: EstimateActionStripProps) {
  const canDelete = estimate.status === "draft";
  if (!canDelete) {
    return null;
  }
  return (
    <div className="flex flex-wrap items-center gap-2">
      <Button type="button" variant="destructive" size="sm" onClick={onDelete}>
        <Trash2 className="mr-1 h-4 w-4" />
        Delete
      </Button>
    </div>
  );
}
```

Phase 44 will replace this body with Send/Convert/MarkLost wiring. Phase 43's minimum is that the file exists, compiles, and renders the Delete button on Draft estimates.

**Update `features/estimates/components/index.ts`** to export every new component:

```typescript
export * from "./ContingencySlider";
export * from "./UncertaintyChipGroup";
export * from "./CreateEstimateForm";
export * from "./EstimatesTable";
export * from "./EstimatesCardList";
export * from "./EstimatesCardSkeleton";
export * from "./EstimatesTableSkeleton";
export * from "./EstimatesDataView";
export * from "./EstimateLineItemsCard";
export * from "./EstimateLineItemsTable";
export * from "./EstimateLineItemsCardList";
export * from "./EstimateActionStrip";
```

Run `cd trade-flow-ui && npm run typecheck && npm run lint`. Fix any issues at the root cause.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/features/estimates</automated>
  </verify>
  <acceptance_criteria>
    - Files `EstimateLineItemsCard.tsx`, `EstimateLineItemsTable.tsx`, `EstimateLineItemsCardList.tsx`, `EstimateActionStrip.tsx` exist under `trade-flow-ui/src/features/estimates/components/`
    - `grep -c "estimate.status === \"draft\"" trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx` returns at least 1 (and returns 0 matches for the pattern `\"draft\" || .*\"sent\"` — narrower gate)
    - `grep -c "useAddEstimateLineItemMutation" trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx` returns at least 1
    - `grep -c "useUpdateEstimateLineItemMutation" trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx` returns at least 1
    - `grep -c "useDeleteEstimateLineItemMutation" trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx` returns at least 1
    - `grep -c "EstimateActionStrip" trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` returns at least 1
    - `grep -c "export \\* from \"./Estimate" trade-flow-ui/src/features/estimates/components/index.ts` returns at least 9
    - `grep -rc "from \"@/features/quotes\"\|from \"@/features/quotes/" trade-flow-ui/src/features/estimates/components/` returns 0
    - `grep -rc "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/features/estimates/components/` returns 0
    - `cd trade-flow-ui && npm run typecheck` exits 0
  </acceptance_criteria>
  <done>All nine estimate components mirrored, draft-only edit gate enforced, EstimateActionStrip stub exists and renders Delete on Draft, barrel exports everything.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 3: Build EstimatesPage with 7 grouped tabs, prerequisite alert, and CreateDocumentDialog</name>
  <files>trade-flow-ui/src/pages/EstimatesPage.tsx</files>
  <read_first>
    - trade-flow-ui/src/pages/QuotesPage.tsx (full file — structural template)
    - trade-flow-ui/src/features/estimates/index.ts (to see exported hooks/components)
    - trade-flow-ui/src/components/CreateDocumentDialog.tsx (from plan 04)
    - trade-flow-ui/src/components/PrerequisiteAlert.tsx
    - trade-flow-ui/src/hooks/useCurrentBusiness.ts
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md D-TAB-01/02/03, D-ROUTE-01, D-EDIT-04
  </read_first>
  <action>
Create NEW file `trade-flow-ui/src/pages/EstimatesPage.tsx`. Structure mirrors `QuotesPage.tsx` with these required differences:

1. Use `useGetEstimatesQuery`, `useDeleteEstimateMutation`, `EstimatesDataView` from `@/features/estimates`
2. Import `CreateDocumentDialog` from `@/components/CreateDocumentDialog` (NOT the old CreateQuoteDialog) and render it with `defaultType="estimate"`
3. Define the tab union type and the tab mapping table verbatim:

```typescript
import type { EstimateStatus } from "@/types";

type EstimateTabValue = "all" | "draft" | "sent" | "responded" | "converted" | "declined" | "expired_lost";

const TAB_STATUS_GROUPS: Record<EstimateTabValue, readonly EstimateStatus[]> = {
  all: ["draft", "sent", "viewed", "responded", "site_visit_requested", "converted", "declined", "expired", "lost"],
  draft: ["draft"],
  sent: ["sent", "viewed"],
  responded: ["responded", "site_visit_requested"],
  converted: ["converted"],
  declined: ["declined"],
  expired_lost: ["expired", "lost"],
};
```

Note that `"deleted"` is NOT in any group — deleted estimates are never rendered per D-TAB-03.

4. Client-side filtering:

```typescript
const filteredEstimates = useMemo(() => {
  const visible = estimates.filter((e) => e.status !== "deleted");
  const allowed = TAB_STATUS_GROUPS[activeTab];
  return visible.filter((e) => allowed.includes(e.status));
}, [estimates, activeTab]);
```

5. Render the seven tabs with the labels:

```jsx
<Tabs value={activeTab} onValueChange={(v) => setActiveTab(v as EstimateTabValue)}>
  <TabsList>
    <TabsTrigger value="all">All</TabsTrigger>
    <TabsTrigger value="draft">Draft</TabsTrigger>
    <TabsTrigger value="sent">Sent</TabsTrigger>
    <TabsTrigger value="responded">Responded</TabsTrigger>
    <TabsTrigger value="converted">Converted</TabsTrigger>
    <TabsTrigger value="declined">Declined</TabsTrigger>
    <TabsTrigger value="expired_lost">Expired/Lost</TabsTrigger>
  </TabsList>
</Tabs>
```

6. Heading: `<h1>Estimates</h1>` + subtitle `"Create and manage rough-price estimates for your customers"`

7. New button labelled `"New Estimate"` with `<Plus />` icon, disabled on missing prerequisites, triggers `setCreateOpen(true)`

8. Render the create dialog:

```jsx
{businessId && (
  <CreateDocumentDialog
    open={createOpen}
    onOpenChange={setCreateOpen}
    businessId={businessId}
    defaultType="estimate"
  />
)}
```

9. Delete confirmation AlertDialog — mirror QuotesPage exactly but with "Delete {estimateToDelete?.number}?" and `deleteEstimate({ businessId, estimateId: estimateToDelete.id })`. Toast success: `"Estimate deleted"`; toast error: `"Failed to delete estimate"`.

10. Wrap in `<DashboardLayout user={user} isLoading={isLoading}>` with an inner `EstimatesContent()` function exactly like QuotesPage.

Do NOT touch `App.tsx` (routing is Plan 06). Do NOT touch navigation.ts (sidebar is Plan 06). This plan only creates the page file — Plan 06 wires it into the router.

Run `cd trade-flow-ui && npm run ci` — everything must pass.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run ci</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/pages/EstimatesPage.tsx` exists
    - `grep -c "useGetEstimatesQuery\|useDeleteEstimateMutation\|EstimatesDataView" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 3
    - `grep -c "CreateDocumentDialog" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 1
    - `grep -c "defaultType=\"estimate\"" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 1
    - `grep -c "TAB_STATUS_GROUPS" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 2
    - `grep -c "\"expired_lost\"" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 2
    - `grep -c "\"responded\"\|\"site_visit_requested\"" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 2
    - `grep -c "\"deleted\"" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 1 (in the exclusion filter, NOT in TAB_STATUS_GROUPS)
    - `grep -c "<TabsTrigger value=\"all\">All</TabsTrigger>" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 1
    - `grep -c "New Estimate\|Estimate deleted\|Failed to delete estimate" trade-flow-ui/src/pages/EstimatesPage.tsx` returns at least 3
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/pages/EstimatesPage.tsx` returns 0
    - `cd trade-flow-ui && npm run ci` exits 0
  </acceptance_criteria>
  <done>EstimatesPage renders the 7 grouped tabs with correct groupings, wires CreateDocumentDialog with defaultType="estimate", handles delete confirm, passes the full CI gate.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| user→client | Tab selection, delete confirm action |
| client→API | useGetEstimatesQuery and useDeleteEstimateMutation hit `/v1/estimates/*` (same guards as Plan 03) |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-05-01 | Information disclosure | Deleted estimates leak into list view | mitigate | EstimatesPage filter explicitly excludes `status === "deleted"` AND Phase 41 D-DRAFT-03 list endpoint already excludes them server-side — defence in depth |
| T-43-05-02 | Tampering | URL manipulation navigates to another business's estimate detail | mitigate | Navigation is only to `/estimates/:id` which Plan 06's EstimateDetailPage hits `/v1/estimates/:id` — Phase 41 policy enforces ownership server-side |
| T-43-05-03 | Tampering | Optimistic delete leaves stale cache after server rejection | mitigate | RTK Query onQueryStarted (Plan 03) rolls back patch on failure; tag invalidation forces refetch |
| T-43-05-04 | Information disclosure | formatRange discloses priceRange to users who shouldn't see it | accept | Any authenticated trader viewing `/estimates` already has ownership scope — the risk is equivalent to viewing quotes, no new vector |
| T-43-05-05 | Denial of service | Client-side filter on large estimate list | accept | Matches existing QuotesPage behaviour; flagged as latent concern for v1.9+ in 43-CONTEXT.md |
</threat_model>

<verification>
`cd trade-flow-ui && npm run ci` must exit 0 at the end of this plan.

Interface-integration check: after this plan, Plan 06 can write `<Route path="/estimates" element={<EstimatesPage />} />` in App.tsx without errors.
</verification>

<success_criteria>
- 9 new components under features/estimates/components/
- EstimatesPage at src/pages/EstimatesPage.tsx with 7 grouped tabs
- EstimatesTable and EstimatesCardList call `formatRange` for the price cell
- EstimateLineItemsCard narrow edit gate (draft only)
- `npm run ci` passes
- No suppressions / cross-imports
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-05-SUMMARY.md`
</output>
