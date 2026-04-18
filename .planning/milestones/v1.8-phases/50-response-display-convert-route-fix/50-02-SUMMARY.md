---
phase: 50-response-display-convert-route-fix
plan: 02
subsystem: ui
tags: [estimate, response-display, routing, react, component]

requires:
  - phase: 50-01
    provides: Backend responseSummary derivation on estimate responses (IEstimateResponseSummaryDto populated when responses[] is non-empty) + frontend EstimateResponseSummary type alignment
  - phase: 45-research
    provides: UI-SPEC for response card (response type labels, icons, colors, decline reason map)
  - phase: 47-research
    provides: D-CONV-02/D-CONV-03 — convert-to-quote flow requires /quotes/:quoteId/edit route
provides:
  - EstimateResponseCard component rendering lastResponseType, timestamp, decline reason, message
  - /quotes/:quoteId/edit route in App.tsx (reuses QuoteDetailPage)
  - Real response card in EstimateDetailPage replacing the Phase 45 placeholder
affects:
  - Phase 45 public response handling (convert navigation now lands on a valid route)
  - Phase 47 convert-to-quote flow (target route exists)

tech-stack:
  added: []
  patterns:
    - "Label/icon/color lookup tables keyed by API enum strings with `?? fallback` for unknown values (mirrors EstimateLostReasonCard)"
    - "Reuse single page component for multiple route variants (`/quotes/:quoteId` and `/quotes/:quoteId/edit` both render QuoteDetailPage)"

key-files:
  created:
    - trade-flow-ui/src/features/estimates/components/EstimateResponseCard.tsx
  modified:
    - trade-flow-ui/src/features/estimates/components/index.ts
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx
    - trade-flow-ui/src/App.tsx
    - .planning/phases/50-response-display-convert-route-fix/deferred-items.md

key-decisions:
  - "Typed the icon map as Record<string, ElementType> using a named type import from react rather than React.ElementType — matches CLAUDE.md 'always use import type for type imports' convention"
  - "Imported EstimateResponseSummary from @/types barrel (not @/types/estimate) to match the surrounding EstimateDetailPage import style and avoid deep paths"
  - "Added /quotes/:quoteId/edit as a separate sibling Route (not a nested child of /quotes/:quoteId) so React Router matches it literally and renders QuoteDetailPage cleanly — matches plan D-CONV-03"
  - "Capitalised the card title to 'Customer Response' per UI-SPEC (§Copywriting Contract). The placeholder used lowercase 'Customer response'; the plan explicitly specifies the capitalised form"

patterns-established:
  - "Estimate summary-state cards (LostReasonCard, ResponseCard) share the same structure: top-of-file lookup tables, single props interface with a pre-typed summary object, early-return when discriminator is null, `?? key` fallback for unknown enum values"

requirements-completed: [RESP-08, CONV-03]

duration: 3 min
completed: 2026-04-16
---

# Phase 50 Plan 02: Response Display Card + Convert-Route Fix Summary

**Replaced the Phase 45 placeholder on EstimateDetailPage with a real `EstimateResponseCard` component that renders proceed/message/decline responses with icon, label, timestamp, decline reason, and customer message; added the missing `/quotes/:quoteId/edit` route so the convert-to-quote flow in EstimateActionStrip navigates to a valid destination.**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-16T06:26:10Z
- **Completed:** 2026-04-16T06:29:48Z
- **Tasks:** 2 (both `type="auto"`, non-TDD)
- **Files created:** 1 (UI)
- **Files modified:** 3 (UI) + 1 (planning artifact)

## Accomplishments

- Created `EstimateResponseCard` with a three-way response-type branch (proceed / message / decline) wired to lookup tables for display label, lucide-react icon, and semantic text colour (`text-green-600 dark:text-green-400` / `text-amber-600 dark:text-amber-400` / `text-destructive`).
- Added decline reason rendering with the `DECLINE_REASON_LABELS` map shared in form with Phase 45's MarkAsLostDialog and plan 50-02 UI-SPEC §Decline Reason Label Map.
- Integrated the card into `EstimateDetailPage` in place of the "Response handling ships in Phase 45" placeholder; empty state copy is now "No customer response yet." (no phase reference).
- Added `<Route path="/quotes/:quoteId/edit" element={<QuoteDetailPage />} />` as a sibling of the existing `/quotes/:quoteId` route, inside the same `AuthenticatedLayout > OnboardingGuard > PaywallGuard` hierarchy.
- `npx tsc --noEmit` (frontend) exits 0; `vitest run` reports 99/99 tests passing; `prettier --check` on all 4 files I touched exits 0.
- `npm run ci` in `trade-flow-api` exits 0 (0 errors, 33 pre-existing warnings).

## Task Commits

Each task was committed atomically in the `trade-flow-ui` sub-repo (no API changes required for this plan):

1. **Task 1** — `a91190c` (feat)
   `feat(50-02): render customer response card on estimate detail`
