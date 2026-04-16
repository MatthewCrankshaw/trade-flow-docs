# Deferred Items — Phase 48

Out-of-scope issues discovered during plan execution. These were NOT fixed because
they are unrelated to the task at hand. They should be handled in a separate quick task.

## Pre-existing format:check failures in trade-flow-ui (discovered during 48-02)

`npm run ci` in `trade-flow-ui` is currently failing on `format:check` step due to
Prettier formatting drift in 5 files that were last modified by Phase 45 and earlier
commits. None of these files were modified by this plan.

Files with format drift:

- `trade-flow-ui/src/components/ui/toggle-group.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateCard.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
- `trade-flow-ui/src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
- `trade-flow-ui/src/pages/PublicEstimatePage.tsx`

Fix: `cd trade-flow-ui && npx prettier --write <files>` then commit as `chore(lint): apply prettier to Phase 45 files`.

## Pre-existing lint warning in trade-flow-ui (discovered during 48-02)

Non-blocking warning (lint exits 0):

- `trade-flow-ui/src/features/onboarding/components/BusinessStep.tsx:51:25` —
  `react-hooks/incompatible-library`: React Compiler cannot safely memoize values
  returned from `watch()` in react-hook-form.

Fix: review react-hook-form usage here; either migrate to `useWatch()` hook or
suppress with a targeted `eslint-disable-next-line` comment with justification.
(Note: project CLAUDE.md discourages suppression; prefer the `useWatch()` refactor.)
