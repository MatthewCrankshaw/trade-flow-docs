# Phase 5: Schedule Creation UI - Research

**Researched:** 2026-03-07
**Domain:** React form dialog with date/time inputs, RTK Query mutations, shadcn/ui components
**Confidence:** HIGH

## Summary

Phase 5 adds a schedule creation dialog to the job detail page. The project has a well-established pattern for form dialogs (VisitTypeFormDialog, CustomerFormDialog, JobFormDialog) using React Hook Form + Valibot + shadcn Dialog. This phase follows that pattern closely, adding date/time-specific inputs.

The key technical work is: (1) adding the shadcn Calendar component (requires installing `react-day-picker` and `date-fns`), (2) building the schedule form dialog with calendar picker + time/duration Select dropdowns, (3) creating the schedule RTK Query API layer (including adding "Schedule" to tagTypes), (4) wiring the dialog into JobDetailPage and adding the empty state to JobOverviewSection, and (5) defining frontend types for Schedule entities.

**Primary recommendation:** Follow the VisitTypeFormDialog pattern exactly -- separate FormContent component inside Dialog wrapper, useController for custom inputs, valibotResolver for validation, RTK Query mutation with cache invalidation.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Separate controls: calendar picker for date, dropdown for start time, dropdown for duration
- Calendar picker: shadcn/ui Calendar component (react-day-picker)
- Time dropdown: 15-minute intervals from 6:00 AM to 9:00 PM (12-hour format: 6:00 AM, 6:15 AM, ...)
- Duration dropdown: fixed presets only -- 30min, 1hr, 1.5hr, 2hr, 3hr, 4hr, Full day (10hrs/600min)
- Default date: today, Default start time: 9:00 AM, Default duration: 1 hour
- Past dates are selectable but shown with a visual hint (muted style or label)
- Primary trigger: existing JobActionStrip button (onSchedule callback already wired up)
- Form opens as a Dialog overlay (consistent with VisitTypeFormDialog, CustomerFormDialog)
- Empty state: illustrated card with icon, "No visits scheduled yet" message, and "Schedule First Visit" CTA button in the overview section
- Empty state CTA opens the creation dialog directly (same as action strip button)
- Post-create flow: dialog closes, success toast shows "Visit scheduled for [date]", job detail page refreshes via RTK Query cache invalidation
- Visit type is optional, starts blank, placeholder "Select visit type (optional)"
- Color-coded dropdown: Select component showing visit type name with its color dot/swatch
- Only active visit types shown in dropdown (deactivated ones excluded)
- If no active visit types exist: dropdown visible but disabled, with helper text
- Optional notes textarea always visible in the form
- Date: today, Start time: 9:00 AM (fixed), Duration: 1 hour, Visit type: blank, Notes: empty

### Claude's Discretion
- Loading skeleton design for the form
- Exact spacing, typography, and component sizing within the dialog
- Form validation error message wording
- Duration dropdown option labels (e.g. "1 hour" vs "1h" vs "60 min")
- Calendar picker configuration (which day starts the week, etc.)
- Visual hint style for past dates (muted color, label, or both)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SCHED-01 | User can create a schedule entry on a job with date, start time, and duration | Schedule form dialog with calendar picker, time Select, duration Select, RTK Query create mutation, Valibot schema |
| VTYPE-01 | User can select a visit type when creating a schedule entry | Visit type Select dropdown populated via useGetVisitTypesQuery, filtered to active only, color-coded options |
| INTG-03 | Empty state displayed when a job has no schedule entries | Empty state card in JobOverviewSection replacing mock data, CTA opens schedule creation dialog |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| react-hook-form | ^7.71.1 | Form state management | Already used in all form dialogs (VisitTypeFormDialog, CustomerFormDialog, JobFormDialog) |
| valibot | ^1.2.0 | Schema validation | Already used for all form schemas via valibotResolver |
| @hookform/resolvers | ^5.2.2 | Valibot integration | Already installed, provides valibotResolver |
| react-day-picker | NEW | Calendar picker | shadcn/ui Calendar component depends on it |
| date-fns | NEW | Date formatting/manipulation | Required peer dependency for react-day-picker; also useful for date formatting in form |
| @radix-ui/react-select | ^2.2.6 | Time/duration dropdowns | Already installed, used by shadcn Select component |
| @reduxjs/toolkit | ^2.11.2 | RTK Query for API layer | Already used for all API endpoints |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| lucide-react | ^0.563.0 | Icons (Calendar, Clock, Plus) | Already installed |
| sonner | ^2.0.7 | Toast notifications | Already installed via toast utility |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| react-day-picker | Native date input | Native input has poor UX, inconsistent cross-browser; shadcn Calendar is the project standard |
| date-fns | Luxon | Backend uses Luxon but frontend has no date library yet; date-fns is react-day-picker's peer dep so it's free |

