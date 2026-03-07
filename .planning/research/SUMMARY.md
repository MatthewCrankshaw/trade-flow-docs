# Project Research Summary

**Project:** Trade Flow v1.1 - Item Tax Rate Linkage
**Domain:** Cross-module reference integration (item -> tax-rate) in a NestJS/MongoDB business management app
**Researched:** 2026-03-07
**Confidence:** HIGH

## Executive Summary

This milestone is a focused data model refactor, not a greenfield feature. The goal is to replace the `defaultTaxRate: number` field on items with a `taxRateId: ObjectId` reference to the existing tax-rate collection, then update the UI to use a dropdown instead of a number input. The codebase already has a proven pattern for exactly this kind of cross-module reference: schedules reference visit types via `visitTypeId`, validated at the service layer, resolved client-side via RTK Query cache. Item-tax rate linkage should follow this pattern verbatim.

No new libraries, dependencies, or architectural patterns are needed. Every tool is already in the codebase. The complexity is not in the design -- it is in the breadth of the change. The `defaultTaxRate` field is referenced in 15+ files across entity, DTO, repository mappers, request validators, response mappers, service validators, quote factories, default items creator, and merge utilities. Missing even one reference causes silent bugs, particularly in the quote creation pipeline where wrong tax rates directly affect invoicing accuracy.

The top three risks are: (1) quote line item factories break silently because they read `defaultTaxRate` directly from items, (2) the business onboarding flow breaks because default items are created before default tax rates, and (3) the UI displays stale or missing tax rate information if cache invalidation is not wired correctly. All three are preventable with thorough grep-based change tracking and integration tests.

## Key Findings

### Recommended Stack

No new technologies required. This is a pure refactor within the existing stack. See [STACK.md](STACK.md) for the full change inventory.

**Core technologies (all existing):**
- **NestJS service-layer validation:** Validate `taxRateId` exists via `TaxRateRetrieverService.findByIdOrFail()` -- same pattern as schedule `visitTypeId` validation
- **`@IsMongoId()` from class-validator:** Structural validation on request DTOs -- already available in the project
- **RTK Query client-side resolution:** Resolve tax rate names from `useGetTaxRatesQuery` cache -- no new API endpoints needed
- **MongoDB migration (IMigration interface):** Convert existing `defaultTaxRate: number` to `taxRateId: ObjectId` references

### Expected Features

See [FEATURES.md](FEATURES.md) for the full feature landscape with competitor analysis.

**Must have (table stakes):**
- Tax rate dropdown on item create/edit forms (replaces numeric input)
- Display linked tax rate name and percentage on item list/detail views
- Default tax rate pre-selection when creating new items
- "No tax" / exempt option for tax-exempt items
- Disabled tax rates hidden from dropdown (only enabled rates selectable)

**Should have (differentiators):**
- Tax rate inline preview in dropdown options (e.g., "VAT - 20%")

**Defer (v2+):**
- Quick-create tax rate from within the item form (medium complexity, not essential)
- Multiple tax rates per item (significant complexity, overkill for sole traders)
- Tax-inclusive pricing toggle per item (business-level setting, not per-item)

**Anti-features (do not build):**
- Tax rate override at item level (overrides happen at quote line item level, already supported)
- Hard deletion of tax rates (use existing `status: disabled` soft-delete)

### Architecture Approach

The architecture is a straightforward cross-module reference following the established schedule-to-visit-type pattern. The API stores only the `taxRateId`, validates it on create/update via the tax rate module's retriever service, and returns only the ID. The UI resolves display details client-side using RTK Query's cached tax rates. No server-side joins, no embedded objects in responses, no new endpoints. See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed component boundaries and data flow diagrams.

**Major components:**
1. **Item Module (API)** -- Entity, DTO, request, response, repository, services, and mapper utilities all change from `defaultTaxRate: number` to `taxRateId: ObjectId/string`
2. **Tax Rate Module (API)** -- Unchanged; already exports `TaxRateRetrieverService` with `findByIdOrFail()`
3. **Item Forms (UI)** -- MaterialItemForm, LabourItemForm, FeeItemForm all replace number input with tax rate select dropdown
4. **Item Display (UI)** -- ItemsTable and ItemsCardList resolve tax rate names from RTK Query cache via `taxRateMap`
5. **Migration Script** -- Converts existing `defaultTaxRate` numeric values to `taxRateId` references by matching rate values within the same business

