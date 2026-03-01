# Phase 5: Schedule Creation UI - Context

**Gathered:** 2026-03-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can create schedule entries on a job through an intuitive dialog form with visit type selection, date/time inputs, and smart defaults. The form is triggered from the job detail page's action strip. Listing, viewing detail, editing, and status transitions are separate phases (6, 7).

</domain>

<decisions>
## Implementation Decisions

### Date and time input
- Calendar picker for date selection (shadcn/ui Calendar component built on react-day-picker)
- Time dropdown with 15-minute intervals for start time selection (8:00 AM, 8:15 AM, 8:30 AM, ...)
- Duration dropdown with pre-set options: 30min, 1hr, 1.5hr, 2hr, 3hr, 4hr, Full day
- 12-hour time format throughout (2:30 PM, not 14:30)

### Form trigger and placement
- Primary trigger: existing JobActionStrip button (onSchedule callback already wired up)
- Form opens as a Dialog overlay on the job detail page (consistent with VisitTypeFormDialog, CustomerFormDialog)
- Empty state: illustrated card with icon, "No visits scheduled yet" message, and "Schedule First Visit" CTA button in the overview section
- Post-create flow: dialog closes, success toast shows "Visit scheduled for [date]", job detail page refreshes via RTK Query cache invalidation

### Visit type selection
- Color-coded dropdown: Select component showing visit type name with its color dot/swatch next to each option
- Visit type is required (every scheduled visit must have a type)
- Only active visit types shown in dropdown (deactivated ones excluded)
- If no visit types exist: form submission disabled with hint message "No visit types yet — create one in Settings"

### Smart defaults and notes
- Default date: today's date
- Default start time: 9:00 AM (fixed)
- Default duration: 1 hour
- Optional notes textarea for visit-specific notes (e.g. "Bring extra sealant", "Customer available after 2pm")
- Past dates are selectable in calendar picker (allows logging visits that already happened)

### Claude's Discretion
- Loading skeleton design for the form
- Exact spacing, typography, and component sizing within the dialog
- Form validation error message wording
- Duration dropdown option labels (e.g. "1 hour" vs "1h" vs "60 min")
- Calendar picker configuration (which day starts the week, etc.)

</decisions>

<specifics>
## Specific Ideas

- Schedule creation should feel lightweight — tradespeople are on-site and need to quickly log or plan a visit
- The empty state CTA should be prominent enough that a new user landing on a job immediately understands they can schedule visits

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
- `JobDetailPage.tsx`: `handleSchedule` placeholder needs to open the new ScheduleFormDialog
- Schedule API endpoints (built in Phase 3-4): POST create, GET list by job
- Visit Types API (Phase 2): GET active visit types for dropdown population
- `JobOverviewSection`: schedule summary card currently uses MOCK_SCHEDULE data

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 05-schedule-creation-ui*
*Context gathered: 2026-03-01*
