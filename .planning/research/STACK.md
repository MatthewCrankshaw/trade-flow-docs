# Stack Research: Job Scheduling for Trade Flow

**Research Date:** 2026-02-21
**Domain:** Job scheduling/visit management for tradespeople business management app
**Context:** Adding scheduling module to existing NestJS 11 + React 19 stack

## Recommendations

### Backend — No New Libraries Needed

**Confidence:** HIGH

The scheduling feature is a CRUD domain entity (schedule entries on jobs), not a background job scheduling system. This is an important distinction:

- **@nestjs/schedule** — NOT needed. This is for cron jobs and background tasks, not appointment/visit management.
- **Agenda / BullMQ** — NOT needed. These are job queues for background processing, not visit scheduling.

The existing NestJS Controller → Service → Repository pattern with MongoDB/Mongoose is sufficient. A Schedule module follows the same pattern as Job, Quote, Customer.

**What to build:**
- `ScheduleModule` with standard NestJS module structure
- Mongoose schema for schedule entries
- Controller with CRUD endpoints
- Creator/Retriever/Updater services (matching existing pattern)
- Repository with MongoDB operations

### Date/Time Handling — date-fns

**Confidence:** HIGH

| Library | Bundle Size | Timezone | Tree-Shakable | Recommendation |
|---------|-------------|----------|---------------|----------------|
| date-fns | ~13KB (used functions only) | Via date-fns-tz | Yes | **Use this** |
| Luxon | ~23KB | Built-in | No | Overkill for this use case |
| Day.js | ~2KB core | Via plugin | Plugin-based | Good but less ecosystem |

**Why date-fns:**
- Tree-shakable — only import what you use
- Largest community and ecosystem
- Works well with both NestJS and React
- `date-fns-tz` available if timezone support needed later
- Functional API fits well with TypeScript

**Why NOT Luxon:**
- Built-in timezone is great, but Trade Flow operates in a single locale (tradesperson's location)
- Heavier bundle for no benefit in this use case
- OOP API less natural in the existing functional patterns

**Why NOT Day.js:**
- Moment.js-compatible API brings legacy patterns
- Plugin system adds complexity

### Frontend — No Calendar Library

**Confidence:** HIGH

For this milestone, schedules appear as a list within job detail tabs — no standalone calendar view. Therefore:

- **react-big-calendar** — NOT needed yet. Future "Today" screen milestone.
- **FullCalendar** — NOT needed. Heavyweight, enterprise-focused.

Build a simple schedule list component within the existing job detail page using RTK Query for data fetching (matches existing pattern).

### MongoDB Schema Considerations

**Confidence:** HIGH

Schedule entries should be a separate collection (not embedded in Job document):
- Allows independent querying (future: "all schedules for today")
- Avoids document size bloat on jobs with many visits
- Enables conflict detection queries across all schedules
- Follows the existing pattern (Quotes are separate from Jobs)

**Indexes needed:**
- `{ jobId: 1, startDate: 1 }` — find schedules for a job, sorted by date
- `{ userId: 1, startDate: 1 }` — find schedules for a user on a date (conflict detection)
- `{ businessId: 1, status: 1 }` — business-level schedule queries

## What NOT to Use

| Library/Approach | Why Not |
|-----------------|---------|
| @nestjs/schedule | For cron jobs, not appointment management |
| Agenda / BullMQ | For background job queues, not visit scheduling |
| react-big-calendar | No calendar view in this milestone |
| FullCalendar | Enterprise complexity not needed |
| Moment.js | Deprecated, large bundle |
| Embedded MongoDB subdocuments | Limits query flexibility, document size issues |

## Summary

This is a straightforward CRUD feature that fits perfectly into the existing stack. No new libraries are needed beyond potentially `date-fns` for date manipulation. The existing patterns (NestJS modules, Mongoose schemas, RTK Query) handle everything.

---
*Research completed: 2026-02-21*
