# Phase 8: Job Detail Integration - Research

**Researched:** 2026-03-07
**Domain:** React UI integration -- replacing mock data with real API data, enhancing schedule summary
**Confidence:** HIGH

## Summary

This phase is a UI-only integration phase with no backend changes. The work involves three targeted modifications: (1) enhancing the `ScheduleSummaryCard` in `JobOverviewSection.tsx` to show per-status badge breakdowns using existing `STATUS_CONFIG` colors, (2) replacing the hardcoded `getNextVisitHint()` function in `JobDetailPage.tsx` with logic that derives from real schedule data, and (3) confirming that `MOCK_SCHEDULES` is already removed from `JobDetailTabs.tsx` (it was -- Phase 6 already wired real data there).

All building blocks exist: `useGetSchedulesByJobQuery` is already called in `JobDetailPage`, schedule data already flows as props to both `JobOverviewSection` and `JobDetailTabs`, `STATUS_CONFIG` in `schedule-utils.ts` provides color classes for all 5 statuses, and `date-fns` `format()` is already used for date formatting. The `JobDetailHeader` already accepts `nextVisitHint` as a string prop, so only the computation logic needs changing.

**Primary recommendation:** This is a straightforward enhancement phase. Enhance `ScheduleSummaryCard` with per-status badges, replace `getNextVisitHint()` with a schedule-aware function, and verify end-to-end flow.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- Stat grid layout with color-coded badges using existing STATUS_CONFIG colors
- Only statuses with count > 0 are shown as badges
- Total count shown as "{N} visits" above the badges
- All statuses included in the total count (scheduled, confirmed, completed, canceled, no-show)
- Next Visit section shows date/time only (no visit type) -- "Thu 12 Mar, 2:00 PM"
- When no upcoming visits exist: display "No upcoming visits"
- Replace hardcoded getNextVisitHint() with logic that derives from real schedule data
- For active jobs: find the next upcoming schedule entry (scheduled or confirmed, future date, ascending sort) and display its date/time
- For active jobs with no upcoming visits: "Not scheduled yet"
- For completed/closed jobs with schedules: show last visit date (most recent past schedule)
- For completed/closed jobs with no schedules: "No visits recorded"
- On-hold jobs: "On hold" (job status takes precedence)
- Leave all non-schedule mock data as-is: MOCK_CUSTOMER, MOCK_JOB_TYPE, MOCK_ACCESS_NOTES, MOCK_COMMERCIAL (JobDetailPage.tsx)
- Leave MOCK_QUOTES, MOCK_INVOICES, MOCK_NOTES in JobDetailTabs.tsx
- No TODO comments needed -- MOCK_ prefix is self-documenting
- Clean up when respective features (quotes, invoices, customer details) ship in future milestones

### Claude's Discretion
- Badge sizing and spacing within the summary card
- Exact date/time format for next visit hint in header (consistent with ScheduleSummaryCard)
- How to pass schedule data to the header hint (prop drilling vs computing in page)
- Whether to extract next-visit logic into a shared utility or keep inline

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| INTG-01 | Schedule entries replace MOCK_SCHEDULES in the job detail page | MOCK_SCHEDULES was already removed in Phase 6. JobDetailTabs now receives real `schedules` prop from JobDetailPage. Remaining work: remove `getNextVisitHint()` mock function and wire real schedule-derived hint. |
| INTG-02 | Job detail page shows schedule count/status summary | Enhance existing `ScheduleSummaryCard` with per-status badge breakdown using `STATUS_CONFIG` colors. Add total visit count display. |

</phase_requirements>

## Standard Stack

### Core (already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19 | Component framework | Project standard |
| TypeScript | 5.9+ | Type safety | Project standard |
| date-fns | (installed) | Date formatting | Already used in ScheduleSummaryCard and schedule-utils |
| shadcn/ui Badge | (installed) | Status badges | Already used throughout for status display |
| RTK Query | (installed) | Data fetching | useGetSchedulesByJobQuery already wired |
| Tailwind CSS | 4.x | Styling | Project standard |

### Supporting
No new libraries needed. All required tools are already installed and in use.

### Alternatives Considered
None -- this phase uses only existing project infrastructure.

## Architecture Patterns

### Current Data Flow (already established)
```
JobDetailPage.tsx
  -> useGetSchedulesByJobQuery(businessId, jobId)  // fetches real data
  -> passes schedules to JobOverviewSection (props)
  -> passes schedules to JobDetailTabs (props)
  -> computes nextVisitHint string -> passes to JobDetailHeader (props)
```

### Pattern 1: Schedule-Derived Next Visit Hint
**What:** Replace the hardcoded `getNextVisitHint(status)` with a function that takes both `jobStatus` and `schedules` array.
**When to use:** In `JobDetailPage.tsx` before rendering `JobDetailHeader`.
**Recommendation:** Compute in `JobDetailPage.tsx` (where both `job.status` and `schedules` are available) and pass the string to `JobDetailHeader`. No need to change the header's interface -- it already accepts `nextVisitHint: string`.

