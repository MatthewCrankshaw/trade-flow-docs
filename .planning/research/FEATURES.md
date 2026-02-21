# Features Research: Job Scheduling for Tradespeople

**Research Date:** 2026-02-21
**Domain:** Scheduling/visit management in field service apps
**Competitors Reviewed:** Tradify, Jobber, ServiceTitan, ServiceM8, Workever

## Table Stakes (Must Have or Users Leave)

These features are expected in any trades scheduling tool:

### 1. Create Schedule Entry on a Job
- **Complexity:** Low
- **What users expect:** Quick creation (under 2 minutes), linked to a specific job
- **Dependencies:** Job entity must exist
- **Notes:** Tradify and Jobber both center scheduling on jobs. This is fundamental.

### 2. Date + Time + Duration
- **Complexity:** Low
- **What users expect:** Pick a date, set a rough time, estimate how long
- **Dependencies:** None
- **Notes:** Every competitor has this. Start time + duration is more natural for trades than start + end.

### 3. View Schedules on a Job
- **Complexity:** Low
- **What users expect:** See all visits for a job in chronological order, with status
- **Dependencies:** Schedule CRUD
- **Notes:** Job detail page is the natural home for this.

### 4. Edit/Reschedule
- **Complexity:** Low
- **What users expect:** Drag or tap to change date/time. Jobs shift constantly in trades.
- **Dependencies:** Schedule CRUD
- **Notes:** Rescheduling is extremely common — jobs overrun, customers cancel, emergencies happen.

### 5. Cancel/Delete Visits
- **Complexity:** Low
- **What users expect:** Mark as canceled (not delete — keep history)
- **Dependencies:** Status system
- **Notes:** Canceled visits should remain visible for history. Soft cancel, not hard delete.

### 6. Visit Status Tracking
- **Complexity:** Low-Medium
- **What users expect:** Know if a visit is scheduled, confirmed, completed, canceled, or no-show
- **Dependencies:** Status state machine
- **Notes:** Jobber and Tradify both have status workflows. No-show is important for trades (customers not home).

### 7. Conflict Visibility
- **Complexity:** Medium
- **What users expect:** See when visits overlap. Don't block — just warn.
- **Dependencies:** Query across all user schedules
- **Notes:** Tradify uses Google Calendar integration for this. Building it natively is better UX.

## Differentiators (Competitive Advantage)

### 8. Two Scheduling Modes (Exact + Arrival Window)
- **Complexity:** Medium
- **What competitors do:** Most only support exact start times. Some support "morning/afternoon" slots.
- **Why it differentiates:** Matches how tradespeople actually communicate with customers. "Between 8 and 12" is extremely common for residential work.
- **Dependencies:** Schedule entity design
- **Notes:** This is a genuine insight that competitors largely miss.

### 9. Visit Types (Per-Trade)
- **Complexity:** Low-Medium
- **What competitors do:** Some have generic categories. Few generate per-trade types.
- **Why it differentiates:** "Boiler service" vs "Emergency callout" vs "Quote visit" — different types have different expectations.
- **Dependencies:** VisitType entity (similar to JobType)
- **Notes:** Matches existing JobType pattern — low implementation cost, high user value.

### 10. Notes on Schedule Entries
- **Complexity:** Low
- **What competitors do:** Most support notes. Some support photos.
- **Why it differentiates:** Per-visit notes capture what happened on each visit, not just on the job overall.
- **Dependencies:** None beyond schedule entity
- **Notes:** "Customer wasn't home, left card" or "Needs follow-up — part on order"

## Anti-Features (Do NOT Build)

### Drag-and-Drop Calendar
- **Why not:** No standalone calendar in this milestone. Adds significant UI complexity.
- **When to reconsider:** "Today" screen milestone.

### Route Optimization
- **Why not:** Solo operator with local work — they know their area. Over-engineering.
- **When to reconsider:** When team features are added and dispatch becomes relevant.

### Automated Scheduling / AI Optimization
- **Why not:** Tradespeople want control. Automated scheduling removes the "natural looseness" they need.
- **When to reconsider:** Never for sole traders. Maybe for larger teams.

### Real-Time Location Tracking
- **Why not:** Privacy concerns, battery drain, solo operator doesn't need to track themselves.
- **When to reconsider:** Team features milestone.

### Recurring Schedules / Templates
- **Why not:** Scope creep. Trades jobs are mostly one-off or infrequent repeat.
- **When to reconsider:** After core scheduling is solid and users request it.

### Customer-Facing Booking Portal
- **Why not:** Tradespeople get work via calls, referrals, and messages — not online booking.
- **When to reconsider:** Much later, if ever. This changes the product's nature.

## Feature Dependencies

```
Job (existing) ← Schedule Entry ← Visit Status
                                 ← Visit Type (new, mirrors JobType)
                                 ← Conflict Detection
                                 ← Notes
```

## Summary

The core feature set (CRUD + statuses + conflicts) is table stakes. The two scheduling modes (exact + window) and per-trade visit types are genuine differentiators that match the user's deep understanding of the domain. Keep anti-features firmly out — they represent the "precision scheduling" trap that doesn't fit tradespeople.

---
*Research completed: 2026-02-21*
