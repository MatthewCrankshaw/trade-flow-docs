---
phase: 50-response-display-convert-route-fix
plan: 01
subsystem: api
tags: [estimate, responseSummary, dto, repository, typescript]

requires:
  - phase: 41-estimate-module-crud
    provides: IEstimateResponseSummaryDto interface and responses[] array on estimates
  - phase: 48-estimate-ui-cleanup
    provides: site_visit_requested removal from frontend types/components
  - phase: 49-revision-types
    provides: EstimateResponseSummary type realignment (pre-satisfied Task 2)
provides:
  - Backend responseSummary derivation from last element of responses[] in EstimateRepository.toDto()
  - Unit test coverage for empty/single-decline/multi-response/proceed cases
  - Confirmed frontend EstimateResponseSummary shape matches API response contract
affects:
  - 50-02-plan (UI card rendering of response data)
  - Phase 45 public response handling (consumes same derivation path)

tech-stack:
  added: []
  patterns:
    - "Derive summary DTOs from source array in repository toDto() rather than hardcoding null"

key-files:
  created: []
  modified:
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts

key-decisions:
  - "Derived responseSummary inline in toDto() using the already-mapped responses[] array (reusing the ISO-stringified respondedAt) instead of duplicating date conversion logic"
  - "Imported IEstimateResponseSummaryDto from its dedicated DTO file rather than a barrel to keep the path explicit and grep-friendly"
  - "Task 2 confirmed already-satisfied by Phase 48-02 and Phase 49-01 commits; no duplicate edits made"

patterns-established:
  - "Pattern: repository toDto() derives denormalised summary fields from source arrays at read time, leaving entity storage unchanged"

requirements-completed: [RESP-08]

duration: 6 min
completed: 2026-04-16
---

# Phase 50 Plan 01: Response Display Backend Fix + Type Alignment Summary

**Backend EstimateRepository.toDto() now derives responseSummary from the last entry in responses[] (previously hardcoded null), with four new unit tests covering empty/single/multi/proceed cases. Frontend type alignment pre-satisfied by earlier phases.**

## Performance

- **Duration:** 6 min
- **Started:** 2026-04-16T06:11:55Z
- **Completed:** 2026-04-16T06:17:59Z
- **Tasks:** 2 (1 TDD, 1 verify-only)
- **Files modified:** 2 (API) + 0 (UI — already at target state)

## Accomplishments

- Fixed the core defect: `EstimateRepository.toDto()` now reads the last element of `responses[]` and populates `IEstimateResponseSummaryDto` instead of returning `responseSummary: null` for every estimate
- Added `describe("responseSummary derivation")` with four tests to `estimate.repository.spec.ts` covering: empty responses → null; single DECLINE with message + reason; multi-response last-wins; PROCEED with null message/reason
- Verified frontend `EstimateResponseSummary` shape (`lastResponseType`, `lastResponseAt`, `lastResponseMessage`, `declineReason` all `string | null`) already matches the API contract
- Verified `site_visit_requested` is absent from committed UI HEAD (types, EstimateDetailPage, EstimateActionStrip)
- Both `npx jest --testPathPatterns=estimate.repository.spec` and `npx tsc --noEmit` exit 0

## Task Commits

Each task was committed atomically in the `trade-flow-api` sub-repo (UI sub-repo required no commits — Task 2 was pre-satisfied):

1. **Task 1 RED** — `6329510` (test)
   `test(50-01): add failing tests for responseSummary derivation`
2. **Task 1 GREEN** — `aeae804` (feat)
   `feat(50-01): derive responseSummary from last entry in responses[]`
3. **Task 2** — No commit needed. Target state already committed in:
   - `trade-flow-ui` `8223315` `chore(48-02): remove dead site_visit_requested status from estimate UI`
   - `trade-flow-ui` `728071e` `feat(49-01): add revision fields to Estimate type and RTK Query endpoints` (also realigned `EstimateResponseSummary` field names and nullability to match the API response shape)

_Note: REFACTOR step of TDD was skipped because the minimal GREEN implementation was already clean (no duplication, clear naming)._

