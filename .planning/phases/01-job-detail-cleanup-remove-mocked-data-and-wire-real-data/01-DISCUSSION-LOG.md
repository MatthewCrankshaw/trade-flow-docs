# Phase 1: Job Detail Cleanup — Remove mocked data and wire real data - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-21
**Phase:** 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
**Areas discussed:** Scope boundary, Empty feature handling, Commercial summary, Access notes & job fields

---

## Scope Boundary

### Q1: Wire real data vs. remove mocks only

| Option | Description | Selected |
|--------|-------------|----------|
| Wire real data (Recommended) | Replace mocks with actual API calls for customer, job type, quotes | ✓ |
| Remove mocks only | Delete all hardcoded data and show empty states everywhere | |
| Wire what's easy, stub the rest | Wire customer + job type, show empty states for the rest | |

**User's choice:** Wire real data
**Notes:** None

### Q2: Sections without backend support — stay visible or remove?

| Option | Description | Selected |
|--------|-------------|----------|
| Keep with empty states (Recommended) | Tabs stay visible with "No invoices yet" etc. | |
| Remove until built | Strip out tabs/sections that have no backend | ✓ |
| Hide behind feature flags | Sections conditionally hidden | |

**User's choice:** Remove until built
**Notes:** "But the timeline should remain for parts that currently exist like schedules, quotes and estimates and job state changes."

### Q3: Timeline — client-side assembly or dedicated backend endpoint?

| Option | Description | Selected |
|--------|-------------|----------|
| Client-side assembly (Recommended) | Frontend fetches separately and merges | |
| Dedicated backend endpoint | New API endpoint aggregates events | ✓ |

**User's choice:** Dedicated backend endpoint
**Notes:** None

### Q4: What events should the timeline endpoint return?

| Option | Description | Selected |
|--------|-------------|----------|
| Job status changes | Created, started, completed, closed | ✓ |
| Schedule events | Visits with date/time, status changes | ✓ |
| Quote events | Created, sent, accepted/rejected | ✓ |
| Estimate events | Created, sent, responded, converted, lost | ✓ |

**User's choice:** All four event types
**Notes:** None

### Q5: Job status history storage approach

| Option | Description | Selected |
|--------|-------------|----------|
| Derive from timestamps (Recommended) | Use createdAt/updatedAt | |
| Add statusHistory array | Embedded array on job entity | |
| Separate events collection | New job_events collection | ✓ |

**User's choice:** Separate events collection
**Notes:** "New job_events collection logging all state changes and changes to other related records that are relevant to the job like quotes, estimates and schedules."

### Q6: Event recording pattern

| Option | Description | Selected |
|--------|-------------|----------|
| Inline in services | Each service writes a job event | ✓ |
| Event listener pattern | Domain events with subscriber | |
| You decide | Claude picks | |

**User's choice:** Inline in services
**Notes:** None

### Q7: Backfill historical events?

| Option | Description | Selected |
|--------|-------------|----------|
| Forward-only (Recommended) | Only new actions create events | ✓ |
| Backfill from existing data | Migration creates events from timestamps | |
| Backfill job creation only | Minimal backfill | |

**User's choice:** Forward-only
**Notes:** None

---

## Empty Feature Handling

### Q1: Tab structure after removing Invoices and Notes

| Option | Description | Selected |
|--------|-------------|----------|
| Keep as tabs (Recommended) | Schedules and Quotes remain as tabs | ✓ |
| Convert to page sections | Remove tab structure, show vertically | |
| You decide | Claude picks | |

**User's choice:** Keep as tabs
**Notes:** None

### Q2: Create Invoice placeholder button

| Option | Description | Selected |
|--------|-------------|----------|
| Remove it | No button until feature built | ✓ |
| Keep as disabled | Greyed out with "Coming soon" tooltip | |
| You decide | Claude picks | |

**User's choice:** Remove it
**Notes:** None

---

## Commercial Summary

### Q1: Financial summary without invoices

| Option | Description | Selected |
|--------|-------------|----------|
| Show quotes only (Recommended) | Total quoted amount + quote statuses | ✓ |
| Remove section entirely | No financial info | |
| Show quotes + estimates | Both quoted and estimated amounts | |

**User's choice:** Show quotes only
**Notes:** None

### Q2: What to display in quotes-only summary

| Option | Description | Selected |
|--------|-------------|----------|
| Total + status breakdown | Total + count by status | |
| Just total quoted amount | Single aggregated number | ✓ |
| You decide | Claude picks | |

**User's choice:** Just total quoted amount
**Notes:** None

---

## Access Notes & Job Fields

### Q1: Access notes — add or defer?

| Option | Description | Selected |
|--------|-------------|----------|
| Add to job entity | Embedded accessNotes object | |
| Defer entirely (Recommended) | Remove section, design later | ✓ |
| Add as freetext field | Single siteNotes field | |

**User's choice:** Defer entirely
**Notes:** None

### Q2: Job type and address display

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, wire both (Recommended) | Add jobTypeId to frontend types, surface address | ✓ |
| Job type only | Wire job type, address can wait | |
| You decide | Claude determines | |

**User's choice:** Yes, wire both
**Notes:** None

---

## Claude's Discretion

- Component structure for job events module
- Exact UI layout adjustments after section removal
- Job event entity field design
- Timeline pagination strategy

## Deferred Ideas

- Access notes / site info — future phase
- Invoices tab — requires invoice module
- Notes tab — requires notes module
- Timeline backfill migration — retroactive events for existing jobs
- Commercial summary invoiced/paid/outstanding — requires invoice feature
