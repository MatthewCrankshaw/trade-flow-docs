# Requirements: Trade Flow — Scheduling Milestone

**Defined:** 2026-02-21
**Core Value:** A job is the centre of the business — Trade Flow helps tradespeople run their entire business from first call to final payment

## v1 Requirements

Requirements for the scheduling milestone. Each maps to roadmap phases.

### Scheduling

- [ ] **SCHED-01**: User can create a schedule entry on a job with date, start time, and duration
- [ ] **SCHED-02**: User can view all schedule entries for a job in chronological order
- [ ] **SCHED-03**: User can edit an existing schedule entry (date, time, duration)
- [ ] **SCHED-04**: User can cancel a schedule entry (status change, entry preserved in history)
- [ ] **SCHED-05**: User can add free-text notes to a schedule entry
- [ ] **SCHED-06**: Schedule assignee defaults to the logged-in user
- [ ] **SCHED-07**: Duration defaults to 1 hour when not specified

### Visit Status

- [ ] **VSTAT-01**: Schedule entries have status: Scheduled, Confirmed, Completed, Canceled, No-show
- [ ] **VSTAT-02**: User can transition status following valid state machine (scheduled→confirmed→completed, scheduled→canceled, scheduled→no-show, confirmed→canceled)
- [ ] **VSTAT-03**: Invalid status transitions are rejected by the API

### Visit Types

- [ ] **VTYPE-01**: User can select a visit type when creating a schedule entry
- [ ] **VTYPE-02**: Default visit types are generated per trade when a business is created
- [ ] **VTYPE-03**: User can create custom visit types
- [ ] **VTYPE-04**: User can view and manage visit types for their business

### Integration

- [ ] **INTG-01**: Schedule entries replace MOCK_SCHEDULES in the job detail page
- [ ] **INTG-02**: Job detail page shows schedule count/status summary
- [ ] **INTG-03**: Empty state displayed when a job has no schedule entries

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Scheduling Modes

- **SMODE-01**: User can create schedule entry with arrival window mode (window start, window end, duration)
- **SMODE-02**: Schedule list clearly distinguishes exact time vs arrival window entries
- **SMODE-03**: Mode selector (exact vs window) in schedule creation form

### Conflict Detection

- **CONF-01**: User sees warning when creating a schedule that overlaps with an existing one
- **CONF-02**: Conflicts are advisory only — user can proceed despite overlap
- **CONF-03**: Overlapping schedules are visually highlighted in the schedule list

### Future

- **FUT-01**: "Today" screen showing all schedules for the current day with quick actions
- **FUT-02**: Customer notification settings on schedule entries
- **FUT-03**: Required items/materials linked to schedule entries
- **FUT-04**: Team member assignment on schedules (multi-user support)
- **FUT-05**: Standalone calendar view

## Out of Scope

| Feature | Reason |
|---------|--------|
| Drag-and-drop calendar | No calendar view this milestone — schedules on job page only |
| Route optimization | Solo operator with local work — over-engineering |
| Automated scheduling / AI | Tradespeople want control, not automation |
| Real-time location tracking | Privacy concerns, solo operator doesn't need self-tracking |
| Recurring schedules / templates | Scope creep — trades jobs are mostly one-off |
| Customer-facing booking portal | Tradespeople get work via calls and referrals, not online booking |
| Google Calendar integration | Not needed when schedules are job-centric, not calendar-centric |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| SCHED-01 | — | Pending |
| SCHED-02 | — | Pending |
| SCHED-03 | — | Pending |
| SCHED-04 | — | Pending |
| SCHED-05 | — | Pending |
| SCHED-06 | — | Pending |
| SCHED-07 | — | Pending |
| VSTAT-01 | — | Pending |
| VSTAT-02 | — | Pending |
| VSTAT-03 | — | Pending |
| VTYPE-01 | — | Pending |
| VTYPE-02 | — | Pending |
| VTYPE-03 | — | Pending |
| VTYPE-04 | — | Pending |
| INTG-01 | — | Pending |
| INTG-02 | — | Pending |
| INTG-03 | — | Pending |

**Coverage:**
- v1 requirements: 17 total
- Mapped to phases: 0
- Unmapped: 17 ⚠️

---
*Requirements defined: 2026-02-21*
*Last updated: 2026-02-21 after initial definition*
