---
phase: 43-estimate-frontend-crud
plan: "04"
subsystem: trade-flow-ui/features/estimates + trade-flow-ui/features/quotes + trade-flow-ui/components
tags: [estimates, quotes, react, components, dialog, frontend]
dependency_graph:
  requires:
    - 43-01 (UncertaintyReason type from src/types/estimate.ts)
    - 43-03 (useCreateEstimateMutation from estimateApi.ts)
  provides:
    - trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx (standalone form extracted from dialog)
    - trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts (UNCERTAINTY_CHIP_LABELS + UNCERTAINTY_CHIP_ORDER)
    - trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx (shadcn Slider wrapper)
    - trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx (multi-select pill buttons)
    - trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx (estimate create form body)
    - trade-flow-ui/src/components/CreateDocumentDialog.tsx (shared dialog shell with toggle)
  affects:
    - Plan 43-06 (wires CreateDocumentDialog into QuotesPage, EstimatesPage, JobDetailPage)
tech_stack:
  added: []
  patterns:
    - Form body extraction pattern (form lives inside DialogContent provided by shell)
    - Shared dialog shell with defaultType? prop hiding/showing toggle
    - Multi-select pill chip group (Button variant toggle + aria-pressed)
    - Shadcn Slider wrapper with constrained min/max/step
key_files:
  created:
    - trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx
    - trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts
    - trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx
    - trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx
    - trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx
    - trade-flow-ui/src/features/estimates/components/index.ts
    - trade-flow-ui/src/components/CreateDocumentDialog.tsx
  modified:
    - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx (thin shell now imports CreateQuoteForm)
    - trade-flow-ui/src/features/quotes/components/index.ts (added CreateQuoteForm export)
    - trade-flow-ui/src/features/estimates/index.ts (added components barrel export)
decisions:
  - "CreateQuoteForm extracted verbatim from CreateQuoteDialog — no internal refactoring, exact behaviour preserved"
  - "CreateEstimateForm duplicates job-picker popover inline (Separation-over-DRY pattern) rather than extracting shared hook"
  - "CreateDocumentDialog lives in src/components/ (not features/) per D-DLG-06"
  - "defaultType prop hides the Quote/Estimate toggle and fixes selectedType; omitting it shows the toggle defaulting to quote"
metrics:
  duration: "10 minutes"
  completed: "2026-04-12"
  tasks_completed: 2
  files_changed: 10
---

# Phase 43 Plan 04: Shared Create Dialog and Estimate Form Summary

**One-liner:** Extracted CreateQuoteForm from its dialog shell, shipped ContingencySlider + UncertaintyChipGroup + UNCERTAINTY_CHIP_LABELS, and built CreateEstimateForm + CreateDocumentDialog shared shell with defaultType-controlled Quote/Estimate toggle.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Extract CreateQuoteForm + ship uncertainty chip lib, ContingencySlider, UncertaintyChipGroup | 9a87159 | CreateQuoteForm.tsx (new), CreateQuoteDialog.tsx (thinned), index.ts, uncertainty-chips.ts, ContingencySlider.tsx, UncertaintyChipGroup.tsx |
| 2 | Build CreateEstimateForm + CreateDocumentDialog shared shell | 5212d44 | CreateEstimateForm.tsx, estimates/components/index.ts, estimates/index.ts, CreateDocumentDialog.tsx |

## What Was Built

**Task 1 — CreateQuoteForm extracted** (`trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx`):

The inner `CreateQuoteForm` function (previously defined inline in `CreateQuoteDialog.tsx`, ~225 lines) was extracted verbatim into its own file with the `CreateQuoteFormProps` interface exported. `CreateQuoteDialog.tsx` became a thin shell (~30 lines) that imports and delegates to `CreateQuoteForm`. Existing consumers (`QuotesPage`, `JobDetailPage`) continue to work unchanged through `CreateQuoteDialog`.