```typescript
// In JobDetailPage.tsx -- replaces getNextVisitHint()
function deriveNextVisitHint(status: JobStatus, schedules?: Schedule[]): string {
  if (status === "on_hold") return "On hold";

  const isTerminalJob = status === "completed" || status === "closed";
  const hasSchedules = schedules && schedules.length > 0;

  if (isTerminalJob) {
    if (!hasSchedules) return "No visits recorded";
    // Most recent past schedule
    const sorted = [...schedules].sort(
      (a, b) => new Date(b.startDateTime).getTime() - new Date(a.startDateTime).getTime()
    );
    return `Last visit: ${format(new Date(sorted[0].startDateTime), "EEE d MMM, h:mm a")}`;
  }

  // Active job statuses: planned, scheduled, in_progress
  if (!hasSchedules) return "Not scheduled yet";

  const now = new Date();
  const upcoming = schedules
    .filter(
      (s) => (s.status === "scheduled" || s.status === "confirmed") &&
        new Date(s.startDateTime) > now
    )
    .sort((a, b) => new Date(a.startDateTime).getTime() - new Date(b.startDateTime).getTime());

  if (upcoming.length === 0) return "Not scheduled yet";
  return `Next: ${format(new Date(upcoming[0].startDateTime), "EEE d MMM, h:mm a")}`;
}
```

### Pattern 2: Per-Status Badge Breakdown in ScheduleSummaryCard
**What:** Enhance the existing `ScheduleSummaryCard` to show color-coded badges for each status that has count > 0.
**When to use:** Replace the current "Completed / Total" two-stat display.

```typescript
// Count schedules by status
const statusCounts = schedules.reduce<Partial<Record<ScheduleStatus, number>>>(
  (acc, s) => {
    acc[s.status] = (acc[s.status] ?? 0) + 1;
    return acc;
  },
  {}
);

// Render only non-zero statuses as badges
{Object.entries(statusCounts).map(([status, count]) => (
  <Badge
    key={status}
    className={cn("text-xs", STATUS_CONFIG[status as ScheduleStatus].className)}
  >
    {count} {STATUS_CONFIG[status as ScheduleStatus].label}
  </Badge>
))}
```

### Pattern 3: Import STATUS_CONFIG from schedule-utils
**What:** The `STATUS_CONFIG` record already exists in `@/features/schedules/components/schedule-utils.ts` with color classes for all 5 statuses.
**When to use:** Import into `JobOverviewSection.tsx` for the badge breakdown.

```typescript
import { STATUS_CONFIG } from "@/features/schedules/components/schedule-utils";
// or via barrel: import { STATUS_CONFIG } from "@/features/schedules";
```

### Anti-Patterns to Avoid
- **Duplicating STATUS_CONFIG colors:** Do not re-define status colors in JobOverviewSection. Import from schedule-utils.
- **Changing JobDetailHeader interface:** The `nextVisitHint: string` prop is sufficient. Do not pass schedules to the header.
- **Adding TODO comments for remaining mocks:** Per user decision, MOCK_ prefix is self-documenting.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Date formatting | Custom date string logic | `date-fns format()` | Already used; handles locale, edge cases |
| Status colors | New color definitions | `STATUS_CONFIG` from schedule-utils | Single source of truth, already tested visually |
| Schedule sorting/filtering | Custom sort in every component | Existing `groupSchedules()` or similar patterns from schedule-utils | Reuse proven logic |

## Common Pitfalls

### Pitfall 1: Stale Next Visit Hint After Schedule Changes
**What goes wrong:** User creates/edits/cancels a schedule but the header hint does not update.
**Why it happens:** `nextVisitHint` is computed from `schedules` which comes from RTK Query. If cache invalidation tags are correct (they are -- `JOB-${jobId}` tags from Phase 5), the data refetches automatically.
**How to avoid:** The hint must be derived reactively from the `schedules` array, not from a stale computation. Computing it inline in the render or via `useMemo` ensures it stays current.
**Warning signs:** Hint shows old date after creating a new schedule.

### Pitfall 2: Timezone Issues in "Next Visit" Display
**What goes wrong:** Schedule shows wrong time because `new Date(isoString)` interprets UTC as local.
**Why it happens:** ISO8601 strings from the API may be in UTC. `date-fns format()` formats in local timezone by default.
**How to avoid:** Use the same formatting approach already established in `ScheduleSummaryCard` (line 193) and `schedule-utils.ts`. The existing code already handles this consistently with `format(new Date(s.startDateTime), "EEE d MMM, h:mm a")`.
**Warning signs:** Times off by several hours.

### Pitfall 3: Rendering Badges for Zero-Count Statuses
**What goes wrong:** Empty badges appear for statuses with no schedules in that status.
**Why it happens:** Iterating over all 5 statuses instead of only those with count > 0.
**How to avoid:** Per user decision, only show badges for statuses with count > 0. Filter before rendering.

