---
phase: 34-use-luxon-datetime-in-dtos-extend-ibaseresourcedto-move-createdat-updatedat-to-entity-only
verified: 2026-03-30T19:30:00Z
status: passed
score: 9/9 must-haves verified
re_verification: false
gaps: []
human_verification:
  - test: "Visually verify date formatting in subscription billing tab and trial banner"
    expected: "Dates display as human-readable strings (e.g. '1 May 2026') not ISO strings"
    why_human: "Luxon toLocaleString formatting depends on system locale; cannot verify display output programmatically without a browser"
  - test: "Verify schedule form date picker pre-fills correctly when editing an existing schedule"
    expected: "Date field shows the existing schedule's date, time field shows formatted HH:mm"
    why_human: "ScheduleFormDialog converts schedule.startDateTime via DateTime.fromISO().toJSDate() and DateTime.fromISO().toFormat() — correctness requires visual inspection in browser"
---

# Phase 34: Luxon DateTime Standardization — Verification Report

**Phase Goal:** Standardize date/time handling across the full stack -- Luxon DateTime in all DTOs, ISubscriptionDto aligned to IBaseResourceDto, shared toDateTime utility, Luxon adopted on the frontend, and conventions documented in both CLAUDE.md files
**Verified:** 2026-03-30T19:30:00Z
**Status:** PASSED
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Shared toDateTime and toOptionalDateTime utility exists in src/core/utilities/ with unit tests | VERIFIED | `trade-flow-api/src/core/utilities/to-date-time.utility.ts` (10 lines, both functions exported); `src/core/test/utilities/to-date-time.utility.spec.ts` (60 lines, 5 tests covering UTC zone, null/undefined, valid Date) |
| 2 | ISubscriptionDto extends IBaseResourceDto and has no createdAt or updatedAt fields | VERIFIED | `subscription.dto.ts` line 4: `export interface ISubscriptionDto extends IBaseResourceDto`; grep of DTO for createdAt/updatedAt returns zero matches |
| 3 | All date fields on ISubscriptionDto are Luxon DateTime (currentPeriodEnd, trialEnd, canceledAt) | VERIFIED | All three fields typed as `DateTime` (optional); no native `Date` type found in the DTO |
| 4 | No DTO across the API uses native JS Date for date fields | VERIFIED | Comprehensive grep of `src/*/data-transfer-objects/*.dto.ts` for `: Date` (excluding `DateTime`) returns zero matches |
| 5 | Date-to-DateTime conversion happens in repository layer via shared utility | VERIFIED | `subscription.repository.ts` imports `toOptionalDateTime` from `@core/utilities/to-date-time.utility` and calls it in `mapToDto()`; `toEntityFields()` helper converts DateTime back to Date for MongoDB writes |
| 6 | Stripe field names are unchanged (D-06) | VERIFIED | `subscription.dto.ts` preserves `stripeSubscriptionId`, `stripeCustomerId`, `stripeLatestCheckoutSessionId` |
| 7 | API CLAUDE.md documents Luxon DateTime and ISO 8601 UTC standards (D-13) | VERIFIED | `trade-flow-api/CLAUDE.md` contains full `## Date/Time Standards` section (lines 146–200) covering DTO standard, ISO 8601 serialization, MongoDB storage, conversion boundary, shared utilities, and rules |
| 8 | Luxon is installed in trade-flow-ui and date-helpers module exists with all required exports | VERIFIED | `trade-flow-ui/package.json` has `"luxon": "^3.7.2"` (dep) and `"@types/luxon": "^3.7.1"` (devDep); `src/lib/date-helpers.ts` exports parseApiDate, formatDate, formatDateTime, formatRelative, daysUntil |
| 9 | UI CLAUDE.md documents Luxon date standards (D-14) | VERIFIED | `trade-flow-ui/CLAUDE.md` contains `## Date/Time Standards` section explicitly stating "Never use new Date() for parsing API date strings", documenting `@/lib/date-helpers`, acceptable Calendar boundary exception |

**Score:** 9/9 truths verified

---

### Required Artifacts

