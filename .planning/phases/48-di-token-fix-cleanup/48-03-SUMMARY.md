---
phase: 48-di-token-fix-cleanup
plan: 03
subsystem: ui
tags: [typescript, react, estimates, type-safety, exhaustiveness]

requires:
  - phase: 48-di-token-fix-cleanup
    provides: "Plan 02 completed the site_visit_requested removal across most of the codebase; this plan closes the one remaining gap (SC-4) in EstimateActionStrip.tsx"
provides:
  - "EstimateActionStrip status guards typed as `readonly EstimateStatus[]` — TypeScript now rejects any dead literal added to these tuples"
  - "site_visit_requested literal fully eliminated from trade-flow-ui/src (SC-4 closed)"
  - "Compiler guard restored for this component — the class of bug that slipped past Plan 02 will be caught at compile time going forward"
affects: [estimates, type-safety, ci-gate]

tech-stack:
  added: []
  patterns:
    - "Union-typed readonly tuple pattern: `const X_STATUSES: readonly EstimateStatus[] = [...]` instead of `as const` + widening cast. Preserves readonly immutability while enforcing union membership via tsc."

key-files:
  created: []
  modified:
    - "trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx"

key-decisions:
  - "Use `readonly EstimateStatus[]` explicit typing instead of `as const` + widening cast. Gives the same readonly guarantee while enforcing union exhaustiveness — exactly the compiler guard that was missing during Plan 02."
  - "Leave inline array literal on line 52 (`canResend`) untouched. It is not one of the three named constants flagged by VERIFICATION.md and its three literals are all valid EstimateStatus members."
  - "Leave non-null assertion on line 80 (`estimate.convertedToQuoteId!`) unchanged. Explicitly flagged by VERIFICATION.md as out of scope."

patterns-established:
  - "Union-typed readonly tuple: prefer `const X: readonly Union[] = [...]` over `const X = [...] as const` when you need `.includes()` semantics without widening casts"

requirements-completed: []

duration: 2 min
completed: 2026-04-16
---

# Phase 48 Plan 03: EstimateActionStrip type-safety hardening Summary

**Closed SC-4 gap by typing three status tuples in `EstimateActionStrip.tsx` as `readonly EstimateStatus[]`, removing the stale `site_visit_requested` literal and three `as readonly string[]` widening casts so the TypeScript compiler enforces union exhaustiveness going forward.**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-16T06:46:23Z
- **Completed:** 2026-04-16T06:49:18Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Removed the dead `"site_visit_requested"` literal from `REVISABLE_STATUSES` — SC-4 from 48-VERIFICATION.md is now fully closed; zero occurrences of `site_visit_requested` remain anywhere under `trade-flow-ui/src/`.
- Replaced three `as readonly string[]` widening casts with explicit `readonly EstimateStatus[]` typing, restoring the TypeScript exhaustiveness guard that was silently disabled during Plan 02.
- Extended the type import to bring `EstimateStatus` alongside `Estimate`, keeping the import site minimal and `import type`-correct per UI conventions.

## Task Commits

1. **Task 1: Replace widening casts with `readonly EstimateStatus[]` typing and remove stale `site_visit_requested` literal** — `7803f1b` in trade-flow-ui (fix)

_Note: The code change was committed to the `trade-flow-ui` sibling repository. The plan metadata SUMMARY.md is committed to the `trade-flow-docs` repository via the orchestrator._

## Files Created/Modified

- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — removed dead literal, replaced widening casts with union-typed readonly tuples, extended type import.

## Decisions Made

- **Typing style for readonly tuples:** `const X: readonly EstimateStatus[] = [...]` was chosen over `const X = [...] as const` + runtime cast. The first form gives the same readonly guarantee, keeps `.includes(estimate.status)` type-checking natively (no cast needed), AND makes the compiler reject any member that is not part of the `EstimateStatus` union. The second form disables that check at the call site.
- **Scope discipline:** Inline array on line 52 (`canResend`) was left alone — it is not one of the three named constants flagged by VERIFICATION.md and its literals are all valid `EstimateStatus` members. Non-null assertion on line 80 was left alone per VERIFICATION.md's explicit out-of-scope note.

## Deviations from Plan

None - plan executed exactly as written. All four changes (import, three tuples, three call sites, untouched line 80) match the plan's specification verbatim.

## Issues Encountered

**Pre-existing CI format:check failures (not caused by this plan):**

`npm run ci` in `trade-flow-ui` exits non-zero because of Prettier formatting drift in 5 files last modified by Phase 45 and earlier commits:

- `src/components/ui/toggle-group.tsx`
- `src/features/public-estimate/components/PublicEstimateCard.tsx`
- `src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
- `src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
- `src/pages/PublicEstimatePage.tsx`

These were already tracked in `.planning/phases/48-di-token-fix-cleanup/deferred-items.md` during Plan 48-02. I re-confirmed during this plan that they are unmodified by my change and updated deferred-items.md to record the re-confirmation. Per the executor `SCOPE BOUNDARY` rule, I did NOT fix them — the rule explicitly states pre-existing failures in unrelated files should be logged to `deferred-items.md` and left alone.

**Substantive CI gates on the in-scope file:**

- Vitest: 99 tests pass (0 fail)
- ESLint: 0 errors (1 pre-existing warning in unrelated `BusinessStep.tsx`, already deferred)
- Prettier on `EstimateActionStrip.tsx`: clean
- TypeScript `tsc -b --noEmit`: 0 errors — **the compiler guard is active and working**

The plan's six grep-based acceptance criteria all PASS:

| Criterion | Expected | Actual |
|-----------|----------|--------|
| `grep -c "site_visit_requested"` | 0 | 0 |
| `grep -c "as readonly string\[\]"` | 0 | 0 |
| `grep -c "as const"` | 0 | 0 |
| `grep -c "readonly EstimateStatus\[\]"` | 3 | 3 |
| `grep -c "EstimateStatus"` | ≥4 | 4 |
| Line 80 preserves `convertedToQuoteId!` | yes | yes |

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- SC-4 gap from 48-VERIFICATION.md is closed; re-verification of 48-VERIFICATION.md would now score 4/4.
- TypeScript exhaustiveness guard is restored for `EstimateActionStrip.tsx` — any future dead literal added to `CONVERTIBLE_STATUSES`, `LOSABLE_STATUSES`, or `REVISABLE_STATUSES` will fail `tsc` and block CI.
- Phase 48 gap-closure is functionally complete; only the cross-cutting pre-existing prettier drift in 5 unrelated files blocks a clean `npm run ci`. That drift is tracked in `deferred-items.md` and should be resolved by a separate quick task (`cd trade-flow-ui && npx prettier --write <files>`).

## Self-Check: PASSED

- File `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` exists and contains the expected changes (verified via Read).
- Commit `7803f1b` exists in the `trade-flow-ui` repository (verified via `git log --oneline -1`).
- All six grep-based acceptance criteria re-run and PASS.
- Vitest, ESLint (errors), Prettier (on modified file), and TypeScript all pass on the in-scope change.
- Pre-existing out-of-scope failures are documented in `deferred-items.md` per SCOPE BOUNDARY rule.

---
*Phase: 48-di-token-fix-cleanup*
*Completed: 2026-04-16*