### Critical Pitfalls

See [PITFALLS.md](PITFALLS.md) for the full list of 10 pitfalls with detection and prevention strategies.

1. **Quote factories break silently** -- `QuoteStandardLineItemFactory` and `QuoteBundleLineItemFactory` read `item.defaultTaxRate` to set tax on quote line items. Must update these to resolve the tax rate ID to a numeric percentage. Prevention: grep all `defaultTaxRate` references; update quote factories in the same phase; add integration test for quote creation from items.
2. **Business onboarding breaks** -- `DefaultBusinessItemsCreatorService` hardcodes `defaultTaxRate: 0`. After the change, it must reference the default tax rate by ID. Additionally, default tax rates may be created AFTER default items (ordering bug). Prevention: verify and fix creation ordering in `BusinessCreatorService.create()`; test full onboarding flow.
3. **Default items creator references nonexistent tax rate** -- If item creation runs before tax rate creation during onboarding, the tax rate ID validation fails. Prevention: ensure `defaultTaxRatesCreator.createDefaultTaxRates()` runs before `defaultItemsCreator.createDefaultItems()`.
4. **Bundle items and tax rate validation** -- Bundles must have `taxRateId: null` (they derive tax from components). The `@ValidateIf` decorator and service validation both check the old field name. Prevention: update all bundle-related conditional validation.
5. **Merge utility handles field type wrong** -- The `mergeExistingItemWithChanges` utility has a `mergeDefaultTaxRate` function for numeric merge logic. Must be replaced with string ID merge following the `visitTypeId` pattern. Prevention: update merge utility and its unit tests.

## Implications for Roadmap

Based on research, the work splits cleanly into two phases with a strict dependency: API changes must complete before UI changes can begin, because the UI depends on the API contract.

### Phase 1: API - Item Tax Rate Reference

**Rationale:** The API contract must change first. The UI cannot show a tax rate dropdown until the API accepts and returns `taxRateId`. All backend work is interdependent and should ship as one coordinated change.
**Delivers:** Updated item entity/DTO/request/response with `taxRateId`, cross-module validation, updated quote factories, updated default items creator, data migration script, new error code.
**Addresses:** Core data model change (table stakes foundation for all features)
**Avoids:** Pitfalls 1 (quote factories), 2 (missing validation), 3 (onboarding ordering), 4 (bundle validation), 7 (merge utility)

**Suggested build order within phase:**
1. Error codes -- services reference them
2. Entity + DTO + Response interfaces -- all layers depend on these types
3. Request classes with `@IsMongoId()` validation
4. Repository `toDto`/`toEntity` mapping
5. Mapper utilities (create, response, merge)
6. Module import (`TaxRateModule` into `ItemModule`)
7. Service validation (creator + updater)
8. Quote factories update (resolve ID to rate)
9. Default items creator update + onboarding ordering fix
10. Data migration script
11. Tests across all changes

### Phase 2: UI - Tax Rate Dropdown and Display

**Rationale:** Depends entirely on Phase 1's API contract. All UI changes are independent of each other and can be built in parallel once the API is deployed.
**Delivers:** Tax rate dropdown on all item forms, tax rate name/percentage display on item list/detail, default rate pre-selection, disabled rate handling.
**Addresses:** Tax rate dropdown (table stakes), display linked tax rate (table stakes), default pre-selection (table stakes), inline preview (differentiator)
**Avoids:** Pitfalls 5 (empty tax rates), 8 (Valibot schema), 9 (cache invalidation), 10 (disabled rates in dropdown)

**Suggested build order within phase:**
1. TypeScript types in `api.types.ts` -- all components depend on these
2. Valibot form schema update
3. `useItemForm` hook update
4. Item form components (Material/Labour/Fee) -- dropdown replaces number input
5. Item display components (ItemsTable/ItemsCardList) -- resolve tax rate names
6. Cache invalidation wiring