**Task 1 — Uncertainty chip library** (`trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts`):

- `UNCERTAINTY_CHIP_LABELS`: `Record<UncertaintyReason, string>` with all 5 entries (site_inspection, hidden_conditions, materials_supply, access_working_space, scope_unclear)
- `UNCERTAINTY_CHIP_ORDER`: readonly tuple of all 5 UncertaintyReason values in display order

**Task 1 — ContingencySlider** (`trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx`):

Wraps shadcn `Slider` with `min=0 max=30 step=5`. Displays live percentage label with `aria-live="polite"` and an explanatory "X% uncertainty buffer" paragraph below.

**Task 1 — UncertaintyChipGroup** (`trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx`):

Multi-select pill button group rendering 5 chips from `UNCERTAINTY_CHIP_ORDER`. Uses `variant="default"` for selected, `variant="outline"` for unselected, `aria-pressed` on each, and a Check icon on selected chips.

**Task 2 — CreateEstimateForm** (`trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx`):

Full estimate create form with:
- Same job-picker popover + CreateJobDialog inline-create pattern as CreateQuoteForm (duplicated per Separation-over-DRY)
- ContingencySlider (default 10%)
- DisplayMode toggle: "Range (£X - £Y)" / "From £X" pill buttons (default "range")
- UncertaintyChipGroup (default [])
- uncertaintyNotes Textarea ("Other notes (optional)")
- notes Textarea ("Internal notes (optional)")
- Submits `CreateEstimateRequest` via `useCreateEstimateMutation`, toasts success, navigates to `/estimates/${result.id}`

**Task 2 — CreateDocumentDialog** (`trade-flow-ui/src/components/CreateDocumentDialog.tsx`):

Shared dialog shell under `src/components/`. Accepts `defaultType?: "quote" | "estimate"`:
- If `defaultType` provided: toggle hidden, `selectedType` initialised from prop
- If `defaultType` omitted: toggle visible (grid-cols-2, aria-group), defaults to "quote"
- Renders `CreateQuoteForm` or `CreateEstimateForm` below toggle based on `selectedType`

## Deviations from Plan

### Auto-fixed: Prettier formatting

After initial write of `CreateEstimateForm.tsx`, Prettier reformatted one multi-line JSX attribute (`<Check className={...} />`) from split to single-line (within 125 char width). Applied via `npx prettier --write`. No behaviour change.

## Known Stubs

None. All components are fully wired — CreateEstimateForm calls the real `useCreateEstimateMutation` hook, and CreateDocumentDialog renders real form bodies. No placeholder data or hardcoded empty values in rendering paths.

## Threat Flags

None beyond the plan's threat model. No new network endpoints, auth paths, or schema changes introduced. The `uncertaintyNotes` and `notes` Textarea inputs are handled as plain React controlled state — React auto-escapes on render (T-43-04-03 mitigated). The ContingencySlider enforces min=0/max=30/step=5 client-side (T-43-04-01 mitigated at UI layer; Phase 41 class-validator is authoritative). UncertaintyChipGroup only emits values from `UNCERTAINTY_CHIP_ORDER` (T-43-04-02 mitigated).

## Self-Check: PASSED

| Item | Result |
|------|--------|
| trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx | FOUND |
| trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx | FOUND (thinned shell) |
| trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts | FOUND |
| trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx | FOUND |
| trade-flow-ui/src/features/estimates/components/index.ts | FOUND |
| trade-flow-ui/src/features/estimates/index.ts (exports components) | FOUND |
| trade-flow-ui/src/components/CreateDocumentDialog.tsx | FOUND |
| Commit 9a87159 (Task 1) | FOUND |
| Commit 5212d44 (Task 2) | FOUND |
| npm run ci exits 0 (82 tests, 0 errors, 0 type errors) | PASSED |
| No suppressions (@ts-ignore, eslint-disable) | PASSED |
