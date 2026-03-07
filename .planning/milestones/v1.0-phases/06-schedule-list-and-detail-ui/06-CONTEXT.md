# Phase 6: Schedule List and Detail UI - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace mock schedule data in the job detail Schedule tab with real schedule entries from the API, displayed chronologically with visit details and status. Users can view schedule details in a dialog and add/edit notes on existing entries. Editing schedule fields and status transitions are Phase 7.

</domain>

<decisions>
## Implementation Decisions

### List layout & info density
- Follow existing patterns: table for desktop (768px+), card list for mobile -- matching visit types and job types sections (Phase 2 pattern)
- Each entry shows: date, start time, duration, visit type (with color dot), status badge
- No assignee column -- solo operator, always the current user
- Small notes icon/indicator on entries that have notes (tells user there's more to see)
- Canceled and no-show entries shown with muted styling (lighter text, faded) -- preserves history

### Status display
- Color-coded badges: Scheduled=blue, Confirmed=green, Completed=gray, Canceled=red, No-show=amber
- Badge is the primary status indicator in both table rows and mobile cards

### Detail dialog
- Clicking a schedule entry opens a detail dialog (not inline expand)
- Dialog shows all schedule fields as read-only text (date, time, duration, visit type, status)
- Notes textarea is the only editable field in this phase
- Phase 7 will add full edit mode to this same dialog
- Save button updates notes via the existing PATCH schedule endpoint

### Notes display & add flow
- Notes displayed inside the detail dialog (not inline in the list)
- Editable textarea in the detail dialog -- user types/edits notes and saves
- Uses the existing PATCH /schedule/:id endpoint to update notes field

### Chronological ordering & grouping
- Two sections: "Upcoming" (future dates, ascending) then "Past" (past dates, descending)
- Labeled section headers ("Upcoming" / "Past") as subtle labels above each group
- Tab header count shows total entries (all statuses)

### Empty state
- Use existing EmptyTabState component pattern (icon + title + description + action). "Schedule Visit" action opens the creation dialog from Phase 5
- Remove the "+Add" button from the schedule tab header -- action strip is the primary creation trigger

### Claude's Discretion
- Loading skeleton design for the list
- Exact column widths and responsive breakpoint details
- Date/time formatting (e.g. "Tue 10 Mar" vs "10/03/2026" vs "March 10")
- Duration display format (e.g. "2 hrs" vs "2h" vs "120 min")
- Detail dialog layout and spacing
- Notes icon choice and placement in list rows
- Muted styling approach for canceled/no-show entries (opacity, strikethrough, or both)
- Status badge color exact shades

</decisions>

<specifics>
## Specific Ideas

- The schedule tab replaces MOCK_SCHEDULES in JobDetailTabs.tsx with real RTK Query data from the scheduleApi built in Phase 5
- Detail dialog is designed to be extended in Phase 7 with edit mode -- structure it so switching from read-only to editable is a mode toggle, not a full rebuild
- The "Upcoming" / "Past" split matches how tradespeople think -- "what's coming up on this job" vs "what we've already done"

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `EmptyTabState` component in JobDetailTabs.tsx -- used by Photos and Files tabs, reuse for schedule empty state
- `TabHeader` component in JobDetailTabs.tsx -- shows title + count (remove Add button for schedule tab)
- `ScheduleFormDialog` (Phase 5) -- creation dialog, can be opened from empty state action
- `scheduleApi` (Phase 5) -- RTK Query endpoints: `useGetSchedulesByJobQuery`, `useCreateScheduleMutation`
- `useScheduleActions` hook (Phase 5) -- wraps mutations with toast notifications
- `Schedule` and `ScheduleStatus` types in `src/types/api.types.ts`
- shadcn/ui components: Dialog, Table, Badge, Card, Button
- Visit type API (Phase 2) -- `useGetVisitTypesQuery` for resolving visitTypeId to name + color

### Established Patterns
- Responsive list: table for desktop, card list for mobile (JobTypesSection, VisitTypesSection)
- RTK Query cache invalidation via tags ("Schedule" tag already added in Phase 5)
- Dialog-based detail views consistent with form dialogs across the app

### Integration Points
- `JobDetailTabs.tsx` -- replace `MOCK_SCHEDULES`, `ScheduleTable` component with real data
- `scheduleApi.ts` -- may need additional endpoint for PATCH/update notes
- Visit types API -- resolve visitTypeId to display name and color dot in list

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 06-schedule-list-and-detail-ui*
*Context gathered: 2026-03-07*
