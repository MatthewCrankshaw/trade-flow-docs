---
phase: 48-di-token-fix-cleanup
plan: 01
subsystem: api
tags: [nestjs, dependency-injection, bullmq, estimate-followups, cleanup]

requires:
  - phase: 44-estimate-followup-automation
    provides: BullMQEstimateFollowupCanceller exported from EstimateFollowupsModule
provides:
  - Correct production DI resolution for ESTIMATE_FOLLOWUP_CANCELLER (real BullMQ canceller, not Noop)
  - Removal of dead empty ConvertEstimateRequest class
affects:
  - 46-followup-queue-automation
  - 47-convert-to-quote-mark-as-lost

tech-stack:
  added: []
  patterns:
    - "NestJS local-provider precedence detection: local providers shadow imported module exports"

key-files:
  created: []
  modified:
    - trade-flow-api/src/estimate/estimate.module.ts
    - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
    - trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts
  deleted:
    - trade-flow-api/src/estimate/requests/convert-estimate.request.ts

key-decisions:
  - "Kept NoopEstimateFollowupCanceller service file intact because unit tests depend on it for isolation from BullMQ. Only removed its registration from EstimateModule providers/exports."
  - "Removed the stale `resolves ESTIMATE_FOLLOWUP_CANCELLER to NoopEstimateFollowupCanceller` assertion from estimate-reviser.service.spec.ts; preserved the test-module `useClass: NoopEstimateFollowupCanceller` wiring used for unit-test isolation."

patterns-established:
  - "Module cleanup: when a local provider shadows an imported module export, remove the local provider rather than the import (NestJS resolves local providers first)."

requirements-completed: []

duration: 7min
completed: 2026-04-16
---

# Phase 48 Plan 01: Fix DI Token Resolution & Cleanup Summary

**Removed the local `NoopEstimateFollowupCanceller` provider shadowing the real `BullMQEstimateFollowupCanceller` in EstimateModule, deleted the empty `ConvertEstimateRequest` class, and updated the stale DI resolution test. Production followup cancellation now resolves correctly.**

## Performance

- **Duration:** ~7 min
- **Tasks:** 2
- **Files modified:** 2 (trade-flow-api)
- **Files deleted:** 1 (trade-flow-api)
- **Files incidentally formatted:** 1 (blocking prettier fix)

## Accomplishments

- Removed `NoopEstimateFollowupCanceller` and `{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }` from `EstimateModule.providers`.
- Removed `ESTIMATE_FOLLOWUP_CANCELLER` from `EstimateModule.exports` (the imported `EstimateFollowupsModule` already exports it via the real BullMQ canceller).
- Cleaned up now-unused `ESTIMATE_FOLLOWUP_CANCELLER` / `NoopEstimateFollowupCanceller` imports from `estimate.module.ts`.
- Deleted `trade-flow-api/src/estimate/requests/convert-estimate.request.ts` (empty class with zero references).
- Removed the stale `resolves ESTIMATE_FOLLOWUP_CANCELLER to NoopEstimateFollowupCanceller` assertion from `estimate-reviser.service.spec.ts` while preserving the unit-test isolation `useClass` wiring.
- `npm run ci` in `trade-flow-api` exits 0 — 835/835 tests passing, lint / prettier / typecheck all clean.

## Task Commits

Each task committed atomically in the `trade-flow-api` repo (`main` branch):

1. **Task 1: Fix DI token override and remove duplicate Noop registration** — `13bba57` (fix)
2. **Task 2: Delete ConvertEstimateRequest and update reviser test** — `8dbefa4` (chore)

## Files Created/Modified

- `trade-flow-api/src/estimate/estimate.module.ts` — Removed local `ESTIMATE_FOLLOWUP_CANCELLER` provider and `NoopEstimateFollowupCanceller` registration; removed unused imports. `EstimateFollowupsModule` import preserved.
- `trade-flow-api/src/estimate/requests/convert-estimate.request.ts` — **Deleted** (empty class, zero references).
- `trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts` — Removed stale DI resolution test asserting the old (incorrect) Noop binding as production behavior. Unit-test module setup using Noop for isolation unchanged.
- `trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts` — Pre-existing prettier errors fixed (deviation, see below).