| Artifact | Expected | Level 1 Exists | Level 2 Substantive | Level 3 Wired | Status |
|----------|----------|----------------|---------------------|---------------|--------|
| `trade-flow-api/src/core/utilities/to-date-time.utility.ts` | Shared Date-to-DateTime conversion | YES | YES — 10 lines, both functions with UTC zone enforcement | YES — imported by subscription.repository.ts | VERIFIED |
| `trade-flow-api/src/core/test/utilities/to-date-time.utility.spec.ts` | Unit tests for utility | YES | YES — 5 tests in 60 lines | N/A (test file) | VERIFIED |
| `trade-flow-api/src/subscription/data-transfer-objects/subscription.dto.ts` | Extends IBaseResourceDto, Luxon DateTime date fields, no timestamps | YES | YES — extends IBaseResourceDto, DateTime fields, no createdAt/updatedAt | YES — consumed by repository, services, test mocks | VERIFIED |
| `trade-flow-api/src/subscription/repositories/subscription.repository.ts` | Uses toOptionalDateTime in mapToDto, toEntityFields for writes | YES | YES — toOptionalDateTime in mapToDto, private toEntityFields handles DateTime-to-Date for writes | YES — connected to DTO and entity | VERIFIED |
| `trade-flow-ui/src/lib/date-helpers.ts` | parseApiDate, formatDate, formatDateTime, formatRelative, daysUntil | YES | YES — all 5 functions implemented with Luxon | YES — imported by subscription-helpers.ts; Luxon DateTime used across 13 component files | VERIFIED |
| `trade-flow-api/CLAUDE.md` (Date/Time Standards section) | Documents Luxon, ISO 8601, conversion boundary, shared utility | YES — section present | YES — covers DTO standard, serialization, MongoDB storage, conversion boundary, rules | N/A (documentation) | VERIFIED |
| `trade-flow-ui/CLAUDE.md` (Date/Time Standards section) | Documents Luxon, parseApiDate, no new Date() for API strings | YES — section present | YES — explicit rules, helper references, Calendar exception documented | N/A (documentation) | VERIFIED |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| subscription.repository.ts | to-date-time.utility.ts | import toOptionalDateTime | WIRED | Line 10: `import { toOptionalDateTime } from "@core/utilities/to-date-time.utility"` |
| subscription.repository.ts | subscription.dto.ts | mapToDto() / toEntityFields() | WIRED | mapToDto returns ISubscriptionDto; toEntityFields converts DateTime back to Date |
| stripe-webhook.processor.ts | ISubscriptionDto | DateTime.fromSeconds() for Stripe timestamps | WIRED | Lines 93, 97, 132 use DateTime.fromSeconds/DateTime.utc() — no raw Date in Stripe field construction |
| subscription-mock-generator.ts | ISubscriptionDto | DateTime.fromISO() for mock date fields | WIRED | currentPeriodEnd and trialEnd use `DateTime.fromISO("...", { zone: "utc" })` — no new Date() in mock DTO |
| trade-flow-ui components | date-helpers.ts | import formatDate, daysUntil | WIRED | subscription-helpers.ts imports daysUntil and formatDate; 13 component files import DateTime from luxon directly |

---

### Data-Flow Trace (Level 4)

Not applicable for this phase. The phase standardizes the type system and serialization pipeline, not data retrieval. The wire format (ISO 8601 strings from JSON.stringify) is unchanged — Luxon DateTime.toJSON() produces the same ISO strings as native Date.toJSON(). No new data sources introduced.

---

### Behavioral Spot-Checks

| Behavior | Evidence | Status |
|----------|----------|--------|
| Git commits from both plans exist in their respective repos | API: 91934ac, 7688cec, 4c6d358, 7061f1f all confirmed in `git log`; UI: 77cd2a8, 4e93cca, a4c4f29 all confirmed | PASS |
| toDateTime/toOptionalDateTime are exported from utility file | File read confirms both functions exported | PASS |
| date-helpers.ts exports all 5 required functions | File read confirms parseApiDate, formatDate, formatDateTime, formatRelative, daysUntil all exported | PASS |
| Subscription test mocks use DateTime not native Date | subscription-mock-generator.ts uses DateTime.fromISO() for all date fields; no new Date() in mock DTO construction | PASS |
| No date-fns imports in migrated UI components | Files modified in plan 2 use Luxon; date-fns references not found in migrated components | PASS |

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| D-01 | 34-01-PLAN | IBaseResourceDto stays as `{ id: string }` — no timestamps added | SATISFIED | base-resource.dto.ts verified: only `id: string` |
| D-02 | ORPHANED (no plan claims it) | Individual DTOs add createdAt/updatedAt as Luxon DateTime only when feature needs them | SATISFIED BY CONVENTION | D-07 rule enforces Luxon DateTime for any date field added; CLAUDE.md states "All date/time fields in DTOs use Luxon DateTime -- never native JS Date". No explicit documentation of the opt-in policy itself, but D-07 enforces correctness when applied |
| D-03 | 34-01-PLAN | ISubscriptionDto must extend IBaseResourceDto | SATISFIED | subscription.dto.ts line 4: `extends IBaseResourceDto` confirmed |
| D-04 | 34-01-PLAN | Strip createdAt and updatedAt from ISubscriptionDto | SATISFIED | grep of DTO for createdAt/updatedAt returns zero matches; entity retains them |
| D-05 | 34-01-PLAN | Convert currentPeriodEnd, trialEnd, canceledAt to Luxon DateTime | SATISFIED | All three typed as `DateTime` in subscription.dto.ts |
| D-06 | 34-01-PLAN | Keep Stripe-specific field names unchanged | SATISFIED | stripeSubscriptionId, stripeCustomerId, stripeLatestCheckoutSessionId preserved |
| D-07 | 34-01-PLAN | No DTO may use native JS Date — all date fields must be Luxon DateTime | SATISFIED | Full grep of all DTO files returns zero `: Date` matches (excluding DateTime) |
| D-08 | 34-01-PLAN | Date-to-DateTime conversion in repository layer only | SATISFIED | toOptionalDateTime called in mapToDto(); webhook processor uses DateTime.fromSeconds (constructs DateTime directly, not raw Date in DTO) |
| D-09 | 34-01-PLAN | Dates in API responses serialize as ISO 8601 UTC strings | SATISFIED | Luxon DateTime.toJSON() calls toISO() — serialization is automatic; subscription.response.ts correctly types fields as `string` for HTTP wire format |
| D-10 | 34-01-PLAN | MongoDB dates use ISO 8601 UTC format for consistency | SATISFIED | Entities use native JS Date (stored as BSON Date by MongoDB, always UTC); toEntityFields converts DateTime back to Date via toJSDate() before writes |
| D-11 | 34-02-PLAN | Add Luxon to trade-flow-ui as a dependency | SATISFIED | package.json confirmed: `"luxon": "^3.7.2"`, `"@types/luxon": "^3.7.1"` |
| D-12 | 34-02-PLAN | Standardize all frontend date parsing/display to Luxon, no raw new Date() for API dates | SATISFIED | 13 component files migrated to Luxon; remaining new Date() calls are Calendar UI boundaries (explicitly permitted); no raw Date API parsing found |
| D-13 | 34-01-PLAN | Update trade-flow-api/CLAUDE.md with Luxon/ISO 8601 standards | SATISFIED | Full Date/Time Standards section present in API CLAUDE.md covering all required topics |
| D-14 | 34-02-PLAN | Update trade-flow-ui/CLAUDE.md with Luxon date standards | SATISFIED | Date/Time Standards section present in UI CLAUDE.md with Luxon, parseApiDate, no-new-Date rule |

