# Phase 6: Schedule List and Detail UI - Research

**Researched:** 2026-03-07
**Domain:** React UI - schedule list display, detail dialog, notes editing
**Confidence:** HIGH

## Summary

Phase 6 replaces the `MOCK_SCHEDULES` data in `JobDetailTabs.tsx` with real schedule data from the RTK Query `scheduleApi` built in Phase 5. The work involves building a responsive schedule list (table on desktop, cards on mobile) with chronological grouping ("Upcoming" / "Past"), a detail dialog for viewing schedule fields with editable notes, and adding an `updateSchedule` RTK Query mutation for the existing PATCH endpoint.

The codebase already has all needed patterns established. The responsive table/card pattern exists in `VisitTypesSection` (Phase 2). The `scheduleApi` has `useGetSchedulesByJobQuery` returning `Schedule[]` with all needed fields. The API PATCH endpoint at `business/:businessId/job/:jobId/schedule/:scheduleId` already accepts `{ notes?: string | null }` partial updates. The `Schedule` type includes `startDateTime` (ISO8601), `durationMinutes`, `visitTypeId`, `status`, and `notes`.

**Primary recommendation:** Follow the VisitTypesSection responsive pattern exactly -- extract ScheduleList as a standalone component, add `updateSchedule` mutation to `scheduleApi`, build a ScheduleDetailDialog with read-only fields and editable notes textarea, and wire it into JobDetailTabs replacing MOCK_SCHEDULES.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Follow existing patterns: table for desktop (768px+), card list for mobile -- matching visit types and job types sections (Phase 2 pattern)
- Each entry shows: date, start time, duration, visit type (with color dot), status badge
- No assignee column -- solo operator, always the current user
- Small notes icon/indicator on entries that have notes
- Canceled and no-show entries shown with muted styling (lighter text, faded)
- Color-coded badges: Scheduled=blue, Confirmed=green, Completed=gray, Canceled=red, No-show=amber
- Clicking a schedule entry opens a detail dialog (not inline expand)
- Dialog shows all schedule fields as read-only text (date, time, duration, visit type, status)
- Notes textarea is the only editable field in this phase
- Phase 7 will add full edit mode to this same dialog
- Save button updates notes via the existing PATCH schedule endpoint
- Notes displayed inside the detail dialog (not inline in the list)
- Two sections: "Upcoming" (future dates, ascending) then "Past" (past dates, descending)
- Labeled section headers ("Upcoming" / "Past") as subtle labels above each group
- Tab header count shows total entries (all statuses)
- Use existing EmptyTabState component pattern. "Schedule Visit" action opens the creation dialog from Phase 5
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

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SCHED-02 | User can view all schedule entries for a job in chronological order | RTK Query `useGetSchedulesByJobQuery` already returns `Schedule[]`; client-side sorting by `startDateTime` into Upcoming/Past groups; responsive table/card display |
| SCHED-05 | User can add free-text notes to a schedule entry | API PATCH endpoint accepts `{ notes?: string \| null }` with max 2000 chars; need `updateSchedule` RTK Query mutation; detail dialog with editable textarea |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19 | Component framework | Project standard |
| TypeScript | 5.9+ | Type safety | Project standard |
| RTK Query | (via Redux Toolkit) | API data fetching and caching | Already used for scheduleApi |
| shadcn/ui | New York style | UI components (Dialog, Table, Badge, Card, Button, Textarea) | Project standard |
| Tailwind CSS | 4.x | Styling | Project standard |
| date-fns | (installed) | Date formatting | Already used in ScheduleFormDialog |
| lucide-react | (installed) | Icons | Project standard |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `useMediaQuery` hook | Custom | Desktop/mobile responsive switching | Already exists at `@/hooks/useMediaQuery` |
| `cn()` utility | Custom | Conditional Tailwind classes | Already exists at `@/lib/utils` |

### Alternatives Considered
None -- all libraries are already established in the project.

**Installation:**
No new packages needed. All dependencies already installed.

## Architecture Patterns

