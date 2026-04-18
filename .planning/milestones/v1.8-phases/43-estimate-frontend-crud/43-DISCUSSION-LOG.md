# Phase 43: Estimate Frontend CRUD - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-11
**Phase:** 43 - estimate-frontend-crud
**Areas discussed:** Create dialog shape, features/estimates scaffolding, ContingencySlider + chips UX, Detail page edit affordance

---

## Gray Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| Create dialog shape | Shared CreateDocumentDialog with Quote/Estimate toggle vs separate CreateEstimateDialog (+ toggle entry point). Tension with separation-over-DRY memory. | ✓ |
| features/estimates scaffolding | How much of features/quotes to mirror verbatim vs factoring shared components. Covers 1:1 duplication vs shared abstraction. | ✓ |
| ContingencySlider + chips UX | Slider implementation, uncertainty chips interaction, display-mode toggle placement. | ✓ |
| Detail page edit affordance | Inline detail-page editing vs dedicated edit dialog; line-item editing strategy. | ✓ |

**User selected:** All four areas.

---

## Area 1: Create dialog shape

### Q1.1: How should the Create Quote / Create Estimate dialog be structured?

| Option | Description | Selected |
|--------|-------------|----------|
| Shared CreateDocumentDialog | Single dialog replaces CreateQuoteDialog. Top-of-form toggle picks Quote or Estimate; form branches to show estimate-only fields when Estimate is selected. Single source of truth for job/customer selection. | |
| Separate dialogs, page-level toggle | Keep CreateQuoteDialog untouched. Build a new CreateEstimateDialog that duplicates job/customer selection + adds estimate fields. The 'toggle' is a pre-step on the parent page. Aligns with Separation-over-DRY for UI. | |
| Shared shell, separate bodies | One outer dialog owns the Quote/Estimate toggle + shared job/customer picker, delegates to <CreateQuoteForm> or <CreateEstimateForm> inner components. Middle-ground: no duplication of the job/customer picker, full separation of type-specific fields. | ✓ |

**User's choice:** Shared shell, separate bodies

**Notes:** "From the tradesperson's perspective, the first decision when creating a document isn't 'what fields do I fill in' — it's 'who is this for and what job is it about.' Customer and job selection are always the first step regardless of document type. The second decision is 'do I know enough to give a fixed price, or is this a rough indication?' That's the quote/estimate toggle. Everything after that diverges — quotes need expiry dates and acceptance mechanics, estimates need contingency sliders and uncertainty notes."

---

### Q1.2: When the dialog opens from Quotes page or Estimates page with a default type, what does the Quote/Estimate toggle do?

| Option | Description | Selected |
|--------|-------------|----------|
| Visible + switchable | Toggle shown with preselected type; trader can flip mid-creation. | |
| Visible but locked | Toggle shown preselected and disabled; trader must cancel to switch. | |
| Hidden when preselected | Toggle only appears from Dashboard or Job Detail; per-page buttons skip the toggle entirely. | ✓ |

**User's choice:** Hidden when preselected

---

### Q1.3: Is the Dashboard 'Create Document' quick action in scope for Phase 43?

| Option | Description | Selected |
|--------|-------------|----------|
| In scope — add the button | Phase 43 adds the Dashboard button with inline customer/job creation sub-flows. | |
| In scope — add, but no inline customer creation | Dashboard button added; missing-customer case shows PrerequisiteAlert. | |
| Defer — not a Phase 43 concern | Dashboard button captured as a Deferred Idea; Phase 43 stays focused on Estimates page + detail page + shared dialog from Quotes/Estimates/JobDetail. | ✓ |

**User's choice:** Defer — not a Phase 43 concern

---

## Area 2: features/estimates scaffolding

### Q2.1: How much of features/quotes should features/estimates duplicate verbatim vs factor out shared pieces?

| Option | Description | Selected |
|--------|-------------|----------|
| 1:1 mirror (Recommended) | Full feature-module duplication with quote→estimate renames. Zero cross-imports. Accepts duplication cost per Separation-over-DRY. | ✓ |
| Mirror + lift shared ui helpers | Duplicate feature components but lift generic helpers (formatRange, etc.) into src/lib/. | |
| Generic DocumentTable with props | Single generic <DocumentsTable documentType="quote"|"estimate"> with config object. Violates Separation-over-DRY for UI. | |

