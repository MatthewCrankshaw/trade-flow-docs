---
phase: 43-estimate-frontend-crud
plan: "05"
subsystem: trade-flow-ui/features/estimates + trade-flow-ui/pages
tags: [estimates, react, components, list, page, frontend]
dependency_graph:
  requires:
    - 43-01 (Estimate, EstimateStatus, EstimateLineItem types)
    - 43-03 (useGetEstimatesQuery, useDeleteEstimateMutation, useAddEstimateLineItemMutation, useUpdateEstimateLineItemMutation, useDeleteEstimateLineItemMutation)
    - 43-04 (CreateDocumentDialog, CreateEstimateForm)
  provides:
    - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesCardSkeleton.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesTableSkeleton.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
    - trade-flow-ui/src/features/estimates/components/index.ts (12 exports)
    - trade-flow-ui/src/pages/EstimatesPage.tsx
  affects:
    - Plan 43-06 (EstimateDetailPage can import EstimateLineItemsCard, EstimateActionStrip, EstimatesDataView)
    - Plan 44 (EstimateActionStrip stub is the extension point for Send/Convert/MarkLost)
tech_stack:
  added: []
  patterns:
    - Responsive DataView pattern (useMediaQuery desktop/mobile switch)
    - formatRange for price range display (not formatAmount)
    - Draft-only edit gate (narrower than quote's draft||sent)
    - 7 grouped status tabs via TAB_STATUS_GROUPS mapping
    - EstimateActionStrip stub pattern (minimal now, Phase 44 extends)
key_files:
  created:
    - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesCardSkeleton.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesTableSkeleton.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
    - trade-flow-ui/src/pages/EstimatesPage.tsx
  modified:
    - trade-flow-ui/src/features/estimates/components/index.ts (12 exports added)
    - trade-flow-ui/src/components/PrerequisiteAlert.tsx (estimates context added)
    - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx (Prettier formatting fix)
decisions:
  - "EstimateLineItemsCard edit gate is draft-only (not draft||sent) per D-EDIT-02 — narrower than QuoteLineItemsCard"
  - "EstimatesDataView includes empty state (mirrors QuotesDataView) even though plan spec showed minimal version"
  - "PrerequisiteAlert extended with estimates context type (Rule 2 auto-fix) — the type union was too narrow"
  - "EstimateActionStrip is a minimal stub with only Delete on draft; Phase 44 extends it with Send/Convert/MarkLost"
metrics:
  duration: "15 minutes"
  completed: "2026-04-13"
  tasks_completed: 3
  files_changed: 12
---

# Phase 43 Plan 05: Estimate List Components and Page Summary

**One-liner:** Nine estimate list/line-item components mirrored from quotes with formatRange price columns, draft-only edit gate, and EstimatesPage with 7 grouped status tabs and CreateDocumentDialog wired to defaultType=estimate.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Mirror EstimatesTable/CardList/DataView/Skeletons with formatRange | f2d21ac (ui) | 5 new component files |
| 2 | Mirror EstimateLineItemsCard/Table/CardList + EstimateActionStrip stub + barrel | 8737dd6 (ui) | 5 new files, index.ts updated |
| 3 | Build EstimatesPage with 7 grouped tabs + CreateDocumentDialog + PrerequisiteAlert fix | 0d3228a (ui) | EstimatesPage.tsx, PrerequisiteAlert.tsx, EstimatesTable.tsx (format fix) |

## What Was Built

**Task 1 — Five list/view components** (`trade-flow-ui/src/features/estimates/components/`):

- `EstimatesTable.tsx`: Desktop table with 10-entry `statusColors` map (all EstimateStatus values), formatRange price column calling `formatRange(estimate.priceRange.low, estimate.priceRange.high, estimate.displayMode, currencyCode)`, navigates to `/estimates/:id`
- `EstimatesCardList.tsx`: Mobile card list with same 10-entry statusColors map, formatRange in the price display area
- `EstimatesCardSkeleton.tsx` / `EstimatesTableSkeleton.tsx`: Structural skeletons renamed from quotes equivalents
- `EstimatesDataView.tsx`: Responsive switcher using `useMediaQuery("(min-width: 768px)")`, includes empty state ("No estimates found")

**Task 2 — Four line-item components + stub** (`trade-flow-ui/src/features/estimates/components/`):

- `EstimateLineItemsCard.tsx`: Line-item container with `isEditable = estimate.status === "draft"` (draft-only per D-EDIT-02); calls `useAddEstimateLineItemMutation` with `estimateId` payload
- `EstimateLineItemsTable.tsx`: Desktop line-item table using `useUpdateEstimateLineItemMutation` and `useDeleteEstimateLineItemMutation` with `estimateId` in all payloads
- `EstimateLineItemsCardList.tsx`: Mobile card list, same mutation hooks and `estimateId` renames throughout
- `EstimateActionStrip.tsx`: Phase 43 stub — renders Delete button on draft estimates only; exports `EstimateActionStripProps` interface Phase 44 will extend with Send/Convert/MarkLost wiring
- `index.ts`: Updated to export all 12 components (3 original + 9 new)

**Task 3 — EstimatesPage** (`trade-flow-ui/src/pages/EstimatesPage.tsx`):

- Seven grouped tabs via `TAB_STATUS_GROUPS` record mapping `EstimateTabValue` → `readonly EstimateStatus[]`
- `deleted` is excluded from ALL groups — client-side filter runs `estimates.filter((e) => e.status !== "deleted")` before tab grouping
- Responded tab groups `responded` + `site_visit_requested`; Sent tab groups `sent` + `viewed`; Expired/Lost groups `expired` + `lost`
- `CreateDocumentDialog` wired with `defaultType="estimate"` — toggle is hidden, form goes straight to estimate creation
- Delete confirmation AlertDialog with `"Estimate deleted"` / `"Failed to delete estimate"` toasts
- Prerequisite alert rendered when business or customers missing

## Deviations from Plan

### Auto-fixed: PrerequisiteAlert context type too narrow (Rule 2)

**Found during:** Task 3  
**Issue:** `PrerequisiteAlert` `context` prop type was `"items" | "customers" | "jobs" | "quotes"` — did not include `"estimates"`. TypeScript error TS2322 on EstimatesPage render.  
**Fix:** Added `"estimates"` to the union type and added `estimates: "create estimates."` entry to `contextActionText` record in `PrerequisiteAlert.tsx`.  
**Files modified:** `trade-flow-ui/src/components/PrerequisiteAlert.tsx`  
**Commit:** 0d3228a

### Auto-fixed: Prettier formatting in EstimatesTable.tsx (Rule 1)

**Found during:** Task 3 CI run  
**Issue:** Prettier flagged EstimatesTable.tsx multi-line `TableRow onClick` attribute (written split across lines but Prettier preferred single-line within 125 char width).  
**Fix:** `npx prettier --write src/features/estimates/components/EstimatesTable.tsx`  
**Files modified:** `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx`  
**Commit:** 0d3228a

### Minor: EstimatesDataView includes empty state

**Found during:** Task 1  
**Issue:** Plan spec showed minimal EstimatesDataView without empty state, but QuotesDataView (the mirror source) includes one. Omitting it would leave an inferior UX.  
**Action:** Included `EstimatesEmptyState` ("No estimates found") matching the QuotesDataView pattern. No plan constraint forbade it.

## Known Stubs

`EstimateActionStrip.tsx` is intentionally a minimal stub per plan D-MIR-02. It renders only the Delete button on draft estimates. Phase 44 will replace the body with Send/Convert/MarkLost wiring. The stub is fully functional for its Phase 43 scope (provides extension point, compiles, renders correct output on draft status).

## Threat Flags

None beyond the plan's threat model. T-43-05-01 (deleted estimates leak) is mitigated — EstimatesPage explicitly filters `e.status !== "deleted"` before tab grouping. T-43-05-02 (URL manipulation) mitigated by server-side Phase 41 EstimatePolicy. No new network endpoints, auth paths, or schema changes introduced in this plan.

## Self-Check: PASSED

| Item | Result |
|------|--------|
| trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimatesCardSkeleton.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimatesTableSkeleton.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimateLineItemsTable.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimateLineItemsCardList.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx | FOUND |
| trade-flow-ui/src/pages/EstimatesPage.tsx | FOUND |
| Commit f2d21ac (Task 1) | FOUND |
| Commit 8737dd6 (Task 2) | FOUND |
| Commit 0d3228a (Task 3) | FOUND |
| npm run ci exits 0 (82 tests, 0 lint errors, 0 type errors) | PASSED |
| formatRange called in EstimatesTable and EstimatesCardList | PASSED |
| isEditable = estimate.status === "draft" (not draft||sent) | PASSED |
| deleted excluded from TAB_STATUS_GROUPS | PASSED |
| No cross-imports from @/features/quotes | PASSED |
| No suppressions (@ts-ignore, eslint-disable) | PASSED |
