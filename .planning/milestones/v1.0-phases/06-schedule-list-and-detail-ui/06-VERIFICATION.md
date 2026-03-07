---
phase: 06-schedule-list-and-detail-ui
verified: 2026-03-07T17:00:00Z
status: human_needed
score: 9/9 must-haves verified
human_verification:
  - test: "Navigate to job detail with schedule entries; verify Upcoming/Past grouping and chronological order"
    expected: "Upcoming section shows future dates ascending, Past shows past dates descending"
    why_human: "Date grouping depends on current time and real data; cannot verify visually via grep"
  - test: "Resize browser below 768px; verify card layout replaces table"
    expected: "Table on desktop, cards on mobile, same data visible in both"
    why_human: "Responsive behavior requires live browser testing"
  - test: "Click a schedule entry; verify detail dialog opens with read-only fields and editable notes"
    expected: "Dialog shows date, time, duration, visit type with color dot, status badge, and notes textarea"
    why_human: "Dialog interaction and field rendering need visual confirmation"
  - test: "Edit notes in detail dialog, click Save; close and reopen dialog"
    expected: "Toast 'Notes saved' appears; notes persist when dialog reopened"
    why_human: "End-to-end PATCH flow requires running app with API backend"
  - test: "Navigate to job with no schedules; click 'Schedule Visit' in empty state"
    expected: "ScheduleFormDialog opens for creation"
    why_human: "Callback wiring through EmptyTabState needs live interaction test"
  - test: "Verify canceled/no-show entries appear muted (lower opacity)"
    expected: "Rows/cards with canceled or no-show status have opacity-60"
    why_human: "Visual opacity difference needs human eye confirmation"
---

# Phase 6: Schedule List and Detail UI Verification Report

**Phase Goal:** Users can see all scheduled visits for a job and read visit details including notes
**Verified:** 2026-03-07T17:00:00Z
**Status:** human_needed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Schedule entries for a job are fetched via RTK Query and displayed in a list | VERIFIED | ScheduleList.tsx receives `schedules` prop; JobDetailTabs passes data from `useGetSchedulesByJobQuery` via JobDetailPage |
| 2 | Schedules are grouped into Upcoming (future, ascending) and Past (past, descending) sections | VERIFIED | `groupSchedules()` in schedule-utils.ts splits by `now`, sorts upcoming ascending, past descending; ScheduleList renders conditional section headers |
| 3 | Desktop shows a table view, mobile shows card view (768px breakpoint) | VERIFIED | ScheduleList.tsx uses `useMediaQuery("(min-width: 768px)")` to switch between ScheduleTable and ScheduleCardList |
| 4 | Each entry shows date, start time, duration, visit type with color dot, status badge | VERIFIED | ScheduleTable.tsx columns: formatScheduleDate, formatScheduleTime, formatDuration, visitType color dot + name, Badge with STATUS_CONFIG. Same data in ScheduleCardList.tsx |
| 5 | Canceled and no-show entries are visually muted (opacity) | VERIFIED | `isMutedStatus()` returns true for "canceled"/"no-show"; both ScheduleTable (line 58) and ScheduleCardList (line 42) apply `opacity-60` via `cn()` |
| 6 | Entries with notes show a small notes icon indicator | VERIFIED | Both ScheduleTable (line 67-69) and ScheduleCardList (line 60-62) conditionally render `<StickyNote>` icon when `schedule.notes` is truthy |
| 7 | Clicking a row/card calls onSelect; detail dialog opens with read-only fields and editable notes | VERIFIED | ScheduleTable/ScheduleCardList call `onSelect(schedule)` on click; JobDetailTabs wires `handleSelectSchedule` to open ScheduleDetailDialog with selected schedule |
| 8 | User can edit notes in detail dialog and save via PATCH endpoint | VERIFIED | ScheduleDetailDialog uses `useScheduleActions().updateNotes()` which calls `useUpdateScheduleMutation`; PATCH to `/v1/business/{bid}/job/{jid}/schedule/{sid}` with notes payload; save button disabled when unchanged or saving |
| 9 | MOCK_SCHEDULES removed; tab uses real API data | VERIFIED | No `MOCK_SCHEDULES` found anywhere in codebase; JobDetailPage passes `schedules` from `useGetSchedulesByJobQuery` to JobDetailTabs |

