---
phase: 49-revision-frontend-ui
verified: 2026-04-16T00:00:00Z
status: human_needed
score: 5/5
overrides_applied: 0
re_verification: false
human_verification:
  - test: "Edit and resend button triggers navigation to new Draft revision"
    expected: "After clicking the button, a new Draft revision is created via POST /v1/estimates/:id/revisions and the browser navigates to /estimates/{newId}"
    why_human: "RTK Query mutation wiring and post-mutation navigation require a live backend and browser session to confirm the success/error path end-to-end"
  - test: "History section collapses and expands, shows correct revision data"
    expected: "Clicking the ChevronsUpDown trigger reveals the revision list with revision number, sent date, and viewed date; collapsed by default"
    why_human: "Collapsible UI behaviour and rendered revision list content require browser interaction to verify"
  - test: "Edit and resend button hidden on Draft estimate"
    expected: "Draft status shows Send Estimate and Delete buttons; no Edit and resend button visible"
    why_human: "Status-conditional rendering requires visual confirmation in a live session with a draft estimate"
---

# Phase 49: Revision Frontend UI — Verification Report

**Phase Goal:** A trader can trigger "Edit and resend" from the estimate detail page and view the full revision history — completing the frontend half of the revision feature whose backend shipped in Phase 42.
**Verified:** 2026-04-16T00:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Trader sees "Edit and resend" button on Sent estimates | VERIFIED | `REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"]`; `canRevise` guard at line 104 renders button conditionally |
| 2 | Trader does NOT see "Edit and resend" on Draft, Converted, Lost, Expired, or Deleted estimates | VERIFIED | Draft: `isDraft` check shows different buttons only; Converted: early return with `EstimateConvertedLink`; Lost/Expired/Deleted: early `return null` via `isTerminal`/`isLost` check at line 83 |
| 3 | Clicking "Edit and resend" creates a new Draft revision and navigates to its detail page | VERIFIED (wiring confirmed, runtime needs human) | `handleRevise` calls `reviseEstimate({ estimateId }).unwrap()` then `navigate('/estimates/${result.id}')` with `toast.success`/`toast.error` handling |
| 4 | Trader sees a collapsed History section on revised estimates showing revision number, send date, and view date | VERIFIED (wiring confirmed, render needs human) | `EstimateRevisionHistory` renders `Collapsible` (collapsed by default), maps `orderedRevisions` showing `revisionNumber`, `sentAt`/`viewedAt` via `formatDateTime`, `(current)` badge, loading skeleton, error state |
| 5 | History section is hidden on root estimates that have never been revised | VERIFIED | `EstimateDetailPage.tsx:259` guards with `estimate.revisionNumber > 1`; root estimates have `revisionNumber === 1` |

