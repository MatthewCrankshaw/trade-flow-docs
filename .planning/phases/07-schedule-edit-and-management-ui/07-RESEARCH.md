# Phase 7: Schedule Edit and Management UI - Research

**Researched:** 2026-03-07
**Domain:** React UI -- form editing, status state machine, action menus, confirmation dialogs
**Confidence:** HIGH

## Summary

Phase 7 adds edit and status management capabilities to existing schedule UI components from Phases 5 and 6. The work is entirely frontend -- all API endpoints already exist (PATCH for updates, POST /transition for status changes). The core approach is extending two existing components: `ScheduleFormDialog` gains an edit mode via a schedule prop, and `ScheduleDetailDialog` gains a three-dot DropdownMenu with edit and transition actions plus AlertDialog confirmations for terminal transitions.

The API backend is fully built with proper validation: `ScheduleUpdaterService` rejects updates on terminal statuses, auto-resets confirmed-to-scheduled when time/duration changes, and validates visit type IDs. `ScheduleTransitionService` validates against the state machine. The frontend needs to add two new RTK Query endpoints (transition mutation and update mutation with broader data type), extend `useScheduleActions`, wire the DropdownMenu, and handle confirmation dialogs.

**Primary recommendation:** Split into two plans -- (1) API layer + edit form mode, (2) DropdownMenu + transitions + confirmations + locked state. This separates data-flow concerns from interaction patterns.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Reuse ScheduleFormDialog for editing -- pass existing schedule data as props to switch to "edit mode"
- Edit triggered from three-dot DropdownMenu in detail dialog header -- clicking "Edit" closes detail dialog, opens ScheduleFormDialog pre-filled
- Edit mode title: "Edit Visit" (vs "Schedule Visit" for create), submit button: "Save Changes" (vs "Schedule Visit")
- Same inline calendar layout as create mode -- identical form, different props
- After saving edit: dialog closes, success toast "Visit updated", list refreshes via RTK Query cache invalidation
- Notes omitted from edit form entirely -- notes editing stays exclusively in the detail dialog
- Edit form fields: date, start time, duration, visit type (no notes, no status)
- Inline warning when editing a confirmed schedule and time/duration fields are dirty: "Changing time will reset status to Scheduled." -- subtle info note, no extra confirmation step
- All transitions accessed from three-dot DropdownMenu in detail dialog top-right header
- Menu contains Edit action + valid status transitions for the current status
- Menu items change dynamically per status:
  - Scheduled: Edit, Confirm, Cancel Visit, Mark No-show
  - Confirmed: Edit, Mark Complete, Cancel Visit
  - Terminal (completed/canceled/no-show): menu hidden entirely
- After successful transition: stay in dialog, status badge updates in place, menu items update to reflect new valid transitions, toast confirms action
- Non-terminal transitions (scheduled->confirmed) happen immediately on click -- no confirmation dialog
- All terminal transitions require confirmation via AlertDialog:
  - Cancel Visit: "This visit will be marked as canceled. This can't be undone." [Go Back] [Cancel Visit]
  - Mark Complete: "Mark this visit as completed? This can't be undone." [Go Back] [Mark Complete]
  - Mark No-show: "Mark this visit as a no-show? This can't be undone." [Go Back] [Mark No-show]
- Three-dot menu hidden entirely for terminal statuses (completed, canceled, no-show)
- Subtle info note below status badge: "Schedule locked. Notes can still be edited."
- Notes textarea remains editable in ALL states (including terminal)
- Save Notes button available for all statuses

### Claude's Discretion
- Exact AlertDialog styling and button colors (destructive variant for Cancel Visit, etc.)
- DropdownMenu icon placement and sizing in the dialog header
- How to pass schedule data to ScheduleFormDialog (new props, or wrapper pattern)
- Status reset warning exact styling and positioning
- Whether Edit action in menu uses pencil icon or text only
- Valibot schema adjustments for edit mode (if any needed vs create schema)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SCHED-03 | User can edit an existing schedule entry (date, time, duration) | ScheduleFormDialog edit mode with schedule prop, PATCH `/schedule/:id` endpoint already exists, `UpdateScheduleRequest` accepts optional startDateTime, durationMinutes, visitTypeId |
| SCHED-04 | User can cancel a schedule entry (status change, entry preserved in history) | Transition endpoint POST `/schedule/:id/transition` with `{ status }` body already exists, AlertDialog confirmation for cancel, canceled entries remain in list with muted styling |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19 | Component framework | Project standard |
| React Hook Form | latest | Form state management | Used in ScheduleFormDialog, VisitTypeFormDialog |
| Valibot | latest | Schema validation | Paired with React Hook Form via @hookform/resolvers |
| RTK Query | (Redux Toolkit) | API state & cache | Project standard for all API calls |
| shadcn/ui | New York | UI components | Project standard, DropdownMenu + AlertDialog available |
| date-fns | latest | Date formatting | Used in ScheduleFormDialog for date formatting |
| Lucide React | latest | Icons | Project icon library |