**Score:** 9/9 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `schedules/api/scheduleApi.ts` | updateSchedule mutation endpoint | VERIFIED | PATCH mutation with invalidatesTags, exports `useUpdateScheduleMutation` (line 80) |
| `schedules/components/schedule-utils.ts` | STATUS_CONFIG, formatters, groupSchedules | VERIFIED | 111 lines; exports STATUS_CONFIG (5 statuses), formatScheduleDate, formatScheduleTime, formatDuration, isMutedStatus, groupSchedules |
| `schedules/components/ScheduleList.tsx` | Orchestrator: data grouping, responsive switch, visit type resolution | VERIFIED | 139 lines; useMediaQuery, useGetVisitTypesQuery, groupSchedules, loading skeletons |
| `schedules/components/ScheduleTable.tsx` | Desktop table view | VERIFIED | 103 lines; Table with 5 columns, click handler, muted styling, notes icon |
| `schedules/components/ScheduleCardList.tsx` | Mobile card view | VERIFIED | 96 lines; role="button" divs with keyboard support, same data as table |
| `schedules/components/ScheduleDetailDialog.tsx` | Detail dialog with read-only fields and editable notes | VERIFIED | 189 lines; Dialog with label/value pairs, Textarea for notes, Save button with loading state, toast notifications |
| `schedules/components/index.ts` | Barrel exports | VERIFIED | Exports ScheduleDetailDialog, ScheduleFormDialog, ScheduleEmptyState, ScheduleList |
| `jobs/components/JobDetailTabs.tsx` | Schedule tab wired to real data | VERIFIED | Props accept schedules/schedulesLoading/onScheduleVisit; renders ScheduleList or EmptyTabState; ScheduleDetailDialog rendered |
| `pages/JobDetailPage.tsx` | Passes schedule data to tabs | VERIFIED | useGetSchedulesByJobQuery called; schedules/schedulesLoading/handleSchedule passed to JobDetailTabs |
| `schedules/hooks/useScheduleActions.ts` | updateNotes method | VERIFIED | 97 lines; exports updateNotes(scheduleId, notes) and isUpdating boolean |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| JobDetailPage.tsx | useGetSchedulesByJobQuery | RTK Query hook | WIRED | Line 91-95: data destructured as `schedules`, passed to JobDetailTabs line 192 |
| JobDetailTabs.tsx | ScheduleList | schedules prop | WIRED | Line 152-157: `<ScheduleList schedules={schedules!} ... onSelect={handleSelectSchedule}>` |
| JobDetailTabs.tsx | ScheduleDetailDialog | selectedSchedule state | WIRED | Lines 103-111: state management; line 205-211: dialog rendered with props |
| ScheduleList.tsx | ScheduleTable / ScheduleCardList | useMediaQuery responsive switch | WIRED | Line 91: `isDesktop` check; line 109: `ViewComponent = isDesktop ? ScheduleTable : ScheduleCardList` |
| ScheduleDetailDialog.tsx | useUpdateScheduleMutation | useScheduleActions hook | WIRED | Line 77: `updateNotes` from `useScheduleActions`; line 108: `await updateNotes(schedule.id, trimmed)` |
| JobDetailTabs.tsx | ScheduleFormDialog | EmptyTabState onAction | WIRED | Line 164: `onAction={onScheduleVisit}`; JobDetailPage line 113: `handleSchedule` opens dialog |
| ScheduleList.tsx | schedule-utils | groupSchedules | WIRED | Line 15: import; line 108: `groupSchedules(schedules)` called |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| SCHED-02 | 06-01, 06-02 | User can view all schedule entries for a job in chronological order | SATISFIED | groupSchedules splits upcoming/past with correct sort order; ScheduleList renders grouped sections; JobDetailTabs wired to real API data |
| SCHED-05 | 06-02 | User can add free-text notes to a schedule entry | SATISFIED | ScheduleDetailDialog textarea with save via PATCH; useUpdateScheduleMutation endpoint; useScheduleActions.updateNotes method; toast feedback |

No orphaned requirements found. REQUIREMENTS.md maps SCHED-02 and SCHED-05 to Phase 6, and both are covered by plans.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None found | - | - | - | - |

No TODOs, FIXMEs, placeholder returns, empty implementations, or console.log-only handlers detected in phase 6 artifacts. The `placeholder` attribute on the notes Textarea (ScheduleDetailDialog.tsx:175) is a legitimate HTML input placeholder.

### Human Verification Required

### 1. Chronological Grouping with Real Data

**Test:** Navigate to a job detail page that has schedule entries with dates spanning past and future
**Expected:** Entries appear under "Upcoming" (future dates ascending) and "Past" (past dates descending) headers
**Why human:** Grouping depends on current time and real data distribution; cannot verify visually via static analysis

### 2. Responsive Table/Card Switch

**Test:** On desktop (>=768px) verify table layout; resize below 768px and verify card layout
**Expected:** Table columns on desktop, stacked cards on mobile, same data visible in both views
**Why human:** CSS media query behavior requires live browser testing

### 3. Detail Dialog Interaction

**Test:** Click a schedule entry; verify detail dialog opens with all read-only fields and editable notes textarea
**Expected:** Dialog shows Date, Time, Duration, Visit Type (with color dot), Status (with badge), and Notes textarea
**Why human:** Dialog rendering and field display need visual confirmation

### 4. Notes Save End-to-End

**Test:** Edit notes in detail dialog, click Save; close dialog and reopen
**Expected:** Toast "Notes saved" appears on success; notes persist when dialog reopened
**Why human:** Requires running app with API backend to test full PATCH flow

### 5. Empty State Action

**Test:** Navigate to a job with no schedules; click "Schedule Visit" button in empty state
**Expected:** ScheduleFormDialog opens for schedule creation
**Why human:** Callback chain through EmptyTabState needs live interaction test

### 6. Muted Status Styling

**Test:** View entries with "canceled" or "no-show" status
**Expected:** Those entries appear visually muted (lower opacity) compared to other entries
**Why human:** Visual opacity difference requires human eye confirmation

### Gaps Summary

No gaps found. All 9 observable truths pass automated verification. All artifacts exist, are substantive (not stubs), and are properly wired. All key links verified. Both requirements (SCHED-02, SCHED-05) are satisfied. TypeScript compiles clean and lint passes with no errors.

The phase requires human verification for 6 items that involve live browser interaction: chronological grouping with real data, responsive layout switching, dialog interaction, notes save persistence, empty state creation trigger, and visual muted styling.

---

_Verified: 2026-03-07T17:00:00Z_
_Verifier: Claude (gsd-verifier)_
