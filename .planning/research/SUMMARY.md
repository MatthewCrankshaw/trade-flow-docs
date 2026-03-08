# Project Research Summary

**Project:** Trade Flow v1.2 - Bundles & Quotes
**Domain:** Bundle management and quote creation for sole-trader business management (NestJS/React)
**Researched:** 2026-03-08
**Confidence:** HIGH

## Executive Summary

Trade Flow v1.2 is a brownfield milestone adding bundle component editing and quote management to an existing NestJS + React application for tradespeople. The research confirms that zero new dependencies are needed -- every required library (cmdk, Radix Collapsible, RTK Query, react-hook-form, valibot) is already installed and has working patterns in the codebase. The core work is building new UI pages and components (quote detail, line items, searchable picker) while fixing existing bundle editing restrictions and a known bundle creation bug.

The recommended approach is dependency-first: fix the bundle creation bug first (it blocks all bundle work), then build the reusable SearchableItemPicker component (needed by both bundle editing and quote line items), then complete bundle CRUD, then wire up the quote UI to the existing API. The API already handles quote creation, line item addition with bundle decomposition, and total calculation with correct Money precision. The frontend work is primarily about consuming these existing endpoints via RTK Query and rendering the data with expandable bundle rows.

The top risks are: (1) the BundleItemForm silently drops bundleConfig on edit due to an `if (!isEditMode)` guard that must be removed alongside the read-only restriction, (2) RTK Query cache staleness if line item mutations do not invalidate the parent quote tag, and (3) bundle component double-counting in totals if `parentLineItemId` is not set correctly in the persistence pipeline. All three have clear prevention strategies documented in the pitfalls research.

## Key Findings

### Recommended Stack

No new packages are required. The existing stack covers all v1.2 needs. See [STACK.md](STACK.md) for full details.

**Core technologies (all existing):**
- **cmdk + Radix Popover**: Searchable combobox pattern -- already working in CreateJobDialog, reuse for bundle item picker and quote customer/item selectors
- **Radix Collapsible + shadcn Table**: Expandable table rows -- use for bundle component display on quote line items
- **RTK Query**: API data fetching -- "Quote" tag already registered in api.ts, endpoints need to be built in a new quoteApi.ts
- **react-hook-form + valibot**: Form management -- established pattern for all forms in the project
- **useCurrency hook**: Currency formatting -- display all server-calculated money values, never do client-side arithmetic

Libraries explicitly NOT needed: @tanstack/react-table (overkill), downshift/react-select (redundant with cmdk), PDF libraries (v1.3), dnd-kit (over-engineering), decimal.js (server handles money).

### Expected Features

See [FEATURES.md](FEATURES.md) for the full feature landscape with dependency graph.

**Must have (table stakes):**
- Fix bundle creation bug (unit field defaults to "bundle") -- unblocks all bundle work
- Component editing on existing bundles -- currently blocked by read-only guard and submit handler exclusion
- Searchable item dropdown -- current flat Select is unusable at scale, reuse Command+Popover pattern
- Improved bundle component display -- show per-component pricing and visual hierarchy
- Quote API integration (RTK Query endpoints) -- currently hardcoded mock data
- Quote creation dialog -- customer picker, title, notes
- Quote detail view with totals -- header, line items, subtotal/tax/total
- Add line items to quotes -- item picker + quantity, triggers server-side pricing
- Bundle line items on quotes with expandable rows -- chevron to reveal component breakdown

**Should have (differentiators for future):**
- Inline quantity editing on quote line items
- Quote duplication
- Quick-add from recent items
- Bundle price override per quote

**Defer (v1.3+):**
- PDF/print quote export (high complexity, major standalone feature)
- Quote acceptance/rejection workflow (explicitly deferred per PROJECT.md)
- Customer-facing quote portal
- Optional bundle component selection on quotes

### Architecture Approach

This is a brownfield build following established patterns throughout. See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed component diagrams, data flows, and implementation patterns.

The architecture uses full array replacement for bundle component editing (PATCH with complete components array, not granular endpoints), quote-level RTK Query caching (no separate line item cache), and a QuoteTransitionService mirroring the existing ScheduleTransitionService for status state machine management. All pricing and total calculation happens server-side; the UI is strictly a display layer for money values.

