---
phase: 38-hard-paywall-and-soft-paywall-removal
plan: 01
subsystem: ui
tags: [react, stripe, paywall, subscription, route-guard]

# Dependency graph
requires:
  - phase: 36-public-landing-page-and-route-restructure
    provides: PaywallGuard shell in route tree
  - phase: 32-subscription-gate-and-subscribe-pages
    provides: useSubscription hook, subscriptionApi RTK Query endpoints
  - phase: 35-no-card-trial-api-endpoint
    provides: Subscription entity with trialEnd/canceledAt fields
provides:
  - PaywallPage component with three variant modes (trial-expired, payment-failed, canceled)
  - PaywallGuard blocking logic with support role bypass
  - Route tree restructure with /subscribe/success and /subscribe/cancel outside PaywallGuard
  - /subscribe redirect to /dashboard
affects: [38-02-soft-paywall-removal, 39-welcome-dashboard]

# Tech tracking
tech-stack:
  added: []
  patterns: [layout-route-guard-with-inline-blocking-page, subscription-state-variant-derivation]

key-files:
  created:
    - trade-flow-ui/src/pages/PaywallPage.tsx
  modified:
    - trade-flow-ui/src/features/auth/components/PaywallGuard.tsx
    - trade-flow-ui/src/features/subscription/contexts/SubscriptionContext.ts
    - trade-flow-ui/src/features/subscription/contexts/SubscriptionProvider.tsx
    - trade-flow-ui/src/App.tsx

key-decisions:
  - "openPaywall made no-op (not deleted) to avoid breaking consumers before Plan 02 cleanup"
  - "Manage Billing opens in new tab via window.open (not same-page redirect like existing usePortalSession)"
  - "derivePaywallVariant uses 24-hour threshold for trial-expired vs canceled detection with null-safe trialEnd/canceledAt checks"

patterns-established:
  - "Layout route guard with inline blocking page: PaywallGuard renders PaywallPage instead of redirecting"
  - "Subscription state variant derivation: status + trialEnd + canceledAt fields determine paywall variant"

requirements-completed: [PAYWALL-01, PAYWALL-02, PAYWALL-03, PAYWALL-04, PAYWALL-05]

# Metrics
duration: 3min
completed: 2026-04-02
---

# Phase 38 Plan 01: Hard Paywall Blocking Page Summary

**Full-screen PaywallPage with three variant modes (trial-expired, payment-failed, canceled) rendered by PaywallGuard blocking logic with support role bypass and route restructure for Stripe callback routes**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-02T11:35:33Z
- **Completed:** 2026-04-02T11:38:31Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- PaywallPage component with three variants showing state-specific headlines, body copy with inline data safety reassurance, Subscribe CTA calling Stripe Checkout, and Manage Billing opening Stripe Portal in new tab
- PaywallGuard blocking logic with support role bypass, loading screen (no paywall flash), and derivePaywallVariant function
- Route tree restructured: /subscribe/success and /subscribe/cancel moved outside PaywallGuard, /subscribe redirects to /dashboard, SubscriptionGatedLayout removed

## Task Commits

Each task was committed atomically:

1. **Task 1: Create PaywallPage component with three variant modes** - `9a38e8d` (feat)
2. **Task 2: Implement PaywallGuard blocking logic, update useSubscription, and adjust route tree** - `108fb3c` (feat)

## Files Created/Modified
- `trade-flow-ui/src/pages/PaywallPage.tsx` - Full-screen blocking page with variant-specific headlines, Subscribe/Manage Billing CTAs, and log out link
- `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` - Subscription checking guard with support bypass, loading state, and variant derivation
- `trade-flow-ui/src/features/subscription/contexts/SubscriptionContext.ts` - Removed openPaywall from context (deprecated, no-op for backward compat)
- `trade-flow-ui/src/features/subscription/contexts/SubscriptionProvider.tsx` - Removed paywallSlice dispatch, openPaywall is no-op
- `trade-flow-ui/src/App.tsx` - Removed SubscribePage import/route, added /subscribe redirect, moved callback routes outside PaywallGuard, removed SubscriptionGatedLayout

## Decisions Made
- Made openPaywall a no-op instead of deleting it entirely -- consumers in 8+ feature files still reference it. Plan 02 handles the full removal of all openPaywall dispatch calls. This avoids TypeScript errors while still removing the paywallSlice dependency from SubscriptionProvider.
- Manage Billing opens Stripe Portal in a new tab (window.open) rather than same-page redirect (window.location.href) -- users should stay on the paywall page while managing billing in another tab.
- Added null-safe checks on trialEnd and canceledAt in derivePaywallVariant since both fields are optional in the Subscription type.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Kept openPaywall as no-op instead of removing entirely**
- **Found during:** Task 2 (useSubscription simplification)
- **Issue:** Removing openPaywall from SubscriptionContextValue would break 8+ consumer files that still destructure and call it. TypeScript compilation would fail.
- **Fix:** Deprecated openPaywall in context interface, made it a no-op function in SubscriptionProvider. Plan 02 removes all consumer calls then deletes the property.
- **Files modified:** SubscriptionContext.ts, SubscriptionProvider.tsx
- **Verification:** `npx tsc --noEmit` passes clean
- **Committed in:** 108fb3c (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Necessary to maintain TypeScript compilation across the codebase. Plan 02 completes the full removal. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- PaywallGuard is fully functional and blocking -- Plan 02 can safely remove all soft paywall infrastructure
- openPaywall no-op maintains backward compatibility until Plan 02 cleanup sweep
- /subscribe/success and /subscribe/cancel routes are accessible for Stripe Checkout callbacks even when paywall is active

## Self-Check: PASSED

- PaywallPage.tsx: FOUND
- PaywallGuard.tsx: FOUND
- Commit 9a38e8d: FOUND
- Commit 108fb3c: FOUND

---
*Phase: 38-hard-paywall-and-soft-paywall-removal*
*Completed: 2026-04-02*