### Recommended File Structure
```
src/features/schedules/
├── api/
│   └── scheduleApi.ts          # ADD updateSchedule mutation
├── components/
│   ├── index.ts                # UPDATE exports
│   ├── ScheduleFormDialog.tsx   # Existing (Phase 5)
│   ├── ScheduleList.tsx         # NEW - orchestrator (data fetching, grouping, responsive switch)
│   ├── ScheduleTable.tsx        # NEW - desktop table view
│   ├── ScheduleCardList.tsx     # NEW - mobile card view
│   └── ScheduleDetailDialog.tsx # NEW - detail view with notes editing
├── hooks/
│   ├── useScheduleActions.ts    # UPDATE - add updateNotes action
│   └── index.ts
└── index.ts
```

### Pattern 1: Responsive Table/Card (from VisitTypesSection)
**What:** Use `useMediaQuery("(min-width: 768px)")` to switch between Table and CardList views
**When to use:** All list displays in this project
**Example:**
```typescript
// Source: VisitTypesSection.tsx (existing pattern)
const isDesktop = useMediaQuery("(min-width: 768px)");

return isDesktop ? (
  <ScheduleTable schedules={schedules} onSelect={handleSelect} />
) : (
  <ScheduleCardList schedules={schedules} onSelect={handleSelect} />
);
```

### Pattern 2: Chronological Grouping
**What:** Split schedules into "Upcoming" and "Past" based on `startDateTime` vs current time
**When to use:** Schedule list display
**Example:**
```typescript
// Client-side grouping from the RTK Query data
const now = new Date();
const upcoming = schedules
  .filter(s => new Date(s.startDateTime) >= now)
  .sort((a, b) => new Date(a.startDateTime).getTime() - new Date(b.startDateTime).getTime());
const past = schedules
  .filter(s => new Date(s.startDateTime) < now)
  .sort((a, b) => new Date(b.startDateTime).getTime() - new Date(a.startDateTime).getTime());
```

### Pattern 3: RTK Query Mutation for PATCH
**What:** Add `updateSchedule` mutation endpoint to existing `scheduleApi`
**When to use:** Notes saving in detail dialog
**Example:**
```typescript
// Add to scheduleApi.ts endpoints
updateSchedule: builder.mutation<
  Schedule,
  { businessId: string; jobId: string; scheduleId: string; data: { notes?: string | null } }
>({
  query: ({ businessId, jobId, scheduleId, data }) => ({
    url: `/v1/business/${businessId}/job/${jobId}/schedule/${scheduleId}`,
    method: "PATCH",
    body: data,
  }),
  transformResponse: (response: StandardResponse<Schedule>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No schedule data returned");
  },
  invalidatesTags: (result, _error, { jobId, scheduleId }) => [
    { type: "Schedule", id: scheduleId },
    { type: "Schedule", id: `JOB-${jobId}` },
  ],
}),
```

### Pattern 4: Detail Dialog (designed for Phase 7 extension)
**What:** Dialog with read-only field display and editable notes, structured for future edit mode
**When to use:** Viewing/editing individual schedule entries
**Key design:** Use a `mode` concept (even if only "view" exists now) so Phase 7 can add "edit" mode without rebuilding
```typescript
// Structure for extensibility
interface ScheduleDetailDialogProps {
  schedule: Schedule | null;
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  jobId: string;
}
// Notes are always editable (not gated by mode) since they're the one thing users can change in Phase 6
```

### Pattern 5: Visit Type Resolution
**What:** Look up visitTypeId to get name + color from visit types query
**When to use:** Displaying visit type with color dot in schedule list
**Example:**
```typescript
// Use existing useGetVisitTypesQuery to build a lookup map
const { data: visitTypes } = useGetVisitTypesQuery(businessId);
const visitTypeMap = useMemo(
  () => new Map(visitTypes?.map(vt => [vt.id, vt]) ?? []),
  [visitTypes]
);
// Then: visitTypeMap.get(schedule.visitTypeId)?.name, .color
```