### Components Already Available
| Component | Source | Used For |
|-----------|--------|----------|
| DropdownMenu | `@/components/ui/dropdown-menu` | Three-dot action menu |
| AlertDialog | `@/components/ui/alert-dialog` | Terminal transition confirmations |
| Dialog | `@/components/ui/dialog` | Already used in both form and detail dialogs |
| Badge | `@/components/ui/badge` | Status badges (already rendered) |
| Calendar | `@/components/ui/calendar` | Date picker in form (already used) |
| Select | `@/components/ui/select` | Time/duration/visit-type dropdowns (already used) |

**Installation:** No new packages needed. All required components are already installed.

## Architecture Patterns

### Existing Component Extension Pattern
```
ScheduleFormDialog.tsx  (Phase 5 -- add edit mode)
  +-- schedule?: Schedule prop (null = create, defined = edit)
  +-- title/button text conditional on mode
  +-- defaultValues from schedule prop
  +-- PATCH instead of POST on submit
  +-- notes field hidden in edit mode

ScheduleDetailDialog.tsx  (Phase 6 -- add actions)
  +-- DropdownMenu in header (three-dot)
  +-- Status transition actions
  +-- AlertDialog for terminal confirmations
  +-- Locked state messaging
  +-- onEdit callback to parent
```

### State Flow for Edit
```
ScheduleDetailDialog (detail view)
  --> User clicks three-dot menu --> "Edit"
  --> onEdit(schedule) callback fires
  --> Parent closes detail dialog, opens ScheduleFormDialog with schedule prop
  --> User edits and saves
  --> PATCH /schedule/:id fires
  --> Cache invalidation refreshes list
  --> Dialog closes, toast shows
```

### State Flow for Transitions
```
ScheduleDetailDialog (detail view)
  --> User clicks three-dot menu --> transition action
  --> If terminal: AlertDialog opens for confirmation
  --> If non-terminal (Confirm): immediate API call
  --> POST /schedule/:id/transition with { status }
  --> On success: dialog stays open, schedule data updates via cache invalidation
  --> Status badge and menu items update reactively
```

### RTK Query Cache Strategy
```typescript
// Transition mutation invalidates same tags as update
invalidatesTags: (_result, _error, { jobId, scheduleId }) => [
  { type: "Schedule", id: scheduleId },
  { type: "Schedule", id: `JOB-${jobId}` },
]
```

### Anti-Patterns to Avoid
- **Local state for schedule data after transition:** Don't store schedule in component state. Let RTK Query cache be the source of truth. After transition, the invalidation will refetch and the component re-renders with new data.
- **Separate edit dialog component:** Don't create a new EditScheduleDialog. Reuse ScheduleFormDialog with a mode prop -- this is a locked decision.
- **Status string comparison without constant map:** Use a `VALID_TRANSITIONS` constant map on the frontend mirroring the backend, not ad-hoc string checks.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Status transition validation | Custom if/else chains | Transition map constant mirroring backend | Backend already validates; frontend map is for UI only (hiding invalid menu items) |
| Confirmation dialogs | Custom modal state | shadcn AlertDialog | Built-in accessible pattern with proper focus management |
| Action menu | Custom popover with buttons | shadcn DropdownMenu | Built-in keyboard navigation, focus management, portal rendering |
| Form state for edit | Manual state management | React Hook Form with defaultValues from schedule | Handles dirty tracking, validation, submission consistently |

## Common Pitfalls

### Pitfall 1: Status String Mismatch (no_show vs no-show)
**What goes wrong:** The API enum uses `no_show` (underscore) but the frontend type definition has `no-show` (hyphen). Sending `no-show` to the transition endpoint will fail validation.
**Why it happens:** The `ScheduleStatus` type in `trade-flow-ui/src/types/api.types.ts` line 332 defines `"no-show"` but `trade-flow-api/src/schedule/enum/schedule-status.enum.ts` defines `NO_SHOW = "no_show"`.
**How to avoid:** Verify what the API actually returns for no-show schedules. If the API returns `no_show`, the frontend type and STATUS_CONFIG need to be updated. If there's already a mapping layer, use it consistently. The transition request body MUST send `no_show` to match the API enum.
**Warning signs:** Status badge not rendering for no-show entries, transition to no-show failing with 400 error.
**Confidence:** HIGH -- verified by reading both enum files directly.