**Orphaned requirement:** D-02 was not claimed by either plan. It is a policy/convention decision (DTOs may add Luxon DateTime timestamps when explicitly needed) that is satisfied by the broader D-07 enforcement. No code gap exists.

---

### Anti-Patterns Found

| File | Pattern | Severity | Assessment |
|------|---------|----------|------------|
| `trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx:118` | `date: new Date()` | Info | ACCEPTABLE — Calendar component default value requires native Date object; plan explicitly permitted `new Date()` for Calendar UI boundaries |
| `trade-flow-ui/src/features/schedules/components/ScheduleFormDialog.tsx:226` | `before: new Date()` | Info | ACCEPTABLE — DayPicker `modifiers.past` requires native Date; same Calendar boundary exception |
| `trade-flow-ui/src/components/ui/calendar.tsx:199` | `day.date.toLocaleDateString()` | Info | ACCEPTABLE — Internal shadcn/ui Calendar component, not authored code; outside migration scope |
| `trade-flow-api/src/subscription/test/repositories/subscription.repository.spec.ts` | Multiple `new Date()` | Info | ACCEPTABLE — These construct `ISubscriptionEntity` objects (MongoDB entity layer), not DTOs. Entities legitimately use native Date |

No blocker anti-patterns found. All flagged usages are at correct layer boundaries.

---

### Human Verification Required

#### 1. Subscription Date Formatting Display

**Test:** Navigate to Settings > Billing tab with an active or trialing subscription
**Expected:** Dates appear as human-readable locale strings (e.g. "1 May 2026") not raw ISO strings like "2026-05-01T00:00:00.000Z"
**Why human:** Luxon `toLocaleString(DateTime.DATE_MED)` output depends on browser locale configuration; cannot be verified programmatically without a running browser environment

#### 2. Schedule Form Date Pre-fill on Edit

**Test:** Open an existing scheduled job visit in edit mode via the ScheduleFormDialog
**Expected:** The date field pre-fills with the schedule's existing date; the time field shows formatted HH:mm from the existing startDateTime
**Why human:** ScheduleFormDialog uses `DateTime.fromISO(schedule.startDateTime).toJSDate()` for the Calendar default and `DateTime.fromISO(schedule.startDateTime).toFormat("HH:mm")` for the time field — correctness requires visual confirmation in a browser

---

### Gaps Summary

No gaps found. All must-haves are verified, all D-series requirements are satisfied, commits exist in both repos, and no blocker anti-patterns were detected.

The only notable finding is that **D-02 was not claimed by any plan** — it is an orphaned requirement in the roadmap. However, it describes a passive policy ("DTOs may add timestamps as Luxon DateTime when needed") rather than an active implementation task. The D-07 rule enforced in both the codebase and CLAUDE.md ensures that when any DTO does add timestamp fields, they will be Luxon DateTime. The requirement is satisfied by convention even without an explicit plan entry.

---

_Verified: 2026-03-30T19:30:00Z_
_Verifier: Claude (gsd-verifier)_
