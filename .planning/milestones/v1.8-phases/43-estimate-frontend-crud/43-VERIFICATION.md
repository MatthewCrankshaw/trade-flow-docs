---
phase: 43-estimate-frontend-crud
verified: 2026-04-13T00:00:00Z
status: human_needed
score: 4/4 must-haves verified (automated checks); 2 items require human testing
overrides_applied: 0
human_verification:
  - test: "Open the app, navigate to a job, click 'Create Document'. Verify the dialog shows a Quote/Estimate toggle. Select Estimate and confirm: ContingencySlider appears (0-30% in 5% steps), display-mode toggle (Range / From), five uncertainty chips (Site inspection needed, Hidden conditions, Materials & supply, Access & working space, Scope unclear until investigation), and a freeform notes textarea. Submit and verify the estimate appears in /estimates list."
    expected: "Dialog shows toggle; estimate form fields are functional; submitted estimate appears in the list with correct number format (E-YYYY-NNN) and price range display."
    why_human: "End-to-end form submission and navigation requires a running app connected to Phase 41 backend. Cannot verify UI rendering or form submission without a live environment."
  - test: "Navigate to an existing Draft estimate detail page. Verify: price range displays as '£X - £Y' or 'From £X' (not raw numbers); ContingencySlider is draggable and persists on blur; line items can be added/edited/deleted; Delete button is visible and functional; non-Draft estimates show all controls disabled."
    expected: "Detail page renders API-returned price range without client-side arithmetic; all Draft controls work; non-Draft states are properly locked."
    why_human: "Requires live app with real estimate data from Phase 41 API to verify the formatRange output matches API-returned values and mutation side-effects are correct."
---

# Phase 43: Estimate Frontend CRUD Verification Report

**Phase Goal:** A trader can visually create and edit estimates from the app with a document-type toggle, contingency slider, and range-or-"from" price display, running against the Phase 41 backend.
**Verified:** 2026-04-13T00:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Shared Create Document dialog shows Quote/Estimate toggle; selecting Estimate reveals ContingencySlider (0-30% in 5%), display-mode toggle, five uncertainty chips, freeform notes field, and submitted estimate appears in list | ✓ VERIFIED (code) / ? HUMAN (runtime) | `CreateDocumentDialog.tsx` has toggle logic with `selectedType` state and `defaultType` prop. `ContingencySlider.tsx` uses min=0/max=30/step=5. `UncertaintyChipGroup.tsx` renders all 5 chips from `UNCERTAINTY_CHIP_ORDER`. `CreateEstimateForm.tsx` calls `useCreateEstimateMutation` on submit. Runtime flow requires human verification. |
| 2 | Estimates list page renders with status tab filtering; each row shows customer name, job title, E-YYYY-NNN, status badge, and price range formatted as `£X - £Y` or `From £X` per display mode | ✓ VERIFIED | `EstimatesPage.tsx` exists with 7 grouped tabs. `EstimatesTable.tsx` calls `formatRange(estimate.priceRange.low, estimate.priceRange.high, estimate.displayMode, currencyCode)`. 10-entry `statusColors` map covers all `EstimateStatus` values. |
| 3 | Estimate detail page shows line items, contingency %, formatted price range, status, customer info, response placeholder, and action buttons with Draft-only edit gate | ✓ VERIFIED | `EstimateDetailPage.tsx` exists with `EstimateEditor` component. `EstimateLineItemsCard` has `isEditable = estimate.status === "draft"`. `EstimateActionStrip` renders Delete only on Draft. Response placeholder card confirmed present ("No customer response yet. Response handling ships in Phase 45.") |
| 4 | UI never multiplies base x contingency on client; `formatRange(low, high, mode)` renders only what the API returns | ✓ VERIFIED | `formatRange` in `currency.ts` (lines 199-211) is a pure renderer: takes `low.total` and `high.total`, formats with `formatAmount`, returns string. No multiplication, no contingency math. Golden-file test exists at `src/lib/__tests__/currency.formatRange.test.ts` with fixture `estimate-response.sample.json` + SHA-256 drift guard. |

**Score:** 4/4 truths verified (automated code checks pass; 2 human items outstanding for runtime behavior)

