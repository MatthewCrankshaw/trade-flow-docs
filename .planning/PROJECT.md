# Trade Flow

## What This Is

A business management application for sole tradespeople -- plumbers, electricians, builders, and other independent contractors -- that replaces scattered tools (paper notes, spreadsheets, generic invoicing apps, WhatsApp, calendar apps) with one streamlined system built around how trades actually work. Everything connects to the job: quotes, schedules, materials, labour, invoices, payments, and customer history.

Two independent codebases: `trade-flow-api` (NestJS/MongoDB) and `trade-flow-ui` (React/Vite), each with their own git repo, managed and deployed independently. Feature branches are created in both repos for coordinated feature work.

## Core Value

A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## Current Milestone: None (planning next)

Previous milestone (v1.1 Item Tax Rate Linkage) shipped 2026-03-08. Run `/gsd:new-milestone` to start next.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

- ✓ User can sign up and authenticate via Firebase -- existing
- ✓ User can create and manage a business -- existing
- ✓ User can create and manage customers -- existing
- ✓ User can create and manage jobs (draft status, job types, descriptions) -- existing
- ✓ User can manage inventory items -- existing
- ✓ User can manage tax rates -- existing
- ✓ User can create quotes on jobs -- existing (basic display)
- ✓ Default job types and items generated per trade on business creation -- existing
- ✓ Onboarding flow guides new users through setup -- existing
- ✓ Email sending via SendGrid for verification and notifications -- existing
- ✓ User can create schedule entries on a job with date, start time, and duration -- v1.0
- ✓ User can view all schedule entries for a job in chronological order -- v1.0
- ✓ User can edit existing schedule entries (date, time, duration, visit type) -- v1.0
- ✓ User can cancel schedule entries (status change, entry preserved) -- v1.0
- ✓ User can add free-text notes to a schedule entry -- v1.0
- ✓ Schedule assignee defaults to logged-in user -- v1.0
- ✓ Duration defaults to 1 hour when not specified -- v1.0
- ✓ Schedule status lifecycle: Scheduled, Confirmed, Completed, Canceled, No-show -- v1.0
- ✓ Valid status transitions enforced by API with clear error messages -- v1.0
- ✓ User can select visit type when creating a schedule entry -- v1.0
- ✓ Default visit types generated per trade on business creation -- v1.0
- ✓ User can create custom visit types -- v1.0
- ✓ User can view and manage visit types for their business -- v1.0
- ✓ Schedule entries replace mock data in job detail page -- v1.0
- ✓ Job detail shows schedule count/status summary -- v1.0
- ✓ Empty state displayed when job has no schedule entries -- v1.0
- ✓ Items reference tax rates by ID instead of storing a numeric value -- v1.1
- ✓ Item create/edit forms show tax rate dropdown from business tax rates -- v1.1
- ✓ Item list/detail views unchanged (tax rate only visible in edit form) -- v1.1
- ✓ Item create/update validates referenced tax rate exists -- v1.1
- ✓ Default items during onboarding reference the correct default tax rate -- v1.1
- ✓ Quote line item factories resolve tax rate percentage from taxRateId -- v1.1

### Active

<!-- Requirements for next milestone -- to be defined via /gsd:new-milestone -->

(None yet — define via `/gsd:new-milestone`)

### Out of Scope

<!-- Explicit boundaries. -->

- Quote acceptance/rejection workflow -- next milestone candidate
- Invoice generation and management -- next milestone candidate
- Payment tracking -- next milestone candidate
- Standalone calendar or "Today" screen -- future milestone (schedules show on job page only)
- Arrival window scheduling mode -- v2 scheduling (SMODE-01/02/03)
- Conflict detection/overlap warnings -- v2 scheduling (CONF-01/02/03)
- Customer notifications for schedule entries -- future
- Required items/materials on schedule entries -- future
- Team member assignment on schedules -- future (solo operator only for now)
- File/photo uploads on jobs -- separate feature
- Job notes (non-schedule) -- separate feature
- Data export/backup -- future
- Audit logging -- future
- Drag-and-drop calendar -- over-engineering for current use case
- Route optimization -- solo operator with local work
- Automated scheduling / AI -- tradespeople want control
- Recurring schedules / templates -- trades jobs are mostly one-off
- Customer-facing booking portal -- tradespeople get work via calls/referrals
- Google Calendar integration -- schedules are job-centric, not calendar-centric