### Installation
```bash
npx shadcn@latest add calendar
# This installs react-day-picker and date-fns automatically
# Also adds src/components/ui/calendar.tsx
```

## Architecture Patterns

### Recommended Project Structure
```
src/
├── features/
│   └── schedules/                    # NEW feature module
│       ├── api/
│       │   ├── scheduleApi.ts        # RTK Query endpoints (create, getByJob)
│       │   └── index.ts
│       ├── components/
│       │   ├── ScheduleFormDialog.tsx # Main form dialog (follows VisitTypeFormDialog pattern)
│       │   ├── ScheduleEmptyState.tsx # Empty state card with CTA
│       │   └── index.ts
│       ├── hooks/
│       │   ├── useScheduleActions.ts  # Create mutation wrapper (follows useVisitTypeActions)
│       │   └── index.ts
│       └── index.ts                   # Barrel export
├── lib/forms/schemas/
│   └── schedule.schema.ts            # NEW Valibot schema for schedule form
├── types/
│   └── api.types.ts                  # ADD Schedule types (ScheduleStatus, Schedule, CreateScheduleRequest)
└── services/
    └── api.ts                        # ADD "Schedule" to tagTypes array
```

### Pattern 1: Form Dialog (VisitTypeFormDialog Pattern)
**What:** Outer Dialog component manages open/close state; inner FormContent component handles form logic
**When to use:** All form dialogs in this project
**Example:**
```typescript
// Outer component: Dialog shell
export function ScheduleFormDialog({ open, onOpenChange, jobId, businessId }: Props) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-md" showCloseButton={false}>
        <DialogHeader>
          <DialogTitle>Schedule Visit</DialogTitle>
          <DialogDescription>...</DialogDescription>
        </DialogHeader>
        <ScheduleFormContent jobId={jobId} businessId={businessId} onClose={() => onOpenChange(false)} />
      </DialogContent>
    </Dialog>
  );
}

// Inner component: Form logic
function ScheduleFormContent({ jobId, businessId, onClose }: ContentProps) {
  const form = useForm<ScheduleFormValues>({
    resolver: valibotResolver(scheduleFormSchema),
    defaultValues: { date: new Date(), startTime: "09:00", durationMinutes: 60, visitTypeId: "", notes: "" },
  });
  // ... submit handler, fields
}
```

### Pattern 2: RTK Query API with Cache Tags
**What:** Schedule API endpoints using injectEndpoints with tag-based invalidation
**When to use:** All API integrations
**Example:**
```typescript
// scheduleApi.ts -- follows visitTypeApi.ts pattern exactly
export const scheduleApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getSchedulesByJob: builder.query<Schedule[], { businessId: string; jobId: string }>({
      query: ({ businessId, jobId }) => `/v1/business/${businessId}/job/${jobId}/schedule`,
      transformResponse: (response: StandardResponse<Schedule>) => response.data,
      providesTags: (result, _error, { jobId }) =>
        result
          ? [...result.map(({ id }) => ({ type: "Schedule" as const, id })),
             { type: "Schedule", id: `JOB-${jobId}` }]
          : [{ type: "Schedule", id: `JOB-${jobId}` }],
    }),
    createSchedule: builder.mutation<Schedule, { businessId: string; jobId: string; data: CreateScheduleRequest }>({
      query: ({ businessId, jobId, data }) => ({
        url: `/v1/business/${businessId}/job/${jobId}/schedule`,
        method: "POST",
        body: data,
      }),
      transformResponse: (response: StandardResponse<Schedule>) => response.data[0],
      invalidatesTags: (_result, _error, { jobId }) => [{ type: "Schedule", id: `JOB-${jobId}` }],
    }),
  }),
});
```