### Anti-Patterns to Avoid
- **Server-side sorting for this use case:** The schedule list per job is small (typically < 50 entries). Client-side sorting is simpler and avoids extra API parameters. The API already returns data sorted by startDateTime ascending.
- **Separate API calls per schedule for notes:** Use the existing list query data. Only call PATCH when saving notes.
- **Building edit mode into the dialog now:** Phase 7 adds edit mode. Structure the dialog to accept it later but do not build it now.
- **Removing MOCK data from other tabs:** Only replace MOCK_SCHEDULES. Other mocks (quotes, invoices, notes) are out of scope.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Date formatting | Custom date string manipulation | `date-fns` `format()` | Already used in ScheduleFormDialog; handles locale, edge cases |
| Responsive switching | CSS-only show/hide with duplicate markup | `useMediaQuery` hook | Existing project pattern, avoids rendering both views |
| Status badge styling | Manual class mapping | Utility function returning Badge variant + className | Keeps color logic centralized for reuse |
| Cache invalidation | Manual refetch after mutation | RTK Query tag invalidation | Already set up with `JOB-${jobId}` tags |

## Common Pitfalls

### Pitfall 1: JobDetailTabs Prop Drilling
**What goes wrong:** Trying to pass businessId/jobId through JobDetailTabs which currently takes only `jobStatus`
**Why it happens:** JobDetailTabs was built as a presentation component with mock data
**How to avoid:** Either pass `businessId` and `jobId` as props to JobDetailTabs, or have the ScheduleList component consume them from route params/context directly. The JobDetailPage already has both values available. Prefer passing as props to keep the component testable.
**Warning signs:** Reaching for useParams inside a deeply nested component

### Pitfall 2: Visit Type Lookup for Null visitTypeId
**What goes wrong:** Schedule entries with `visitTypeId: null` cause lookup errors or show "undefined"
**Why it happens:** visitTypeId is optional when creating schedules
**How to avoid:** Guard with `schedule.visitTypeId ? visitTypeMap.get(schedule.visitTypeId) : null` and show a fallback (e.g., no visit type dot, or "General Visit")
**Warning signs:** Blank or "undefined" in visit type column

### Pitfall 3: Timezone Issues with Upcoming/Past Split
**What goes wrong:** Entries near midnight show in wrong group
**Why it happens:** `startDateTime` is ISO8601 UTC; comparing to `new Date()` uses local time
**How to avoid:** Be consistent -- compare UTC to UTC, or parse both to the same reference. The split does not need to be exact to the second; date-based comparison is sufficient.
**Warning signs:** Entries jumping between Upcoming/Past near midnight

### Pitfall 4: Notes Textarea Dirty State
**What goes wrong:** User edits notes, navigates away, loses changes without warning
**Why it happens:** Dialog closes on backdrop click without checking for unsaved changes
**How to avoid:** Track dirty state; optionally disable backdrop close when dirty, or just save on close. Keep it simple for Phase 6 -- a Save button is sufficient.
**Warning signs:** User complaints about lost notes

### Pitfall 5: EmptyTabState Callback for Schedule Creation
**What goes wrong:** EmptyTabState's "Schedule Visit" button does not open the ScheduleFormDialog
**Why it happens:** EmptyTabState currently has no onClick handler wired up (just renders a Button)
**How to avoid:** Add `onAction` callback prop to EmptyTabState or wire the button via the parent. The existing EmptyTabState will need modification to accept an action callback.
**Warning signs:** Button renders but does nothing when clicked

## Code Examples

### Status Badge Color Map
```typescript
// Centralized status display config
const STATUS_CONFIG: Record<ScheduleStatus, { label: string; className: string }> = {
  scheduled: { label: "Scheduled", className: "bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400" },
  confirmed: { label: "Confirmed", className: "bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400" },
  completed: { label: "Completed", className: "bg-gray-100 text-gray-800 dark:bg-gray-900/30 dark:text-gray-400" },
  canceled: { label: "Canceled", className: "bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400" },
  "no-show": { label: "No Show", className: "bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-400" },
};
```

