---
phase: 49
plan: "01"
subsystem: trade-flow-ui
tags: [frontend, rtk-query, revisions, estimates, collapsible, react-router]
requires:
  - Phase 42 D-REV-01 (POST /v1/estimates/:id/revisions — no body, clone-only)
  - Phase 42 D-DET-01 (GET /v1/estimates/:id/revisions returns oldest-first list of IEstimateResponse)
  - Phase 42 response shape (parentEstimateId, rootEstimateId, revisionNumber, isCurrent)
provides:
  - Frontend revision workflow (Edit and resend button) closing REV-02
  - Frontend revision history section closing REV-04
  - useReviseEstimateMutation RTK Query hook
  - useGetEstimateRevisionsQuery RTK Query hook
  - EstimateRevisionHistory reusable component
affects:
  - EstimateActionStrip (new button in action group)
  - EstimateDetailPage (new history card)
  - Estimate type (4 new fields consumed across estimates feature)
tech-stack:
  added: []
  patterns:
    - "RTK Query mutation + query pair mirrors existing estimate mutation pattern (unwrapSingle, tag invalidation)"
    - "Radix Collapsible wrapped in shadcn Card for collapsed-by-default history display"
    - "Conditional render guarded by revisionNumber > 1 to avoid history on root estimates"
    - "Post-mutation navigation uses returned estimate.id (new draft) per Phase 42 revise contract"
key-files:
  created:
    - trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx
    - trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts
    - .planning/phases/49-revision-frontend-ui/49-01-SUMMARY.md
    - .planning/phases/49-revision-frontend-ui/deferred-items.md
  modified:
    - trade-flow-ui/src/types/estimate.ts
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
    - trade-flow-ui/src/features/estimates/components/index.ts
    - trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx
decisions:
  - "Toast imported from @/lib/toast (not sonner directly) to match existing EstimateActionStrip convention (CLAUDE.md conventions)"
  - "Viewed timestamp uses existing Estimate.viewedAt field (frontend mapping of backend firstViewedAt)"
  - "Revision history card placed between lost-reason and line-items cards (plan says after action strip / before line items; chose above line items for visual grouping with other detail cards)"
  - "Test assertion refined from endpoint.initiate return inspection to endpoint surface presence check (initiate returns a thunk, not a plain object)"
  - "SendEstimateForm test mock Estimate updated to include 4 new revision fields to keep the type literal valid"
metrics:
  duration: "~6min"
  completed: 2026-04-16
  tasks: 3
  files: 6
requirements:
  - REV-02
  - REV-04
---

# Phase 49 Plan 01: Revision Frontend UI Summary

**One-liner:** Wire the frontend revision feature — Estimate type gains revisionNumber/isCurrent/parent/root fields, two new RTK Query hooks (revise mutation + revisions query), Edit-and-resend button on sent/viewed/responded/site-visit-requested estimates, and a collapsible History card on revised estimates.

## What Changed

Three atomic tasks executed against the `trade-flow-ui` repository on `main`, closing the two gap-closure requirements REV-02 and REV-04:

1. **Estimate type extended + RTK Query endpoints added (TDD, RED+GREEN).** `src/types/estimate.ts` gains `parentEstimateId: string | null`, `rootEstimateId: string | null`, `revisionNumber: number`, `isCurrent: boolean`. `src/features/estimates/api/estimateApi.ts` gains `reviseEstimate` mutation (POST `/v1/estimates/:id/revisions`, no body, invalidates both the specific estimate and LIST tags) and `getEstimateRevisions` query (GET `/v1/estimates/:id/revisions`, returns `Estimate[]` from `StandardResponse`, provides a unique tag keyed by `${estimateId}-revisions`). Both hooks are exported from the barrel destructure. A new test file `estimateApi.revisions.test.ts` covers hook presence, endpoint registration, and revision-field type shape.

