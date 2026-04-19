---
phase: 57-impersonation-frontend
plan: 02
subsystem: ui
tags: [react, redux, impersonation, banner, guard-bypass, navigation, rtk-query]

# Dependency graph
requires:
  - phase: 57-01
    provides: impersonationSlice, impersonationApi, token-switching prepareHeaders, baseQuery wrapper
provides:
  - useImpersonation hook with startSession/endSession lifecycle
  - ImpersonationBanner fixed amber banner with Return to Support button
  - OnboardingGuard impersonation bypass
  - PaywallGuard impersonation bypass
  - DashboardLayout integration with banner, top padding, expired session watcher, and nav switching
affects: [57-03, 57-04, impersonation-dialog]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - useRef + useEffect pattern for detecting Redux state transitions (active-to-inactive) in a layout component
    - Impersonation bypass pattern in route guards (check Redux state before existing guard logic)
    - Fixed overlay banner with z-[60] to sit above Radix overlays (z-50)

key-files:
  created:
    - trade-flow-ui/src/features/support/hooks/useImpersonation.ts
    - trade-flow-ui/src/features/support/components/ImpersonationBanner.tsx
  modified:
    - trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx
    - trade-flow-ui/src/features/auth/components/PaywallGuard.tsx
    - trade-flow-ui/src/config/navigation.ts
    - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
    - trade-flow-ui/src/features/auth/components/__tests__/OnboardingGuard.test.tsx
    - trade-flow-ui/src/features/auth/components/__tests__/PaywallGuard.test.tsx

key-decisions:
  - "navigation.ts already refactored to getBusinessNavigationItems()/getSupportNavigationItems() — added getNavigationItems(user, isImpersonating) wrapper; DashboardLayout NavContent adapted to use isImpersonating prop directly"
  - "endSession navigates before dispatching endImpersonation to ensure navigate(/support) fires first; ref watcher in DashboardLayout fires afterward but navigate is idempotent on same route"
  - "pt-10 applied to outermost flex container (not main) so both sidebar and content area are pushed down equally"

# Metrics
duration: 7min
completed: 2026-04-19
---

# Phase 57 Plan 02: Impersonation UX Layer Summary

**Fixed amber banner, useImpersonation hook, guard bypasses, and nav switching — complete impersonation UX layer**

## Performance

- **Duration:** ~7 min
- **Started:** 2026-04-19T17:09:06Z
- **Completed:** 2026-04-19T17:16:29Z
- **Tasks:** 2
- **Files modified:** 8

## Accomplishments

- Created `useImpersonation` hook encapsulating full session lifecycle: API calls, Redux dispatches (`startImpersonation`/`endImpersonation`), RTK cache reset (`apiSlice.util.resetApiState()`), navigation, and toast notifications
- Created `ImpersonationBanner` — fixed `top-0 left-0 right-0 z-[60]` amber banner with `role="alert"` and `aria-live="assertive"`, showing target user name and a "Return to Support" button with loading state
- Added impersonation bypass to `OnboardingGuard` as the first check after loading (before support role check)
- Added impersonation bypass to `PaywallGuard` as the first check (before support role check)
- Added `getNavigationItems(user, isImpersonating)` wrapper to `navigation.ts` — returns business nav when impersonating
- Updated `DashboardLayout` to: render `ImpersonationBanner`, apply `pt-10` when impersonating, pass `isImpersonating` to `NavContent`, and watch active-to-inactive state transitions via `useRef`/`useEffect` to auto-navigate to `/support` with `toast.warning("Impersonation session expired")` on expiry (D-14)
- Updated guard tests to wrap with Redux `Provider` and added impersonation bypass test cases

## Task Commits

Each task was committed atomically:

1. **Task 1: Create useImpersonation hook and add guard bypasses and navigation switching** — `57b5e7e` (feat)
2. **Task 2: Create ImpersonationBanner and integrate into DashboardLayout** — `03120f5` (feat)

## Files Created/Modified

