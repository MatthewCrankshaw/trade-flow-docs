---
phase: 37-onboarding-wizard-pages
plan: 02
subsystem: ui
tags: [react, onboarding, wizard, react-hook-form, valibot, rtk-query, trade-selection]

# Dependency graph
requires:
  - phase: 37-onboarding-wizard-pages-01
    provides: OnboardingWizard container, ProfileStep, trade constants, useOnboardingStep hook
  - phase: 35-no-card-trial-api-endpoint
    provides: POST /v1/subscription/trial endpoint for no-card trial creation
provides:
  - BusinessStep form with business name input and trade card grid selection
  - TradeCard accessible button component with selected/unselected visual states
  - TradeCardGrid responsive grid (2-col mobile, 3-col tablet+) for 10 trade options
  - SetupLoadingScreen with sequential business creation and trial activation
  - startTrial RTK Query mutation (POST /v1/subscription/trial)
  - Complete onboarding flow from Step 2 through dashboard redirect
affects: [39 (old onboarding removal), 38 (paywall -- trial now auto-started during onboarding)]

# Tech tracking
tech-stack:
  added: []
  patterns: [sequential API mutation chain with partial-completion retry, valibot cross-field validation with v.forward]

key-files:
  created:
    - trade-flow-ui/src/features/onboarding/components/TradeCard.tsx
    - trade-flow-ui/src/features/onboarding/components/TradeCardGrid.tsx
    - trade-flow-ui/src/features/onboarding/components/BusinessStep.tsx
    - trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx
  modified:
    - trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx
    - trade-flow-ui/src/features/onboarding/components/index.ts
    - trade-flow-ui/src/features/subscription/api/subscriptionApi.ts

key-decisions:
  - "Used v.forward with v.check for cross-field validation (customTrade required when primaryTrade is other)"
  - "SetupLoadingScreen refetches businesses on retry to detect partial completion (business created but trial failed)"
  - "422 from trial endpoint treated as success -- subscription already exists is not an error during onboarding"
  - "Business creation payload uses TradeInfo structure with primaryTrade + customTradeName matching existing API contract"

patterns-established:
  - "Sequential mutation chain with partial-completion detection for multi-step API workflows"
  - "Trade card grid with accessible button elements and aria-pressed for selection state"

requirements-completed: [ONBD-02, ONBD-03, ONBD-04]

# Metrics
duration: 4min
completed: 2026-04-02
---

# Phase 37 Plan 02: Business Step and Setup Loading Summary

**BusinessStep form with trade card grid selection and SetupLoadingScreen executing sequential business creation (GB/GBP) plus no-card trial activation with partial-completion retry**

## Performance

- **Duration:** 4 min
- **Started:** 2026-04-02T11:28:20Z
- **Completed:** 2026-04-02T11:32:05Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments
- TradeCard, TradeCardGrid, and BusinessStep components with accessible trade selection (10 options in responsive grid)
- SetupLoadingScreen with sequential business creation (hardcoded GB/GBP) and trial activation via POST /v1/subscription/trial
- Retry handles partial completion by checking if business already exists before re-creating
- OnboardingWizard fully wired with BusinessStep replacing placeholder and SetupLoadingScreen replacing loading stub

## Task Commits

Each task was committed atomically:

1. **Task 1: Create TradeCard, TradeCardGrid, and BusinessStep components** - `2d9bd89` (feat)
2. **Task 2: Create SetupLoadingScreen, wire RTK mutations, complete wizard integration** - `ea0788a` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/onboarding/components/TradeCard.tsx` - Accessible button with selected/unselected visual states and aria-pressed
- `trade-flow-ui/src/features/onboarding/components/TradeCardGrid.tsx` - Responsive grid (2-col/3-col) rendering 10 trade options
- `trade-flow-ui/src/features/onboarding/components/BusinessStep.tsx` - Step 2 form with business name, trade selection, and Other custom input
- `trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx` - Full-screen loading/error state with sequential API calls and retry
- `trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx` - Replaced placeholders with BusinessStep and SetupLoadingScreen
- `trade-flow-ui/src/features/onboarding/components/index.ts` - Added barrel exports for all new components
- `trade-flow-ui/src/features/subscription/api/subscriptionApi.ts` - Added startTrial mutation for POST /v1/subscription/trial

## Decisions Made
- Used `v.forward(v.check(...))` for cross-field validation requiring customTrade when primaryTrade is "other"
- SetupLoadingScreen refetches businesses on retry to detect partial completion (business created but trial failed)
- 422 from trial endpoint treated as success since it means subscription already exists (D-09)
- Business creation uses existing `BusinessRequest` type with `TradeInfo` structure (`primaryTrade` + `customTradeName`)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Added startTrial RTK Query mutation**
- **Found during:** Task 2
- **Issue:** No startTrial mutation existed in subscriptionApi.ts; SetupLoadingScreen needs it to start the free trial
- **Fix:** Added startTrial mutation with POST /v1/subscription/trial, invalidatesTags: ["Subscription"], and exported useStartTrialMutation
- **Files modified:** trade-flow-ui/src/features/subscription/api/subscriptionApi.ts
- **Verification:** TypeScript compilation passes
- **Committed in:** ea0788a (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 missing critical)
**Impact on plan:** Auto-fix necessary for trial activation. No scope creep.

## Issues Encountered
None

## Known Stubs
None -- all components are fully implemented with real data flows.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Full onboarding flow is complete: Step 1 (profile name) -> Step 2 (business name + trade) -> loading screen -> dashboard
- Phase 38 (paywall) can proceed independently
- Phase 39 (old onboarding removal) can proceed -- new wizard is fully functional

---
*Phase: 37-onboarding-wizard-pages*
*Completed: 2026-04-02*
