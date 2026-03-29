---
phase: 32-subscription-gate-and-subscribe-pages
plan: 02
subsystem: ui
tags: [react, subscription, paywall, stripe, radix-dialog, rtk-query]

# Dependency graph
requires:
  - phase: 32-01
    provides: SubscriptionProvider, useSubscription, useCheckout, useVerifySession hooks, paywallSlice Redux state
provides:
  - PaywallModal component with feature-aware title and checkout CTA
  - PersistentCta component for header (desktop) and banner (mobile)
  - DashboardBanner read-only mode reminder component
  - PricingCard shared pricing display component
  - SubscriptionGatedLayout route wrapper
  - SubscribePage, SubscribeSuccessPage, SubscribeCancelPage
  - Route tree restructured with subscription gate nesting
  - SubscriptionProvider wrapping all authenticated routes
affects: [32-03, subscription-enforcement, settings-billing]

# Tech tracking
tech-stack:
  added: []
  patterns: [soft-gate-paywall, route-nesting-for-gating, subscription-context-at-auth-level]

key-files:
  created:
    - trade-flow-ui/src/features/subscription/components/PricingCard.tsx
    - trade-flow-ui/src/features/subscription/components/PaywallModal.tsx
    - trade-flow-ui/src/features/subscription/components/PersistentCta.tsx
    - trade-flow-ui/src/features/subscription/components/DashboardBanner.tsx
    - trade-flow-ui/src/features/subscription/components/SubscriptionGatedLayout.tsx
    - trade-flow-ui/src/pages/SubscribePage.tsx
    - trade-flow-ui/src/pages/SubscribeSuccessPage.tsx
    - trade-flow-ui/src/pages/SubscribeCancelPage.tsx
  modified:
    - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
    - trade-flow-ui/src/App.tsx
    - trade-flow-ui/src/features/subscription/index.ts

key-decisions:
  - "SubscriptionProvider placed inside AuthenticatedLayout wrapping all authenticated routes -- subscribe pages and business routes both access subscription context"
  - "PaywallModal closes on route change via useLocation effect -- prevents stale modal after navigation"
  - "PersistentCta renders in header actions area before onboarding button -- subscription CTA takes priority"

patterns-established:
  - "Soft gate pattern: SubscriptionGatedLayout is semantic grouping only, actual enforcement via openPaywall per-action"
  - "Route nesting: subscribe and settings routes outside gate, business routes inside SubscriptionGatedLayout"
  - "No flash of CTA: components render null while isLoading is true"

requirements-completed: [GATE-01, GATE-02, GATE-03, GATE-05, ACQ-04]

# Metrics
duration: 3min
completed: 2026-03-29
---

# Phase 32 Plan 02: Subscription Gate UI Summary

**Subscription paywall modal, persistent CTAs, pricing card, dashboard banner, three subscribe pages, and gated route tree with SubscriptionProvider**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-29T18:54:44Z
- **Completed:** 2026-03-29T18:58:02Z
- **Tasks:** 2
- **Files modified:** 11

## Accomplishments
- Built 5 subscription UI components: PaywallModal (feature-aware dialog with checkout), PersistentCta (desktop header button + mobile banner), DashboardBanner (read-only mode reminder), PricingCard (shared pricing display), and SubscriptionGatedLayout (route wrapper)
- Created 3 subscribe pages: /subscribe (pricing + checkout), /subscribe/success (polling/verified/timed-out states), /subscribe/cancel (retry CTA)
- Restructured App.tsx route tree: subscribe and settings routes outside gate (GATE-02, GATE-03), business routes inside SubscriptionGatedLayout, SubscriptionProvider wrapping all authenticated routes

## Task Commits

Each task was committed atomically:

1. **Task 1: Create subscription UI components and integrate into DashboardLayout** - `9e656e8` (feat)
2. **Task 2: Create subscribe pages and wire App.tsx routes with SubscriptionProvider** - `57387e8` (feat)

## Files Created/Modified
- `src/features/subscription/components/PricingCard.tsx` - Shared pricing card with Trade Flow Pro plan details and feature list
- `src/features/subscription/components/PaywallModal.tsx` - Feature-aware paywall dialog using Redux paywall state with checkout CTA
- `src/features/subscription/components/PersistentCta.tsx` - Desktop header button and mobile banner CTA for inactive users
- `src/features/subscription/components/DashboardBanner.tsx` - Read-only mode reminder banner with subscribe CTA
- `src/features/subscription/components/SubscriptionGatedLayout.tsx` - Semantic route grouping wrapper rendering Outlet
- `src/pages/SubscribePage.tsx` - Subscribe page with PricingCard and checkout hook
- `src/pages/SubscribeSuccessPage.tsx` - Success page with polling, verified, and timed-out states
- `src/pages/SubscribeCancelPage.tsx` - Cancel page with Try again link
- `src/components/layouts/DashboardLayout.tsx` - Added PaywallModal and PersistentCta integration
- `src/App.tsx` - Added SubscriptionProvider, route nesting with SubscriptionGatedLayout, and three subscribe routes
- `src/features/subscription/index.ts` - Updated barrel exports with all new components

## Decisions Made
- SubscriptionProvider placed inside AuthenticatedLayout wrapping all authenticated routes so both subscribe pages and business routes have access to subscription context
- PaywallModal uses useLocation effect to auto-close on route change, preventing stale modal state
- PersistentCta renders before onboarding button in header actions area for visual priority

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All subscription UI components ready for Plan 03 to wire per-action gate enforcement (openPaywall calls on write actions in each page)
- DashboardBanner ready to be rendered on DashboardPage in Plan 03
- Route tree correctly structured for soft gate behavior

## Self-Check: PASSED

- All 8 created files exist on disk
- Commits 9e656e8 and 57387e8 verified in git log
- TypeScript compilation passes (npx tsc --noEmit)
- ESLint passes (npm run lint)

---
*Phase: 32-subscription-gate-and-subscribe-pages*
*Completed: 2026-03-29*
