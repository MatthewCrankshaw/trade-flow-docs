---
phase: 37-onboarding-wizard-pages
plan: 01
subsystem: ui
tags: [react, onboarding, wizard, react-hook-form, valibot, rtk-query, route-guard]

# Dependency graph
requires:
  - phase: 36-public-landing-page-and-route-restructure
    provides: OnboardingGuard pass-through shell, route tree structure
provides:
  - OnboardingGuard redirect logic (checks user name + business existence)
  - OnboardingWizard container with step state management
  - ProfileStep form (Step 1 -- display name entry via PATCH /v1/user/me)
  - OnboardingProgress accessible progress bar component
  - Trade constants with 10 trade options and Lucide icon mappings
  - useOnboardingStep hook for wizard resume detection
  - updateUser RTK Query mutation (PATCH /v1/user/me)
  - /onboarding route wired into App.tsx with dual OnboardingGuard wrapping
affects: [37-02 (BusinessStep, SetupLoadingScreen), 38 (paywall), 39 (old onboarding removal)]

# Tech tracking
tech-stack:
  added: []
  patterns: [onboarding wizard with useState step management, route guard with loading state protection]

key-files:
  created:
    - trade-flow-ui/src/features/onboarding/constants/trades.ts
    - trade-flow-ui/src/features/onboarding/components/OnboardingProgress.tsx
    - trade-flow-ui/src/features/onboarding/components/ProfileStep.tsx
    - trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx
    - trade-flow-ui/src/features/onboarding/hooks/useOnboardingStep.ts
    - trade-flow-ui/src/features/onboarding/components/index.ts
    - trade-flow-ui/src/features/onboarding/hooks/index.ts
    - trade-flow-ui/src/features/onboarding/index.ts
    - trade-flow-ui/src/pages/OnboardingPage.tsx
  modified:
    - trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx
    - trade-flow-ui/src/services/userApi.ts
    - trade-flow-ui/src/services/index.ts
    - trade-flow-ui/src/App.tsx

key-decisions:
  - "Used API User.name field (not Firebase displayName) for onboarding guard check -- matches server-side persistence"
  - "Added updateUser RTK Query mutation to userApi.ts since none existed for PATCH /v1/user/me"
  - "Dual OnboardingGuard wrapping in App.tsx -- one for /onboarding route, one for main app routes"

patterns-established:
  - "Onboarding wizard uses useState for step management (not React Router nested routes)"
  - "OnboardingGuard shows loading spinner while data fetches to prevent flash redirect"

requirements-completed: [ONBD-01, ONBD-05, ONBD-06, ONBD-07]

# Metrics
duration: 4min
completed: 2026-04-02
---

# Phase 37 Plan 01: Onboarding Wizard Foundation Summary

**OnboardingGuard redirect logic with wizard container, ProfileStep display name form, progress bar, and trade constants for the mandatory onboarding flow**

## Performance

- **Duration:** 4 min
- **Started:** 2026-04-02T11:22:00Z
- **Completed:** 2026-04-02T11:26:00Z
- **Tasks:** 2
- **Files modified:** 13

## Accomplishments
- OnboardingGuard checks user name and business existence, redirects incomplete users to /onboarding with loading state protection
- OnboardingWizard container manages step state with resume detection (skips to Step 2 if display name exists but no business)
- ProfileStep form validates display name with Valibot and saves via PATCH /v1/user/me RTK Query mutation
- Accessible progress bar with role="progressbar" and aria attributes
- Trade constants file with 10 trade options matching API enum values

## Task Commits

Each task was committed atomically:

1. **Task 1: Create onboarding feature module** - `e526310` (feat)
2. **Task 2: Implement OnboardingGuard, wizard container, and page** - `37905cc` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/onboarding/constants/trades.ts` - 10 trade options with Lucide icons
- `trade-flow-ui/src/features/onboarding/components/OnboardingProgress.tsx` - Accessible progress bar
- `trade-flow-ui/src/features/onboarding/components/ProfileStep.tsx` - Step 1 display name form with RHF + Valibot
- `trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx` - Wizard container with step management
- `trade-flow-ui/src/features/onboarding/hooks/useOnboardingStep.ts` - Resume detection hook
- `trade-flow-ui/src/features/onboarding/index.ts` - Feature barrel exports
- `trade-flow-ui/src/pages/OnboardingPage.tsx` - Route-level page shell
- `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` - Redirect logic replacing pass-through shell
- `trade-flow-ui/src/services/userApi.ts` - Added updateUser mutation
- `trade-flow-ui/src/services/index.ts` - Re-exported useUpdateUserMutation
- `trade-flow-ui/src/App.tsx` - Added /onboarding route with dual OnboardingGuard wrapping

## Decisions Made
- Used API `User.name` field for the onboarding guard check (not Firebase `displayName`) since the API is the source of truth for persistent profile data
- Created `updateUser` RTK Query mutation in existing `userApi.ts` (PATCH /v1/user/me with `{ name }` payload) since no user update mutation existed
- Implemented dual OnboardingGuard wrapping in App.tsx: one instance wraps `/onboarding` routes (allows user to stay), another wraps main app routes (redirects if incomplete)
- BusinessStep placeholder renders inline text for now; Plan 02 will implement the full component

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Added updateUser RTK Query mutation**
- **Found during:** Task 1 (ProfileStep implementation)
- **Issue:** No PATCH /v1/user/me mutation existed in userApi.ts; ProfileStep needs it to save display name
- **Fix:** Added updateUser mutation with invalidatesTags: ["User"] and re-exported from services barrel
- **Files modified:** trade-flow-ui/src/services/userApi.ts, trade-flow-ui/src/services/index.ts
- **Verification:** TypeScript compilation passes; mutation properly typed
- **Committed in:** e526310 (Task 1 commit)

**2. [Rule 3 - Blocking] Added /onboarding route to App.tsx**
- **Found during:** Task 2 (OnboardingPage wiring)
- **Issue:** Phase 36 created OnboardingGuard shell and import but did not add /onboarding route to App.tsx
- **Fix:** Added /onboarding route inside OnboardingGuard wrapper, wrapped main app routes with second OnboardingGuard instance
- **Files modified:** trade-flow-ui/src/App.tsx
- **Verification:** TypeScript compilation passes; route structure matches Phase 36 D-08 pattern
- **Committed in:** 37905cc (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (1 missing critical, 1 blocking)
**Impact on plan:** Both auto-fixes necessary for the onboarding flow to function. No scope creep.

## Issues Encountered
None

## Known Stubs
- `OnboardingWizard.tsx` line 34: BusinessStep renders placeholder text "Business setup step coming soon..." -- Plan 02 implements BusinessStep
- `OnboardingWizard.tsx` line 27: SetupLoadingScreen renders basic spinner placeholder -- Plan 02 implements SetupLoadingScreen

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Plan 02 can implement BusinessStep and SetupLoadingScreen, plugging into the wizard container
- OnboardingGuard is live and will redirect new users to /onboarding
- Trade constants ready for TradeCardGrid in Plan 02

---
*Phase: 37-onboarding-wizard-pages*
*Completed: 2026-04-02*