2. **Edit-and-resend button wired into EstimateActionStrip (REV-02).** New `REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"]` constant mirrors the existing `CONVERTIBLE_STATUSES` / `LOSABLE_STATUSES` pattern. `useReviseEstimateMutation` is consumed with `isRevising` loading state. `handleRevise` awaits `.unwrap()`, fires a success toast, and navigates to `/estimates/${result.id}` (the newly returned draft revision). Button renders before Convert-to-Quote with `FilePenLine` icon; swaps to `Loader2` + "Creating revision..." while pending.

3. **EstimateRevisionHistory component built and wired (REV-04).** New component in `src/features/estimates/components/` wraps a Radix `Collapsible` inside a shadcn `Card`. Header shows a `History` icon + "History" title + ghost `ChevronsUpDown` trigger. Content reverses the API's oldest-first response to render newest-first, each revision showing "Revision N" (+ "(current)" in muted style for `isCurrent`), a "Sent {formatDateTime(sentAt)}" line (or "Draft -- not yet sent" when `sentAt` is null), and a "Viewed {formatDateTime(viewedAt)}" line when both `sentAt` and `viewedAt` exist. Loading state renders two animate-pulse skeletons; error state renders "Could not load revision history." Barrel export updated. `EstimateDetailPage` renders the component conditionally on `estimate.revisionNumber > 1`, placed above the line-items card so it sits visually with the other detail cards.

## Task Execution

| Task | Name                                                                                        | Commit    | Files                                                                                                                                                                                                                                                                                                                                |
| ---- | ------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1-RED  | Add failing tests for estimate revision RTK Query endpoints                               | ab0e029   | trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts (created)                                                                                                                                                                                                                                         |
| 1-GREEN | Extend Estimate type and add RTK Query revision endpoints                                | 728071e   | trade-flow-ui/src/types/estimate.ts, trade-flow-ui/src/features/estimates/api/estimateApi.ts, trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts, trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx                                                                |
| 2    | Add "Edit and resend" button to EstimateActionStrip (REV-02)                              | 288f64a   | trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx                                                                                                                                                                                                                                                            |
| 3    | Build EstimateRevisionHistory and wire into EstimateDetailPage (REV-04)                   | 72c72bc   | trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx (created), trade-flow-ui/src/features/estimates/components/index.ts, trade-flow-ui/src/pages/EstimateDetailPage.tsx, trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts (prettier)                                        |

All commits are on the `trade-flow-ui` repo, branch `main`.

## Verification Evidence

- `cd trade-flow-ui && npm run test` — **99 passed (13 files)**. Includes the 5 new tests in `estimateApi.revisions.test.ts`.
- `cd trade-flow-ui && npm run typecheck` — exit 0, zero errors.
- `cd trade-flow-ui && npm run lint` — zero errors (one pre-existing warning in `BusinessStep.tsx` unrelated to this phase).
- `npx prettier --check` on all Phase 49 created/modified files — **All matched files use Prettier code style!**

### Acceptance criteria coverage

- REV-02: `REVISABLE_STATUSES` constant present, `useReviseEstimateMutation` consumed, button renders conditionally on `canRevise`, success path navigates to `/estimates/${result.id}`, failure path emits `toast.error("Failed to create revision. Please try again.")`. Button hidden on Draft, Converted, Lost, Expired, Deleted.
- REV-04: `EstimateRevisionHistory.tsx` created with `useGetEstimateRevisionsQuery`, `Collapsible`/`CollapsibleContent`/`CollapsibleTrigger`, `formatDateTime` formatting, "History" title, "Could not load revision history." error, "Draft -- not yet sent" for unsent drafts, "(current)" badge, `animate-pulse` skeletons. `EstimateDetailPage.tsx` references `EstimateRevisionHistory` and guards with `estimate.revisionNumber > 1`.

## Deviations from Plan

### Plan-directive deviations (all low-risk, documented)

**1. [Rule 3 - Project convention] Toast import source**
- **Found during:** Task 2
- **Issue:** Plan action step 1 specified `import { toast } from "sonner"` but the existing `EstimateActionStrip.tsx` imports from `@/lib/toast` (project-wide wrapper providing typed, iconed toast helpers). Mixing sources would fragment the UX (different icons/styles).
- **Fix:** Kept the existing `@/lib/toast` import. `toast.success`/`toast.error` already work identically.
- **Files modified:** trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
- **Commit:** 288f64a