**Major components:**
1. **SearchableItemPicker** (NEW shared) -- Combobox for item selection, used in bundle editing and quote line item addition
2. **QuoteDetailPage** (NEW page) -- Full quote view with header, line items list, expandable bundles, totals, status actions
3. **LineItemsList + BundleExpandRow** (NEW components) -- Renders line items with collapsible bundle component rows
4. **AddLineItemDialog** (NEW component) -- Item picker + quantity input for adding items to quotes
5. **quoteApi.ts** (NEW RTK Query slice) -- getQuotes, getQuote, createQuote, addLineItem, removeLineItem, updateQuoteStatus
6. **QuoteTransitionService** (NEW API service) -- Status state machine (draft -> sent -> accepted/rejected/expired)
7. **BundleItemForm + BundleComponentsList** (MODIFY) -- Remove edit-mode restrictions, wire component editing to API PATCH

### Critical Pitfalls

See [PITFALLS.md](PITFALLS.md) for the full list of 13 pitfalls with detection and prevention strategies.

1. **BundleItemForm drops bundleConfig on edit (Pitfall 2)** -- The submit handler has `if (!isEditMode)` guard that skips bundleConfig. Must remove this AND the `isReadOnly` guard in BundleComponentsList in the same commit, or editing renders but saves nothing.
2. **RTK Query cache staleness after line item mutations (Pitfall 4)** -- Mutations must invalidate `{ type: "Quote", id: quoteId }` to trigger quote detail refetch. Use single quote-level cache granularity, not separate line item tags.
3. **Bundle component double-counting in totals (Pitfall 3)** -- QuoteTotalsCalculator skips items with parentLineItemId. If the persistence pipeline fails to set this field on component line items, totals are 2x actual. Verify end-to-end with tests.
4. **Bundle edits do not propagate to existing quotes (Pitfall 1)** -- Intentional: quotes are point-in-time snapshots. Do NOT build sync logic. Communicate this clearly in the UI.
5. **Duplicate components in bundle config (Pitfall 6)** -- Add uniqueness validation on component itemId in both Valibot schema (client) and BundleConfigValidator (server). Filter already-selected items from the picker dropdown.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Bundle Bug Fix and Foundation
**Rationale:** The bundle creation bug blocks all bundle work. Fixing it first and extending the API PATCH endpoint to accept components establishes the foundation for everything else.
**Delivers:** Working bundle creation with correct unit field; API support for component editing via PATCH; reusable SearchableItemPicker component.
**Addresses:** Fix bundle creation bug, searchable item dropdown, API component editing contract.
**Avoids:** Pitfall 10 (unit field bug), Pitfall 2 (submit handler guard -- establish correct API contract first).

### Phase 2: Bundle Component Editing
**Rationale:** Depends on Phase 1 (API contract and SearchableItemPicker). Completes bundle CRUD which is a prerequisite for meaningful bundle-on-quote testing.
**Delivers:** Full bundle component editing in edit mode; improved bundle display with pricing; duplicate prevention.
**Addresses:** Component editing on existing bundles, improved bundle component display.
**Avoids:** Pitfall 2 (both guards removed together), Pitfall 6 (duplicate components), Pitfall 13 (quantity validation), Pitfall 11 (form state reset).

### Phase 3: Quote API Integration and List View
**Rationale:** Independent of bundle editing (can run in parallel with Phase 2). Wires the quote UI to the existing API, replacing hardcoded mock data. Must establish the RTK Query cache strategy before building interactive quote features.
**Delivers:** quoteApi.ts with all endpoints; quotes list from real API data; quote creation dialog.
**Addresses:** Quote listing from API, quote creation dialog.
**Avoids:** Pitfall 4 (define cache invalidation strategy upfront).

### Phase 4: Quote Detail and Line Items
**Rationale:** Depends on Phase 3 (quoteApi.ts) and benefits from Phase 1 (SearchableItemPicker). This is the core quote experience.
**Delivers:** Quote detail page with line items; add line items to quotes; expandable bundle rows; totals display; status transitions.
**Addresses:** Quote detail view, add line items, bundle line items on quotes, quote totals, quote status transitions.
**Avoids:** Pitfall 3 (double-counting -- verify parentLineItemId pipeline), Pitfall 7 (visual hierarchy for bundles), Pitfall 8 (display server-calculated totals only), Pitfall 9 (graceful fallback for deleted items), Pitfall 12 (no optimistic updates on totals).

### Phase Ordering Rationale

