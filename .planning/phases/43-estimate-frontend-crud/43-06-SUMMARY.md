---
phase: 43-estimate-frontend-crud
plan: "06"
subsystem: trade-flow-ui/pages + trade-flow-ui/features/estimates + trade-flow-ui/features/quotes + trade-flow-ui/config
tags: [estimates, routing, navigation, react, frontend, migration]
dependency_graph:
  requires:
    - 43-01 (Estimate, EstimateStatus, EstimateDisplayMode, UncertaintyReason types)
    - 43-03 (useGetEstimateQuery, useUpdateEstimateMutation, useDeleteEstimateMutation)
    - 43-04 (CreateDocumentDialog, ContingencySlider, UncertaintyChipGroup)
    - 43-05 (EstimateLineItemsCard, EstimateActionStrip, EstimatesPage)
  provides:
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx (detail page with inline-edit Draft controls)
    - trade-flow-ui/src/App.tsx (/estimates and /estimates/:estimateId routes)
    - trade-flow-ui/src/config/navigation.ts (Estimates sidebar link)
    - trade-flow-ui/src/pages/JobDetailPage.tsx (migrated to CreateDocumentDialog)
    - trade-flow-ui/src/pages/QuotesPage.tsx (migrated to CreateDocumentDialog)
    - trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx (Create Document button)
    - trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx (Create Document label)
  affects:
    - Phase 44 (EstimateActionStrip stub is the extension point for Send/Convert/MarkLost)
    - Phase 45 (response summary placeholder card will be wired with real data)
tech_stack:
  added: []
  patterns:
    - Split EstimateEditor child component pattern (inner component receives estimate as prop so useState initialises from server value on mount, key={estimate.id} forces remount on navigation)
    - Pessimistic single-field PATCH pattern (onBlur for text, onChange for toggles/sliders)
    - AlertDialog delete confirmation pattern (mirrors QuoteDetailPage)
decisions:
  - "EstimateEditor split from EstimateDetailContent to satisfy react-hooks/set-state-in-effect and react-hooks/refs ESLint rules — useState initialiser runs once on mount with correct server value"
  - "key={estimate.id} on EstimateEditor ensures remount when navigating between different estimates"
  - "JobActionStrip and JobDetailTabs both updated to say Create Document (two render paths for the same action)"
  - "Pre-existing BusinessStep.tsx react-hooks/incompatible-library warning (0 errors, 1 warning) is out of scope — not introduced by this plan"
metrics:
  duration: "12 minutes"
  completed: "2026-04-13"
  tasks_completed: 2
  files_changed: 10
---

# Phase 43 Plan 06: Estimate Detail Routing and Migration Summary

**One-liner:** EstimateDetailPage with inline-edit Draft controls and delete, /estimates routes wired, ClipboardList sidebar link added, all CreateQuoteDialog consumers migrated to CreateDocumentDialog, CreateQuoteDialog.tsx deleted, full CI gate green.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Build EstimateDetailPage with inline-edit Draft controls and delete | bfd2d99 (ui) | EstimateDetailPage.tsx (new) |
| 2 | Wire /estimates routing, sidebar link, migrate JobDetailPage + QuotesPage, delete CreateQuoteDialog | f7ff302 (ui) | App.tsx, navigation.ts, JobDetailPage.tsx, QuotesPage.tsx, JobActionStrip.tsx, JobDetailTabs.tsx, quotes/components/index.ts, CreateQuoteDialog.tsx (deleted) |

## What Was Built

**Task 1 — EstimateDetailPage** (`trade-flow-ui/src/pages/EstimateDetailPage.tsx`):

Component split into `EstimateDetailContent` (outer — handles loading/error/not-found) and `EstimateEditor` (inner — receives `estimate: Estimate` and `businessId: string` as props, holds all local draft state and mutation handlers). `key={estimate.id}` forces remount when navigating to a different estimate, so `useState` always initialises from the correct server value.

Sections in render order:
- Back link → `/estimates`
- Header: `estimate.number`, status badge (10-entry `statusColors` map), `customerName`, job link, `estimateDate`
- `EstimateActionStrip` (delete button visible on draft only)
- Price card: `formatRange` output, display-mode pill toggle (Range / From), `ContingencySlider`, base total paragraph
- Uncertainty card: `UncertaintyChipGroup` + `uncertaintyNotes` Textarea (onBlur PATCH)
- `EstimateLineItemsCard` (inner edit gate is already `status === "draft"`)
- Internal notes card: Textarea (onBlur PATCH)
- Response summary placeholder: "No customer response yet. Response handling ships in Phase 45."
- Delete confirmation AlertDialog: "Delete {number} for {customerName}? This cannot be undone." → `useDeleteEstimateMutation` → toast + navigate to `/estimates`

All controls disabled when `!isEditable` (non-Draft). Pessimistic PATCH with `isUpdating` spinner on display-mode/contingency changes.

**Task 2 — Routing, navigation, migration**:

