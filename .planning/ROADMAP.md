# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2026-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2026-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (in progress)

## Phases

<details>
<summary>v1.0 Scheduling (Phases 1-8) -- SHIPPED 2026-03-07</summary>

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

<details>
<summary>v1.1 Item Tax Rate Linkage (Phases 9-10) -- SHIPPED 2026-03-08</summary>

- [x] Phase 9: Item Tax Rate API (4/4 plans) -- completed 2026-03-08
- [x] Phase 10: Item Tax Rate UI (2/2 plans) -- completed 2026-03-08

Full details: `.planning/milestones/v1.1-ROADMAP.md`

</details>

### v1.2 Bundles & Quotes (In Progress)

**Milestone Goal:** Complete the partially-built bundle feature (fix creation bug, enable component editing, searchable item picker, improved display) and wire up the quote system with bundle support (API integration, creation flow, detail view with expandable bundle lines).

- [x] **Phase 11: Bundle Bug Fix and Foundation** - Fix bundle creation bug and build reusable SearchableItemPicker component (completed 2026-03-08)
- [x] **Phase 12: Bundle Component Editing** - Enable full component editing on existing bundles with improved display (completed 2026-03-08)
- [ ] **Phase 13: Quote API Integration** - Wire quote UI to existing API with creation flow and status transitions
- [ ] **Phase 14: Quote Detail and Line Items** - Build quote detail view with line item management and expandable bundle rows

## Phase Details

### Phase 11: Bundle Bug Fix and Foundation
**Goal**: Users can create bundles without errors and select items through a fast, searchable picker
**Depends on**: Nothing (first phase of v1.2)
**Requirements**: BNDL-01, BNDL-03
**Success Criteria** (what must be TRUE):
  1. User can create a new bundle item and the unit field correctly defaults to "bundle" without form errors
  2. User can search and filter items by name when selecting components for a bundle via a combobox dropdown
  3. SearchableItemPicker component works in both bundle creation and bundle editing contexts
**Plans**: 1 plan

Plans:
- [ ] 11-01-PLAN.md -- Fix bundle unit bug, create SearchableItemPicker, integrate into BundleComponentsList

### Phase 12: Bundle Component Editing
**Goal**: Users can fully manage bundle components on existing bundles with clear visual display
**Depends on**: Phase 11 (SearchableItemPicker, bug fix)
**Requirements**: BNDL-02, BNDL-04
**Success Criteria** (what must be TRUE):
  1. User can add new component items to an existing bundle via the edit form
  2. User can remove component items and change quantities on an existing bundle
  3. User can view bundle components in a structured list showing item name, quantity, and unit (e.g., "10 x metres")
  4. Changes to bundle components persist correctly after saving the edit form
**Plans**: 2 plans

Plans:
- [ ] 12-01-PLAN.md -- Extend API update endpoint for bundle components, update frontend type
- [ ] 12-02-PLAN.md -- Enhance component display with two-line format and type badges (edit form + items list)

### Phase 13: Quote API Integration
**Goal**: Users can create quotes, view them in a list from real API data, and manage quote status
**Depends on**: Nothing (independent of Phase 12; can run after Phase 11 or in parallel with Phase 12)
**Requirements**: QUOT-01, QUOT-02, QUOT-04
**Success Criteria** (what must be TRUE):
  1. User can create a new quote linked to a job and customer via a creation dialog
  2. User can view a list of all quotes populated from real API data (no mock/hardcoded data)
  3. User can transition a quote through its status lifecycle (Draft to Sent to Accepted/Rejected)
**Plans**: TBD

Plans:
- [ ] 13-01: TBD
- [ ] 13-02: TBD

### Phase 14: Quote Detail and Line Items
**Goal**: Users can view full quote details with line items, add items and bundles to quotes, and see calculated totals
**Depends on**: Phase 13 (quoteApi.ts endpoints), Phase 11 (SearchableItemPicker)
**Requirements**: QUOT-03, QLIT-01, QLIT-02, QLIT-03, QLIT-04
**Success Criteria** (what must be TRUE):
  1. User can view a quote detail page showing header information and a list of line items
  2. User can add standard items (material, labour, fee) to a quote with quantity
  3. User can add bundle items to a quote and see them as a single rolled-up line
  4. User can expand a bundle line item to reveal its individual component breakdown
  5. User can view quote totals (subtotal, tax, total) calculated from line items with correct bundle handling (no double-counting)
**Plans**: TBD

Plans:
- [ ] 14-01: TBD
- [ ] 14-02: TBD
- [ ] 14-03: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 11 -> 12 -> 13 -> 14
(Note: Phases 12 and 13 have no cross-dependency and could execute in parallel)

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
| 10. Item Tax Rate UI | v1.1 | 2/2 | Complete | 2026-03-08 |
| 11. Bundle Bug Fix and Foundation | 1/1 | Complete    | 2026-03-08 | - |
| 12. Bundle Component Editing | 2/2 | Complete    | 2026-03-08 | - |
| 13. Quote API Integration | v1.2 | 0/2 | Not started | - |
| 14. Quote Detail and Line Items | v1.2 | 0/3 | Not started | - |
