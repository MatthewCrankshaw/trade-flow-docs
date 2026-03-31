---
phase: 32-subscription-gate-and-subscribe-pages
plan: 03
subsystem: ui
tags: [react, subscription, paywall, gating, redux]

# Dependency graph
requires:
  - phase: 32-02
    provides: PaywallModal, DashboardBanner, useSubscription hook, paywallSlice
provides:
  - Paywall-gated write actions across all 7 business pages
  - DashboardBanner on dashboard for inactive users
  - Feature-aware paywall labels for each write action
affects: [33-trial-banner-and-billing-settings-tab]

# Tech tracking
tech-stack:
  added: []
  patterns: [subscription guard pattern in page-level and component-level handlers]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/pages/CustomersPage.tsx
    - trade-flow-ui/src/pages/JobsPage.tsx
    - trade-flow-ui/src/pages/JobDetailPage.tsx
    - trade-flow-ui/src/pages/QuotesPage.tsx
    - trade-flow-ui/src/pages/QuoteDetailPage.tsx (via QuoteActionStrip + QuoteLineItemsCard)
    - trade-flow-ui/src/pages/ItemsPage.tsx
    - trade-flow-ui/src/pages/DashboardPage.tsx
    - trade-flow-ui/src/features/business/components/BusinessDetails.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx

key-decisions:
  - "Paywall guards added directly in sub-components (QuoteActionStrip, QuoteLineItemsCard, BusinessDetails) where handlers live, rather than threading callbacks through props"
  - "Dashboard quick actions (Create Job, Add Customer) also gated for consistency"

patterns-established:
  - "Subscription guard pattern: if (!isActive) { openPaywall('Label'); return; } at top of handler body"

requirements-completed: [GATE-01, GATE-04]

# Metrics
duration: 5min
completed: 2026-03-29
---

# Phase 32 Plan 03: Paywall Trigger Wiring Summary

**Paywall guards wired into all business page write-action handlers with feature-aware labels, plus DashboardBanner on dashboard**

## Performance

- **Duration:** 5 min
- **Started:** 2026-03-29T18:59:54Z
- **Completed:** 2026-03-29T19:05:01Z
- **Tasks:** 1
- **Files modified:** 9

## Accomplishments
- All write actions across 7 business pages gated by subscription status check
- PaywallModal shows feature-specific title for each action (14 distinct labels)
- DashboardBanner rendered on dashboard for inactive users
- Active/trialing/support users pass through to existing logic unimpeded
- TypeScript compiles cleanly; ESLint passes

## Task Commits

Each task was committed atomically:

1. **Task 1: Wire paywall triggers into all business page write-action handlers and add DashboardBanner** - `96f1734` (feat) [trade-flow-ui]

## Files Created/Modified
- `trade-flow-ui/src/pages/CustomersPage.tsx` - Guards on create, edit, toggle status
- `trade-flow-ui/src/pages/JobsPage.tsx` - Guard on create job
- `trade-flow-ui/src/pages/JobDetailPage.tsx` - Guards on schedule, quote creation, status change
- `trade-flow-ui/src/pages/QuotesPage.tsx` - Guard on create quote
- `trade-flow-ui/src/pages/ItemsPage.tsx` - Guards on create and edit item
- `trade-flow-ui/src/pages/DashboardPage.tsx` - DashboardBanner added, quick actions gated
- `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` - Guards on edit business and edit location
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` - Guards on send, transition, accept/reject
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` - Guard on add line item

## Decisions Made
- Paywall guards placed directly in sub-components (QuoteActionStrip, QuoteLineItemsCard, BusinessDetails) where write-action handlers live, rather than threading guard callbacks through multiple prop layers. This is cleaner and follows existing component autonomy patterns.
- Dashboard quick action buttons also gated (Create Job, Add Customer) even though dashboard is not a core business page -- consistency matters for the paywall experience.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Added paywall guards to dashboard quick actions**
- **Found during:** Task 1 (DashboardPage modification)
- **Issue:** Plan only specified adding DashboardBanner to dashboard but dashboard has "Create New Job" and "Add Customer" quick action buttons that are write actions
- **Fix:** Added subscription guards to both quick action button click handlers
- **Files modified:** trade-flow-ui/src/pages/DashboardPage.tsx
- **Verification:** TypeScript compiles, pattern consistent with other pages
- **Committed in:** 96f1734

---

**Total deviations:** 1 auto-fixed (1 missing critical)
**Impact on plan:** Essential for completeness -- leaving dashboard quick actions ungated would bypass the paywall.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - all paywall guards are fully wired to the useSubscription hook from Plan 01.

## Next Phase Readiness
- All soft-gate enforcement complete across business pages
- Ready for Phase 33 (trial banner and billing settings tab)
- PaywallModal integration tested via TypeScript compilation

---
*Phase: 32-subscription-gate-and-subscribe-pages*
*Completed: 2026-03-29*