**User's choice:** 1:1 mirror

---

### Q2.2: Where does the new formatRange helper live, and what does it return?

| Option | Description | Selected |
|--------|-------------|----------|
| src/lib/currency.ts export | formatRange added to the existing currency module. Golden-file test in src/lib/__tests__/. | ✓ |
| features/estimates/lib/formatRange.ts | Estimate-specific helper co-located with consumers. | |
| useFormatRange hook | Hook wrapping useBusinessCurrency() to avoid passing currencyCode at every call site. | |

**User's choice:** src/lib/currency.ts export

---

### Q2.3: What should Estimate.priceRange look like in the TypeScript type?

| Option | Description | Selected |
|--------|-------------|----------|
| Match API DTO exactly | priceRange: { low: { subTotal, taxTotal, total }, high: { subTotal, taxTotal, total } }. Full fidelity. | ✓ |
| Slimmer UI shape | priceRange: { low: number, high: number } — just inclusive totals. | |
| Defer — mirror Phase 41 DTO | Decide at plan time once Phase 41 DTO is locked. | |

**User's choice:** Match API DTO exactly

---

### Q2.4: Does EstimatesPage get its own /estimates route, and where does it sit in the nav?

| Option | Description | Selected |
|--------|-------------|----------|
| New route + sidebar link | Add /estimates and /estimates/:estimateId routes. Add 'Estimates' sidebar link next to 'Quotes'. | ✓ |
| New route, no new nav link | Routes but no sidebar link. Only reachable from JobDetailPage and direct URL. | |

**User's choice:** New route + sidebar link

---

### Q2.5: Which tabs should appear on EstimatesPage for filtering?

| Option | Description | Selected |
|--------|-------------|----------|
| All statuses as tabs | 10 tabs (deleted excluded). | |
| Grouped status tabs (Recommended) | 7 tabs: All, Draft, Sent (sent+viewed), Responded (responded+site_visit_requested), Converted, Declined, Expired/Lost. | ✓ |
| Key-states tabs only | 6 tabs; less-used statuses only visible in the 'All' view. | |

**User's choice:** Grouped status tabs

---

## Area 3: ContingencySlider + chips UX

### Q3.1: What drives the ContingencySlider component?

| Option | Description | Selected |
|--------|-------------|----------|
| Install @radix-ui/react-slider | New dependency; shadcn-style slider.tsx + ContingencySlider on top. Accessible out of the box. | ✓ |
| RadioGroup with 7 buttons | 7 pill buttons at 0/5/10/15/20/25/30%. Zero new deps. | |
| Native range input | Plain <input type="range">. Zero deps, weaker a11y. | |

**User's choice:** Install @radix-ui/react-slider

---

### Q3.2: Which direction should Phase 43 take on uncertainty chips?

Clarification raised by user: *"Trade Flow supports 10 trades via BusinessTrade enum (plumber, electrician, heating_gas_engineer, carpenter_joiner, builder_general_contractor, painter_decorator, tiler_flooring, landscaping_gardening, handyman, other). Are the original four chips (site inspection, pipework, materials, access) relevant to all trades? Is this sufficiently covering enough uncertainties to be a useful feature?"*

Claude audited against the ten trades: `pipework` is irrelevant for 6 of 10 trades (electricians, carpenters, builders, painters, tilers, landscapers). The other three chips are universal but miss the real per-trade uncertainty sources. Four options were presented:

| Option | Description | Selected |
|--------|-------------|----------|
| Option A — generic chips | Ship 5 trade-agnostic presets: Site inspection needed, Hidden conditions, Materials & supply, Access & working space, Scope unclear until investigation. Update CONT-04 and ROADMAP success criterion wording. | ✓ |
| Option D — generic chips + defer trade-aware (Recommended) | Same as A plus explicit Future requirement for per-trade chip dispatch. | |
| Option B — trade-aware chips now | Read business.trade, render per-trade chip array. 10 chip sets to design. High scope. | |
| Option C — freeform notes only | Drop the chip preset concept entirely. Single textarea with placeholder hint. | |

**User's choice:** Option A — generic chips

**Notes:** Final stored values and UI labels:
- `site_inspection` → "Site inspection needed"
- `hidden_conditions` → "Hidden conditions"
- `materials_supply` → "Materials & supply"
- `access_working_space` → "Access & working space"
- `scope_unclear` → "Scope unclear until investigation"