## Context

- **Brownfield project:** Existing codebase with established patterns (see `.planning/codebase/`)
- **Backend pattern:** Strict Controller -> Service -> Repository layering (NestJS)
- **Frontend pattern:** Feature-based modules with Redux RTK Query for server state
- **Auth:** Firebase JWT (RS256) with server-side public key validation
- **API contract:** Standardized response format: `{ data: T[], pagination?, errors? }`
- **Scheduling shipped (v1.0):** Visit types (CRUD + defaults per trade) and schedules (create/list/edit/cancel/status transitions) fully integrated into job detail page
- **Item tax rate linkage shipped (v1.1):** Items reference tax rates by ID; API validates references; UI shows tax rate dropdown on item forms; quote factories resolve rates
- **Two scheduling modes deferred:** Only exact start time shipped; arrival window mode deferred to v2
- **Conflict detection deferred:** Overlap warnings deferred to v2
- **Codebase size:** ~18.2k LOC API (TypeScript) + ~20.3k LOC UI (TypeScript/TSX)

## Constraints

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns -- see `.planning/codebase/CONVENTIONS.md`
- **Two repos:** Changes span `trade-flow-api` and `trade-flow-ui` with coordinated feature branches
- **Solo operator:** Schedule assignee is always the logged-in user for now, but data model supports team assignment later
- **No standalone calendar:** Schedules only appear within a job's detail page (v1.0 scope)

## Key Decisions

<!-- Decisions that constrain future work. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Start Time + Duration (not Start + End) | Matches how tradespeople think -- "I'll be there at 9, it'll take 2 hours" | ✓ Good -- natural UX |
| Two scheduling modes deferred to v2 | Keep v1 scope tight; exact time covers majority of use cases | ✓ Good -- shipped faster |
| Warn on conflicts deferred to v2 | Tradespeople sometimes intentionally double-book; advisory-only adds complexity | ✓ Good -- no blocker |
| Schedules on job page only (no calendar) | Keep scope tight; "Today" screen is a future milestone | ✓ Good -- job-centric focus |
| Visit types generated per trade | Consistent with existing job types pattern; reduces setup friction | ✓ Good -- zero-config onboarding |
| Data model supports team assignment | Even though solo-only now, avoid costly migration later | ✓ Good -- future-proofed |
| Separate MongoDB collection for schedules | Not embedded in Job -- allows independent querying and indexing | ✓ Good -- clean separation |
| Structured filter format (filter:field:op=value) | Project-wide API filtering standard from Phase 4 | ✓ Good -- reusable pattern |
| Merged date+startTime into single startDateTime | ISO8601 field simplifies parsing and timezone handling | ✓ Good -- cleaner API |
| Luxon DateTime enforced in all DTOs | Consistent date handling across API | ✓ Good -- eliminated date bugs |
| taxRateId reference instead of numeric defaultTaxRate | Proper relational model; tax rate changes propagate automatically | ✓ Good -- clean data model |
| Client-side tax rate resolution via RTK Query cache | No server joins needed; UI resolves tax rate details from cache | ✓ Good -- simpler API |
| TaxRateRepository directly (not RetrieverService) for validation | Avoids unnecessary auth check; item creation itself is business-scoped | ✓ Good -- simpler validation |
| Tax rate only visible in edit form (not list/detail) | User decision: tax rate is secondary detail | ✓ Good -- clean UI |

---
*Last updated: 2026-03-08 after v1.1 milestone completed*