- Phase 1 must come first: the bundle bug and SearchableItemPicker are dependencies for nearly everything else
- Phases 2 and 3 can run in parallel since they have no cross-dependencies (bundle editing vs. quote API wiring)
- Phase 4 depends on Phase 3 (needs quoteApi.ts) and reuses components from Phase 1 (SearchableItemPicker)
- Grouping bundle work (Phases 1-2) and quote work (Phases 3-4) reduces context switching
- The architecture research confirms steps 1, 3, 6, 9 from the build order can start in parallel -- this maps to Phase 1 foundation work

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 2 (Bundle Component Editing):** The interaction between BundleItemForm submit handler, API merge utility, and BundleConfigValidator needs careful tracing. The `mergeExistingItemWithChanges` utility must be verified to handle bundleConfig.components replacement correctly.
- **Phase 4 (Quote Detail and Line Items):** The parentLineItemId pipeline through QuoteBundleLineItemFactory -> QuoteLineItemCreator -> QuoteTotalsCalculator needs end-to-end verification. Also, Money value object serialization format needs to be confirmed before building display logic.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Bug Fix and Foundation):** All patterns are well-documented in the codebase. The SearchableItemPicker is a direct extraction of the CreateJobDialog combobox pattern.
- **Phase 3 (Quote API Integration):** RTK Query endpoint creation follows the exact pattern in itemApi.ts. Tag strategy is straightforward and documented in architecture research.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies verified against package.json and existing usage in codebase. Zero new dependencies. |
| Features | HIGH | Feature list derived from existing codebase gaps, PROJECT.md scope, and domain analysis of tradesperson quoting tools. |
| Architecture | HIGH | All patterns derived from existing codebase (brownfield). Every recommendation mirrors a working pattern already in the project. |
| Pitfalls | HIGH | Pitfalls identified from codebase analysis with specific file/line references. Top risks have concrete prevention strategies. |

**Overall confidence:** HIGH -- This is brownfield work on a well-structured codebase. All research is grounded in verified codebase patterns, not external assumptions.

### Gaps to Address

- **Money value object serialization format:** The exact JSON shape of Money fields in API responses needs confirmation before building quote display. Check `money.value-object.ts` and response mappers during Phase 4 planning.
- **API PATCH endpoint for bundleConfig.components:** The architecture research specifies the contract but it needs to be built. Verify that `mergeExistingItemWithChanges` correctly handles the `components` field during Phase 2 implementation.
- **Quote line item `name` field:** Pitfall 9 identifies that deleted items may break display. Verify whether quote line items already snapshot the item name or if this needs a schema addition. Handle gracefully in the UI either way.
- **Existing bundle data with `unit: "bundle"`:** Determine whether existing bad data needs a migration or can be fixed lazily on next edit.

## Sources

### Primary (HIGH confidence)
- Codebase: `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` -- Command+Popover combobox pattern
- Codebase: `trade-flow-ui/src/components/ui/collapsible.tsx` -- Radix Collapsible component
- Codebase: `trade-flow-ui/src/features/items/components/forms/shared/BundleComponentsList.tsx` -- current Select picker and read-only guard
- Codebase: `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` -- submit handler `isEditMode` guard
- Codebase: `trade-flow-api/src/quote/controllers/quote.controller.ts` -- existing quote endpoints with nested line items
- Codebase: `trade-flow-api/src/quote/services/quote-updater.service.ts` -- addLineItem flow with bundle decomposition
- Codebase: `trade-flow-api/src/schedule/services/schedule-transition.service.ts` -- transition pattern to mirror
- Codebase: `trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts` -- PATCH merge pattern
- Codebase: `trade-flow-ui/src/services/api.ts` -- base API slice with "Quote" tag registered
- Codebase: `trade-flow-ui/package.json` -- verified installed dependency versions

### Secondary (MEDIUM confidence)
- [commercetools Bundle Management](https://docs.commercetools.com/foundry/best-practice-guides/product-bundles) -- restrict direct component line item modifications
- [Trax Group ERP Bundle Issues](https://traxgroup.com/erp-bundle-management-issues/) -- common pitfalls with bundle pricing
- [Shadcn Combobox Patterns](https://shadcnstudio.com/docs/components/combobox) -- searchable dropdown variants

### Tertiary (LOW confidence)
- [YourTradebase Quoting](https://www.yourtradebase.com/electricians/quote-software) -- tradesperson quoting workflow patterns
- [Extraflow Quoting](https://extraflow.io/industries/estimating-and-quoting-software-for-plumbing) -- plumber quoting workflows
- [Extensiv Bundle/Kit UI](https://help.extensiv.com/om-products/creating-bundleskits-through-the-ui) -- bundle component editing UI patterns

---
*Research completed: 2026-03-08*
*Ready for roadmap: yes*
