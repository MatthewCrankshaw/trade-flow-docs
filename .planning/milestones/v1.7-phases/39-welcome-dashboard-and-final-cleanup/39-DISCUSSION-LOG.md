# Phase 39: Welcome Dashboard and Final Cleanup - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 39-welcome-dashboard-and-final-cleanup
**Areas discussed:** Welcome message, Getting-started checklist, Old onboarding cleanup, Dashboard layout

---

## Welcome Message

### Greeting Style

| Option | Description | Selected |
|--------|-------------|----------|
| Name-based greeting | "Welcome, Matt!" or "Hey Matt, let's get started" -- uses display name from onboarding Step 1 | ✓ |
| Generic welcome | "Welcome to Trade Flow!" -- no personalisation | |
| Time-based greeting | "Good morning, Matt!" -- personalised with time of day + display name | |

**User's choice:** Name-based greeting
**Notes:** None

### Visibility

| Option | Description | Selected |
|--------|-------------|----------|
| When checklist complete | Welcome message disappears along with getting-started widget once all checklist items are done | ✓ |
| Always visible | Welcome message stays permanently on the dashboard | |
| First session only | Shows only on first visit after onboarding | |

**User's choice:** When checklist complete
**Notes:** None

### Tone

| Option | Description | Selected |
|--------|-------------|----------|
| Casual and direct | "Welcome, Matt! Let's get your business up and running." -- matches existing Phase 36/37 tone | ✓ |
| Minimal | "Welcome, Matt" with no subtitle | |

**User's choice:** Casual and direct
**Notes:** Carrying forward from Phase 36 D-01 and Phase 37 D-08

---

## Getting-Started Checklist

### Widget Style

| Option | Description | Selected |
|--------|-------------|----------|
| Card with checkboxes | Card component with checkbox-style items, checked items show completion | ✓ |
| Step-by-step progress | Vertical stepper with numbered steps and progress indicators | |
| Inline links | Simple text links below the welcome message | |

**User's choice:** Card with checkboxes
**Notes:** None

### Completion Detection

| Option | Description | Selected |
|--------|-------------|----------|
| API data check | Check if user has jobs and sent quotes via API calls | |
| Backend flag | New field on user/business entity | |
| Local storage | Track completion client-side | |

**User's choice:** Other -- "Using the onboarding progress record"
**Notes:** User wants to leverage the existing onboarding progress record rather than querying jobs/quotes or adding new fields

### All Done Behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Widget disappears | Getting-started card and welcome message hide. Dashboard shows normal content only. | ✓ |
| Completion celebration | Brief confetti/animation + "You're all set!" then widget disappears | |
| Collapsed state | Widget collapses to minimal "All done!" bar | |

**User's choice:** Widget disappears
**Notes:** None

### Item Detail Level

| Option | Description | Selected |
|--------|-------------|----------|
| Title + short description | "Create your first job" with subtitle like "Add a job to start tracking your work" | ✓ |
| Title only | Just "Create your first job" and "Send your first quote" | |

**User's choice:** Title + short description
**Notes:** None

---

## Old Onboarding Cleanup

### Cleanup Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Old onboarding removal | Remove the pre-Phase 37 onboarding system (old wizard, old routes, old state) | ✓ |
| General polish | No specific system to remove -- general tidying | |
| Both | Remove old system AND general cleanup | |

**User's choice:** Old onboarding removal
**Notes:** None

### Removal Inventory

| Option | Description | Selected |
|--------|-------------|----------|
| Researcher discovers | Let researcher scan codebase for full removal inventory | ✓ |
| I know what to remove | User lists specific files/components | |

**User's choice:** Researcher discovers
**Notes:** None

---

## Dashboard Layout

### Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Top of page | Welcome message and checklist card at the very top, above existing content | ✓ |
| Full-page takeover | Welcome content replaces entire dashboard for new users | |
| Side panel | Welcome widget in sidebar alongside existing content | |

**User's choice:** Top of page
**Notes:** None

### Empty State

| Option | Description | Selected |
|--------|-------------|----------|
| Existing empty states | Current dashboard content renders below welcome widget -- widget is additive | ✓ |
| Simplified empty view | Hide/simplify dashboard sections for new users | |
| You decide | Claude's discretion | |

**User's choice:** Existing empty states
**Notes:** None

### Returning Users

| Option | Description | Selected |
|--------|-------------|----------|
| Standard dashboard only | Once checklist complete, just the normal dashboard -- no welcome remnants | ✓ |
| Returning greeting | "Welcome back, Matt" header persists permanently | |

**User's choice:** Standard dashboard only
**Notes:** None

---

## Claude's Discretion

- Welcome message exact copy (subtitle wording, greeting format)
- Checklist item descriptions exact wording
- Card dimensions, padding, spacing
- Checkbox visual styling
- Link behavior on checklist items
- Responsive layout
- Transition when widget disappears
- Order of old onboarding removal operations

## Deferred Ideas

None -- discussion stayed within phase scope