**Score:** 5/5 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/types/estimate.ts` | Revision fields on Estimate type | VERIFIED | Lines 88-91: `parentEstimateId: string \| null`, `rootEstimateId: string \| null`, `revisionNumber: number`, `isCurrent: boolean` — all present, non-optional, correct types |
| `trade-flow-ui/src/features/estimates/api/estimateApi.ts` | reviseEstimate mutation and getEstimateRevisions query | VERIFIED | Lines 189-205: both endpoints defined; lines 221-222: `useReviseEstimateMutation` and `useGetEstimateRevisionsQuery` exported from destructure |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | Edit and resend button with status-conditional visibility | VERIFIED | Line 15: `REVISABLE_STATUSES` constant; line 8: `useReviseEstimateMutation` imported; lines 104-109: conditional button with `FilePenLine` icon, loading state, `handleRevise` handler |
| `trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx` | Collapsible history card with revision list | VERIFIED | File exists, 69 lines; `Collapsible`/`CollapsibleContent`/`CollapsibleTrigger`, `useGetEstimateRevisionsQuery`, `formatDateTime`, "History" title, "(current)", "Draft -- not yet sent", "Could not load revision history.", `animate-pulse` skeleton |
| `trade-flow-ui/src/pages/EstimateDetailPage.tsx` | History section rendered for revised estimates | VERIFIED | Line 38: `EstimateRevisionHistory` imported; line 259: `{estimate.revisionNumber > 1 && <EstimateRevisionHistory estimateId={estimate.id} />}` |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `EstimateActionStrip.tsx` | `POST /v1/estimates/:id/revisions` | `useReviseEstimateMutation` | WIRED | Line 8: `useReviseEstimateMutation` imported; line 44: `const [reviseEstimate, { isLoading: isRevising }] = useReviseEstimateMutation()`; line 71: `reviseEstimate({ estimateId: estimate.id }).unwrap()` |
| `EstimateRevisionHistory.tsx` | `GET /v1/estimates/:id/revisions` | `useGetEstimateRevisionsQuery` | WIRED | Line 5: `useGetEstimateRevisionsQuery` imported; line 13: `const { data: revisions, isLoading, isError } = useGetEstimateRevisionsQuery(estimateId)` |
| `EstimateActionStrip.tsx` | react-router navigate | `navigate` after successful revision | WIRED | Line 2: `useNavigate` imported; line 40: `const navigate = useNavigate()`; line 73: `navigate('/estimates/${result.id}')` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `EstimateRevisionHistory.tsx` | `revisions` | `useGetEstimateRevisionsQuery(estimateId)` → `GET /v1/estimates/:id/revisions` → backend (Phase 42) | Yes — live API query, not hardcoded | FLOWING |
| `EstimateActionStrip.tsx` | `result` (new revision) | `useReviseEstimateMutation` → `POST /v1/estimates/:id/revisions` → backend (Phase 42) | Yes — mutation returns backend response, `result.id` navigated to | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED — frontend React components require browser rendering; no CLI-runnable entry points for UI behavior checks.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|---------|
| REV-02 | 49-01-PLAN.md | User can revise a Sent estimate via "Edit and resend" action | SATISFIED | `REVISABLE_STATUSES`, `useReviseEstimateMutation`, `handleRevise` with navigation, button hidden on non-revisable statuses |
| REV-04 | 49-01-PLAN.md | Estimate detail shows collapsed History section with previous revisions | SATISFIED | `EstimateRevisionHistory` component with `Collapsible`, `useGetEstimateRevisionsQuery`, guarded by `revisionNumber > 1` |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `EstimateActionStrip.tsx` | 15 | `"site_visit_requested"` in `REVISABLE_STATUSES` is not a member of `EstimateStatus` union | Warning | Dead code — the string can never match because `EstimateStatus` does not include it; the `as readonly string[]` cast silences the type-system guard. Does NOT affect correctness of the button hiding requirement. (Review WR-01) |
| `EstimateDetailPage.tsx` | 326 | `estimateId!` non-null assertion passed to `useGetEstimateQuery` | Warning | Silences strict null check; `skip: !estimateId` prevents the query firing but the cache key could become `"undefined"`. Pre-existing pattern, not introduced by Phase 49. (Review WR-03) |
| `EstimateActionStrip.tsx` | 80 | `estimate.convertedToQuoteId!` non-null assertion | Warning | Pre-existing; not introduced by Phase 49. (Review WR-03) |
| `estimateApi.ts` | 189-198 | `reviseEstimate` does not invalidate the `${estimateId}-revisions` cache tag | Info | Latent staleness: after creating a revision, the History card on the source estimate would show stale data if navigated back. Current UX navigates away immediately so bug is dormant. (Review IN-05) |
| `EstimateRevisionHistory.tsx` | 15 | `orderedRevisions` naming is ambiguous (doesn't indicate direction) | Info | Readability only; functionality unaffected. (Review IN-02) |
| `EstimateRevisionHistory.tsx` | 56 | `revision.sentAt && revision.viewedAt` — `sentAt` guard is logically redundant | Info | A revision cannot have `viewedAt` without `sentAt`; extra check is harmless. (Review IN-04) |

No placeholder strings, empty returns, or TODO comments found in Phase 49 files. The two Textarea `placeholder` prop hits in `EstimateDetailPage.tsx` (lines 243, 271) are HTML input placeholder attributes, not stub code.

### Human Verification Required

#### 1. Edit and resend happy path

**Test:** Open the Trade Flow UI in a browser, navigate to a Sent estimate (status "sent"), click the "Edit and resend" button.
**Expected:** Button shows "Creating revision..." with spinner during the API call, then navigates to `/estimates/{newRevisionId}` for the newly created Draft revision, and a "New revision created" success toast appears.
**Why human:** RTK Query mutation wiring and post-mutation navigation require a live backend (Phase 42 endpoint) and browser session.

#### 2. Edit and resend error path

**Test:** With network devtools, block the `POST /v1/estimates/:id/revisions` request, then click "Edit and resend".
**Expected:** Toast shows "Failed to create revision. Please try again." and navigation does not occur. Button re-enables after error.
**Why human:** Error path requires network manipulation in a live browser.

#### 3. History section collapse/expand and data display

**Test:** Navigate to an estimate where `revisionNumber > 1` (a revised estimate). Verify the History card is visible but collapsed. Click the ChevronsUpDown toggle.
**Expected:** Card expands showing a list of revisions, each with "Revision N" heading, "(current)" suffix on the current revision, sent date formatted via `formatDateTime`, and viewed date when available.
**Why human:** Collapsible animation and formatted date rendering require visual inspection.

#### 4. History section absent on root estimates

**Test:** Navigate to a root estimate (one that has never been revised, `revisionNumber === 1`). Verify the History card does not appear between the lost-reason card area and the line items card.
**Expected:** No History section is visible on the page.
**Why human:** Conditional rendering requires visual confirmation with a real estimate record.

#### 5. Button hidden on Draft estimate

**Test:** Navigate to a Draft estimate. Confirm the action strip shows "Send Estimate" and "Delete" buttons only.
**Expected:** No "Edit and resend" button visible on Draft status estimates.
**Why human:** Status-conditional rendering requires visual confirmation in a live session.

### Gaps Summary

No functional gaps found. All five observable truths are verified at the wiring level (artifacts exist, are substantive, are wired, and data flows from live API endpoints). The three human verification items are confirmations of runtime behaviour, not indicators of missing implementation. The code review warnings (WR-01: dead `site_visit_requested` value, WR-03: non-null assertions, IN-05: stale revisions cache tag) are code quality issues, not blockers for goal achievement.

The `site_visit_requested` entry in `REVISABLE_STATUSES` is a dead value (not in `EstimateStatus`) but does not break any success criterion — the button is correctly hidden for all required statuses (Draft, Converted, Lost, Expired, Deleted).

---

_Verified: 2026-04-16T00:00:00Z_
_Verifier: Claude (gsd-verifier)_
