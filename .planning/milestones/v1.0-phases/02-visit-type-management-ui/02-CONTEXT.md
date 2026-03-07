# Phase 2: Visit Type Management UI - Context

**Gathered:** 2026-02-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Frontend for viewing, creating, editing, and deactivating visit types within the My Business settings area. Users can manage both default (auto-generated) and custom visit types. The backend CRUD API already exists from Phase 1.

</domain>

<decisions>
## Implementation Decisions

### List layout & type distinction
- Follow the existing JobTypesSection pattern: table for desktop (768px+), card list for mobile
- Default visit types get a subtle primary-tinted background/border plus a "Default" badge — same as job types
- Each item shows name and description only (no status display in the row)
- List sorted by creation order (not alphabetical, not defaults-first)

### Create experience
- Dialog form (modal) matching the JobTypeFormDialog pattern
- Fields: name (required), description (optional), color (preset palette of 8-12 colors)
- Color is a new field — the backend will need a color property added to the visit type model
- Server-side uniqueness validation only — let the API enforce and display the error

### Navigation & placement
- New "Scheduling" tab in Business Details, alongside existing Job Types and Tax Rates tabs
- Within the Scheduling tab, a "Visit Types" section header with the list below
- Section header approach leaves room for future schedule settings in the same tab

### Edit & deactivation
- Edit via the same dialog form in edit mode — click a visit type to open it
- Name field is always locked (visible but disabled) — names are immutable once created, for both default and custom types
- Description and color are editable
- Deactivate/activate from the list row action menu (dropdown) AND from the dialog action menu (top-right)
- All visit types can be deactivated (defaults included), with the API enforcing at least one must remain active
- Use the items page pattern for status handling: a status filter to switch between active/inactive views (not a simple toggle)

### Empty state
- Illustration + "No visit types yet" message with a prominent "Create Visit Type" button
- The "all inactive" empty state is not possible — API enforces at least one active type

### Loading & error states
- Skeleton loading matching the layout: skeleton rows for desktop, skeleton cards for mobile (reference: JobTypesCardSkeleton)
- Toast notifications for action failures (create, edit, deactivate)
- Full-page error state if the list fails to load

### Claude's Discretion
- Exact skeleton design and count
- Toast message wording
- Specific preset color palette choices
- Color swatch UI component implementation
- Empty state illustration choice

</decisions>

<specifics>
## Specific Ideas

- "Follow the same design pattern as the job types, with a list for desktop screen sizes and card list for mobile screen sizes"
- Reference implementation for status filtering and enable/disable actions: Items page (ItemsDataView)
- Reference implementation for list/card layout and form dialog: Job Types (JobTypesSection, JobTypeFormDialog)
- Reference implementation for action menus: both list row dropdown AND dialog top-right action menu, consistent across job types and items

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-visit-type-management-ui*
*Context gathered: 2026-02-28*