### Pitfall 2: ScheduleFormDialog Key Reset
**What goes wrong:** Opening edit mode for different schedules shows stale form data from the previous edit.
**Why it happens:** React Hook Form caches defaultValues. Without a key change, the form doesn't re-initialize.
**How to avoid:** Key the ScheduleFormContent by `schedule?.id ?? "create"` so form state resets when switching between different schedules or between create/edit modes.
**Warning signs:** Editing schedule A, closing, editing schedule B shows schedule A's data.

### Pitfall 3: Stale Detail Dialog After Transition
**What goes wrong:** After a status transition, the dialog still shows old status because it uses the prop passed from the parent, not the latest cache data.
**Why it happens:** The parent passes a `schedule` object from the list render. After transition, cache invalidation triggers a refetch, but the dialog needs to consume the updated data.
**How to avoid:** After transition mutation, the `invalidatesTags` on the list query will cause refetch. The detail dialog should either: (a) use the schedule ID to read from cache directly via `selectFromResult`, or (b) the parent should pass the latest schedule from the refetched list. Option (a) is cleaner.
**Warning signs:** Status badge doesn't update after clicking Confirm, menu items don't change.

### Pitfall 4: Notes Field Visibility in Edit Mode
**What goes wrong:** Notes appear in the edit form even though they should only be editable in the detail dialog.
**Why it happens:** The existing ScheduleFormDialog has a notes field.
**How to avoid:** Conditionally render the notes field only when in create mode (no schedule prop). The edit Valibot schema should omit the notes field entirely, or the notes field can simply be hidden and the form submission should not include notes.

### Pitfall 5: Update API Accepts Notes But Edit Form Should Not Send Them
**What goes wrong:** The PATCH endpoint accepts `notes` in the body. If the edit form sends `notes: undefined` or `notes: null`, it could accidentally clear notes.
**Why it happens:** The `UpdateScheduleRequest` has an optional `notes` field.
**How to avoid:** When building the PATCH request body from the edit form, explicitly exclude `notes` from the payload. Only send `startDateTime`, `durationMinutes`, and `visitTypeId`.

### Pitfall 6: Confirmed Status Reset Warning Timing
**What goes wrong:** The warning about status resetting to "Scheduled" shows even when time/duration haven't changed.
**Why it happens:** Checking status alone without checking dirty state of specific fields.
**How to avoid:** Use React Hook Form's `formState.dirtyFields` to check if `startTime` or `durationMinutes` fields are actually dirty. Only show warning when `schedule.status === "confirmed"` AND (`dirtyFields.startTime` OR `dirtyFields.durationMinutes` OR `dirtyFields.date`).

## Code Examples

### Frontend Transition Map (mirror backend)
```typescript
// Source: trade-flow-api/src/schedule/enum/schedule-transitions.ts
export const VALID_TRANSITIONS: Record<ScheduleStatus, ScheduleStatus[]> = {
  scheduled: ["confirmed", "canceled", "no_show"],
  confirmed: ["completed", "canceled"],
  completed: [],
  canceled: [],
  no_show: [],  // NOTE: use no_show to match API, not no-show
};

export const TERMINAL_STATUSES: ScheduleStatus[] = ["completed", "canceled", "no_show"];

export const isTerminalStatus = (status: ScheduleStatus): boolean =>
  TERMINAL_STATUSES.includes(status);
```

### ScheduleFormDialog Edit Mode Props Pattern
```typescript
interface ScheduleFormDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  jobId: string;
  businessId: string;
  schedule?: Schedule;  // undefined = create, defined = edit
}
```

### Edit Form Default Values from Schedule
```typescript
// Parse existing schedule into form values for edit mode
const editDefaults = schedule ? {
  date: new Date(schedule.startDateTime),
  startTime: format(new Date(schedule.startDateTime), "HH:mm"),
  durationMinutes: schedule.durationMinutes,
  visitTypeId: schedule.visitTypeId ?? "",
} : {
  date: new Date(),
  startTime: "09:00",
  durationMinutes: 60,
  visitTypeId: "",
  notes: "",
};
```

### Transition RTK Query Endpoint
```typescript
transitionSchedule: builder.mutation<
  Schedule,
  {
    businessId: string;
    jobId: string;
    scheduleId: string;
    status: ScheduleStatus;
  }
>({
  query: ({ businessId, jobId, scheduleId, status }) => ({
    url: `/v1/business/${businessId}/job/${jobId}/schedule/${scheduleId}/transition`,
    method: "POST",
    body: { status },
  }),
  transformResponse: (response: StandardResponse<Schedule>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No schedule data returned");
  },
  invalidatesTags: (_result, _error, { jobId, scheduleId }) => [
    { type: "Schedule", id: scheduleId },
    { type: "Schedule", id: `JOB-${jobId}` },
  ],
}),
```