## Files Created/Modified

- `trade-flow-api/src/estimate/repositories/estimate.repository.ts` — Added `IEstimateResponseSummaryDto` import; compute `lastResponse` from `responses[]` and build summary DTO; replaced hardcoded `responseSummary: null` in `toDto()` return.
- `trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` — Added nested `describe("responseSummary derivation")` inside `toEntity / toDto round-trip` with four `it(...)` tests.
- `.planning/phases/50-response-display-convert-route-fix/deferred-items.md` — Created to log pre-existing out-of-scope UI CI issues that are not fixed by this plan.

## Decisions Made

- **Reuse mapped `responses` for summary derivation.** The existing `responses` local variable already performs `respondedAt: r.respondedAt.toISOString()`. Deriving `responseSummary` from this local (rather than from `entity.responses`) avoids a second date-to-ISO conversion and keeps the source of truth aligned.
- **Explicit DTO import path.** Used `@estimate/data-transfer-objects/estimate-response-summary.dto` directly rather than a barrel re-export, matching existing imports in the file (e.g. `estimate.dto`, `estimate-line-item.dto`).
- **Do not duplicate Task 2 edits already committed by earlier phases.** Upon reading `trade-flow-ui/src/types/estimate.ts` and other Task 2 targets at `main` HEAD, all target-state assertions from the plan were already true (Phase 48-02 removed `site_visit_requested`; Phase 49-01 realigned `EstimateResponseSummary`). No duplicate commits were made.

## Deviations from Plan

### Rule 1 / Scope — Task 2 pre-satisfied by earlier phases

**1. [Rule 1-adjacent — No change needed] Frontend type alignment and status removal already committed**

- **Found during:** Task 2 initial read
- **Issue:** `trade-flow-ui` `main` HEAD already contained the exact target state prescribed by Task 2. `EstimateResponseSummary` had already been realigned to `lastResponseType`/`lastResponseAt`/`lastResponseMessage`/`declineReason` (all `string | null`) in commit `728071e` (Phase 49-01). `site_visit_requested` had already been removed from `EstimateStatus`, `statusColors`, and `canResend` in commit `8223315` (Phase 48-02).
- **Fix:** Verified acceptance criteria against committed HEAD via grep, confirmed all 8 grep-based criteria pass and `npx tsc --noEmit` exits 0. Made no duplicate edits.
- **Files modified:** None.
- **Verification:** Grep output confirms 1 match for each of `lastResponseType: string | null;`, `lastResponseAt: string | null;`, `lastResponseMessage: string | null;` and 0 matches for `  responseType: string`, `  respondedAt: string`, `site_visit_requested` across all three target files.
- **Commit:** N/A — no code change.

### Scope boundary — Pre-existing UI CI failures (deferred, not fixed)

**2. [Scope boundary] `npm run ci` in trade-flow-ui fails at `format:check` on 5 unrelated files**

- **Found during:** Final verification (plan `<verification>` step 4)
- **Issue:** `prettier --check` reports formatting issues in `src/components/ui/toggle-group.tsx`, `src/features/public-estimate/components/PublicEstimateCard.tsx`, `.../PublicEstimateDeclineForm.tsx`, `.../PublicEstimateResponseButtons.tsx`, and `src/pages/PublicEstimatePage.tsx`. Also 1 lint error (unused `EstimateRevisionHistory` import in `EstimateDetailPage.tsx`) and 1 lint warning (`react-hooks/incompatible-library` in `BusinessStep.tsx`).
- **Fix:** Not applied. None of these files are modified by plan 50-01; they were last touched in Phase 45 commits. Per deviation scope rule ("Only auto-fix issues DIRECTLY caused by the current task's changes"), these are out of scope.
- **Files modified:** `.planning/phases/50-response-display-convert-route-fix/deferred-items.md` (new) logs the discovery for a future housekeeping pass.
- **Verification:** `git log --oneline -3` on the affected files confirms last modification dates pre-date this plan.
- **Commit:** N/A — deferred.

### Uncommitted-by-others working tree state

**3. [Scope boundary / destructive-git prohibition] Pre-existing uncommitted edit in EstimateActionStrip.tsx left untouched**

