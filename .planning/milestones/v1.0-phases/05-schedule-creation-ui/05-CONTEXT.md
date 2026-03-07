# Phase 5: Schedule Creation UI - Context

**Gathered:** 2026-03-07 (updated)
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can create schedule entries on a job through an intuitive dialog form with visit type selection, date/time inputs, and smart defaults. The form is triggered from the job detail page's action strip and from the empty state CTA. Listing, viewing detail, editing, and status transitions are separate phases (6, 7).

</domain>

<decisions>
## Implementation Decisions

### Date and time input
- Separate controls: calendar picker for date, dropdown for start time, dropdown for duration
- Calendar picker: shadcn/ui Calendar component (react-day-picker)
- Time dropdown: 15-minute intervals from 6:00 AM to 9:00 PM (12-hour format: 6:00 AM, 6:15 AM, ...)
- Duration dropdown: fixed presets only — 30min, 1hr, 1.5hr, 2hr, 3hr, 4hr, Full day (10hrs/600min)
- Default date: today
- Default start time: 9:00 AM
- Default duration: 1 hour
- Past dates are selectable (for logging visits that already happened) but shown with a visual hint (muted style or label) to reduce accidental selection

### Form trigger and placement
- Primary trigger: existing JobActionStrip button (onSchedule callback already wired up)
- Form opens as a Dialog overlay on the job detail page (consistent with VisitTypeFormDialog, CustomerFormDialog)
- Empty state: illustrated card with icon, "No visits scheduled yet" message, and "Schedule First Visit" CTA button in the overview section
- Empty state CTA opens the creation dialog directly (same as action strip button)
- Post-create flow: dialog closes, success toast shows "Visit scheduled for [date]", job detail page refreshes via RTK Query cache invalidation

### Visit type selection
- Visit type is **optional** (matches API), with a nudge — dropdown shows placeholder "Select visit type (optional)"
- Starts blank (no pre-selection), encouraging active choice without forcing it
- Color-coded dropdown: Select component showing visit type name with its color dot/swatch next to each option
- Only active visit types shown in dropdown (deactivated ones excluded)
- If no active visit types exist: dropdown visible but disabled, with helper text "No active visit types -- create one in Settings". Form still submittable without a visit type.

### Notes field
- Optional notes textarea always visible in the form (not collapsed/hidden)
- Visit-specific notes (e.g. "Bring extra sealant", "Customer available after 2pm")

### Smart defaults summary
- Date: today
- Start time: 9:00 AM (fixed)
- Duration: 1 hour
- Visit type: blank (no default)
- Notes: empty

### Claude's Discretion
- Loading skeleton design for the form
- Exact spacing, typography, and component sizing within the dialog
- Form validation error message wording
- Duration dropdown option labels (e.g. "1 hour" vs "1h" vs "60 min")
- Calendar picker configuration (which day starts the week, etc.)
- Visual hint style for past dates (muted color, label, or both)

</decisions>

<specifics>
## Specific Ideas

- Schedule creation should feel lightweight -- tradespeople are on-site and need to quickly log or plan a visit
- The empty state CTA should be prominent enough that a new user landing on a job immediately understands they can schedule visits
- Past date visual hint should be subtle -- not a warning, just awareness (e.g. muted date text or small "past date" label)

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `JobActionStrip` (src/features/jobs/): already has `onSchedule` prop wired with placeholder handler
- `VisitTypeFormDialog` (src/features/visit-types/): reference pattern for React Hook Form + Valibot + Dialog form
- `CustomerFormDialog`, `JobFormDialog`, `ItemFormDialog`: additional form dialog references
- shadcn/ui components: Dialog, Select, Calendar, Button, Label, Textarea, FormField, FieldError
- `useCurrentBusiness` hook: provides businessId for API calls
- `toast` utility (`@/lib/toast`): success/error notifications
- RTK Query base API (`@/services/api.ts`): authenticated API calls with cache management

### Established Patterns
- Form dialogs: React Hook Form + Valibot schema + `valibotResolver` + `FormProvider`
- API layer: RTK Query `injectEndpoints` with `builder.query`/`builder.mutation` + cache tag invalidation
- Feature structure: `src/features/{name}/` with components/, hooks/, api/, index.ts barrel
- Form schemas: `src/lib/forms/schemas/` with Valibot validation schemas
- State: Redux Toolkit store with typed hooks (`useAppDispatch`, `useAppSelector`)

### Integration Points
- `JobDetailPage.tsx`: `handleSchedule` placeholder (line 106) needs to open the new ScheduleFormDialog
- `CreateScheduleRequest` (API): startDateTime (ISO8601), durationMinutes (optional, 15-min increments, min 15, max 1440), visitTypeId (optional), assigneeId (optional), notes (optional, max 2000)
- Schedule API endpoints (Phase 3-4): POST create, GET list by job
- Visit Types API (Phase 2): GET active visit types for dropdown population
- `JobOverviewSection`: schedule summary card currently uses MOCK_SCHEDULE data
- `JobDetailTabs`: MOCK_SCHEDULES data to be replaced in Phase 6

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 05-schedule-creation-ui*
*Context gathered: 2026-03-01, updated: 2026-03-07*