### Date/Time Formatting (Recommended)
```typescript
import { format, isToday, isTomorrow, isPast } from "date-fns";

// Date: "Tue 10 Mar" (short, scannable -- matches MOCK_SCHEDULES style)
function formatScheduleDate(isoString: string): string {
  const date = new Date(isoString);
  if (isToday(date)) return "Today";
  if (isTomorrow(date)) return "Tomorrow";
  return format(date, "EEE d MMM");
}

// Time: "9:00 AM"
function formatScheduleTime(isoString: string): string {
  return format(new Date(isoString), "h:mm a");
}

// Duration: "2 hrs" for whole hours, "1.5 hrs" for half, "45 min" for < 1 hour
function formatDuration(minutes: number): string {
  if (minutes < 60) return `${minutes} min`;
  const hours = minutes / 60;
  if (Number.isInteger(hours)) return `${hours} hr${hours > 1 ? "s" : ""}`;
  return `${hours} hrs`;
}
```

### Muted Row Styling for Canceled/No-Show
```typescript
// Apply to table rows and cards
const isMuted = schedule.status === "canceled" || schedule.status === "no-show";
// Table row:
<TableRow className={cn(isMuted && "opacity-60")}>
// Card:
<Card className={cn(isMuted && "opacity-60")}>
```

### Update Schedule Mutation Usage
```typescript
// In detail dialog
const [updateSchedule, { isLoading: isSaving }] = useUpdateScheduleMutation();

const handleSaveNotes = async () => {
  try {
    await updateSchedule({
      businessId,
      jobId,
      scheduleId: schedule.id,
      data: { notes: notesValue.trim() || null },
    }).unwrap();
    toast.success("Notes saved");
  } catch {
    toast.error("Failed to save notes");
  }
};
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate date + startTime fields | Single `startDateTime` ISO8601 | Phase quick-2 | All parsing uses `new Date(startDateTime)` |
| Mock data in JobDetailTabs | Real RTK Query data | This phase | Replaces MOCK_SCHEDULES |

## Open Questions

1. **TabHeader "+Add" button removal scope**
   - What we know: CONTEXT.md says remove the Add button from schedule tab header. TabHeader is shared across all tabs.
   - What's unclear: Should TabHeader be modified to conditionally show/hide the button, or should the schedule tab use a different header component?
   - Recommendation: Make the Add button optional in TabHeader via an `onAdd` prop -- when provided, show button; when omitted, hide it. This keeps other tabs working while removing it from schedule.

2. **EmptyTabState action callback**
   - What we know: EmptyTabState renders a Button but has no onClick handler currently
   - What's unclear: Whether to modify the shared EmptyTabState or create a schedule-specific empty state
   - Recommendation: Add optional `onAction` prop to EmptyTabState. When provided, wire it to the button's onClick. Minimal change, backward compatible.

## Sources

### Primary (HIGH confidence)
- `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` - Current mock data structure, TabHeader, EmptyTabState, ScheduleTable
- `trade-flow-ui/src/features/schedules/api/scheduleApi.ts` - Existing RTK Query endpoints
- `trade-flow-ui/src/features/visit-types/components/VisitTypesSection.tsx` - Responsive table/card pattern
- `trade-flow-ui/src/features/visit-types/components/VisitTypesCardList.tsx` - Mobile card pattern
- `trade-flow-ui/src/types/api.types.ts` - Schedule, ScheduleStatus, VisitType types
- `trade-flow-api/src/schedule/controllers/schedule.controller.ts` - PATCH endpoint confirmation
- `trade-flow-api/src/schedule/requests/update-schedule.request.ts` - Notes field: `@MaxLength(2000)`, nullable
- `trade-flow-api/src/schedule/responses/schedule.response.ts` - API response shape
- `trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx` - Dialog pattern, visit type resolution
- `trade-flow-ui/src/pages/JobDetailPage.tsx` - Integration point, businessId/jobId availability

### Secondary (MEDIUM confidence)
- None needed -- all findings verified from source code

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in use in the project
- Architecture: HIGH - follows established responsive list pattern from Phase 2
- Pitfalls: HIGH - identified from direct code inspection of integration points

**Research date:** 2026-03-07
**Valid until:** 2026-04-07 (stable -- no external dependency changes expected)