**2. [Rule 3 - API field naming] Viewed timestamp field**
- **Found during:** Task 3
- **Issue:** Plan action specified reading `revision.firstViewedAt`. The frontend `Estimate` type has no `firstViewedAt` field — it maps the backend `firstViewedAt` to `viewedAt`.
- **Fix:** Use `revision.viewedAt` in `EstimateRevisionHistory.tsx` (grep confirmed zero occurrences of `firstViewedAt` in frontend).
- **Files modified:** trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx
- **Commit:** 72c72bc

**3. [Rule 3 - Test assertion correctness] RTK Query endpoint initiate return shape**
- **Found during:** Task 1 GREEN
- **Issue:** Initial test asserted on `endpoint.initiate(args).endpointName`, but `initiate` returns an async thunk (callable), not a plain action object. Test failed after GREEN because of incorrect assertion target.
- **Fix:** Refined assertions to check `endpoint.initiate` and `endpoint.select` are functions — a stable surface exposed by RTK Query for every registered endpoint.
- **Files modified:** trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts
- **Commit:** 728071e

**4. [Rule 3 - Type safety] SendEstimateForm test mock**
- **Found during:** Task 1 GREEN
- **Issue:** Extending the `Estimate` interface with 4 required revision fields made the existing mock literal in `SendEstimateForm.test.tsx` fail the type contract (even though `tsc -b` excludes tests).
- **Fix:** Added the 4 revision fields to the mock (parentEstimateId: null, rootEstimateId: null, revisionNumber: 1, isCurrent: true) to keep the mock a valid `Estimate`.
- **Files modified:** trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx
- **Commit:** 728071e

### Placement refinement

- **Plan wording:** "Place this AFTER the `EstimateActionStrip` and BEFORE the line items card, at the same level as other detail cards."
- **Chosen placement:** Directly before the line-items card (after the lost-reason card), so the History card visually groups with the other estimate detail cards rather than being sandwiched between the status header and the price card.

## Known Stubs

None. All data fetched live via `useGetEstimateRevisionsQuery` against the shipped backend endpoint. All display fields (`sentAt`, `viewedAt`, `revisionNumber`, `isCurrent`) are real API fields, not hardcoded.

## Deferred Issues

See `.planning/phases/49-revision-frontend-ui/deferred-items.md`:
- 5 pre-existing Prettier format failures in Phase 45 files (PublicEstimate*, toggle-group). Not touched by Phase 49; logged so `/gsd:quick` can restore CI format compliance.
- 1 pre-existing `react-hooks/incompatible-library` warning on `BusinessStep.tsx:51` (RHF `watch()`). Pre-existing, non-blocking warning.

CI format gate is the only quality check that does not currently pass `exit 0` in the repo — purely from pre-existing files unrelated to this plan.

## Threat Flags

None new. All threats enumerated in the plan's `<threat_model>` (T-49-01..04) are mitigated by backend controls (EstimatePolicy ownership via JWT) or frontend loading-state gating (`isRevising` disables the button). No new network surface beyond the two endpoints that shipped in Phase 42.

## Self-Check: PASSED

### Files

- FOUND: trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx
- FOUND: trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts
- FOUND: .planning/phases/49-revision-frontend-ui/deferred-items.md

### Commits

- FOUND: ab0e029 (test RED)
- FOUND: 728071e (feat Task 1 GREEN)
- FOUND: 288f64a (feat Task 2)
- FOUND: 72c72bc (feat Task 3)

## TDD Gate Compliance

Task 1 was executed with `tdd="true"`. The RED→GREEN sequence is observable in git log:

1. `ab0e029 test(49-01): add failing tests for estimate revision RTK Query endpoints` — 4 failing runtime assertions confirmed before GREEN.
2. `728071e feat(49-01): add revision fields to Estimate type and RTK Query endpoints` — moved from 2/5 passing to 5/5 passing.

No REFACTOR commit was needed; the GREEN implementation was already minimal.
