# Roadmap: Trade Flow - Scheduling Milestone

## Overview

This milestone adds job scheduling to Trade Flow, enabling tradespeople to create and manage visit schedules on their jobs. The build progresses from visit type infrastructure (a prerequisite data type) through schedule backend CRUD and status management, into frontend UI for creating, viewing, editing, and managing schedules, culminating in integration with the existing job detail page. Each phase delivers a verifiable capability across the two-repo architecture (trade-flow-api and trade-flow-ui).

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Visit Type Backend** - Data model, CRUD API, and default visit type generation per trade (completed 2026-02-23)
- [ ] **Phase 2: Visit Type Management UI** - Frontend for viewing, managing, and creating custom visit types
- [ ] **Phase 3: Schedule Data Model and Create API** - Schedule collection, create endpoint with smart defaults
- [ ] **Phase 4: Schedule Status and CRUD API** - Status state machine, list/edit/cancel/notes endpoints
- [ ] **Phase 5: Schedule Creation UI** - Create schedule dialog with visit type selection and empty state
- [ ] **Phase 6: Schedule List and Detail UI** - Chronological schedule list view with notes display
- [ ] **Phase 7: Schedule Edit and Management UI** - Edit form, cancel action, and status transition controls
- [ ] **Phase 8: Job Detail Integration** - Replace mock data, add schedule summary to job detail page

## Phase Details

### Phase 1: Visit Type Backend
**Goal**: Visit types exist as a managed data type with sensible defaults generated per trade
**Depends on**: Nothing (first phase)
**Requirements**: VTYPE-02
**Success Criteria** (what must be TRUE):
  1. When a new business is created for a specific trade, default visit types appropriate to that trade are automatically generated
  2. Visit types are stored in their own collection and retrievable via API by business ID
  3. The visit type data model supports name, description, and business association
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — Visit Type CRUD module (entity, repository, services, controller, policy, configuration)
- [ ] 01-02-PLAN.md — Default visit type generation at onboarding + MongoDB indexes

### Phase 2: Visit Type Management UI
**Goal**: Users can view their visit types and create custom ones for their business
**Depends on**: Phase 1
**Requirements**: VTYPE-03, VTYPE-04
**Success Criteria** (what must be TRUE):
  1. User can navigate to a visit types management area and see all visit types for their business
  2. User can create a new custom visit type with a name and it appears in the list
  3. Default visit types generated during onboarding are visible alongside custom ones
**Plans**: 2 plans

Plans:
- [x] 02-01-PLAN.md -- Backend patch (isDefault field, remove active-only filter) + frontend API/types/schema + Scheduling tab (completed 2026-02-28)
- [ ] 02-02-PLAN.md -- Visit types list UI, form dialog, color picker, status filter, and management actions

### Phase 3: Schedule Data Model and Create API
**Goal**: Schedules can be created via the API with smart defaults that match how tradespeople think
**Depends on**: Phase 1
**Requirements**: SCHED-06, SCHED-07
**Success Criteria** (what must be TRUE):
  1. A schedule entry can be created via API with date, start time, and duration on a specific job
  2. When duration is not provided, it defaults to 1 hour
  3. The schedule assignee automatically defaults to the authenticated user
  4. The schedule data model supports job reference, visit type reference, date/time, duration, notes, status, and assignee (modeled for future team support)
**Plans**: 2 plans

Plans:
- [x] 03-01-PLAN.md -- Schedule data model, repository, policy, mappers (SCHED-06/SCHED-07 defaults), module wiring, migration, error codes (completed 2026-03-01)
- [ ] 03-02-PLAN.md -- Creator service (cross-module validation), controller (POST endpoint), unit tests, OpenAPI spec