### Pattern 3: Date/Time Composition for API
**What:** Form stores separate date + startTime + durationMinutes; on submit, compose into ISO8601 startDateTime
**When to use:** Schedule creation form submit handler
**Example:**
```typescript
// Form values use separate fields for UX
interface ScheduleFormValues {
  date: Date;           // From Calendar picker
  startTime: string;    // "09:00" from time Select
  durationMinutes: number; // 60 from duration Select
  visitTypeId: string;  // "" or valid ID
  notes: string;
}

// On submit, compose into API request
const handleSubmit = form.handleSubmit(async (values) => {
  // Combine date + time into ISO8601
  const [hours, minutes] = values.startTime.split(":").map(Number);
  const startDate = new Date(values.date);
  startDate.setHours(hours, minutes, 0, 0);
  const startDateTime = startDate.toISOString();

  const request: CreateScheduleRequest = {
    startDateTime,
    durationMinutes: values.durationMinutes,
    ...(values.visitTypeId && { visitTypeId: values.visitTypeId }),
    ...(values.notes.trim() && { notes: values.notes.trim() }),
  };
  await createSchedule({ businessId, jobId, data: request });
});
```

### Pattern 4: Visit Type Color-Coded Select
**What:** Custom Select items with color dots
**When to use:** Visit type dropdown in schedule form
**Example:**
```typescript
<Select value={field.value} onValueChange={field.onChange} disabled={activeVisitTypes.length === 0}>
  <SelectTrigger>
    <SelectValue placeholder="Select visit type (optional)" />
  </SelectTrigger>
  <SelectContent>
    {activeVisitTypes.map((vt) => (
      <SelectItem key={vt.id} value={vt.id}>
        <span className="flex items-center gap-2">
          <span className="h-3 w-3 rounded-full shrink-0" style={{ backgroundColor: vt.color }} />
          {vt.name}
        </span>
      </SelectItem>
    ))}
  </SelectContent>
</Select>
{activeVisitTypes.length === 0 && (
  <p className="text-xs text-muted-foreground">No active visit types -- create one in Settings</p>
)}
```

### Anti-Patterns to Avoid
- **Direct state management for form:** Don't use useState for form fields; always use React Hook Form with FormProvider
- **Inline API calls:** Don't call fetch/axios directly; use RTK Query mutations through a custom hook (useScheduleActions)
- **Embedded form in page:** Don't put form logic in JobDetailPage; extract to dedicated ScheduleFormDialog component
- **Hardcoded time zones:** Don't assume local timezone; compose dates carefully and let toISOString() handle UTC conversion

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Calendar date picker | Custom calendar grid | shadcn Calendar (react-day-picker) | Edge cases with months, years, keyboard nav, accessibility |
| Form state management | useState per field | React Hook Form + valibotResolver | Validation, error state, dirty tracking all handled |
| Select dropdown | Custom dropdown | shadcn Select (@radix-ui/react-select) | Keyboard nav, accessibility, portal rendering |
| API caching/invalidation | Manual refetching | RTK Query with tag-based invalidation | Automatic cache management, deduplication |
| Date formatting | Manual string manipulation | date-fns format() | Locale handling, edge cases |

**Key insight:** Every UI pattern in this form already exists in the codebase. The only truly new element is the Calendar component, which is a standard shadcn add.

## Common Pitfalls

### Pitfall 1: Calendar Component Not Installed
**What goes wrong:** shadcn/ui Calendar is not in the project yet -- no react-day-picker or date-fns in package.json
**Why it happens:** Previous phases were backend-only or used simple inputs
**How to avoid:** Run `npx shadcn@latest add calendar` before building the form; this installs react-day-picker, date-fns, and creates the component file
**Warning signs:** Import errors for Calendar component

### Pitfall 2: "Schedule" Missing from RTK Query tagTypes
**What goes wrong:** Cache invalidation silently fails; after creating a schedule, the list doesn't update
**Why it happens:** apiSlice.tagTypes in services/api.ts currently only lists ["User", "Business", "Customer", "Item", "Quote", "Migration", "TaxRate", "JobType", "Job", "VisitType"] -- no "Schedule"
**How to avoid:** Add "Schedule" to the tagTypes array in services/api.ts before creating schedule endpoints
**Warning signs:** Creating a schedule succeeds but the empty state doesn't disappear

