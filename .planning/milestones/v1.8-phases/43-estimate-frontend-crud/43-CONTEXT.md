# Phase 43: Estimate Frontend CRUD - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning

<domain>
## Phase Boundary

A trader can visually create, view, edit, list, and soft-delete estimates from `trade-flow-ui`, running against the Phase 41 backend. Ships to a new `features/estimates/` module that mirrors `features/quotes/` 1:1, adds new `/estimates` and `/estimates/:estimateId` routes, adds an "Estimates" link to the dashboard sidebar, introduces a shared `CreateDocumentDialog` shell that wraps the refactored `CreateQuoteForm` plus a new `CreateEstimateForm`, introduces a new `ContingencySlider` (built on a new shadcn `Slider` component that installs `@radix-ui/react-slider`), ships five trade-agnostic uncertainty chip buttons plus a freeform notes textarea, renders the price as `£X - £Y` or `From £X` via a new `formatRange` helper whose output is pinned by a golden-file test, and mirrors `QuoteLineItemsCard` as `EstimateLineItemsCard` for Draft-only line-item editing.

**Out of scope (explicitly deferred):**
- Dashboard "Create Document" quick-action button (and any inline customer/job creation wizard). Captured in Deferred Ideas.
- Inline editing of customer/job on the estimate detail page. Change-customer/change-job flows are follow-up work.
- Trade-aware uncertainty chip sets per `BusinessTrade`. Captured as a new Future requirement in `REQUIREMENTS.md`.
- Estimate send / email flow (Phase 44).
- Public customer estimate page (Phase 45).
- Estimate revisions UI (Phase 42).
- Convert-to-quote / mark-as-lost UI (Phase 47).
- Non-Draft transition UI. Phase 43 only renders non-Draft statuses read-only; no UI drives transitions.

**Doc edits this phase must ship (in addition to code):**
- Update `.planning/REQUIREMENTS.md` CONT-04 wording to replace the old four-chip list with the five new trade-agnostic values.
- Update `.planning/ROADMAP.md` Phase 43 success criterion #1 to match the new chip list.
- Update `.planning/milestones/v1.8-ROADMAP.md` Phase 43 success criterion #1 to match.
- Add new Future requirement (SMART-04 or equivalent id — planner to pick consistent id) in `.planning/REQUIREMENTS.md` Future section: *"Trade-specific uncertainty chip presets per BusinessTrade (plumber shows pipework-specific chips, electrician shows wiring-specific chips, etc.)."*

</domain>

<decisions>
## Implementation Decisions

### Create Dialog Shape (D-DLG)

