# Phase 7: Schedule Edit and Management UI - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can edit existing schedule entries (date, time, duration, visit type), cancel entries, and transition statuses through valid states. All actions happen from the detail dialog via a three-dot action menu. The edit form reuses ScheduleFormDialog in edit mode. Terminal status transitions require confirmation. Notes remain editable in all states.

</domain>

<decisions>
## Implementation Decisions

### Edit mode & form design
- Reuse ScheduleFormDialog for editing -- pass existing schedule data as props to switch to "edit mode"
- Edit triggered from three-dot DropdownMenu in detail dialog header -- clicking "Edit" closes detail dialog, opens ScheduleFormDialog pre-filled
- Edit mode title: "Edit Visit" (vs "Schedule Visit" for create), submit button: "Save Changes" (vs "Schedule Visit")
- Same inline calendar layout as create mode -- identical form, different props
- After saving edit: dialog closes, success toast "Visit updated", list refreshes via RTK Query cache invalidation
- Notes omitted from edit form entirely -- notes editing stays exclusively in the detail dialog (quick edit without opening full form)
- Edit form fields: date, start time, duration, visit type (no notes, no status)
- Inline warning when editing a confirmed schedule and time/duration fields are dirty: "Changing time will reset status to Scheduled." -- subtle info note, no extra confirmation step

### Status transition controls
- All transitions accessed from three-dot DropdownMenu in detail dialog top-right header
- Menu contains Edit action + valid status transitions for the current status
- Menu items change dynamically per status:
  - Scheduled: Edit, Confirm, Cancel Visit, Mark No-show
  - Confirmed: Edit, Mark Complete, Cancel Visit
  - Terminal (completed/canceled/no-show): menu hidden entirely
- After successful transition: stay in dialog, status badge updates in place, menu items update to reflect new valid transitions, toast confirms action
- Non-terminal transitions (scheduled->confirmed) happen immediately on click -- no confirmation dialog

### Cancel flow & confirmation
- All terminal transitions require confirmation via AlertDialog:
  - Cancel Visit: "This visit will be marked as canceled. This can't be undone." [Go Back] [Cancel Visit]
  - Mark Complete: "Mark this visit as completed? This can't be undone." [Go Back] [Mark Complete]
  - Mark No-show: "Mark this visit as a no-show? This can't be undone." [Go Back] [Mark No-show]
- Non-terminal transition (Confirm) is immediate -- no confirmation dialog needed
- After confirmation: transition fires, dialog stays open, status updates inline, toast shows result

### Locked state messaging
- Three-dot menu hidden entirely for terminal statuses (completed, canceled, no-show)
- Subtle info note below status badge: "Schedule locked. Notes can still be edited."
- Notes textarea remains editable in ALL states (including terminal) -- useful for post-visit documentation
- Save Notes button available for all statuses
- "Locked" applies to schedule fields only (date, time, duration, visit type, status) -- not notes

### Claude's Discretion
- Exact AlertDialog styling and button colors (destructive variant for Cancel Visit, etc.)
- DropdownMenu icon placement and sizing in the dialog header
- How to pass schedule data to ScheduleFormDialog (new props, or wrapper pattern)
- Status reset warning exact styling and positioning
- Whether Edit action in menu uses pencil icon or text only
- Valibot schema adjustments for edit mode (if any needed vs create schema)

</decisions>

<specifics>
## Specific Ideas

- The three-dot menu pattern consolidates all actions (edit + transitions) into one discoverable place -- no action buttons scattered across the dialog
- ScheduleFormDialog reuse keeps the create and edit experience consistent -- same calendar, same dropdowns, same validation
- Notes staying editable after terminal status is important for tradespeople -- they often add notes after completing a visit ("Job went well", "Customer happy") or after a no-show ("Called twice, no answer")
- The inline status reset warning for confirmed schedules prevents confusion -- tradesperson understands why a confirmed visit went back to scheduled after they changed the time

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ScheduleFormDialog` (Phase 5): create form with calendar, time/duration dropdowns, visit type selector -- add edit mode props
- `ScheduleDetailDialog` (Phase 6): read-only detail view with notes editing -- add three-dot menu and locked state messaging
- `useScheduleActions` hook: wraps mutations with toast -- extend with transition and update actions
- `scheduleApi` (RTK Query): has create, list, update mutations -- needs transition endpoint added
- `STATUS_CONFIG` in schedule-utils: color-coded badge config for all 5 statuses
- shadcn/ui: DropdownMenu, AlertDialog, Dialog, Badge components available
- `useGetVisitTypesQuery`: for visit type dropdown population in edit form
- `scheduleFormSchema` (Valibot): validation schema for create form -- may need edit variant

### Established Patterns
- Dialog-based forms: React Hook Form + Valibot + FormProvider (VisitTypeFormDialog, ScheduleFormDialog)
- RTK Query cache invalidation via tags ("Schedule" tag with job-scoped IDs)
- Toast notifications for success/error on mutations
- Key-based component reset: ScheduleDetailContent keyed by schedule.id

### Integration Points
- `ScheduleDetailDialog.tsx`: add DropdownMenu in header, locked state note, conditional notes read-only
- `ScheduleFormDialog.tsx`: add edit mode (schedule prop, title/button text, pre-fill values, PATCH instead of POST)
- `scheduleApi.ts`: add transition mutation endpoint (POST /transition with { status })
- `useScheduleActions.ts`: add transitionStatus and updateSchedule methods
- Phase 4 API: PATCH /schedule/:id for field updates, POST /schedule/:id/transition for status changes

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 07-schedule-edit-and-management-ui*
*Context gathered: 2026-03-07*