- `trade-flow-ui/src/features/support/hooks/useImpersonation.ts` — Hook managing start/end session lifecycle with API calls, Redux actions, cache reset, navigation, and toasts
- `trade-flow-ui/src/features/support/components/ImpersonationBanner.tsx` — Fixed amber banner with `z-[60]`, `role="alert"`, `aria-live="assertive"`, target user name display, and "Return to Support" button
- `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` — Added impersonation bypass before onboarding check
- `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` — Added impersonation bypass before paywall check
- `trade-flow-ui/src/config/navigation.ts` — Added `getNavigationItems(user, isImpersonating)` wrapper function
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` — Integrated banner, conditional `pt-10`, `isImpersonating` prop to `NavContent`, expired session watcher
- `trade-flow-ui/src/features/auth/components/__tests__/OnboardingGuard.test.tsx` — Added Redux Provider, `renderGuard` helper, impersonation bypass test
- `trade-flow-ui/src/features/auth/components/__tests__/PaywallGuard.test.tsx` — Added Redux Provider, `makeStore` helper, impersonation bypass test

## Decisions Made

- **Navigation adapter:** The plan described a `getNavigationItems(user, isImpersonating)` signature, but the codebase had already been refactored to separate `getBusinessNavigationItems()` and `getSupportNavigationItems()`. Added a `getNavigationItems` wrapper function to satisfy the plan's interface contract while keeping the existing split functions. `DashboardLayout.NavContent` uses `isImpersonating` directly to pick the correct nav set.
- **endSession order:** `navigate("/support")` is called before `dispatch(endImpersonation())` so the deliberate end flow completes navigation first. The `useRef` watcher in `DashboardLayout` still fires on the state transition but `navigate("/support")` is idempotent when already on that route.
- **pt-10 placement:** Applied to the outermost `div.flex.min-h-screen` container so both the sidebar and main content area shift down uniformly behind the fixed banner.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Guard tests broke after adding useAppSelector to guards**

- **Found during:** Task 1 CI run
- **Issue:** `OnboardingGuard` and `PaywallGuard` now call `useAppSelector` which requires a Redux `Provider`. Existing tests rendered components without a Provider, causing "could not find react-redux context value" errors.
- **Fix:** Added `makeStore()` helper and `Provider` wrapper to both test files. Refactored `OnboardingGuard.test.tsx` to use a shared `renderGuard()` helper. Added new test cases for the impersonation bypass in both guards.
- **Files modified:** `OnboardingGuard.test.tsx`, `PaywallGuard.test.tsx`
- **Commits:** included in `57b5e7e`

**2. [Rule 1 - Bug] Unused `error` variable in endSession catch block**

- **Found during:** Task 1 CI lint run
- **Issue:** ESLint `@typescript-eslint/no-unused-vars` error on `catch (error)` in `useImpersonation.ts` endSession — error is not rethrown (per D-12 design: do not clear state on end failure).
- **Fix:** Changed `catch (error)` to `catch` (bare catch clause, TypeScript 4+ syntax).
- **Files modified:** `useImpersonation.ts`
- **Commits:** included in `57b5e7e`

**3. [Rule 1 - Bug] navigation.ts Prettier formatting**

- **Found during:** Task 1 CI format check
- **Issue:** The added `getNavigationItems` wrapper function had formatting that didn't match Prettier's output (function signature line length).
- **Fix:** Ran `prettier --write` on `navigation.ts`.
- **Files modified:** `navigation.ts`
- **Commits:** included in `57b5e7e`

## Known Stubs

None — all impersonation UX state is wired to live Redux state (`state.impersonation.active`, `state.impersonation.targetUser`). The banner renders `null` when not impersonating. No placeholder data.

## Threat Flags

No new threat surface beyond the plan's threat model. All four mitigations implemented:
- T-57-04: `apiSlice.util.resetApiState()` called on both `startSession` AND `endSession`
- T-57-05: `OnboardingGuard` bypass gated by `state.impersonation.active` (set only via authenticated backend call)
- T-57-06: `PaywallGuard` bypass gated by `state.impersonation.active`
- T-57-07: `ImpersonationBanner` uses `z-[60]` ensuring visibility above all Radix overlays; no dismiss mechanism

---
*Phase: 57-impersonation-frontend*
*Completed: 2026-04-19*