- **D-DLG-01:** Shared shell, separate bodies. A new `CreateDocumentDialog` component is the outer dialog. It wraps two inner forms: `CreateQuoteForm` (existing logic extracted from `CreateQuoteDialog`) and new `CreateEstimateForm`. The shell owns the Quote/Estimate toggle.
- **D-DLG-02:** `CreateDocumentDialog` accepts a `defaultType?: "quote" | "estimate"` prop. When set, the toggle is hidden and the shell renders the corresponding body directly. When unset, the toggle is visible and the trader picks before the body renders.
- **D-DLG-03:** Entry points wired in Phase 43:
  - `QuotesPage` → `<CreateDocumentDialog defaultType="quote" ... />` (toggle hidden, lands in `CreateQuoteForm`).
  - New `EstimatesPage` → `<CreateDocumentDialog defaultType="estimate" ... />` (toggle hidden, lands in `CreateEstimateForm`).
  - `JobDetailPage` → `<CreateDocumentDialog prefilledJobId prefilledJobTitle />` (no `defaultType`; toggle is visible because the trader hasn't committed to a document type yet). After the trader picks, the corresponding body renders with the job and customer already prefilled.
- **D-DLG-04:** Dashboard "Create Document" quick-action button is **deferred** to a follow-up phase. Phase 43 does not touch `DashboardPage`. Captured in `<deferred>`.
- **D-DLG-05:** `CreateQuoteDialog` is refactored: the form body moves into a new `CreateQuoteForm` component under `features/quotes/components/`, and the existing `CreateQuoteDialog.tsx` is deleted. Every caller of `CreateQuoteDialog` is rewritten to use `CreateDocumentDialog`. `JobDetailPage`'s current "Create Quote" button becomes "Create Document".
- **D-DLG-06:** `CreateDocumentDialog` lives under `src/components/` (not under `features/quotes/` or `features/estimates/`) because it is shared UI that belongs to neither feature exclusively. Planner may alternatively place it under a new `src/features/documents/` folder if that is a cleaner home — decide at plan time based on whether any non-dialog shared document primitives emerge.
- **D-DLG-07:** The toggle UX inside `CreateDocumentDialog` (when visible): two equal-width pill buttons side by side, labelled "Quote" and "Estimate", using the existing Button component with `variant="default"` / `variant="outline"` for selected/unselected. No radix tabs — the toggle is a visual segmented control, not a content-swapping tab component.

### features/estimates Scaffolding (D-MIR)

- **D-MIR-01:** 1:1 mirror of `features/quotes/` → `features/estimates/`. Every quote component has an estimate equivalent with `quote → estimate` / `Quote → Estimate` renames. Zero cross-imports between the two features. This matches the backend-side Separation-over-DRY decision from Phase 41 (D-LI-02) and applies the same rule to the UI layer. Accepts the duplication cost.
- **D-MIR-02:** Components to mirror, each as its own file under `features/estimates/components/`:
  - `CreateQuoteForm` → `CreateEstimateForm` (new; body for `CreateDocumentDialog`)
  - `QuotesTable` → `EstimatesTable`
  - `QuotesCardList` → `EstimatesCardList`
  - `QuotesCardSkeleton` → `EstimatesCardSkeleton`
  - `QuotesTableSkeleton` → `EstimatesTableSkeleton`
  - `QuotesDataView` → `EstimatesDataView`
  - `QuoteLineItemsCard` → `EstimateLineItemsCard`
  - `QuoteLineItemsTable` → `EstimateLineItemsTable`
  - `QuoteLineItemsCardList` → `EstimateLineItemsCardList`
  - `QuoteActionStrip` → `EstimateActionStrip` (may start as a minimal stub in Phase 43; Phase 44 fills in Send actions, Phase 47 fills in Convert/MarkLost)
- **D-MIR-03:** Not mirrored in Phase 43 (belongs to later phases):
  - `SendQuoteDialog` → `SendEstimateDialog` is Phase 44.
  - `SendQuoteForm` → `SendEstimateForm` is Phase 44.
- **D-MIR-04:** API mirror: new `features/estimates/api/estimateApi.ts` with endpoints:
  - `useGetEstimatesQuery(businessId)` — paginated list
  - `useGetEstimateQuery(estimateId)` — detail
  - `useCreateEstimateMutation()` — POST with contingencyPercent, displayMode, uncertaintyReasons[], uncertaintyNotes, customerId, jobId, estimateDate, notes
  - `useUpdateEstimateMutation()` — PATCH for core fields (contingencyPercent, displayMode, uncertaintyReasons, uncertaintyNotes, notes, customerId, jobId). Draft-only; non-Draft PATCH returns API error.
  - `useAddEstimateLineItemMutation()` — POST line item
  - `useUpdateEstimateLineItemMutation()` — PATCH line item (quantity, unitPrice)
  - `useDeleteEstimateLineItemMutation()` — DELETE line item
  - `useDeleteEstimateMutation()` — soft-delete entire estimate via the endpoint Phase 41 ships (transition-to-deleted or DELETE verb; planner verifies against Phase 41 D-DRAFT-03 and the controller shape in Phase 41 D-TXN-05)
- **D-MIR-05:** RTK Query tag type: new `"Estimate"` tag registered in `src/services/api.ts` alongside the existing `"Quote"` tag. Tag invalidation mirrors the quoteApi pattern (id-per-row + `LIST` sentinel).
- **D-MIR-06:** New `Estimate`, `EstimateLineItem`, `EstimateStatus`, `CreateEstimateRequest`, `UpdateEstimateRequest` types in `src/types/estimate.ts`, exported from `src/types/index.ts`. Does NOT extend or unionise with `Quote` — fully independent type definitions.
- **D-MIR-07:** `Estimate.priceRange` TypeScript shape **matches the API DTO exactly**, both endpoints carry full totals-shaped objects:
  ```typescript
  priceRange: {
    low:  { subTotal: number; taxTotal: number; total: number };
    high: { subTotal: number; taxTotal: number; total: number };
  }
  ```
  `Estimate.totals` stays `{ subTotal: number; taxTotal: number; total: number }` (base, pre-contingency), mirroring Quote.totals. Both fields are always present per Phase 41 D-CONT-06. UI reads both; planner verifies field-for-field against Phase 41's `IEstimatePriceRangeDto` at plan time.
- **D-MIR-08:** `EstimateStatus` type values mirror Phase 41 D-TXN-01 exactly: `"draft" | "sent" | "viewed" | "responded" | "site_visit_requested" | "converted" | "declined" | "expired" | "lost" | "deleted"`.

### Routing and Navigation (D-ROUTE)

- **D-ROUTE-01:** New `/estimates` route rendering `EstimatesPage`. New `/estimates/:estimateId` route rendering `EstimateDetailPage`. Both routes registered in `App.tsx` under the same protected-route wrapper as `/quotes` and `/quotes/:quoteId`.
- **D-ROUTE-02:** New "Estimates" sidebar link in the dashboard navigation, positioned directly after "Quotes". Icon: `FileText` (same as quotes) or a distinct `Receipt` / `ClipboardList` icon from lucide-react — planner's call, pick one that visually reads differently from quotes.
- **D-ROUTE-03:** `JobDetailPage` continues to show related quotes. Related estimates rendering (a separate card or combined "Documents" card) is Planner's discretion — if simple, include it in Phase 43; if non-trivial, defer to a follow-up. **Minimum requirement:** the existing "Create Quote" button on `JobDetailPage` must be renamed to "Create Document" and must route through `CreateDocumentDialog` with prefilled job/customer (toggle visible).

### Status Tab Filtering (D-TAB)

- **D-TAB-01:** `EstimatesPage` renders 7 grouped tabs (not one tab per status):
  1. **All** — every estimate except `deleted`
  2. **Draft** — `draft`
  3. **Sent** — `sent` + `viewed` (viewed is a read-only tracking transition, not meaningfully distinct to the list user)
  4. **Responded** — `responded` + `site_visit_requested`
  5. **Converted** — `converted`
  6. **Declined** — `declined`
  7. **Expired/Lost** — `expired` + `lost`
- **D-TAB-02:** Tab filtering is client-side (same pattern as `QuotesPage` today): fetch all estimates once, filter in a memo on `activeTab`. API-side status filter is not wired in Phase 43.
- **D-TAB-03:** Deleted estimates are never shown in the list. The API is expected to exclude `deleted` from the list endpoint (Phase 41 D-DRAFT-03 partial-index behaviour). Planner verifies against Phase 41's list query.

### ContingencySlider (D-SLD)

- **D-SLD-01:** Install `@radix-ui/react-slider` (new dependency in `trade-flow-ui/package.json`). Build `src/components/ui/slider.tsx` following the shadcn New York style — same surface area as existing `radix-group.tsx` / `tabs.tsx` component wrappers.
- **D-SLD-02:** New `ContingencySlider` component lives under `features/estimates/components/ContingencySlider.tsx`. Wraps the base `Slider` with `min={0}`, `max={30}`, `step={5}`, `defaultValue={[10]}`. Displays the current value as "10%" adjacent to the slider. Fires an `onValueChange(percent: number)` callback with a single number (not an array).
- **D-SLD-03:** The slider label shows "Contingency (%)". Below the slider, the current value is echoed as a visible "X% uncertainty buffer" label so the trader sees the selected number without squinting at the handle.
- **D-SLD-04:** `ContingencySlider` is used in both `CreateEstimateForm` (during create) and `EstimateDetailPage` (inline-editable while status=Draft, disabled otherwise — per D-EDIT-01).
- **D-SLD-05:** Keyboard/ARIA behaviour comes from Radix. Arrow keys move by step=5. PageUp/PageDown move by step=10 (two steps). Home/End jump to min/max. Planner must verify these work after install — add to Phase 43's CI gate manual checklist if automated a11y tests don't already cover slider.

### Uncertainty Chips and Notes (D-CHIP)

- **D-CHIP-01:** Five trade-agnostic chip values (replace the plumber-biased four from the roadmap prose):

  | Stored value | UI label |
  |---|---|
  | `site_inspection` | Site inspection needed |
  | `hidden_conditions` | Hidden conditions |
  | `materials_supply` | Materials & supply |
  | `access_working_space` | Access & working space |
  | `scope_unclear` | Scope unclear until investigation |

  Rationale in `<specifics>` below. These five cover the uncertainty space across all ten `BusinessTrade` values (plumber, electrician, heating_gas_engineer, carpenter_joiner, builder_general_contractor, painter_decorator, tiler_flooring, landscaping_gardening, handyman, other) without requiring trade-aware dispatch.
- **D-CHIP-02:** Chips are rendered as toggleable pill buttons in a wrapped row. Selected state uses existing Button `variant="default"`; unselected uses `variant="outline"`. Tap to toggle on/off. Multi-select (zero or more). No "none of the above" option — an empty array is valid.
- **D-CHIP-03:** Stored values are **not** defined as a TypeScript enum in Phase 43. They live as a union type: `UncertaintyReason = "site_inspection" | "hidden_conditions" | "materials_supply" | "access_working_space" | "scope_unclear"`. Under `features/estimates/lib/` a small module exports the union type and a `UNCERTAINTY_CHIP_LABELS: Record<UncertaintyReason, string>` map that drives the UI labels. Adding a new chip later requires updating both constants in one place.
- **D-CHIP-04:** Freeform `uncertaintyNotes` is a single `<Textarea>` below the chip row, labelled "Other notes (optional)" with a placeholder like `e.g. supplier has boiler on 2-week lead, prices may shift`. Saved to `uncertaintyNotes: string` on the estimate entity (Phase 41 D-UNC-01 reserves this field).
- **D-CHIP-05:** Chips + notes appear in both `CreateEstimateForm` and on `EstimateDetailPage` (inline-editable for Draft, read-only otherwise).
- **D-CHIP-06:** Phase 43 does NOT ship trade-aware chip dispatch. A new Future requirement is added to `REQUIREMENTS.md` for v1.9+: *"SMART-04 (or next available id): trade-specific uncertainty chip presets per BusinessTrade."* Planner owns the exact id and wording when updating REQUIREMENTS.md.

### Display Mode (D-MOD)

- **D-MOD-01:** `displayMode: "range" | "from"` toggle appears in `CreateEstimateForm` (default `"range"`). Rendered as two pill buttons using the same segmented-control pattern as the Quote/Estimate toggle in `CreateDocumentDialog`.
- **D-MOD-02:** `displayMode` is **inline-editable** on `EstimateDetailPage` next to the price display, Draft-only. Flipping the toggle fires `useUpdateEstimateMutation` with `{ displayMode }`. API returns both `low` and `high` regardless of mode (per Phase 41 D-CONT-07) — the toggle is a pure UI relabel, zero recomputation.
- **D-MOD-03:** Non-Draft estimates show the display mode as a read-only visual hint (a small label next to the price, e.g. "range display" / "minimum-price display"), not as an interactive toggle.

### formatRange Helper (D-FMT)

- **D-FMT-01:** New `formatRange(low, high, mode, currencyCode)` helper exported from `src/lib/currency.ts`. Signature:
  ```typescript
  formatRange(
    low: { subTotal: number; taxTotal: number; total: number },
    high: { subTotal: number; taxTotal: number; total: number },
    mode: "range" | "from",
    currencyCode: string
  ): string
  ```
  Returns `"£120 - £135"` in range mode or `"From £120"` in from mode. Uses existing dinero.js-based currency formatting; wraps or extends `formatAmount` rather than reimplementing.
- **D-FMT-02:** `formatRange` operates on the **inclusive-of-tax total** (`low.total` and `high.total`). It does not render exclTax ranges. If a future view needs the exclTax range, add a new helper rather than extending `formatRange` with a mode.
- **D-FMT-03:** `formatRange` is called by:
  - `EstimatesTable` (list row total column)
  - `EstimatesCardList` (mobile list card total)
  - `EstimateDetailPage` (header price display)
- **D-FMT-04:** **Golden-file test** lives in `src/lib/__tests__/currency.formatRange.test.ts`. Fixture approach:
  - A small array of synthetic `{ low, high, mode, currencyCode, expected }` tuples covering: range mode with/without tax, from mode with/without tax, different currency codes (GBP, EUR as smoke), zero-edge cases.
  - The test asserts `formatRange(...args) === expected` exactly.
  - A **separate** golden-file assertion mirrors a real API response (captured as a JSON fixture under `src/lib/__tests__/fixtures/estimate-response.sample.json`). The API-agreement part of success criterion #4 is satisfied by: (1) the JSON fixture is copy-pasted from the Phase 41 backend test output so it matches the real contract byte-for-byte, (2) the test asserts `formatRange(fixture.priceRange.low, fixture.priceRange.high, fixture.displayMode, fixture.currencyCode)` produces the expected string, (3) a hash of the fixture is committed alongside it so any drift is caught in CI.
  - Planner decides whether to add a Phase 41-side helper script that dumps the fixture, or to copy-paste manually. Copy-paste is acceptable given both codebases are in the same repo and planner can run the backend test locally.

### Detail Page Edit Affordance (D-EDIT)

- **D-EDIT-01:** Draft estimate core fields (`contingencyPercent`, `displayMode`, `uncertaintyReasons`, `uncertaintyNotes`, `notes`) are **inline-editable** on `EstimateDetailPage`. Each control PATCHes on change (for the slider/toggle) or on blur (for the textareas). No Edit dialog. Optimistic updates via RTK Query where safe; pessimistic fallback via `isLoading` spinners.
- **D-EDIT-02:** `EstimateLineItemsCard` is a 1:1 mirror of `QuoteLineItemsCard`. `SearchableItemPicker` for add, inline quantity/unit-price editing (where the existing quote pattern supports it), delete per row. `isEditable = status === "draft"` — narrower than the quote pattern (`"draft" || "sent"`) because Phase 41 D-TXN-05 makes Draft the only writable estimate status.
- **D-EDIT-03:** Non-Draft statuses render every core field and every line-item row as read-only. Slider disabled, chip buttons disabled, textarea disabled, line-item picker hidden, delete buttons hidden. No banner — the disabled controls are sufficient signal.
- **D-EDIT-04:** Delete UX mirrors quotes exactly: Delete button visible on the detail page only when `status === "draft"`. Clicking opens an `AlertDialog` with the same copy pattern as `QuotesPage` delete confirm ("Delete E-2026-001? This cannot be undone."). On confirm, `useDeleteEstimateMutation()` fires, toast success/error, navigate back to `/estimates`.
- **D-EDIT-05:** Customer and job are **not** inline-editable on the detail page in Phase 43. Changing customer or job on a Draft estimate is deferred. Captured in `<deferred>`.
- **D-EDIT-06:** Line-item editing uses three separate mutation endpoints (add, update, delete) rather than a bulk PATCH, mirroring the quote pattern exactly. Invalidation tags scoped per estimate id so only the affected detail page refetches.

### Status Badge Colours (D-BADGE)

- **D-BADGE-01:** Estimate status badges are defined per page (`EstimatesPage`, `EstimatesTable`, `EstimateDetailPage`) using a local `statusColors: Record<EstimateStatus, "default" | "secondary" | "destructive" | "outline">` map. No shared helper — matches the existing quote pattern exactly.
- **D-BADGE-02:** Recommended initial mapping (planner may tune during design pass):
  - `draft` → `secondary`
  - `sent` → `outline`
  - `viewed` → `outline`
  - `responded` → `default`
  - `site_visit_requested` → `default`
  - `converted` → `default`
  - `declined` → `destructive`
  - `expired` → `secondary`
  - `lost` → `destructive`
  - `deleted` → `destructive` (never rendered in the list; only shown if the detail page is hit directly)

### Claude's Discretion (planner may override with rationale)

- Exact icon for the "Estimates" sidebar link (something visually distinct from the quote `FileText` icon — e.g. `Receipt`, `ClipboardList`, `ClipboardPen`).
- Exact home for `CreateDocumentDialog` (`src/components/` vs new `src/features/documents/`). Pick based on whether other shared document primitives emerge during planning.
- Whether `EstimateActionStrip` ships as a real component in Phase 43 or a minimal stub that later phases (44, 47) extend. Minimum: at least the component file exists so Phase 44's SendEstimateDialog has somewhere to hang.
- Exact ARIA labels on the chip toggle buttons and `ContingencySlider`.
- Placeholder copy on the `uncertaintyNotes` textarea.
- Whether `JobDetailPage` renders a "Related Documents" card combining quotes and estimates, or two separate cards, or only updates the existing "Create Quote" → "Create Document" button with no related-documents render for estimates. Minimum requirement is the button rename + routing.
- Whether optimistic updates are used on the inline detail-page PATCHes. Safe default: pessimistic with a subtle spinner.
- Id prefix for the new Future requirement (`SMART-04` or next available id after the existing SMART-01..03). Planner assigns.
- Whether the golden-file test captures the fixture automatically from Phase 41 test output or by hand. Both are acceptable; hand is simpler.

### Folded Todos

None — `.planning/STATE.md` shows "Pending Todos: None." No todos were folded into this phase.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Milestone roadmap and restructure decisions
- `.planning/milestones/v1.8-ROADMAP.md` — Phase 43 goal, requirements, success criteria, dependency graph. NOTE: success criterion #1 wording needs updating in Phase 43 to reflect the five new chip values (D-CHIP-01).
- `.planning/ROADMAP.md` — same update to Phase 43 success criterion #1 required.
- `.planning/notes/2026-04-11-v1.8-restructure-decisions.md` — D-01 (entity-boundary separation over DRY). Phase 43 extends this rule to the UI layer (D-MIR-01).

### Requirements
- `.planning/REQUIREMENTS.md` — CONT-03, CONT-04 (Phase 43's direct requirements). CONT-04 wording needs updating as part of Phase 43 work. Also needs a new Future requirement added (SMART-04 or next id) for trade-aware chips.

### Phase 41 decisions that Phase 43 must honour
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` — canonical source for the backend contract. Specifically:
  - **D-CONT-05** — `IEstimatePriceRangeDto` is derived on every read path; Phase 43 type must match exactly (D-MIR-07).
  - **D-CONT-06** — both `totals` (base) and `priceRange` (range) are present on `IEstimateDto`. Phase 43 renders both.
  - **D-CONT-07** — API returns `low` and `high` regardless of `displayMode`; UI owns the labelling switch. Flipping mode never recomputes (D-MOD-02).
  - **D-TXN-01** — `EstimateStatus` values and spelling (`site_visit_requested` with underscore, etc.).
  - **D-TXN-05** — Draft is the only writable status. Phase 43 enforces this client-side via `isEditable = status === "draft"` (D-EDIT-02).
  - **D-DRAFT-03** — soft-delete semantics; list excludes `deleted`.
  - **D-UNC-01/02** — `uncertaintyReasons?: string[]` and `uncertaintyNotes?: string` are accepted by the Phase 41 CRUD surface. Phase 43 is where the chip values are defined (D-CHIP-01).
  - **D-RESP-01** — `responseSummary` shape reserved in Phase 41; Phase 43 renders it as a placeholder card on the detail page (always null in Phase 43).
- `.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md` — Phase 41's research into the backend contract (read §DTO shapes and §endpoints for field names).
- `.planning/phases/41-estimate-module-crud-backend/41-*-PLAN.md` — eight plan files; Phase 43 planner references the `41-07-estimate-crud-services-PLAN.md` and `41-08-controller-module-wiring-and-docs-PLAN.md` for the final endpoint shapes.

### Research
- `.planning/research/ARCHITECTURE.md` §2 (Data Model) — reference only; Phase 41 is the canonical shape now.
- `.planning/research/PITFALLS.md` §Pitfall 7 (type discrimination) — applies to the UI too: `Quote` and `Estimate` types are independent, never unionised.
- `.planning/research/FEATURES.md` — estimate-specific UX expectations if present.

### Codebase maps
- `.planning/codebase/CONVENTIONS.md` — frontend naming, import organisation, component structure. Phase 43 follows these without exception.
- `.planning/codebase/STRUCTURE.md` — where features/, components/, lib/, pages/, types/ live.
- `.planning/codebase/ARCHITECTURE.md` §Frontend Architecture — feature module conventions, RTK Query patterns.

### Existing UI code to mirror (in trade-flow-ui)
- `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` — dialog shell + form body (to be split into `CreateDocumentDialog` + `CreateQuoteForm`)
- `trade-flow-ui/src/features/quotes/components/QuotesTable.tsx` — list table pattern, badge mapping, row click navigation
- `trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx` — mobile card list
- `trade-flow-ui/src/features/quotes/components/QuotesCardSkeleton.tsx`, `QuotesTableSkeleton.tsx` — loading states
- `trade-flow-ui/src/features/quotes/components/QuotesDataView.tsx` — responsive desktop/mobile switcher
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` — line-item editing container (isEditable gate, SearchableItemPicker integration)
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx`, `QuoteLineItemsCardList.tsx` — line-item display/edit rows
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` — action button strip on the detail page (minimal stub for Phase 43; Phase 44 extends)
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` — RTK Query endpoint shape, invalidation tags, transformResponse pattern
- `trade-flow-ui/src/features/quotes/index.ts` — barrel export pattern
- `trade-flow-ui/src/pages/QuotesPage.tsx` — list page wrapping, tabs, prerequisite alert, create dialog orchestration
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` — detail page layout, skeleton, status badge, totals card, notes card
- `trade-flow-ui/src/types/quote.ts` — Quote/QuoteLineItem/QuoteStatus/CreateQuoteRequest type shape
- `trade-flow-ui/src/lib/currency.ts` — existing `formatAmount` / `useBusinessCurrency` that `formatRange` (D-FMT-01) wraps
- `trade-flow-ui/src/hooks/useCurrency.ts`, `useCurrentBusiness.ts` — business context hooks used throughout
- `trade-flow-ui/src/components/SearchableItemPicker.tsx` — item picker reused by `EstimateLineItemsCard`
- `trade-flow-ui/src/components/PrerequisiteAlert.tsx` — missing-business/customers alert used on `QuotesPage` pattern
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` — page wrapper + sidebar nav (where the new Estimates link lands)
- `trade-flow-ui/src/services/api.ts` — RTK Query base slice, tag type registration (`"Estimate"` tag added here)
- `trade-flow-ui/src/App.tsx` — router where `/estimates` and `/estimates/:estimateId` routes land

### shadcn/Radix component patterns
- `trade-flow-ui/src/components/ui/radio-group.tsx` — existing Radix component wrapper shape (import, `forwardRef`, `cn()` usage, shadcn New York styling). New `slider.tsx` follows this exact pattern.
- `trade-flow-ui/src/components/ui/button.tsx`, `button.variants.ts` — variant system that chip toggles and toggle-pills reuse
- `trade-flow-ui/src/components/ui/dialog.tsx` — dialog primitives used by `CreateDocumentDialog`
- `trade-flow-ui/src/components/ui/tabs.tsx` — tab component used for `EstimatesPage` status filter tabs
- `trade-flow-ui/src/components/ui/alert-dialog.tsx` — confirm-delete dialog primitive

### UI conventions
- `trade-flow-ui/CLAUDE.md` — React 19, TS strict, `@/` alias, `import type`, `cn()` helper, Luxon DateTime for dates, `useBusinessCurrency()` for money, `test` + `typecheck` + `lint` + `format:check` + `ci` gates.
- `trade-flow-ui/.github/copilot-instructions/design-system/` (if present) — colours, typography, spacing, component inventory
- `.planning/codebase/TESTING.md` — vitest + @testing-library/react conventions (where the golden-file test for `formatRange` lives)

### No external specs
Phase 43 has no ADRs or external feature docs beyond what the milestone roadmap and Phase 41 context cover. All decisions are captured here.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- **`features/quotes/components/CreateQuoteDialog.tsx`** — the template for `CreateDocumentDialog` + `CreateQuoteForm` + `CreateEstimateForm`. The existing file has a clean split: outer `Dialog` wrapper and inner form component already separated. Lifting the inner into `CreateQuoteForm` is a near-zero-risk refactor. The outer shell becomes `CreateDocumentDialog` after adding the toggle and `defaultType` prop.
- **`features/quotes/components/QuoteLineItemsCard.tsx`** — already implements `isEditable` gating, `SearchableItemPicker` integration, add/update/delete mutation calls. The estimate version changes the gate expression and the mutation names but the structure is identical.
- **`features/quotes/api/quoteApi.ts`** — the RTK Query endpoint shape, transformResponse helpers, invalidation tags, and optimistic update pattern (`onQueryStarted` with `updateQueryData`) are the exact template for `estimateApi.ts`.
- **`src/lib/currency.ts` + `src/hooks/useCurrency.ts`** — the dinero.js-based money formatting already handles the hard parts. `formatRange` wraps two calls to `formatAmount`, assembles the string, returns it. Testing is trivial.
- **`src/components/SearchableItemPicker.tsx`** — already generic; reused by `EstimateLineItemsCard` unchanged.
- **`src/components/PrerequisiteAlert.tsx`** — already generic; reused by `EstimatesPage` for the "no business" / "no customers" prerequisite state.
- **`src/components/ui/` radix wrappers** — `radio-group.tsx`, `tabs.tsx`, `dialog.tsx`, `alert-dialog.tsx`, `button.tsx`, `card.tsx`, `badge.tsx`, `input.tsx`, `label.tsx`, `textarea.tsx`, `separator.tsx`, `skeleton.tsx` — all reused by estimates components unchanged.
- **`@radix-ui/react-slider`** — **not yet installed**. Phase 43 adds this dependency and the corresponding `src/components/ui/slider.tsx` wrapper. Counts as a single new dependency addition.

### Established Patterns

- **Feature barrel exports** — `features/quotes/index.ts` re-exports all components and API hooks. `features/estimates/index.ts` follows the same pattern. Pages import from the feature barrel, never from individual files.
- **Page-level tab filtering** — `QuotesPage` owns the tab state and memo-filters the quotes array. `EstimatesPage` mirrors this. No tab logic in the feature module.
- **RTK Query tag invalidation** — id-per-row + `LIST` sentinel, with cross-feature invalidations where relevant (quote mutations invalidate `Job LIST` because jobs carry a "has quote" flag). Estimate mutations should invalidate `Job LIST` for the same reason if jobs will carry a "has estimate" flag — planner verifies against Phase 41 job response shape.
- **Optimistic updates on delete** — `quoteApi.deleteQuote` uses `onQueryStarted` with `updateQueryData` for instant row removal with rollback on error. Estimate delete mirrors this.
- **Status badge local mapping** — each page and table defines its own `statusColors` record. No shared helper. Mirrored in estimates (D-BADGE-01).
- **Prerequisite alert pattern** — `QuotesPage` computes `missingPrerequisites: PrerequisiteType[]` based on `hasBusiness` / `hasCustomers` and gates the create button on `canCreateQuote`. `EstimatesPage` mirrors this exactly.
- **Responsive data view switcher** — `QuotesDataView` switches between `QuotesTable` (desktop) and `QuotesCardList` (mobile) via `useMediaQuery("(min-width: 768px)")`. `EstimatesDataView` mirrors this.
- **Luxon DateTime for all date display** — `DateTime.fromISO(quote.quoteDate).toFormat("d MMM yyyy")`. Mirrored for `estimate.estimateDate`.
- **`isEditable` gating** — quote uses `status === "draft" || status === "sent"`. Estimate narrows to `status === "draft"` only (D-EDIT-02).
- **Toast notifications** — `toast.success` / `toast.error` from `@/lib/toast`. Reused as-is.

### Integration Points

- **`src/services/api.ts`** — add `"Estimate"` to the `tagTypes` array. New `estimateApi` injects endpoints the same way `quoteApi` does.
- **`src/App.tsx`** — add two new `<Route>` entries under the existing protected-route wrapper: `path="/estimates"` and `path="/estimates/:estimateId"`.
- **`src/components/layouts/DashboardLayout.tsx`** (or the sidebar component it renders) — add the new "Estimates" nav link in the same section as "Quotes".
- **`src/pages/JobDetailPage.tsx`** — rename the "Create Quote" button to "Create Document" and route it through `CreateDocumentDialog` with prefilled job/customer (toggle visible). Potentially add a related-estimates render (planner discretion).
- **`src/pages/QuotesPage.tsx`** — replace `CreateQuoteDialog` usage with `CreateDocumentDialog defaultType="quote"`. One import swap, one prop addition.
- **`src/types/index.ts`** — re-export new types from `src/types/estimate.ts`.
- **`src/components/ui/`** — new `slider.tsx` file lands here (shadcn New York style wrapping `@radix-ui/react-slider`).
- **`trade-flow-ui/package.json`** — new dependency `@radix-ui/react-slider`.

### Known Pitfalls That Apply

- **Pitfall 7 (type discrimination)** — `Quote` and `Estimate` types are fully independent. Never unionise them, never share a `document` parent interface. The `CreateDocumentDialog` outer shell is a UI pattern, not a data contract.
- **Duplication maintenance cost** — Bug fixes in estimate components won't automatically propagate to quote components. Phase 43 accepts this cost per Separation-over-DRY.
- **Currency formatting edge cases** — dinero.js handles sub-unit rounding; never do `(amount / 100).toFixed(2)` manually. `formatRange` must delegate every display concern to `formatAmount`.
- **Optimistic update + tag invalidation** — when using `onQueryStarted` patches alongside `invalidatesTags`, make sure both target the same cache scope, otherwise you get double-fetch flicker.
- **Status filter tabs + client-side filtering** — if the estimate list grows large, client-side filtering starts to drag. Out of scope for Phase 43 (same as quotes today), but flag as a latent concern for v1.9+ if traders start hitting the wall.

</code_context>

<specifics>
## Specific Ideas

### Rationale for the five trade-agnostic chips (D-CHIP-01)

The roadmap prose originally listed four chips — `site inspection`, `pipework`, `materials`, `access` — which were drafted with a plumber in mind. Trade Flow supports ten trades via `BusinessTrade`: plumber, electrician, heating_gas_engineer, carpenter_joiner, builder_general_contractor, painter_decorator, tiler_flooring, landscaping_gardening, handyman, other. A trade-by-trade audit shows that `pipework` is irrelevant for 6 of 10 trades (electricians, carpenters, builders, painters, tilers, landscapers), and the three remaining "universal" chips miss the real uncertainty sources for most non-plumbing trades:

- Electrician: wiring condition, consumer unit capacity, hidden circuits
- Carpenter: timber rot, hidden structural issues, on-site measurements
- Builder: foundations, weather, planning permission, hidden damage
- Painter: surface prep, substrate condition, lead/damp
- Tiler: substrate levelling, existing floor condition
- Landscaper: ground conditions, drainage, weather, plant availability

Rather than build trade-aware chip dispatch (ten configs + an OTHER fallback + product copy work), Phase 43 ships five trade-agnostic chips that capture the underlying uncertainty categories every trade shares:

1. **Site inspection needed** — "I haven't physically been there yet"
2. **Hidden conditions** — "I don't know what's behind walls / under floor / under ground" (covers pipework, wiring, substrate, soil, damp, rot, timber condition, structural)
3. **Materials & supply** — "Stock, lead times, prices" (covers all trades — supplier delays are universal)
4. **Access & working space** — "Parking, stairs, tight spots, working hours" (covers all trades)
5. **Scope unclear until investigation** — "Real work only visible once we open it up" (covers the diagnostic / investigative class of work)

The freeform `uncertaintyNotes` textarea handles trade-specific nuances that don't fit the five categories. Trade-aware chip dispatch is captured as a new Future requirement (SMART-04 or equivalent id) for v1.9+ when there's content-design bandwidth.

### Rationale for the shared shell + separate bodies dialog pattern (D-DLG-01)

From a tradesperson's perspective, the first decision when creating a document isn't "what fields do I fill in" — it's "who is this for and what job is it about." Customer and job selection is always the first step regardless of document type. The second decision is "do I know enough to give a fixed price, or is this a rough indication?" That's the quote/estimate toggle. Everything after that diverges — quotes need expiry dates and acceptance mechanics, estimates need contingency sliders and uncertainty notes.

The shared shell owns the toggle and (in the toggle-visible path) the customer/job picker. The bodies render type-specific fields. The Separation-over-DRY preference still holds at the entity-boundary level because each body is a fully independent form component with its own types, validators, and mutation calls. The shell is pure UI composition, not a data discriminator.

In practice, because the Dashboard entry point is deferred and the Quotes/Estimates pages skip the toggle via `defaultType`, the only place the toggle is actually exercised in Phase 43 is `JobDetailPage`. That's still worth the pattern because it gives Phase 43 a natural place to add the Dashboard entry later without restructuring.

### Phase 41 contract that Phase 43 renders

Phase 41 D-CONT-06 puts both `totals` and `priceRange` on every `IEstimateDto`:

```typescript
interface IEstimateDto {
  // ...
  totals: {
    subTotal: Money;
    taxTotal: Money;
    total: Money;
  };
  priceRange: {
    low:  { subTotal: Money; taxTotal: Money; total: Money };
    high: { subTotal: Money; taxTotal: Money; total: Money };
  };
  contingencyPercent: number;
  displayMode: "range" | "from";
  uncertaintyReasons?: string[];
  uncertaintyNotes?: string;
  responseSummary: IEstimateResponseSummaryDto | null;
  // ...
}
```

Phase 43 mirrors this as plain-number TypeScript (not dinero.js Money objects — the UI's Quote type already uses plain `number` for subTotal/taxTotal/total, and Phase 43 matches that convention). Phase 41 serialises Money as minor-units integers over the wire; `useBusinessCurrency().formatAmount(n)` handles the conversion. Phase 43 delegates every money concern to `formatAmount` and never introspects the integer values.

### formatRange example outputs

```
formatRange({subTotal: 10000, taxTotal: 2000, total: 12000},
            {subTotal: 11000, taxTotal: 2200, total: 13200},
            "range", "GBP")
→ "£120.00 - £132.00"

formatRange({...}, {...}, "from", "GBP")
→ "From £120.00"

formatRange with zero tax (taxRate 0 on all line items):
→ "£100.00 - £110.00"  or  "From £100.00"
```

Currency symbol, decimal places, and thousands separators all come from `formatAmount`. `formatRange` does no formatting of its own beyond the dash and "From " prefix.

### EstimatesPage tab grouping (D-TAB-01 reasoning)

Showing ten tabs (one per status) clutters the tab row and doesn't reflect how a trader thinks about their pipeline. The 7-tab grouping is how a sole trader mentally categorises their estimates:

- **Draft** — "still working on it"
- **Sent** — "waiting for the customer" (whether or not they've opened the link — `viewed` is an observability metric, not a state the trader acts on differently)
- **Responded** — "customer has engaged" (responded with any type, including site visit request — all require trader action)
- **Converted** — "turned into a real job"
- **Declined** — "customer said no"
- **Expired/Lost** — "dead" (auto-expired or manually marked lost — trader treats both the same)

If usage data later shows traders want finer filtering (e.g. separating `viewed` from `sent` to see who's opened the link), the tab grouping can be split in a follow-up.

### ContingencySlider visual sketch

```
  Contingency (uncertainty buffer)

  0%  ---------●---------  30%
              10%

  Base £120 becomes a range of £120 - £132.
  (API returns the exact numbers; UI shows them verbatim.)
```

The echoed "Base £X becomes a range of £Y - £Z" string is a live preview that reads from the current `totals` + computed `priceRange` returned by the API. During create (before an estimate exists), the preview shows placeholder text "Add line items to see your price range." Once line items exist, every PATCH of `contingencyPercent` triggers a refetch that updates the preview.

### Uncertainty chip + notes visual sketch

```
  Why is this an estimate?  (optional, pick any that apply)

  [ ✓ Site inspection needed ]  [ Hidden conditions ]  [ Materials & supply ]
  [ Access & working space ]    [ ✓ Scope unclear until investigation ]

  Other notes (optional)
  ┌──────────────────────────────────────────┐
  │ Customer mentioned existing pipes may    │
  │ be lead — would confirm on site visit.   │
  └──────────────────────────────────────────┘
```

Selected chips show the check mark prefix and `variant="default"` styling; unselected chips show `variant="outline"`. Empty notes textarea is fine — both fields are optional on the API.

### Golden-file test strategy (D-FMT-04)

```typescript
// src/lib/__tests__/currency.formatRange.test.ts
import { describe, it, expect } from "vitest";
import { formatRange } from "@/lib/currency";
import estimateFixture from "./fixtures/estimate-response.sample.json";

describe("formatRange", () => {
  it("matches explicit tuples", () => {
    const cases = [
      {
        low:  { subTotal: 10000, taxTotal: 2000, total: 12000 },
        high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
        mode: "range",
        currency: "GBP",
        expected: "£120.00 - £132.00",
      },
      // ... more tuples
    ];
    for (const c of cases) {
      expect(formatRange(c.low, c.high, c.mode, c.currency)).toBe(c.expected);
    }
  });

  it("agrees with a real API response (golden-file)", () => {
    const { priceRange, displayMode, currencyCode } = estimateFixture;
    expect(formatRange(priceRange.low, priceRange.high, displayMode, currencyCode))
      .toBe(estimateFixture.expectedFormattedRange);
  });
});
```

The fixture `estimate-response.sample.json` is copied from Phase 41's backend test output (specifically from the `EstimateRetriever` or `EstimateTotalsCalculator` spec's expected output) and includes an added `expectedFormattedRange: "£X.XX - £Y.YY"` field that the frontend computes by hand and commits. Any Phase 41-side change to the `IEstimatePriceRangeDto` shape breaks this test, forcing a coordinated update. Phase 43's planner chooses whether to script the fixture dump or copy-paste by hand — both are acceptable.

### Golden-file test edge cases

- Zero tax (all line items at `taxRate: 0`) — `low.total === low.subTotal`, `high.total === high.subTotal`
- Zero contingency (`contingencyPercent === 0`) — `low.total === high.total`; `range` mode renders `"£X - £X"`, `from` mode renders `"From £X"`
- Maximum contingency (`30%`) — plausible span
- Non-GBP currency — smoke test that `formatAmount` is doing the currency work

</specifics>

<deferred>
## Deferred Ideas

- **Dashboard "Create Document" quick action** — Phase 43 does not add a button to `DashboardPage`. A future phase will add it along with any necessary inline customer creation sub-flow. When it ships, it reuses `CreateDocumentDialog` unchanged (toggle visible, no defaultType).
- **Inline customer/job editing on estimate detail page** — Changing the customer or job of a Draft estimate post-create is deferred. Trader workaround: delete the Draft and recreate. Future phase will add either inline popovers or a dedicated "Reassign" dialog.
- **Trade-aware uncertainty chip presets per `BusinessTrade`** — Phase 43 ships five trade-agnostic chips. A new Future requirement is added to `REQUIREMENTS.md` (next available id, likely SMART-04) for per-trade chip vocabularies — plumber-specific pipework chips, electrician-specific wiring chips, builder-specific structural chips, etc. Requires product content design and is tracked for v1.9+.
- **Richer `EstimatesPage` tab filtering** — If usage data shows traders want to separate `viewed` from `sent` or isolate `site_visit_requested`, the 7-tab grouping can be split. Not a Phase 43 concern.
- **API-side pagination and status filter** — Client-side `filter()` on the full estimate array matches the existing quote pattern but won't scale. Deferred alongside the equivalent quote concern.
- **Related-documents card on `JobDetailPage`** — Showing related estimates next to related quotes on the job detail page. Planner may fold this into Phase 43 if simple; otherwise defer to a follow-up.
- **Bulk estimate actions** — Multi-select + bulk delete / bulk mark-lost. Not in the roadmap; not in Phase 43.
- **Estimate templates / presets** — Already listed as out-of-scope in REQUIREMENTS.md for v1.8.
- **Estimate PDF generation** — Out of scope per REQUIREMENTS.md (deferred with quote PDF).
- **Optimistic updates on inline detail-page PATCHes** — Planner may add these during implementation if the UX reads as laggy. Safe default is pessimistic with a subtle spinner.

### Reviewed Todos (not folded)
None — `.planning/STATE.md` shows "Pending Todos: None."

</deferred>

---

*Phase: 43-estimate-frontend-crud*
*Context gathered: 2026-04-11*