- `App.tsx`: Two new routes under `PaywallGuard` — `path="/estimates"` → `EstimatesPage`, `path="/estimates/:estimateId"` → `EstimateDetailPage`
- `navigation.ts`: `ClipboardList` icon imported; "Estimates" nav item added between Quotes and Items
- `JobDetailPage.tsx`: `CreateQuoteDialog` import replaced with `CreateDocumentDialog` from `@/components/CreateDocumentDialog`; dialog JSX updated (no `defaultType`, toggle visible)
- `JobActionStrip.tsx`: "Create Quote" button label renamed to "Create Document"
- `JobDetailTabs.tsx`: `actionLabel="Create Quote"` renamed to `"Create Document"`
- `QuotesPage.tsx`: `CreateQuoteDialog` replaced with `CreateDocumentDialog defaultType="quote"`
- `quotes/components/index.ts`: `CreateQuoteDialog` barrel export removed
- `CreateQuoteDialog.tsx`: File deleted; zero references remain in `trade-flow-ui/src/`

## Deviations from Plan

### Auto-fixed: setState-in-useEffect ESLint rule (Rule 1)

**Found during:** Task 1 lint run
**Issue:** Initial implementation used `useEffect(() => { setNotesDraft(...); }, [estimate?.notes])` to sync draft state from server. The `react-hooks/set-state-in-effect` ESLint rule (error-level) disallowed this pattern.
**Fix:** Split the component — `EstimateDetailContent` (outer) handles loading/error states and renders `<EstimateEditor key={estimate.id} estimate={estimate} businessId={businessId} />`. `EstimateEditor` receives the non-null `estimate` as a prop and initialises `useState` directly from it on mount. `key={estimate.id}` ensures remount when navigating to a different estimate. No `useEffect` or ref access during render needed.
**Files modified:** `trade-flow-ui/src/pages/EstimateDetailPage.tsx`
**Commit:** bfd2d99

### Auto-fixed: useRef access during render ESLint rule (Rule 1)

**Found during:** Task 1 lint run (second attempt)
**Issue:** After the first fix attempt used `useRef` to track the last synced estimate ID during render, the `react-hooks/refs` rule (error-level) disallowed both reading and writing `ref.current` during render.
**Fix:** The component split pattern (above) resolved both issues simultaneously.
**Commit:** bfd2d99

### Auto-fixed: Prettier formatting (Rule 1)

**Found during:** Task 2 CI run
**Issue:** Prettier flagged `navigation.ts` (import reformatted to multi-line) and `EstimateDetailPage.tsx` (inline string collapsed).
**Fix:** `npx prettier --write src/config/navigation.ts src/pages/EstimateDetailPage.tsx`
**Commit:** f7ff302

### Minor: JobActionStrip.tsx and JobDetailTabs.tsx also updated

**Found during:** Task 2 implementation
**Issue:** The plan specified renaming the button in `JobDetailPage.tsx`, but the actual "Create Quote" button text lives in `JobActionStrip.tsx` (rendered by `JobDetailPage`), and a second "Create Quote" action label lives in `JobDetailTabs.tsx` (the empty-state action).
**Action:** Updated both files. No plan constraint forbade it; both are under `features/jobs/` which is directly consumed by `JobDetailPage`.

## Known Stubs

`EstimateActionStrip` renders only the Delete button on Draft estimates — Phase 43 scope. Phase 44 will extend it with Send/Convert/MarkLost wiring. The detail page's "Customer response" card is intentionally a placeholder; Phase 45 wires `estimate.responseSummary`.

## Threat Flags

None beyond the plan's threat model. T-43-06-05 (old CreateQuoteDialog survives migration) is fully mitigated — file deleted, barrel export removed, zero grep matches across `trade-flow-ui/src/`. T-43-06-03 (XSS via uncertaintyNotes/notes rendered back) is mitigated — Textarea renders as a controlled `<textarea>` with no `dangerouslySetInnerHTML`.

## Self-Check: PASSED

| Item | Result |
|------|--------|
| trade-flow-ui/src/pages/EstimateDetailPage.tsx | FOUND |
| trade-flow-ui/src/App.tsx contains EstimatesPage + EstimateDetailPage (4 references) | PASSED |
| path="/estimates" and path="/estimates/:estimateId" in App.tsx | PASSED |
| navigation.ts Estimates title + href + ClipboardList (2 references) | PASSED |
| JobDetailPage.tsx uses CreateDocumentDialog (2 references, 0 CreateQuoteDialog) | PASSED |
| JobActionStrip.tsx says "Create Document" | PASSED |
| QuotesPage.tsx uses CreateDocumentDialog defaultType="quote" (0 CreateQuoteDialog) | PASSED |
| CreateQuoteDialog.tsx DELETED | PASSED |
| quotes/components/index.ts: 0 CreateQuoteDialog references | PASSED |
| Zero CreateQuoteDialog references across trade-flow-ui/src/ | PASSED |
| Commit bfd2d99 (Task 1) | FOUND |
| Commit f7ff302 (Task 2) | FOUND |
| npm run ci exits 0 (82 tests, 0 lint errors, 0 format issues, 0 type errors) | PASSED |
| No suppressions (@ts-ignore, eslint-disable) | PASSED |