### Pitfall 3: ISO8601 DateTime Composition
**What goes wrong:** Date is created in local timezone but API expects UTC; times shift by offset
**Why it happens:** `new Date()` uses local time; `toISOString()` converts to UTC
**How to avoid:** The simplest approach is to construct the ISO string manually: `${format(date, 'yyyy-MM-dd')}T${startTime}:00.000Z` -- this treats the user's selected date/time as the intended UTC value (acceptable for v1 with solo operators)
**Warning signs:** Schedule shows wrong time in API response

### Pitfall 4: Visit Type Select with Empty Value
**What goes wrong:** Form sends empty string as visitTypeId; API rejects it (expects 24-char ObjectId or omitted)
**Why it happens:** Select component uses "" as initial/cleared value
**How to avoid:** On submit, only include visitTypeId in request when it's a non-empty string
**Warning signs:** API 400 error when submitting without visit type

### Pitfall 5: Popover Inside Dialog Z-Index
**What goes wrong:** Calendar popover renders behind the Dialog overlay
**Why it happens:** Both Dialog and Popover use portals; z-index conflicts
**How to avoid:** Use the Calendar inline within the dialog (not in a Popover) OR ensure the Popover has a higher z-index. Inline calendar within dialog content is the simpler approach for mobile-first design.
**Warning signs:** Calendar appears behind dialog backdrop

## Code Examples

### Schedule Form Valibot Schema
```typescript
// src/lib/forms/schemas/schedule.schema.ts
import * as v from "valibot";

export const scheduleFormSchema = v.object({
  date: v.date("Please select a date"),
  startTime: v.pipe(
    v.string(),
    v.minLength(1, "Please select a start time"),
  ),
  durationMinutes: v.pipe(
    v.number(),
    v.minValue(15, "Duration must be at least 15 minutes"),
  ),
  visitTypeId: v.optional(v.string()),
  notes: v.optional(
    v.pipe(
      v.string(),
      v.maxLength(2000, "Notes must be less than 2000 characters"),
    ),
  ),
});

export type ScheduleFormValues = v.InferOutput<typeof scheduleFormSchema>;
```

### Time Options Generator
```typescript
// Generate 15-min interval time options from 6:00 AM to 9:00 PM
function generateTimeOptions(): { value: string; label: string }[] {
  const options: { value: string; label: string }[] = [];
  for (let hour = 6; hour <= 21; hour++) {
    for (let min = 0; min < 60; min += 15) {
      if (hour === 21 && min > 0) break; // Stop at 9:00 PM
      const h24 = `${hour.toString().padStart(2, "0")}:${min.toString().padStart(2, "0")}`;
      const h12 = hour > 12 ? hour - 12 : hour === 0 ? 12 : hour;
      const ampm = hour >= 12 ? "PM" : "AM";
      const label = `${h12}:${min.toString().padStart(2, "0")} ${ampm}`;
      options.push({ value: h24, label });
    }
  }
  return options;
}
// Result: [{ value: "06:00", label: "6:00 AM" }, { value: "06:15", label: "6:15 AM" }, ...]
```

### Duration Options
```typescript
const DURATION_OPTIONS = [
  { value: 30, label: "30 minutes" },
  { value: 60, label: "1 hour" },
  { value: 90, label: "1.5 hours" },
  { value: 120, label: "2 hours" },
  { value: 180, label: "3 hours" },
  { value: 240, label: "4 hours" },
  { value: 600, label: "Full day" },
] as const;
```

### Frontend Schedule Types
```typescript
// Add to src/types/api.types.ts
export type ScheduleStatus = "scheduled" | "confirmed" | "completed" | "canceled" | "no-show";

export interface Schedule {
  id: string;
  jobId: string;
  visitTypeId: string | null;
  assigneeId: string;
  startDateTime: string; // ISO8601
  durationMinutes: number;
  notes: string | null;
  status: ScheduleStatus;
}

export interface CreateScheduleRequest {
  startDateTime: string; // ISO8601
  durationMinutes?: number;
  visitTypeId?: string;
  assigneeId?: string;
  notes?: string;
}
```

