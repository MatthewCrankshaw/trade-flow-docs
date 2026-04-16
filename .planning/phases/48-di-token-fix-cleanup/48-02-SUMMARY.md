---
phase: 48-di-token-fix-cleanup
plan: 02
subsystem: ui
tags: [react, typescript, estimate, type-union, dead-code]

requires:
  - phase: 45-public-customer-page-response-handling
    provides: backend removal of site_visit_requested status enum
provides:
  - Clean EstimateStatus type union in UI matching backend enum exactly
  - TypeScript exhaustiveness now enforces that all future estimate status values stay in sync
affects:
  - 46-followup-queue-automation
  - 47-convert-to-quote-mark-as-lost

tech-stack:
  added: []
  patterns:
    - "Type union narrowing as dead-branch detection"

key-files:
  created: []
  modified:
    - trade-flow-ui/src/types/estimate.ts
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx
    - trade-flow-ui/src/pages/EstimatesPage.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx
    - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx

key-decisions:
  - "Removed the dead status key from all Record<EstimateStatus, ...> badge maps rather than leaving defensive no-op entries; TypeScript's exhaustiveness check on Record now guarantees parity with the narrowed union."

patterns-established:
  - "Dead-status cleanup: narrow the source type first, then let tsc surface every stale reference"

requirements-completed: []

duration: 3min
completed: 2026-04-16
---

# Phase 48 Plan 02: Remove site_visit_requested from Estimate UI Summary

**Dropped the dead `site_visit_requested` status from the EstimateStatus type union and purged all 6 frontend references so the UI type exactly matches the backend enum.**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-16T06:07:52Z
- **Completed:** 2026-04-16T06:11:12Z
- **Tasks:** 1
- **Files modified:** 6 (trade-flow-ui) + 2 docs (worktree)

## Accomplishments

- Removed `"site_visit_requested"` from the `EstimateStatus` type union in `trade-flow-ui/src/types/estimate.ts`.
- Removed the corresponding key from three `Record<EstimateStatus, ...>` status-to-badge-variant maps (EstimateDetailPage, EstimatesCardList, EstimatesTable).
- Removed the stale entries from both filter arrays in `TAB_STATUS_GROUPS` (`all` and `responded`) in EstimatesPage.
- Removed the stale entry from the `canResend` status list in EstimateActionStrip.
- Verified zero remaining references via `grep -r "site_visit_requested" src/` (0 matches).
- Vitest: 94/94 passing. TypeScript: 0 errors. ESLint: 0 errors.

## Task Commits

Each task committed atomically in the `trade-flow-ui` repo (`main` branch):

1. **Task 1: Remove site_visit_requested from EstimateStatus type and all component references** — `8223315` (chore)

## Files Created/Modified

- `trade-flow-ui/src/types/estimate.ts` — Narrowed `EstimateStatus` union by removing `"site_visit_requested"`.
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` — Removed `site_visit_requested: "default"` key from `statusColors` Record.
- `trade-flow-ui/src/pages/EstimatesPage.tsx` — Removed `"site_visit_requested"` from `TAB_STATUS_GROUPS.all` and `TAB_STATUS_GROUPS.responded`.
- `trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx` — Removed key from `statusColors` Record.
- `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` — Removed key from `statusColors` Record.
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — Removed `"site_visit_requested"` from the `canResend` literal array.
- `.planning/phases/48-di-token-fix-cleanup/deferred-items.md` — Created to document pre-existing trade-flow-ui CI drift that is out of scope for this plan.
- `.planning/phases/48-di-token-fix-cleanup/48-02-SUMMARY.md` — This file.

## Decisions Made

- When pruning the Record maps, I removed the keys entirely instead of mapping them to a default value. Keeping stale keys would mean TypeScript no longer checks the Record against the narrowed union, which is the whole point of this cleanup.

## Deviations from Plan

None for in-scope work — the six edits match the plan exactly, and TypeScript compilation, unit tests, and lint all confirm zero dangling references.

Scope-boundary call-out: the plan's acceptance criterion of "`npm run ci` in trade-flow-ui exits 0" is currently blocked by **pre-existing** formatting drift in 5 unrelated files last touched by Phase 45 work. Those files are NOT modified by this plan; auto-fixing them would silently reformat unrelated code paths. Per the agent scope-boundary rule, those failures were captured in `deferred-items.md` rather than fixed here. See the "CI Gate Caveat" section below.

## Issues Encountered

**CI Gate Caveat**

`npm run ci` in `trade-flow-ui` currently fails on the `format:check` step. All failures are in files unrelated to this plan and pre-date it:

- `trade-flow-ui/src/components/ui/toggle-group.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateCard.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
- `trade-flow-ui/src/pages/PublicEstimatePage.tsx`

The 6 files this plan modified were confirmed clean by `npx prettier --check` (`All matched files use Prettier code style!`).

Applicable CI checks that DO pass against this plan's changes:

- Unit tests: 94/94 passing (Vitest)
- TypeScript: 0 errors (`tsc -b --noEmit` clean)
- ESLint: 0 errors (1 pre-existing warning in `BusinessStep.tsx`, non-blocking)
- Prettier on modified files only: clean

A separate quick task is needed to `npx prettier --write` the 5 drifted files and clear the CI gate for the milestone. This is logged in `.planning/phases/48-di-token-fix-cleanup/deferred-items.md`.

## User Setup Required

None — pure code cleanup, no external service or env configuration.

## Next Phase Readiness

- The `EstimateStatus` type union is now the single source of truth and matches the backend enum exactly.
- TypeScript's exhaustiveness check on `Record<EstimateStatus, ...>` ensures any future additions/removals to the backend enum will immediately surface as build errors across these 6 files.
- Phase 48 Plan 01 (API-side, separate worktree) proceeds independently; neither plan shares files with the other.
- Before the milestone can deploy, the pre-existing format drift in `trade-flow-ui` (see deferred-items.md) must be resolved so `npm run ci` can exit 0 again.

## Self-Check: PASSED

Verified claims:

- `trade-flow-ui/src/types/estimate.ts`: FOUND, clean of "site_visit_requested".
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx`: FOUND, clean.
- `trade-flow-ui/src/pages/EstimatesPage.tsx`: FOUND, clean.
- `trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx`: FOUND, clean.
- `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx`: FOUND, clean.
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx`: FOUND, clean.
- Commit `8223315` on `trade-flow-ui` `main`: FOUND.
- `grep -r "site_visit_requested" trade-flow-ui/src/` returns zero matches.

---
*Phase: 48-di-token-fix-cleanup*
*Completed: 2026-04-16*
