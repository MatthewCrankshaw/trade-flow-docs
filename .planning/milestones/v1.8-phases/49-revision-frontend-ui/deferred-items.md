# Deferred Items — Phase 49

Out-of-scope issues discovered during Phase 49 execution. NOT fixed here.

## Pre-existing Prettier Format Failures (trade-flow-ui)

Files fail `prettier --check "src/**/*.{ts,tsx}"` but were last modified in Phase 45 (commits `62edbd5`, `eb1b584`, `59e5de6`, `e4cc213`). Not touched by Phase 49.

- `src/components/ui/toggle-group.tsx`
- `src/features/public-estimate/components/PublicEstimateCard.tsx`
- `src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
- `src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
- `src/pages/PublicEstimatePage.tsx`

**Recommended action:** Schedule a quick task to run `npx prettier --write` on these files to restore CI format gate compliance. Currently blocking `npm run ci` exit 0.

## Pre-existing Lint Warning (trade-flow-ui)

- `src/features/onboarding/components/BusinessStep.tsx:51` — `react-hooks/incompatible-library` warning on React Hook Form's `watch()`. Pre-existing. Non-blocking (warning, not error).