- **Found during:** Task 2 commit-staging
- **Issue:** `trade-flow-ui` working tree had an uncommitted modification to `src/features/estimates/components/EstimateActionStrip.tsx` introducing a `REVISABLE_STATUSES` constant that includes `"site_visit_requested"`. This modification predated this plan (the UI sub-repo is shared across worktrees).
- **Fix:** Not applied. The prohibition on destructive git operations inside a worktree and scope-boundary rules require leaving pre-existing uncommitted work alone. The committed HEAD is clean (no `site_visit_requested`), which is what Task 2 acceptance criteria apply to.
- **Files modified:** None.
- **Verification:** `git show HEAD:src/features/estimates/components/EstimateActionStrip.tsx | grep site_visit` returns no matches at committed HEAD.
- **Commit:** N/A — not our work to commit or revert.

---

**Total deviations:** 3 (0 auto-fixed, 2 deferred as out-of-scope, 1 no-op because already-satisfied)
**Impact on plan:** No change to plan scope. The backend fix (the actual defect in RESP-08) was delivered with TDD coverage; frontend target state was confirmed already correct; pre-existing UI CI issues are logged in `deferred-items.md` for a separate housekeeping pass.

## Issues Encountered

None beyond the deviations documented above. The `--testPathPattern` Jest flag has been renamed to `--testPathPatterns` in Jest 30; the plan's verification command was updated accordingly on the fly.

## User Setup Required

None — no external service configuration.

## Next Phase Readiness

- Backend now emits `responseSummary` populated on every estimate that has at least one customer response — ready for plan 50-02 to render the UI card using these fields.
- Frontend `EstimateResponseSummary` type already aligned — no follow-up rename needed in plan 50-02.
- Pre-existing UI format/lint issues logged to `deferred-items.md` are independent of plan 50-02 readiness.

## TDD Gate Compliance

Task 1 is `tdd="true"` and followed the RED-GREEN-REFACTOR cycle:

- **RED gate:** `6329510` (`test(50-01): ...`) — 3 of 4 new tests fail (confirmed with `npx jest` showing `Tests: 3 failed, 38 skipped, 1 passed`). The single passing test was the "empty responses → null" case, which coincidentally matched the hardcoded null pre-fix; this is acceptable because the other three tests clearly demonstrate the missing behavior.
- **GREEN gate:** `aeae804` (`feat(50-01): ...`) — all 4 tests pass (`Tests: 42 passed, 42 total` on full `estimate.repository.spec`).
- **REFACTOR gate:** Skipped — the minimal GREEN implementation is already clean (no duplication, descriptive naming, types satisfied without casts).

## Self-Check: PASSED

- `trade-flow-api/src/estimate/repositories/estimate.repository.ts` — FOUND, grep criteria 1-6 pass
- `trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts` — FOUND, grep criteria 7-8 pass (1 `describe("responseSummary derivation"`, 4 `it(` blocks inside)
- `trade-flow-ui/src/types/estimate.ts` — FOUND, target state matches (1 each for `lastResponseType`, `lastResponseAt`, `lastResponseMessage` with `string | null`; 0 matches for old field names)
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` — FOUND, 0 matches for `site_visit_requested`
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — FOUND, 0 matches for `site_visit_requested` at committed HEAD
- Commits exist:
  - `git log --oneline --all | grep 6329510` → FOUND (trade-flow-api)
  - `git log --oneline --all | grep aeae804` → FOUND (trade-flow-api)
- Plan verification commands:
  - `npx jest --testPathPatterns=estimate.repository.spec` → exit 0, 42/42 tests pass
  - `npx tsc --noEmit` (trade-flow-ui) → exit 0
  - `npm run ci` (trade-flow-api) → exit 0 (33 pre-existing warnings, 0 errors)
  - `npm run ci` (trade-flow-ui) → exit 1 at `format:check` on 5 unrelated files (deferred per scope boundary; documented in `deferred-items.md`)

---
*Phase: 50-response-display-convert-route-fix*
*Completed: 2026-04-16*
