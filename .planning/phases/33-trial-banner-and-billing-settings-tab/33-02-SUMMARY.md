---
phase: 33-trial-banner-and-billing-settings-tab
plan: 02
subsystem: ui
tags: [react, subscription, billing, stripe-portal, tabs, settings]

# Dependency graph
requires:
  - phase: 33-trial-banner-and-billing-settings-tab
    plan: 01
    provides: calculateDaysRemaining, formatBillingDate utility functions
  - phase: 32-subscription-gate-and-subscribe-pages
    provides: SubscriptionContext, useSubscription hook, subscription types
provides:
  - SubscriptionStatusCard component with full CTA matrix
  - BillingTab composition wrapper
  - usePortalSession hook for Stripe Billing Portal redirect
  - createPortalSession RTK Query mutation
  - Settings page tab routing via URL query params
affects: [33-HUMAN-UAT]

# Tech tracking
tech-stack:
  added: []
  patterns: [url-based-tab-routing, cta-matrix-status-display]

key-files:
  created:
    - trade-flow-ui/src/features/subscription/components/SubscriptionStatusCard.tsx
    - trade-flow-ui/src/features/subscription/components/BillingTab.tsx
    - trade-flow-ui/src/features/subscription/hooks/usePortalSession.ts
  modified:
    - trade-flow-ui/src/features/subscription/api/subscriptionApi.ts
    - trade-flow-ui/src/features/subscription/index.ts
    - trade-flow-ui/src/pages/SettingsPage.tsx

key-decisions:
  - "URL query param (?tab=billing) for tab routing preserves Stripe Portal return_url context"
  - "getStatusDisplay helper function encapsulates CTA matrix logic for testability"

patterns-established:
  - "URL-based tab routing via useSearchParams for Settings page tabs"

requirements-completed: [BILL-04, BILL-05]

# Metrics
duration: 3min
completed: 2026-03-29
---

# Phase 33 Plan 02: Billing Settings Tab Summary

**Settings page billing tab with subscription status card showing CTA matrix (trialing/active/cancelling/inactive), Stripe Portal redirect via usePortalSession hook, and URL-based tab routing**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-29T19:56:04Z
- **Completed:** 2026-03-29T19:59:19Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Built SubscriptionStatusCard with full CTA matrix: trialing (amber), active (green), cancelling (orange), inactive (muted)
- Created usePortalSession hook and createPortalSession RTK Query mutation for Stripe Billing Portal redirect
- Refactored SettingsPage with shadcn Tabs and URL query param routing (?tab=billing)

## Task Commits

Each task was committed atomically:

1. **Task 1: Add createPortalSession mutation and usePortalSession hook** - `be62a2e` (feat)
2. **Task 2: Build SubscriptionStatusCard, BillingTab, and refactor SettingsPage with tab routing** - `ddf79fc` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/subscription/api/subscriptionApi.ts` - Added createPortalSession mutation calling POST /v1/subscription/portal
- `trade-flow-ui/src/features/subscription/hooks/usePortalSession.ts` - Hook wrapping portal mutation with window.location redirect
- `trade-flow-ui/src/features/subscription/components/SubscriptionStatusCard.tsx` - Status card with badge, status line, and CTA button for each subscription state
- `trade-flow-ui/src/features/subscription/components/BillingTab.tsx` - Composition wrapper for billing tab content
- `trade-flow-ui/src/features/subscription/index.ts` - Updated barrel exports with new components and hooks
- `trade-flow-ui/src/pages/SettingsPage.tsx` - Refactored with Tabs component and URL-based tab routing

## Decisions Made
- URL query param (?tab=billing) for tab routing -- preserves context when returning from Stripe Portal
- getStatusDisplay helper function encapsulates CTA matrix logic in SubscriptionStatusCard for clean rendering

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Billing tab fully wired and accessible via /settings?tab=billing
- SubscriptionStatusCard handles all subscription states with correct badges and CTAs
- Ready for human UAT verification

## Self-Check: PASSED