### Empty State Component Pattern
```typescript
// Follows the existing pattern from AccessNotesCard empty state
function ScheduleEmptyState({ onSchedule }: { onSchedule: () => void }) {
  return (
    <Card>
      <CardHeader className="pb-3">
        <CardTitle className="flex items-center gap-2 text-sm font-medium">
          <Calendar className="h-4 w-4 text-muted-foreground" />
          Schedule Summary
        </CardTitle>
      </CardHeader>
      <CardContent className="flex flex-col items-center py-6 text-center">
        <CalendarPlus className="mb-3 h-10 w-10 text-muted-foreground" />
        <p className="mb-1 text-sm font-medium">No visits scheduled yet</p>
        <p className="mb-4 text-xs text-muted-foreground">
          Schedule your first visit to get started
        </p>
        <Button size="sm" className="gap-1.5" onClick={onSchedule}>
          <Plus className="h-4 w-4" />
          Schedule First Visit
        </Button>
      </CardContent>
    </Card>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| react-day-picker v8 | react-day-picker v9 | 2024 | New API, breaking changes; shadcn handles this |
| Separate date + startTime API fields | Single startDateTime ISO8601 field | Phase quick-2 | Frontend must compose date + time into single field |
| MOCK_SCHEDULE in JobOverviewSection | Real schedule data via RTK Query | This phase | Replace mock with actual API data or empty state |

**Deprecated/outdated:**
- MOCK_SCHEDULE constant in JobDetailPage: will be replaced by real RTK Query data or empty state

## Open Questions

1. **Calendar inline vs popover within dialog**
   - What we know: Calendar in a Popover inside a Dialog can have z-index issues. Inline calendar takes more space but avoids the problem.
   - What's unclear: Which approach works better on mobile (mobile-first design requirement)
   - Recommendation: Try inline Calendar first within the dialog; if too tall, switch to Popover with explicit z-50+ class

2. **Schedule list query scope for empty state detection**
   - What we know: Need to know if job has 0 schedules to show empty state. The GET by job endpoint exists.
   - What's unclear: Whether to query all schedules or just non-canceled ones for empty state check
   - Recommendation: Query all schedules for the job (no status filter); show empty state only when total is 0

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vite + TypeScript (typecheck + lint only, no test runner) |
| Config file | tsconfig.json, eslint.config.js |
| Quick run command | `npm run typecheck` |
| Full suite command | `npm run typecheck && npm run lint` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SCHED-01 | Create schedule with date/time/duration | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |
| VTYPE-01 | Visit type selection in schedule form | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |
| INTG-03 | Empty state when no schedules | typecheck + lint | `cd trade-flow-ui && npm run typecheck && npm run lint` | N/A |

### Sampling Rate
- **Per task commit:** `cd trade-flow-ui && npm run typecheck && npm run lint`
- **Per wave merge:** `cd trade-flow-ui && npm run build` (full production build)
- **Phase gate:** Full build green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `npx shadcn@latest add calendar` -- install react-day-picker + date-fns + Calendar component
- [ ] Add "Schedule" to tagTypes in `src/services/api.ts`
- [ ] Add Schedule types to `src/types/api.types.ts` and re-export from `src/types/index.ts`
- [ ] Add schedule schema to `src/lib/forms/schemas/` and re-export from index

## Sources

### Primary (HIGH confidence)
- Project codebase: VisitTypeFormDialog.tsx -- established form dialog pattern
- Project codebase: visitTypeApi.ts -- established RTK Query API pattern
- Project codebase: useVisitTypeActions.ts -- established mutation hook pattern
- Project codebase: schedule.controller.ts, create-schedule.request.ts -- backend API contract
- Project codebase: schedule.response.ts -- API response shape
- Project codebase: package.json -- current dependency versions
- Project codebase: services/api.ts -- tagTypes configuration

### Secondary (MEDIUM confidence)
- shadcn/ui Calendar component -- standard installation via CLI

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in use except react-day-picker (standard shadcn add)
- Architecture: HIGH - follows identical patterns from Phase 2 (VisitTypeFormDialog)
- Pitfalls: HIGH - identified through codebase analysis (missing tagType, ISO8601 composition, z-index)

**Research date:** 2026-03-07
**Valid until:** 2026-04-06 (30 days -- stable stack, no fast-moving deps)