## Decisions Made

- **Kept `NoopEstimateFollowupCanceller` service file.** It is still used by unit tests for isolation from BullMQ. Only its registration in EstimateModule was removed, not the class itself.
- **Removed the stale test rather than updating it.** The removed assertion validated the old (broken) DI resolution. Asserting the new behavior would be redundant — the unit-test module already uses `useClass: NoopEstimateFollowupCanceller` for test isolation, so the service under test never touches BullMQ.
- **Kept the unit-test `useClass` wiring.** Unit tests should not reach out to a real BullMQ queue; the Noop-in-test pattern is correct even after production DI was fixed.

## Deviations from Plan

**[Rule 3 — Blocking] Fixed pre-existing prettier errors in `estimate-to-quote-converter.service.ts`**

- **Found:** during Task 2 while running `npm run ci` to verify the acceptance criterion `npm run ci in trade-flow-api exits 0`.
- **Cause:** Two prettier format errors at lines 123 and 130 introduced in an earlier commit (`ed2206b` — Phase 47 circular DI fix). These errors blocked the plan's own CI acceptance gate.
- **Fix applied:** `npx prettier --write` on the single file (whitespace / line-wrap only, no logic change).
- **Committed together with Task 2** (`8dbefa4`) to keep the atomic deliverable intact.
- **Scope rationale:** Rule 3 of the executor scope-boundary rules permits blocking in-scope CI fixes to be included in the plan; this change was strictly format-only, within the subsystem, and necessary for the plan's acceptance criterion.

## Issues Encountered

**Orchestrator tooling gap (non-blocking):** The executor agent's `Write` and `Bash` tools were denied after the code work completed but before `48-01-SUMMARY.md` could be created in the worktree. The orchestrator recovered by writing this SUMMARY directly in the main repo after verifying all acceptance criteria via spot-checks:

- `grep -c NoopEstimateFollowupCanceller src/estimate/estimate.module.ts` → 0
- `grep -c ESTIMATE_FOLLOWUP_CANCELLER src/estimate/estimate.module.ts` → 0
- `grep -c EstimateFollowupsModule src/estimate/estimate.module.ts` → 2
- `test ! -f src/estimate/requests/convert-estimate.request.ts` → DELETED
- `test -f src/estimate/services/noop-estimate-followup-canceller.service.ts` → PRESERVED
- Commits `13bba57` and `8dbefa4` present on `trade-flow-api` `main`.

## User Setup Required

None — pure backend cleanup. No new env vars, no new external services, no database schema change.

## Next Phase Readiness

- Production follow-up cancellation now correctly resolves to `BullMQEstimateFollowupCanceller`. All callers of `IEstimateFollowupCanceller.cancel(...)` in the estimate revise / convert / mark-as-lost / customer-response paths will now cancel the real BullMQ job instead of silently no-op'ing.
- `EstimateModule` has zero local providers for `ESTIMATE_FOLLOWUP_CANCELLER`; `EstimateFollowupsModule` is the single source of truth.
- Unit-test isolation remains intact — the `useClass: NoopEstimateFollowupCanceller` pattern in `estimate-reviser.service.spec.ts` is documented and correct.
- `trade-flow-api` CI gate is fully green and unblocked for downstream deployment.

## Self-Check: PASSED

Verified claims:

- `trade-flow-api/src/estimate/estimate.module.ts`: no `NoopEstimateFollowupCanceller`, no `ESTIMATE_FOLLOWUP_CANCELLER`, `EstimateFollowupsModule` preserved.
- `trade-flow-api/src/estimate/requests/convert-estimate.request.ts`: DELETED.
- `trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts`: PRESERVED.
- Commits `13bba57` and `8dbefa4` present on `trade-flow-api` `main` branch.
- `npm run ci` in `trade-flow-api`: 835/835 tests passing, lint / prettier / typecheck clean (per executor report).

---
*Phase: 48-di-token-fix-cleanup*
*Completed: 2026-04-16*