A Future requirement (SMART-04 or next available id) for trade-aware chip dispatch will be added to REQUIREMENTS.md as part of Phase 43's doc edits. Phase 43 also must update CONT-04 wording in REQUIREMENTS.md and Phase 43 success criterion #1 in ROADMAP.md + milestones/v1.8-ROADMAP.md to reflect the new five.

---

### Q3.3: How should the five uncertainty chips be rendered and selected?

| Option | Description | Selected |
|--------|-------------|----------|
| Toggle buttons (Recommended) | Five pill buttons in a wrapped row. variant="default" when selected, variant="outline" when not. Freeform notes textarea below. | ✓ |
| Checkbox list | Five checkboxes using existing checkbox.tsx. More vertical space; less tap-friendly. | |
| Combobox multi-select | command.tsx-based multi-select dropdown. Saves space but adds click depth. | |

**User's choice:** Toggle buttons

---

### Q3.4: Where is the display-mode toggle (range / 'from £X') editable, and can it change post-create?

| Option | Description | Selected |
|--------|-------------|----------|
| Create + edit on detail (Recommended) | Toggle in CreateEstimateForm (default='range'). Also editable inline on EstimateDetailPage next to the price display while status=Draft. PATCH sends displayMode. | ✓ |
| Create only | Toggle only in CreateEstimateForm. Immutable post-create. | |
| Create + edit in dialog | Create + dedicated Edit dialog for post-create changes. | |

**User's choice:** Create + edit on detail

---

## Area 4: Detail page edit affordance

### Q4.1: How does a trader edit a Draft estimate's core fields?

| Option | Description | Selected |
|--------|-------------|----------|
| Inline on detail page (Recommended) | Draft estimates show editable controls directly on the detail page. Slider/chips/notes/display-mode all live PATCH on change/blur. No dialog. | ✓ |
| Dedicated Edit dialog | Detail shows read-only; an Edit button opens an EditEstimateDialog that mirrors CreateEstimateForm. | |
| Hybrid: inline quick-edit, dialog for bulk | Slider/chips/mode inline; notes/customer/job require a dialog. | |

**User's choice:** Inline on detail page

---

### Q4.2: How does a trader edit/add/remove estimate line items on the detail page?

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror QuoteLineItemsCard exactly (Recommended) | Build EstimateLineItemsCard as 1:1 mirror. SearchableItemPicker, inline edit, delete per row. Draft-only. | ✓ |
| Read-only line items in Phase 43 | Detail page shows line items but doesn't let you edit them. Trader must delete and recreate to change them. | |

**User's choice:** Mirror QuoteLineItemsCard exactly

---

### Q4.3: What does 'Delete' on a Draft estimate look like from the detail page?

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror quote delete confirm | Delete button (Draft only); AlertDialog confirm matching QuotesPage copy pattern. | ✓ |
| Dropdown menu with delete | More/kebab dropdown containing Delete. | |

**User's choice:** Mirror quote delete confirm

---

## Claude's Discretion

Areas where the user deferred to Claude / planner judgement:

- Exact icon for the "Estimates" sidebar link (distinct from quote FileText).
- Exact home for `CreateDocumentDialog` (`src/components/` vs new `src/features/documents/`).
- Whether `EstimateActionStrip` ships as a real component or a minimal stub in Phase 43.
- Exact ARIA labels on chip toggle buttons and `ContingencySlider`.
- Placeholder copy on the `uncertaintyNotes` textarea.
- Whether `JobDetailPage` renders a "Related Documents" card combining quotes and estimates.
- Whether optimistic updates are used on the inline detail-page PATCHes.
- Id prefix for the new Future requirement (`SMART-04` or next available id).
- Whether the golden-file test fixture is captured automatically from Phase 41 test output or by hand.

## Deferred Ideas

Captured in 43-CONTEXT.md `<deferred>` section:

- Dashboard "Create Document" quick action
- Inline customer/job editing on estimate detail page
- Trade-aware uncertainty chip presets per BusinessTrade (tracked as new Future requirement)
- Richer EstimatesPage tab filtering (split viewed/sent, isolate site_visit_requested)
- API-side pagination and status filter
- Related-documents card on JobDetailPage
- Bulk estimate actions
- Optimistic updates on inline detail-page PATCHes
