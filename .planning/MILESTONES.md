# Milestones

## v1.2 Bundles & Quotes (Shipped: 2026-03-15)

**Phases completed:** 4 phases, 12 plans, 25 tasks
**Timeline:** 7 days (2026-03-08 to 2026-03-14)
**Execution time:** ~30 min (avg 3min/plan)
**Codebase:** ~19k LOC API + ~22.3k LOC UI (TypeScript)
**Git range:** feat(11-01) to feat(14-06) — 23 feature commits across 2 repos

**Key accomplishments:**
- Fixed bundle creation bug and built reusable SearchableItemPicker component (Popover+Command combobox)
- Full bundle component editing — add/remove/update quantities on existing bundles with two-line display and type badges
- Quote system wired to API — creation dialog (dual-context), list with real data and tab filtering, status transitions (Draft/Sent/Accepted/Rejected)
- Quote detail page with line item CRUD — add items/bundles, inline edit quantity/price, soft-delete, expandable bundle rows with native table sub-rows
- Bundle pricing strategy moved to item-level setting (BundleItemForm radio group) with immediate-add in quote flow
- Tax-inclusive line totals, correct bundle tax calculation (Money.percentageOf fix), responsive mobile card layout

---

## v1.1 Item Tax Rate Linkage (Shipped: 2026-03-08)

**Phases completed:** 2 phases, 6 plans
**Timeline:** 1 day (2026-03-08)
**Execution time:** ~1 hour (avg 4min/plan)
**Codebase:** ~18.2k LOC API + ~20.3k LOC UI (TypeScript)

**Key accomplishments:**
- Item entity migrated from numeric defaultTaxRate to taxRateId ObjectId reference across entity, DTO, repository, requests, response, and mappers
- Tax rate existence validation on both item create and update services with unit tests
- Onboarding flow reordered — tax rates created before items, default tax rate ID threaded to item creator
- Quote factories resolve tax rate percentage from taxRateId via TaxRateRepository
- Item forms (Material, Labour, Fee) show tax rate dropdown with "Name (rate%)" format and default pre-selection
- OpenAPI spec fully migrated from defaultTaxRate to taxRateId

---

## v1.0 Scheduling (Shipped: 2026-03-07)

**Phases completed:** 8 phases, 16 plans, 0 tasks

**Timeline:** 15 days (2026-02-21 to 2026-03-07)
**Execution time:** 1.5 hours (avg 6min/plan)
**Codebase:** ~17.4k LOC API + ~20.3k LOC UI (TypeScript)

**Key accomplishments:**
- Visit type system with CRUD backend, default generation per trade, and full management UI
- Schedule data model and API with cross-module validation, smart defaults, and structured filter parser
- Status state machine (Scheduled/Confirmed/Completed/Canceled/No-show) with validated transitions
- Schedule creation UI with calendar picker, time/duration selects, visit type dropdown, and empty state
- Responsive schedule list (table/card), detail dialog with editable notes, and status badges
- Edit mode, dropdown menu with status transitions, confirmation dialogs, and locked-state messaging
- Job detail integration replacing mock data with schedule-derived hints and per-status badge breakdown

---

