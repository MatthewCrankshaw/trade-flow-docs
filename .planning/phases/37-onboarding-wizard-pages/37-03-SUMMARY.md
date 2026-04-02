---
phase: 37-onboarding-wizard-pages
plan: 03
subsystem: ui
tags: [react, tailwind, stripe, subscription, trial, badge, responsive]

requires:
  - phase: 33-stripe-billing-ui
    provides: subscription feature module with TrialChip, useSubscription, subscriptionApi, Stripe portal integration
provides:
  - TrialBadge component with desktop/mobile responsive variants and urgency color shifts
  - Stripe Billing Portal opens in new tab on badge click
  - DashboardLayout header updated with TrialBadge replacing TrialChip
affects: [38-hard-paywall, 39-landing-page]

tech-stack:
  added: []
  patterns:
    - "Responsive badge with hidden/md:inline-flex for desktop and md:hidden for mobile variants"
    - "Urgency color tiers: secondary (>10d), yellow warning (<=10d), destructive (<=3d)"

key-files:
  created:
    - trade-flow-ui/src/features/subscription/components/TrialBadge.tsx
    - trade-flow-ui/src/features/subscription/components/index.ts
  modified:
    - trade-flow-ui/src/features/subscription/index.ts
    - trade-flow-ui/src/components/layouts/DashboardLayout.tsx

key-decisions:
  - "Replaced TrialChip with TrialBadge in DashboardLayout header -- TrialBadge uses shadcn Badge with urgency variants instead of custom amber chip"
  - "Portal opens in new tab (window.open) instead of same-tab navigation to settings, matching plan spec for TRIAL-03"

patterns-established:
  - "Responsive badge pattern: desktop shows full text with icon, mobile shows compact abbreviation"

requirements-completed: [TRIAL-02, TRIAL-03]

duration: 2min
completed: 2026-04-02
---

# Phase 37 Plan 03: Trial Badge Summary

**Responsive TrialBadge component with desktop/mobile variants, urgency color shifts (yellow at <=10 days, destructive at <=3), and Stripe Billing Portal click-to-manage**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-02T11:20:53Z
- **Completed:** 2026-04-02T11:22:52Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- TrialBadge component with desktop (Clock icon + "N days left") and mobile ("Nd") variants
- Urgency color tiers: secondary normally, yellow at <=10 days, destructive at <=3 days
- Click opens Stripe Billing Portal in new tab via createPortalSession mutation
- Replaced existing TrialChip in DashboardLayout header with new TrialBadge

## Task Commits

Each task was committed atomically:

1. **Task 1: Create TrialBadge component with urgency variants and Billing Portal click handler** - `1d2c747` (feat)
2. **Task 2: Integrate TrialBadge into DashboardLayout header** - `51ad1ea` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/subscription/components/TrialBadge.tsx` - Responsive trial badge with urgency colors and portal click handler
- `trade-flow-ui/src/features/subscription/components/index.ts` - Barrel export for TrialBadge
- `trade-flow-ui/src/features/subscription/index.ts` - Added TrialBadge to feature barrel exports
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` - Replaced TrialChip with TrialBadge in header

## Decisions Made
- Replaced TrialChip with TrialBadge in header (TrialChip navigated to settings, TrialBadge opens Billing Portal in new tab per spec)
- Used existing `useSubscription` hook and `calculateDaysRemaining` helper rather than reimplementing from scratch
- Used `useCreatePortalSessionMutation` with `window.open` for new-tab portal access instead of same-tab redirect

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Replaced TrialChip instead of adding alongside**
- **Found during:** Task 2
- **Issue:** DashboardLayout already had a TrialChip in the exact position where TrialBadge should go. Adding both would show duplicate trial indicators.
- **Fix:** Replaced TrialChip import and usage with TrialBadge in DashboardLayout header
- **Files modified:** trade-flow-ui/src/components/layouts/DashboardLayout.tsx
- **Verification:** TypeScript compilation passes, no duplicate trial indicators
- **Committed in:** 51ad1ea (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug prevention)
**Impact on plan:** Necessary to avoid duplicate trial indicators in header. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - component wires directly to existing subscription API hooks and context.

## Next Phase Readiness
- TrialBadge is live in the DashboardLayout header
- Old TrialChip component still exists in codebase (not deleted) for backward compatibility
- Phase 38 can proceed with hard paywall work independently

---
*Phase: 37-onboarding-wizard-pages*
*Completed: 2026-04-02*