### Phase 4: Schedule Status and CRUD API
**Goal**: The API supports full schedule lifecycle including status transitions with validation
**Depends on**: Phase 3
**Requirements**: VSTAT-01, VSTAT-02, VSTAT-03
**Success Criteria** (what must be TRUE):
  1. Schedule entries carry a status field with values: Scheduled, Confirmed, Completed, Canceled, No-show
  2. Valid status transitions succeed (scheduled to confirmed, confirmed to completed, scheduled to canceled, scheduled to no-show, confirmed to canceled)
  3. Invalid status transitions are rejected by the API with a clear error message
  4. Full CRUD is available: list schedules by job, update schedule fields, cancel (status change preserving history), and add/edit notes
**Plans**: 3 plans

Plans:
- [x] 04-01-PLAN.md -- State machine, services (retriever/updater/transition), repository list+update, error codes, index migration (completed 2026-03-01)
- [x] 04-02-PLAN.md -- Controller endpoints (list/get/update/transition), unit tests, OpenAPI spec (completed 2026-03-01)
- [ ] 04-03-PLAN.md -- Gap closure: structured filter parser, case-insensitive status fix, replace ad-hoc query params

### Phase 5: Schedule Creation UI
**Goal**: Users can create schedule entries on a job through an intuitive form with visit type selection
**Depends on**: Phase 2, Phase 3, Phase 4
**Requirements**: SCHED-01, VTYPE-01, INTG-03
**Success Criteria** (what must be TRUE):
  1. User can open a creation form from the job detail page and create a schedule entry with date, start time, and duration
  2. User can select a visit type from a dropdown populated with their business visit types when creating a schedule
  3. When a job has no schedule entries, an informative empty state is displayed with a prompt to create the first one
  4. The creation form uses smart defaults (1-hour duration, current user as assignee) so minimal input is needed
**Plans**: TBD

Plans:
- [ ] 05-01: TBD

### Phase 6: Schedule List and Detail UI
**Goal**: Users can see all scheduled visits for a job and read visit details including notes
**Depends on**: Phase 5
**Requirements**: SCHED-02, SCHED-05
**Success Criteria** (what must be TRUE):
  1. User can view all schedule entries for a job displayed in chronological order
  2. Each schedule entry shows date, time, duration, visit type, status, and assignee
  3. User can add free-text notes to a schedule entry and see them displayed on the entry
**Plans**: TBD

Plans:
- [ ] 06-01: TBD

### Phase 7: Schedule Edit and Management UI
**Goal**: Users can modify, cancel, and manage the status of their schedule entries
**Depends on**: Phase 6
**Requirements**: SCHED-03, SCHED-04
**Success Criteria** (what must be TRUE):
  1. User can edit an existing schedule entry (change date, time, duration, visit type)
  2. User can cancel a schedule entry and it remains visible in the list with canceled status
  3. User can transition schedule status through valid states (e.g., mark as confirmed, mark as completed)
  4. Invalid status transitions are prevented in the UI (unavailable options are not shown)
**Plans**: TBD

Plans:
- [ ] 07-01: TBD

### Phase 8: Job Detail Integration
**Goal**: The job detail page shows real schedule data with a useful summary replacing all mock data
**Depends on**: Phase 7
**Requirements**: INTG-01, INTG-02
**Success Criteria** (what must be TRUE):
  1. MOCK_SCHEDULES data is completely removed from JobDetailTabs.tsx and replaced with real API data
  2. The job detail page shows a schedule count and status summary (e.g., "3 visits: 1 completed, 1 confirmed, 1 scheduled")
  3. All schedule functionality works end-to-end from job detail: create, view, edit, cancel, status transitions
**Plans**: TBD

Plans:
- [ ] 08-01: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Visit Type Backend | 2/2 | Complete    | 2026-02-23 |
| 2. Visit Type Management UI | 1/2 | In progress | - |
| 3. Schedule Data Model and Create API | 1/2 | In progress | - |
| 4. Schedule Status and CRUD API | 1/2 | In progress | - |
| 5. Schedule Creation UI | 0/? | Not started | - |
| 6. Schedule List and Detail UI | 0/? | Not started | - |
| 7. Schedule Edit and Management UI | 0/? | Not started | - |
| 8. Job Detail Integration | 0/? | Not started | - |