### Deferred Items

None. All phase 43 success criteria are addressed in this phase.

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/types/estimate.ts` | Estimate TypeScript contracts (10 exports) | ✓ VERIFIED | 10 export count confirmed via grep |
| `trade-flow-ui/src/types/index.ts` | Barrel re-exports estimate types | ✓ VERIFIED | Modified in plan 43-01 |
| `trade-flow-ui/src/components/ui/slider.tsx` | shadcn Slider wrapper over Radix primitive | ✓ VERIFIED | File exists |
| `trade-flow-ui/src/lib/currency.ts` | `formatRange` helper | ✓ VERIFIED | Function at lines 199-211; pure renderer confirmed |
| `trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` | Golden-file test | ✓ VERIFIED | File exists |
| `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` | Golden-file fixture | ✓ VERIFIED | File exists |
| `trade-flow-ui/src/features/estimates/api/estimateApi.ts` | RTK Query slice with 8 hooks | ✓ VERIFIED | 8 hooks confirmed via grep |
| `trade-flow-ui/src/features/estimates/index.ts` | Feature barrel | ✓ VERIFIED | Exists |
| `trade-flow-ui/src/components/CreateDocumentDialog.tsx` | Shared dialog with Quote/Estimate toggle | ✓ VERIFIED | Toggle logic confirmed; `defaultType` prop hides toggle when provided |
| `trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` | Extracted quote form | ✓ VERIFIED | Extracted from CreateQuoteDialog |
| `trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts` | 5-chip labels and order | ✓ VERIFIED | All 5 chip identifiers confirmed |
| `trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` | Slider with min=0/max=30/step=5 | ✓ VERIFIED | Constraints confirmed via grep |
| `trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx` | Multi-select pill buttons | ✓ VERIFIED | Exists |
| `trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` | Estimate create form | ✓ VERIFIED | Exists; calls `useCreateEstimateMutation` |
| `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` | Desktop table with formatRange | ✓ VERIFIED | formatRange import and call confirmed |
| `trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx` | Mobile card list | ✓ VERIFIED | Exists |
| `trade-flow-ui/src/features/estimates/components/EstimatesDataView.tsx` | Responsive switcher | ✓ VERIFIED | Exists |
| `trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx` | Line items with draft-only edit gate | ✓ VERIFIED | `isEditable = estimate.status === "draft"` confirmed |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | Action strip stub | ✓ VERIFIED | Exists; Delete on Draft only |
| `trade-flow-ui/src/pages/EstimatesPage.tsx` | Estimates list page | ✓ VERIFIED | Exists; 7 grouped tabs |
| `trade-flow-ui/src/pages/EstimateDetailPage.tsx` | Estimate detail page | ✓ VERIFIED | Exists; EstimateEditor pattern |
| `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` | DELETED (replaced by CreateDocumentDialog) | ✓ VERIFIED | File absent; zero references across `trade-flow-ui/src/` |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `App.tsx` | `EstimatesPage` | `path="/estimates"` route | ✓ WIRED | Confirmed line 106 in App.tsx |
| `App.tsx` | `EstimateDetailPage` | `path="/estimates/:estimateId"` route | ✓ WIRED | Confirmed line 107 in App.tsx |
| `navigation.ts` | `/estimates` | `ClipboardList` icon + "Estimates" link | ✓ WIRED | Confirmed lines 60-63 in navigation.ts |
| `EstimatesTable` | `formatRange` | import from `@/lib/currency` | ✓ WIRED | Confirmed import and call in EstimatesTable.tsx |
| `CreateDocumentDialog` | `CreateEstimateForm` | `selectedType === "estimate"` branch | ✓ WIRED | Confirmed in CreateDocumentDialog.tsx |
| `QuotesPage` | `CreateDocumentDialog` | `defaultType="quote"` | ✓ WIRED | Confirmed line 105 in QuotesPage.tsx |
| `JobDetailPage` | `CreateDocumentDialog` | no defaultType (toggle visible) | ✓ WIRED | Confirmed import in JobDetailPage.tsx |
| `estimateApi.ts` | `/v1/estimates` routes | 8 RTK Query endpoints | ✓ WIRED | 8 hooks confirmed; injected into `apiSlice` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| `EstimatesTable.tsx` | `estimates` from `useGetEstimatesQuery` | RTK Query `GET /v1/estimates` | Yes — JWT-scoped API call to Phase 41 backend | ✓ FLOWING |
| `EstimateDetailPage.tsx` | `estimate` from `useGetEstimateQuery(estimateId)` | RTK Query `GET /v1/estimates/:id` | Yes — API call parametrized on route param | ✓ FLOWING |
| `formatRange` | `low`, `high` from API response | `estimate.priceRange.{low,high}` | Yes — renders API-returned values only; no client arithmetic | ✓ FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED — cannot verify form submission, navigation, and API round-trips without a running server connected to Phase 41 backend. See Human Verification Required section.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| CONT-03 | 43-04, 43-06 | User can toggle estimate price display to "From £X" mode | ✓ SATISFIED | `displayMode` toggle in `CreateEstimateForm` and `EstimateDetailPage`; `formatRange` returns `"From £X"` when `mode === "from"` |
| CONT-04 | 43-01, 43-04 | Five uncertainty reason chips (trade-agnostic) plus freeform notes textarea | ✓ SATISFIED | `UncertaintyChipGroup` with all 5 identifiers confirmed; `uncertaintyNotes` textarea in `CreateEstimateForm` and `EstimateDetailPage` |

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `EstimateDetailPage.tsx` line 261 | Response summary placeholder text | ℹ Info | Intentional — Phase 45 wires `estimate.responseSummary`. Placeholder is plan-documented and does not prevent SC #3 from being met (SC #3 explicitly calls for "a placeholder for response summary"). |
| `EstimateActionStrip.tsx` | Minimal stub (Delete only on Draft) | ℹ Info | Intentional — Phase 44 extends with Send/Convert/MarkLost. Stub is plan-documented and functional for Phase 43 scope. |

No blockers found. No `@ts-ignore`, `eslint-disable`, `TODO`, or `FIXME` markers introduced by this phase (pre-existing `BusinessStep.tsx` react-hooks warning predates phase 43 and is out of scope per plan 43-01 and 43-03 summaries).

### Human Verification Required

#### 1. Create Estimate End-to-End Flow

**Test:** Open the app, navigate to a job detail page, click "Create Document". Confirm the dialog shows a Quote/Estimate toggle. Select "Estimate". Verify the ContingencySlider appears and is draggable in 5% steps (0-30), a Range/From display-mode toggle appears, five uncertainty chips appear, and a notes textarea appears. Fill in the form and submit. Confirm the new estimate appears in the /estimates list with E-YYYY-NNN number and price range.
**Expected:** Full create flow works end-to-end against the Phase 41 backend; estimate appears in list with correct formatting.
**Why human:** Form submission, navigation, and API round-trip require a live environment with Phase 41 backend running.

#### 2. Estimate Detail Page Interactions

**Test:** Navigate to an existing Draft estimate detail page. Verify: (a) price displays as "£X - £Y" (range mode) or "From £X" (from mode) using API-returned values; (b) ContingencySlider is interactive and PATCHes on change; (c) display-mode toggle PATCHes on change; (d) line items can be added, updated, and deleted; (e) Delete button triggers confirmation dialog and removes the estimate; (f) navigate to a non-Draft estimate and verify all edit controls are disabled/locked.
**Expected:** All Draft controls functional; non-Draft states properly locked; price display matches API values without client-side arithmetic.
**Why human:** Requires a running app with real estimate data from Phase 41 API to verify mutations, state transitions, and that formatRange output is consistent with the backend's computed priceRange.

### Gaps Summary

No gaps found. All 4 success criteria are satisfied by verified code artifacts. The 2 human verification items are runtime behavioral checks that require a live environment — they are not code gaps.

The response summary placeholder in `EstimateDetailPage.tsx` and the minimal `EstimateActionStrip` are intentional stubs explicitly documented in the plans, required by later phases (44/45), and consistent with the phase 43 success criteria which calls for "a placeholder for response summary."

---

_Verified: 2026-04-13T00:00:00Z_
_Verifier: Claude (gsd-verifier)_