### Phase Ordering Rationale

- API before UI is a hard dependency -- the UI consumes the API contract
- Within the API phase, the order follows the dependency chain: types first, then mappers, then services, then consumers (quote factories, default creator)
- Quote factory updates MUST be in the API phase, not deferred -- they are a breaking change if left out
- Migration runs last in Phase 1 because it needs the new schema in place
- UI types first within Phase 2 because all components depend on the type definitions
- Forms before display because create/edit is the primary user interaction

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1 (API):** The quote factory update needs careful analysis -- the factory reads from item DTOs and the exact resolution path (where to look up the tax rate percentage from the ID) needs to be determined during planning. The onboarding ordering also needs verification of the actual call sequence in `BusinessCreatorService.create()`.

Phases with standard patterns (skip research-phase):
- **Phase 2 (UI):** All UI patterns are well-established. The schedule form dropdown and schedule table resolution patterns are direct templates. No research needed.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | No new tech. All patterns verified against actual codebase source files. |
| Features | HIGH | Competitor analysis (QuickBooks, Xero, Invoice Ninja, FreshBooks) confirms feature expectations. Edge cases well-documented. |
| Architecture | HIGH | Architecture derived entirely from existing codebase patterns (schedule -> visit-type). No speculation. |
| Pitfalls | HIGH | Pitfalls identified through direct codebase grep of `defaultTaxRate` references. Quote factory risk is real and well-understood. |

**Overall confidence:** HIGH

All research was conducted against the actual codebase, not external documentation or inference. The cross-module reference pattern is proven and in production. The primary risk is not architectural -- it is thoroughness of the change across 20+ files.

### Gaps to Address

- **Onboarding call ordering:** The exact order of `defaultTaxRatesCreator` vs `defaultItemsCreator` in `BusinessCreatorService.create()` needs verification during Phase 1 planning. PITFALLS.md flags this but the actual order was not confirmed.
- **Quote factory resolution path:** The quote factories need to resolve `taxRateId` to a numeric rate. Whether this uses `TaxRateRetrieverService.findByIdOrFail()` or a lighter lookup needs to be decided during implementation. The quote module may need its own import of `TaxRateModule`.
- **Pitfall 6 disagreement resolved:** PITFALLS.md suggests embedding resolved tax rate details in the item API response to avoid N+1 queries. ARCHITECTURE.md and STACK.md both recommend client-side resolution from RTK Query cache (matching the schedule pattern). **Recommendation: follow the established pattern (client-side resolution).** Tax rates are a small, already-cached dataset. The N+1 concern is invalid because the UI fetches all tax rates in one query, not per-item.
- **Data migration edge cases:** Items with `defaultTaxRate` values that do not match any existing tax rate in the business need a fallback strategy. Decide during Phase 1 planning whether to set to `null` with a warning log, or flag for manual resolution.

## Sources

### Primary (HIGH confidence)
- Direct codebase analysis of `trade-flow-api` and `trade-flow-ui` source files
- Pattern precedent: `ScheduleCreatorService` cross-module validation of `visitTypeId`
- Pattern precedent: `ScheduleTable` client-side resolution of visit type names
- Pattern precedent: `TaxRateModule` exports and `TaxRateRetrieverService` interface

### Secondary (MEDIUM confidence)
- [QuickBooks: Delete sales tax rates and agencies](https://quickbooks.intuit.com/learn-support/en-us/help-article/sales-taxes/delete-sales-tax-rates-agencies/L6JKrQWgQ_US_en_US) -- tax rate deactivation vs deletion
- [Invoice Ninja: Tax Settings](https://invoiceninja.github.io/docs/user-guide/taxes) -- per-item tax rate dropdown patterns
- [FreshBooks: How do I create an invoice?](https://support.freshbooks.com/hc/en-us/articles/216631328-How-do-I-create-an-Invoice-) -- tax line item UX patterns
- [Xero: Suspend, archive or delete a tax return](https://central.xero.com/s/article/Suspend-archive-or-delete-a-tax-return) -- archive vs delete approach

---
*Research completed: 2026-03-07*
*Ready for roadmap: yes*
