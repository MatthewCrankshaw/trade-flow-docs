# Deferred Items — Phase 50

Out-of-scope discoveries logged during execution of 50-01-PLAN.md. These are pre-existing issues NOT caused by plan 50-01 and are not fixed here per deviation scope rules.

## Pre-existing UI `format:check` failures

Discovered while running `npm run ci` in `trade-flow-ui` as part of plan 50-01 verification. `prettier --check` reports formatting issues in 5 files none of which are touched by this plan:

- `src/components/ui/toggle-group.tsx`
- `src/features/public-estimate/components/PublicEstimateCard.tsx`
- `src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
- `src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
- `src/pages/PublicEstimatePage.tsx`

Last modified by Phase 45 commits (`4bd01a0`, `eb1b584`), long before Phase 50. The per-file typecheck (`npx tsc --noEmit`) required by this plan passes. The `format:check` failure is an independent project-wide issue that a later housekeeping pass should address.

## Pre-existing UI lint warning/error

From the same `npm run ci` run in `trade-flow-ui`:

- `src/features/onboarding/components/BusinessStep.tsx:51` — `react-hooks/incompatible-library` warning (React Compiler cannot memoize react-hook-form's `watch()`). Pre-existing.
- `src/pages/EstimateDetailPage.tsx:37` — `'EstimateRevisionHistory' is defined but never used` error. Introduced in a commit outside this plan; EstimateDetailPage is not modified by this plan.

## Pending uncommitted UI modification (not mine)

`src/features/estimates/components/EstimateActionStrip.tsx` has an uncommitted working-tree modification (introducing `REVISABLE_STATUSES` which includes `"site_visit_requested"`). This modification was present when this plan started and was not authored as part of plan 50-01. Left untouched per destructive-git prohibition and scope boundary.

## Plan 50-02 follow-up: same pre-existing format:check failures

Plan 50-02 ran `npm run ci` in `trade-flow-ui` as part of Task 2 verification and hit the same 5 format:check failures as plan 50-01 (same files, same pre-Phase-50 commits). All files changed by plan 50-02 (`EstimateResponseCard.tsx`, `index.ts`, `EstimateDetailPage.tsx`, `App.tsx`) pass `prettier --check` cleanly in isolation. The blocker for a green `npm run ci` is still the 5 unrelated Phase 45 files listed above; a single `prettier --write` pass on those files will clear both plans' CI gates.

Lint and typecheck continue to report 0 errors; the 1 lint warning (`BusinessStep.tsx:51`) is also pre-existing and orthogonal.