2. **Task 2** — `8ce2abb` (feat)
   `feat(50-02): add /quotes/:quoteId/edit route for convert flow`

## Files Created/Modified

- `trade-flow-ui/src/features/estimates/components/EstimateResponseCard.tsx` (new, 64 lines) — Named-export React function component. Props: `{ responseSummary: EstimateResponseSummary }`. Top-of-file declares `RESPONSE_TYPE_LABELS`, `RESPONSE_TYPE_ICONS`, `RESPONSE_TYPE_COLORS`, and `DECLINE_REASON_LABELS` Records. Early-returns `null` when `lastResponseType` is null, renders icon + label + timestamp + optional reason + optional message using the design-system tokens from UI-SPEC §Layout Details.
- `trade-flow-ui/src/features/estimates/components/index.ts` — Added `export * from "./EstimateResponseCard";` between `EstimateLostReasonCard` and `EstimateConvertedLink` (preserves alphabetical-ish grouping).
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` — Added `EstimateResponseCard` to the existing `@/features/estimates/components` barrel import; replaced the 9-line placeholder Card (title "Customer response", body "Response handling ships in Phase 45") with the new conditional render (title "Customer Response", body uses `EstimateResponseCard` when `estimate.responseSummary` is present, otherwise shows "No customer response yet.").
- `trade-flow-ui/src/App.tsx` — Inserted `<Route path="/quotes/:quoteId/edit" element={<QuoteDetailPage />} />` immediately after the existing `/quotes/:quoteId` route, inside the `PaywallGuard` > `OnboardingGuard` > `AuthenticatedLayout` block.
- `.planning/phases/50-response-display-convert-route-fix/deferred-items.md` — Appended a short follow-up note documenting that plan 50-02 hit the same pre-existing format:check failures as plan 50-01 (same 5 files, all last touched in Phase 45). My 4 files pass `prettier --check` independently.

## Decisions Made

- **`import type { ElementType } from "react"` over `React.ElementType`.** CLAUDE.md mandates `import type` for all type imports; using a named type import also avoids introducing a namespace import of the `React` object (unused here since JSX auto-imports the runtime).
- **Import `EstimateResponseSummary` from `@/types`, not `@/types/estimate`.** Matches the style already in `EstimateDetailPage.tsx` (`import type { Estimate, EstimateStatus, ... } from "@/types"`) and keeps the component insulated from internal module structure.
- **Empty-state copy changed from "No customer response yet. Response handling ships in Phase 45." to "No customer response yet."** Plan explicitly forbids the "Phase 45" reference and UI-SPEC §Copywriting Contract gives the exact new copy.
- **Card title case-correction from "Customer response" to "Customer Response".** UI-SPEC lists "Customer Response" (both words capitalised). The plan replacement block uses the capitalised form. Matches the visual rhythm of other card titles on this page (e.g. "Internal Notes", "Price", "Uncertainty").
- **Declared `/quotes/:quoteId/edit` as a flat sibling, not a nested child.** The existing quote routes are flat under the PaywallGuard block, not nested. Adding the edit route as a sibling keeps the routing table uniform and matches D-CONV-03 in the plan interfaces.
- **Did not remove the stale `site_visit_requested` literal in `EstimateActionStrip`'s `REVISABLE_STATUSES`.** Out of scope — that file is not in this plan's `files_modified` list, the literal is dead-code-safe (no EstimateStatus value can equal it), and the destructive-git prohibition in my parallel-executor context forbids touching files I didn't modify. Logged implicitly in `deferred-items.md` via plan 50-01's earlier entry.

## Deviations from Plan

### Scope boundary — pre-existing trade-flow-ui `npm run ci` format:check failures still blocking a green UI CI gate

**1. [Scope boundary — deferred] `npm run ci` in trade-flow-ui exits 1 at `format:check` on 5 files untouched by this plan**

- **Found during:** Task 2 verification (`<automated>` was `cd trade-flow-ui && npm run ci`).
- **Issue:** `prettier --check "src/**/*.{ts,tsx}"` reports formatting drift in exactly the same 5 files that plan 50-01 already documented:
  - `src/components/ui/toggle-group.tsx`
  - `src/features/public-estimate/components/PublicEstimateCard.tsx`
  - `src/features/public-estimate/components/PublicEstimateDeclineForm.tsx`
  - `src/features/public-estimate/components/PublicEstimateResponseButtons.tsx`
  - `src/pages/PublicEstimatePage.tsx`
- **Fix:** Not applied. Per deviation scope rule ("Only auto-fix issues DIRECTLY caused by the current task's changes") and the destructive-git prohibition, I cannot touch files outside my plan's `files_modified` list. All 4 files I did modify pass `prettier --check` cleanly in isolation (`npx prettier --check EstimateResponseCard.tsx index.ts EstimateDetailPage.tsx App.tsx` → "All matched files use Prettier code style!"). The 5 failing files were last modified in Phase 45 commits (`4bd01a0`, `eb1b584`) per `git log` in the UI repo — all pre-dating Phase 50.
- **Files modified:** `.planning/phases/50-response-display-convert-route-fix/deferred-items.md` (follow-up note appended).
- **Verification:** `tests` step passes (99/99), `lint` step reports 0 errors (1 pre-existing warning in `BusinessStep.tsx`), `format:check` fails on the 5 pre-existing files, `typecheck` passes. A single `npx prettier --write` on the 5 files would green the CI gate and should be done as a housekeeping quick task.
- **Commit:** N/A for the UI repo (would be out-of-scope edits); deferred-items.md update rides with the final planning metadata commit.

**Total deviations:** 1 (0 auto-fixed, 1 deferred as out-of-scope).
**Impact on plan:** Neither acceptance criteria of Task 2 ("`npm run ci` exits 0") nor the phase success criteria ("Both repos pass full CI gate") can be claimed strictly green, but the UI CI failure is the same pre-existing issue already deferred by plan 50-01 and is orthogonal to plan 50-02's scope. The code additions of plan 50-02 are themselves format/lint/typecheck/test-clean.

## Issues Encountered

- Initial `npm run ci | tail -60; echo "CI_EXIT=$?"` reported `CI_EXIT=0` because `$?` captured `tail`'s exit code, not `npm run ci`'s. Re-ran with a cleaner capture to get the true exit code (`1`), which confirmed the pre-existing format:check failure. No impact on code changes.

## User Setup Required

None — no external service configuration, no env vars, no migrations.

## Next Phase Readiness

- Phase 45 (Public Customer Page & Response Handling) can now rely on EstimateDetailPage showing real response data end-to-end: backend populates `responseSummary` (plan 50-01), the frontend type matches (pre-satisfied), and the card renders it (plan 50-02).
- Phase 47 (Convert to Quote & Mark as Lost) already had `EstimateActionStrip` issuing `navigate(\`/quotes/\${result.quoteId}/edit\`)` after a successful convert, which would have crashed or fallen through to the `*` redirect. With the route now registered, the convert flow lands on `QuoteDetailPage` as required by D-CONV-02/D-CONV-03.
- UI CI gate remains red on 5 unrelated Phase 45 files. The pre-existing issue is the only blocker to a fully green deployment-ready CI state; a single `prettier --write` quick task will close it.

## Self-Check: PASSED

**Files exist:**

- `trade-flow-ui/src/features/estimates/components/EstimateResponseCard.tsx` → FOUND
- `trade-flow-ui/src/features/estimates/components/index.ts` → FOUND (contains `export * from "./EstimateResponseCard";`)
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` → FOUND (contains `EstimateResponseCard` import + JSX, 0 matches for "Phase 45")
- `trade-flow-ui/src/App.tsx` → FOUND (contains `/quotes/:quoteId/edit` once)

**Grep acceptance criteria (Task 1):**

- `RESPONSE_TYPE_LABELS: Record<string, string>` → 1 match
- `proceed: "Happy to Proceed"` → 1 match
- `message: "Customer Message"` → 1 match
- `decline: "Declined"` → 1 match
- `DECLINE_REASON_LABELS: Record<string, string>` → 1 match
- `formatDateTime` → 1 match (used inline)
- `ThumbsUp` / `ThumbsDown` / `MessageSquare` → 3 matches total (imported from lucide-react)
- `features/estimates/components/index.ts` contains `EstimateResponseCard` → 1 match
- `EstimateDetailPage.tsx` contains `EstimateResponseCard` → 2 matches (import + JSX)
- `EstimateDetailPage.tsx` contains `No customer response yet.` → 1 match
- `EstimateDetailPage.tsx` does NOT contain `Response handling ships in Phase 45` → 0 matches

**Grep acceptance criteria (Task 2):**

- `App.tsx` contains `path="/quotes/:quoteId/edit"` → 1 match
- `App.tsx` edit route renders `<QuoteDetailPage />` → confirmed (line 108)
- Edit route is sibling of `/quotes/:quoteId` inside PaywallGuard hierarchy → confirmed (lines 107-108, both inside the `<Route element={<PaywallGuard />}>` block opened at line 101)

**Commits exist:**

- `git log --oneline --all | grep a91190c` → FOUND (trade-flow-ui)
- `git log --oneline --all | grep 8ce2abb` → FOUND (trade-flow-ui)

**Plan verification commands:**

- `cd trade-flow-ui && npx tsc -b --noEmit` → exit 0
- `cd trade-flow-ui && npx vitest run` → exit 0 (99/99 tests pass)
- `cd trade-flow-ui && npx prettier --check <4 plan files>` → exit 0 ("All matched files use Prettier code style!")
- `cd trade-flow-api && npm run ci` → exit 0 (0 errors, 33 pre-existing warnings)
- `cd trade-flow-ui && npm run ci` → exit 1 at `format:check` on 5 pre-existing Phase 45 files (documented deviation, unchanged from plan 50-01)

---
*Phase: 50-response-display-convert-route-fix*
*Completed: 2026-04-16*