### Update Schedule Endpoint (broader data type)
```typescript
// Existing updateSchedule only accepts { notes?: string | null }
// Needs to accept full update fields for edit mode
updateSchedule: builder.mutation<
  Schedule,
  {
    businessId: string;
    jobId: string;
    scheduleId: string;
    data: {
      startDateTime?: string;
      durationMinutes?: number;
      visitTypeId?: string | null;
      notes?: string | null;
    };
  }
>({
  // ... same query shape, just broader data type
})
```

### DropdownMenu with Dynamic Items
```typescript
// Source: shadcn/ui DropdownMenu pattern
import { MoreVertical } from "lucide-react";

<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="ghost" size="icon">
      <MoreVertical className="h-4 w-4" />
    </Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end">
    <DropdownMenuItem onClick={onEdit}>Edit</DropdownMenuItem>
    <DropdownMenuSeparator />
    {validTransitions.map((status) => (
      <DropdownMenuItem
        key={status}
        onClick={() => handleTransition(status)}
      >
        {TRANSITION_LABELS[status]}
      </DropdownMenuItem>
    ))}
  </DropdownMenuContent>
</DropdownMenu>
```

### AlertDialog Confirmation Pattern
```typescript
// Source: shadcn/ui AlertDialog pattern
<AlertDialog open={confirmOpen} onOpenChange={setConfirmOpen}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>{confirmConfig.title}</AlertDialogTitle>
      <AlertDialogDescription>{confirmConfig.description}</AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Go Back</AlertDialogCancel>
      <AlertDialogAction
        onClick={handleConfirmedTransition}
        className={confirmConfig.actionClassName}
      >
        {confirmConfig.actionLabel}
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate create/edit dialog components | Single dialog with mode prop | Project convention from VisitTypeFormDialog | Less code, consistent UX |
| Local state for API data | RTK Query cache as source of truth | Project convention | Automatic refetch, no stale data |

## Open Questions

1. **no_show vs no-show mismatch**
   - What we know: API enum = `no_show`, frontend type = `no-show`, STATUS_CONFIG key = `no-show`
   - What's unclear: Whether this has been tested end-to-end with actual no-show data. The API response mapper sends the raw enum value (`no_show`), so the frontend type is likely wrong.
   - Recommendation: Fix the frontend `ScheduleStatus` type to use `no_show` and update `STATUS_CONFIG` key to match. This should be done in this phase since transitions will exercise this code path for the first time. **This is a bug fix, not a feature -- address it first.**

2. **Detail dialog data freshness after transition**
   - What we know: Currently the detail dialog receives `schedule` as a prop from the parent list
   - What's unclear: Whether RTK Query cache invalidation + parent re-render is fast enough to feel seamless
   - Recommendation: Use `useGetSchedulesByJobQuery` with `selectFromResult` to read the specific schedule from cache, so the dialog auto-updates when the cache is invalidated. Alternatively, use the mutation result directly to optimistically update.

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/schedule/enum/schedule-transitions.ts` - Valid transition map
- `trade-flow-api/src/schedule/enum/schedule-status.enum.ts` - Status enum values (no_show)
- `trade-flow-api/src/schedule/controllers/schedule.controller.ts` - PATCH and transition endpoints
- `trade-flow-api/src/schedule/services/schedule-updater.service.ts` - Auto status reset on confirmed edit
- `trade-flow-api/src/schedule/requests/update-schedule.request.ts` - Accepted update fields
- `trade-flow-api/src/schedule/requests/transition-schedule.request.ts` - Transition request shape
- `trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx` - Existing form structure
- `trade-flow-ui/src/features/schedules/components/ScheduleDetailDialog.tsx` - Existing detail structure
- `trade-flow-ui/src/features/schedules/api/scheduleApi.ts` - Existing RTK Query endpoints
- `trade-flow-ui/src/features/schedules/hooks/useScheduleActions.ts` - Existing action hook
- `trade-flow-ui/src/features/schedules/components/schedule-utils.ts` - STATUS_CONFIG and helpers
- `trade-flow-ui/src/lib/forms/schemas/schedule.schema.ts` - Existing Valibot schema
- `trade-flow-ui/src/types/api.types.ts` - Schedule and ScheduleStatus types

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in use, no new dependencies
- Architecture: HIGH - extending existing components with established patterns
- Pitfalls: HIGH - identified from direct code review of both API and UI codebases
- Status mismatch bug: HIGH - verified by reading both enum definitions

**Research date:** 2026-03-07
**Valid until:** 2026-04-07 (stable codebase, no external dependencies changing)