### Pitfall 4: Incorrect "Last Visit" for Completed Jobs
**What goes wrong:** Showing a future scheduled visit as "last visit" for a completed job.
**Why it happens:** Sorting by date descending without filtering to past dates only, or not considering all statuses.
**How to avoid:** For completed/closed jobs, sort all schedules by `startDateTime` descending and take the first. Per context decision, use the most recent schedule regardless of its status -- this answers "when did we last work on this?"

## Code Examples

### Current ScheduleSummaryCard (to be enhanced)
The existing card at `JobOverviewSection.tsx:173-225` currently shows:
- Next Visit (date/time or "No upcoming visits")
- Two stats: Completed count and Total Visits count

The enhancement replaces the two-stat display with:
- "{N} visits" total count
- Color-coded badges for each status with count > 0

### Existing STATUS_CONFIG (reuse as-is)
```typescript
// From schedule-utils.ts -- DO NOT duplicate
export const STATUS_CONFIG: Record<ScheduleStatus, { label: string; className: string }> = {
  scheduled: { label: "Scheduled", className: "bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400" },
  confirmed: { label: "Confirmed", className: "bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400" },
  completed: { label: "Completed", className: "bg-gray-100 text-gray-800 dark:bg-gray-900/30 dark:text-gray-400" },
  canceled: { label: "Canceled", className: "bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400" },
  no_show: { label: "No Show", className: "bg-amber-100 text-amber-800 dark:bg-amber-900/30 dark:text-amber-400" },
};
```

### Files to Modify (complete list)
1. **`JobDetailPage.tsx`** -- Replace `getNextVisitHint()` with schedule-aware `deriveNextVisitHint()`, pass `schedules` to the computation
2. **`JobOverviewSection.tsx`** -- Enhance `ScheduleSummaryCard` with per-status badge breakdown, import `STATUS_CONFIG`
3. **No changes to `JobDetailTabs.tsx`** -- Already uses real schedule data (no MOCK_SCHEDULES)
4. **No changes to `JobDetailHeader.tsx`** -- Interface already correct (`nextVisitHint: string`)
5. **No changes to `schedule-utils.ts`** -- STATUS_CONFIG already exported

### Barrel Export Check
Ensure `STATUS_CONFIG` is exported from the schedules feature barrel (`@/features/schedules/index.ts`). If not, either add it to the barrel or import directly from the component path.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `MOCK_SCHEDULES` in JobDetailTabs | Real `schedules` prop from RTK Query | Phase 6 | Already done -- no MOCK_SCHEDULES exists |
| Hardcoded `getNextVisitHint(status)` | Derive from real schedule data + job status | This phase | Header shows real next visit info |
| Simple "Completed / Total" stats | Per-status badge breakdown | This phase | Richer schedule overview |

## Open Questions

1. **Should `deriveNextVisitHint` be extracted to `schedule-utils.ts`?**
   - What we know: The function needs both `JobStatus` and `Schedule[]`, coupling it to both domains
   - What's unclear: Whether other components will need the same logic
   - Recommendation: Keep it in `JobDetailPage.tsx` initially (it is the only consumer). Extract later if reuse emerges. This is within Claude's discretion per CONTEXT.md.

2. **Should STATUS_CONFIG barrel export be verified?**
   - What we know: `schedule-utils.ts` exports it, but the barrel (`@/features/schedules/index.ts`) may not re-export it
   - Recommendation: Check and add to barrel if missing, or import directly from the utils file

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None detected in trade-flow-ui |
| Config file | None |
| Quick run command | `npm run typecheck` (in trade-flow-ui) |
| Full suite command | `npm run lint && npm run typecheck` (in trade-flow-ui) |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| INTG-01 | Mock schedules replaced with real data | manual | Visual verification on job detail page | N/A |
| INTG-02 | Schedule count/status summary shown | manual | Visual verification of badge breakdown | N/A |

### Sampling Rate
- **Per task commit:** `cd trade-flow-ui && npm run typecheck && npm run lint`
- **Per wave merge:** Same (no test suite)
- **Phase gate:** Typecheck + lint green, manual visual verification

### Wave 0 Gaps
None -- no test infrastructure exists for UI project. TypeScript type checking and ESLint serve as automated validation. Manual verification covers visual correctness.

## Sources

### Primary (HIGH confidence)
- Direct code inspection of all 5 relevant source files in the project
- CONTEXT.md decisions from user discussion session
- REQUIREMENTS.md for INTG-01, INTG-02 specifications

### Secondary (MEDIUM confidence)
- None needed -- this is a project-internal integration task with no external library research required

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All libraries already in use, no new additions
- Architecture: HIGH - Data flow already established, only computation logic changes
- Pitfalls: HIGH - Based on direct code inspection of existing patterns

**Research date:** 2026-03-07
**Valid until:** 2026-04-07 (stable -- internal refactoring only)
