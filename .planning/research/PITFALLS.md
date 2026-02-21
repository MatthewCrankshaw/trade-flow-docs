# Pitfalls Research: Job Scheduling for Tradespeople

**Research Date:** 2026-02-21
**Domain:** Scheduling implementation for field service / trades apps
**Sources:** Field service industry analysis, competitor reviews, common implementation failures

## Critical Pitfalls

### 1. Over-Engineering the Scheduling Model

**The mistake:** Building a complex calendar/scheduling engine when all you need is CRUD entries with date/time fields. Introducing concepts like "time slots," "availability windows," "booking rules," or "capacity management" for a solo operator.

**Warning signs:**
- Creating an "availability" entity before anyone asked for it
- Building slot-based booking logic
- Adding recurring schedule patterns in v1
- Referencing Google Calendar API integration early

**Prevention strategy:**
- Schedule entries are simple documents: job + user + date + time + duration + status
- No availability system — the conflict checker is sufficient
- No calendar integration — just a list view on the job page
- YAGNI: the "Today" screen and calendar view come later

**Phase relevance:** Phase 1 (schema design) — keep the model flat and simple.

---

### 2. Timezone Handling Mistakes

**The mistake:** Storing times without timezone context, or over-engineering timezone support. Tradespeople work in one locale, but if dates are stored as UTC without proper conversion, "9am appointment" could display as the wrong time.

**Warning signs:**
- Storing raw Date objects without considering timezone
- Displaying UTC times to users
- Converting times multiple times (double-conversion bugs)
- Building timezone selector UI for solo operators

**Prevention strategy:**
- Store dates as UTC in MongoDB (standard practice)
- Convert to local time in the UI layer only
- Use a single timezone per business (configurable later, hardcoded for now)
- date-fns handles this cleanly with `format` and `parse`
- Test with times near midnight to catch off-by-one-day bugs

**Phase relevance:** Phase 1 (schema) and Phase 3 (UI rendering).

---

### 3. Conflict Detection That Blocks Instead of Warns

**The mistake:** Making conflict detection a hard blocker. Tradespeople intentionally stack jobs — "I'll pop in to check the boiler while I'm in the area, then head to the main job." Blocking overlaps frustrates power users.

**Warning signs:**
- Throwing validation errors for overlapping schedules
- Preventing save on conflict
- Complex "is this slot available?" logic

**Prevention strategy:**
- Conflict detection returns warnings, not errors
- UI shows conflicts visually (yellow highlight, not red blocker)
- User explicitly acknowledges the overlap and proceeds
- Never prevent a schedule from being created

**Phase relevance:** Phase 2 (conflict detection) — build as advisory, not blocking.

---

### 4. Status Transitions Without Guard Rails

**The mistake:** Allowing any status to transition to any other status, leading to inconsistent data. Or the opposite — making transitions so rigid that users can't fix mistakes.

**Warning signs:**
- No state machine definition
- "Completed" entries being marked "Scheduled" again
- Users unable to correct a wrongly-set status
- No audit trail of status changes

**Prevention strategy:**
- Define explicit valid transitions (see Architecture research)
- Validate transitions in the service layer, not just UI
- Allow "reset" transitions for corrections (e.g., confirmed → scheduled if mistake)
- Keep it simple: 5 statuses is enough. Don't add "in-transit" or "on-site."

**Phase relevance:** Phase 1 (service layer) — enforce from day one.

---

### 5. Two Scheduling Modes Adds UI Complexity

**The mistake:** Making the scheduling form confusing by showing all fields for both modes simultaneously. Users don't understand the difference between "exact" and "window" modes if presented poorly.

**Warning signs:**
- Form shows 6+ fields at once
- Users confused about which fields to fill
- Mode selector is hidden or unclear
- Both modes look the same in the schedule list

**Prevention strategy:**
- Clear mode toggle at the top of the form (radio buttons or tabs)
- Each mode shows ONLY its relevant fields
- Smart defaults: exact mode defaults duration to 1 hour; window mode defaults to 4-hour window
- List view clearly distinguishes: "9:00 AM (2hrs)" vs "8:00 AM – 12:00 PM window (2hrs)"

**Phase relevance:** Phase 3 (UI) — design the form carefully.

---

### 6. Ignoring the "Quick Creation" Requirement

**The mistake:** Building a schedule form that takes 5+ minutes to fill out. Tradespeople create schedules while on the phone with customers — speed is critical.

**Warning signs:**
- More than 5 fields on the creation form
- Required fields that need lookup (e.g., searching for a visit type)
- No smart defaults
- Multi-step form wizard

**Prevention strategy:**
- Minimum required fields: date, time, duration (mode defaults to "exact")
- Auto-fill: user = logged-in user, status = "scheduled", visit type = default
- Smart defaults: duration = 1 hour, mode = exact
- Optional fields expandable: notes, visit type override
- Target: create a schedule in under 60 seconds

**Phase relevance:** Phase 3 (UI) — measure creation time during testing.

---

### 7. Not Replacing Mock Data Properly

**The mistake:** Building the schedule feature but not properly disconnecting the MOCK_SCHEDULES in JobDetailTabs. Leaving mock data paths, stale types, or conditional rendering logic.

**Warning signs:**
- MOCK_SCHEDULES still referenced somewhere
- Schedule list shows mock data in some conditions
- Types from mock data conflict with real types
- UI works differently when API returns empty vs mock data present

**Prevention strategy:**
- Explicitly remove all MOCK_SCHEDULES references
- Build the schedule tab to handle empty state gracefully ("No visits scheduled yet")
- Test with zero schedules, one schedule, and many schedules
- Search codebase for "MOCK_SCHEDULE" after integration

**Phase relevance:** Final integration phase — explicit cleanup step.

---

### 8. Visit Types as Afterthought

**The mistake:** Skipping visit types or making them free-text, then trying to add structure later. Or building visit types completely differently from the existing JobType pattern, creating inconsistency.

**Warning signs:**
- Visit type stored as string instead of reference
- No default visit types on business creation
- Visit type CRUD built with different patterns than JobType
- Visit type selection is a text field, not a dropdown

**Prevention strategy:**
- Mirror JobType pattern exactly: schema, service, repository
- Generate default visit types per trade (same approach as default job types)
- Reference by ID, not string
- Add to business creation flow alongside default job types

**Phase relevance:** Phase 1 (backend first) — establish before schedule CRUD uses it.

---

## Summary: Top 3 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Over-engineering the model | HIGH | Delays delivery, adds complexity | Keep it flat CRUD, no availability system |
| Slow schedule creation UX | MEDIUM | Users won't adopt the feature | Smart defaults, minimal required fields, measure creation time |
| Timezone display bugs | MEDIUM | Wrong times shown, missed appointments | Store UTC, convert in UI only, test near midnight |

---
*Research completed: 2026-02-21*
