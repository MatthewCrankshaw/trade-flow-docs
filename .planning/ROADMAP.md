# Roadmap: Trade Flow

## Milestones

- ✅ **v1.0 Scheduling** -- Phases 1-8 (shipped 2026-03-07)
- 🚧 **v1.1 Item Tax Rate Linkage** -- Phases 9-10 (in progress)

## Phases

<details>
<summary>✅ v1.0 Scheduling (Phases 1-8) -- SHIPPED 2026-03-07</summary>

- [x] Phase 1: Visit Type Backend (2/2 plans) -- completed 2026-02-23
- [x] Phase 2: Visit Type Management UI (2/2 plans) -- completed 2026-02-28
- [x] Phase 3: Schedule Data Model and Create API (2/2 plans) -- completed 2026-03-01
- [x] Phase 4: Schedule Status and CRUD API (3/3 plans) -- completed 2026-03-01
- [x] Phase 5: Schedule Creation UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 6: Schedule List and Detail UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 7: Schedule Edit and Management UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 8: Job Detail Integration (1/1 plan) -- completed 2026-03-07

Full details: `.planning/milestones/v1.0-ROADMAP.md`

</details>

### 🚧 v1.1 Item Tax Rate Linkage (In Progress)

**Milestone Goal:** Update items to reference tax rates by ID instead of storing a numeric value, connecting the existing tax rate system to items across API and UI.

- [x] **Phase 9: Item Tax Rate API** - Items reference tax rates by ID with cross-module validation, updated quote factories, and fixed onboarding (completed 2026-03-08)
- [ ] **Phase 10: Item Tax Rate UI** - Tax rate dropdown on item forms and tax rate display on item lists

## Phase Details

### Phase 9: Item Tax Rate API
**Goal**: Items store and validate a tax rate ID reference, and all downstream consumers (quotes, onboarding) work correctly with the new field
**Depends on**: Phase 8 (v1.0 complete)
**Requirements**: ITAX-01, ITAX-02, ITAX-03, ITAX-04, ITAX-05, QUOT-01
**Success Criteria** (what must be TRUE):
  1. Creating an item with a valid tax rate ID succeeds; creating with a nonexistent or wrong-business tax rate ID returns a validation error
  2. Updating an item's tax rate ID validates the new reference and persists the change
  3. Item API responses include the tax rate ID field (not a numeric rate)
  4. Creating a quote from items with tax rate references correctly resolves the tax percentage on each line item
  5. Signing up and completing onboarding creates default items that reference the correct default tax rate (not a hardcoded number)
**Plans:** 4/4 plans complete

Plans:
- [ ] 09-01-PLAN.md -- Replace defaultTaxRate with taxRateId across item data layer (entity, DTO, repo, requests, response, mappers)
- [ ] 09-02-PLAN.md -- Add tax rate ID validation to item creator and updater services
- [ ] 09-03-PLAN.md -- Fix onboarding flow and update quote factories for tax rate resolution

### Phase 10: Item Tax Rate UI
**Goal**: Users select tax rates from a dropdown when creating or editing items, and see linked tax rate details when viewing items
**Depends on**: Phase 9
**Requirements**: ITMUI-01, ITMUI-02, ITMUI-03
**Success Criteria** (what must be TRUE):
  1. Item create form shows a dropdown of the business's tax rates (name and percentage visible) instead of a numeric input
  2. Item edit form shows the same dropdown with the item's current tax rate pre-selected
  3. Item list and detail views display the linked tax rate name and percentage (not a raw ID)
**Plans:** 2 plans

Plans:
- [ ] 10-01-PLAN.md -- Update Item types, form schema, and useItemForm hook to use taxRateId
- [ ] 10-02-PLAN.md -- Replace numeric tax input with tax rate dropdown in Material, Labour, and Fee forms

## Progress

**Execution Order:**
Phases execute in numeric order: 9 -> 10

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Visit Type Backend | v1.0 | 2/2 | Complete | 2026-02-23 |
| 2. Visit Type Management UI | v1.0 | 2/2 | Complete | 2026-02-28 |
| 3. Schedule Data Model and Create API | v1.0 | 2/2 | Complete | 2026-03-01 |
| 4. Schedule Status and CRUD API | v1.0 | 3/3 | Complete | 2026-03-01 |
| 5. Schedule Creation UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 6. Schedule List and Detail UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 7. Schedule Edit and Management UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 8. Job Detail Integration | v1.0 | 1/1 | Complete | 2026-03-07 |
| 9. Item Tax Rate API | v1.1 | 4/4 | Complete | 2026-03-08 |
| 10. Item Tax Rate UI | v1.1 | 0/2 | Not started | - |
