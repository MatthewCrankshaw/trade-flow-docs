# Phase 39: Welcome Dashboard and Final Cleanup - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Personalised welcome experience on the dashboard for new users who have just completed onboarding. A welcome message using the user's display name and a getting-started checklist widget with "Create your first job" and "Send your first quote" items. Checklist tracks completion via the onboarding progress record, and the entire welcome section (message + widget) disappears once all items are complete. Also removes the old pre-Phase 37 onboarding system (old wizard, old routes, old state).

</domain>

<decisions>
## Implementation Decisions

### Welcome Message
- **D-01:** Name-based greeting using display name from onboarding Step 1 -- e.g., "Welcome, Matt! Let's get your business up and running."
- **D-02:** Casual and direct tone -- matches existing tradesperson audience tone established in Phase 36 D-01 and Phase 37 D-08
- **D-03:** Welcome message disappears when all checklist items are complete -- tied to widget lifecycle, not tracked separately

### Getting-Started Checklist
- **D-04:** Card component with checkbox-style items -- checked items show completion. Uses existing Card component pattern.
- **D-05:** Two checklist items: "Create your first job" and "Send your first quote" -- each links to the relevant page
- **D-06:** Each item has a title + short description (e.g., "Create your first job" with subtitle "Add a job to start tracking your work")
- **D-07:** Completion detection uses the onboarding progress record -- NOT direct job/quote count queries or new backend fields
- **D-08:** Widget disappears entirely once all checklist items are complete -- no celebration animation, no collapsed state

### Old Onboarding Removal
- **D-09:** Remove the old pre-Phase 37 onboarding system -- old wizard components, old routes, old onboarding state
- **D-10:** Researcher should discover the full removal inventory from the codebase -- no pre-known list of files

### Dashboard Layout
- **D-11:** Welcome message and checklist card positioned at the top of the dashboard page, above any existing content
- **D-12:** Below the welcome widget, existing dashboard content (including empty states) renders as normal -- welcome widget is additive, not a takeover
- **D-13:** Once checklist is complete, dashboard is the standard dashboard with no welcome remnants -- returning users see nothing special

### Claude's Discretion
- Welcome message exact copy (subtitle wording, greeting format)
- Checklist item descriptions exact wording
- Card dimensions, padding, spacing relative to existing dashboard content
- Checkbox visual styling (icon, color, animation on completion)
- How links work on checklist items (whole row clickable vs button)
- Responsive layout of the checklist card (mobile vs desktop)
- Transition when widget disappears (instant vs fade)
- Order of old onboarding removal operations

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.7 Requirements
- `.planning/REQUIREMENTS.md` -- Full v1.7 requirements; Phase 39 covers DASH-01, DASH-02, DASH-03

### Roadmap
- `.planning/ROADMAP.md` §Phase 39 -- Phase goal, success criteria, dependency on Phase 37

### Prior Phase Context (REQUIRED -- decisions carry forward)
- `.planning/phases/37-onboarding-wizard-pages/37-CONTEXT.md` -- Display name saved in Step 1 (D-17), onboarding guard logic (D-26-D-28), post-setup flow (D-15-D-19)
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` -- Route tree structure (D-05-D-08), DashboardLayout location, casual tone (D-01)
- `.planning/phases/38-hard-paywall-and-soft-paywall-removal/38-CONTEXT.md` -- Soft paywall removal inventory (D-12-D-15) as a pattern for old onboarding removal

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Card component (shadcn/ui) -- used throughout the app for content containers
- Lucide icons -- for checklist item icons and completion checkmarks
- Sonner toast -- for any completion notifications if needed

### Established Patterns
- Dashboard page exists inside DashboardLayout route structure
- RTK Query for API data fetching (getting onboarding progress state)
- Redux slices for client-side state management
- Mobile-first responsive design with `md:` breakpoints

### Integration Points
- Dashboard page component -- welcome widget added at the top
- Onboarding progress record -- data source for checklist completion detection
- DashboardLayout route tree -- old onboarding routes removed from here

</code_context>

<specifics>
## Specific Ideas

- Onboarding progress record is the single source of truth for checklist completion -- researcher should investigate what fields/states this record provides and how to query it for "has created first job" / "has sent first quote" status
- Old onboarding removal follows the same pattern as Phase 38's soft paywall removal -- systematic discovery and clean sweep

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 39-welcome-dashboard-and-final-cleanup*
*Context gathered: 2026-04-02*
