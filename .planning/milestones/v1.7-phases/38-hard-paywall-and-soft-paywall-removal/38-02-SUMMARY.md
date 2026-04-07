---
phase: 38-hard-paywall-and-soft-paywall-removal
plan: 02
subsystem: ui
tags: [react, subscription, paywall, cleanup, dead-code-removal]

# Dependency graph
requires:
  - phase: 38-hard-paywall-and-soft-paywall-removal
    plan: 01
    provides: Hard paywall (PaywallGuard + PaywallPage) and openPaywall no-op bridge
provides:
  - Complete removal of soft paywall infrastructure (7 files deleted, 10+ files cleaned)
  - Redux store without paywallSlice
  - DashboardLayout without PaywallModal, PersistentCta, or DashboardBanner
  - Feature pages with no openPaywall dispatch calls
affects: [39-welcome-dashboard]

# Tech tracking
tech-stack:
  added: []
  patterns: []

key-files:
  created: []
  modified:
    - trade-flow-ui/src/store/index.ts
    - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
    - trade-flow-ui/src/features/subscription/index.ts
    - trade-flow-ui/src/features/subscription/contexts/SubscriptionContext.ts
    - trade-flow-ui/src/features/subscription/contexts/SubscriptionProvider.tsx
    - trade-flow-ui/src/pages/CustomersPage.tsx
    - trade-flow-ui/src/pages/JobsPage.tsx
    - trade-flow-ui/src/pages/JobDetailPage.tsx
    - trade-flow-ui/src/pages/QuotesPage.tsx
    - trade-flow-ui/src/pages/ItemsPage.tsx
    - trade-flow-ui/src/pages/DashboardPage.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx
    - trade-flow-ui/src/features/business/components/BusinessDetails.tsx

key-decisions:
  - "Removed openPaywall from SubscriptionContextValue interface entirely (not just deprecated) since all consumers cleaned in same commit"
  - "DashboardBanner import and render removed from DashboardPage (was soft paywall upsell, not needed with hard paywall)"

patterns-established: []

requirements-completed: [PAYWALL-06]

# Metrics
duration: 5min
completed: 2026-04-02
---

# Phase 38 Plan 02: Soft Paywall Removal Summary

**Deleted 7 soft paywall files and removed all openPaywall dispatch calls from 10 feature pages, leaving PaywallGuard as the sole subscription enforcement mechanism**

## Performance

- **Duration:** 5 min
- **Started:** 2026-04-02T11:40:43Z
- **Completed:** 2026-04-02T11:46:24Z
- **Tasks:** 2
- **Files modified:** 21 (7 deleted + 14 edited)

## Accomplishments
- Deleted 7 soft paywall files: SubscriptionGatedLayout, PaywallModal, PersistentCta, DashboardBanner, PricingCard, paywallSlice, SubscribePage
- Removed paywallReducer from Redux store configuration
- Stripped openPaywall/isActive guards from all feature pages (CustomersPage, JobsPage, JobDetailPage, QuotesPage, ItemsPage, DashboardPage, QuoteActionStrip, QuoteLineItemsCard, BusinessDetails)
- Cleaned SubscriptionContext interface and SubscriptionProvider of openPaywall property
- Cleaned feature barrel exports of deleted component re-exports
- Zero soft paywall references remain in codebase (verified via grep)

## Task Commits

Each task was committed atomically:

1. **Task 1: Delete soft paywall components, Redux slice, and SubscribePage** - `67cff43` (refactor)
2. **Task 2: Remove all openPaywall dispatch calls from feature pages** - `22fa823` (refactor)

## Files Created/Modified
- `trade-flow-ui/src/features/subscription/components/SubscriptionGatedLayout.tsx` - DELETED
- `trade-flow-ui/src/features/subscription/components/PaywallModal.tsx` - DELETED
- `trade-flow-ui/src/features/subscription/components/PersistentCta.tsx` - DELETED
- `trade-flow-ui/src/features/subscription/components/DashboardBanner.tsx` - DELETED
- `trade-flow-ui/src/features/subscription/components/PricingCard.tsx` - DELETED
- `trade-flow-ui/src/store/slices/paywallSlice.ts` - DELETED
- `trade-flow-ui/src/pages/SubscribePage.tsx` - DELETED
- `trade-flow-ui/src/store/index.ts` - Removed paywallReducer from store
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` - Removed PaywallModal and PersistentCta imports/renders
- `trade-flow-ui/src/features/subscription/index.ts` - Removed 5 deleted component re-exports
- `trade-flow-ui/src/features/subscription/contexts/SubscriptionContext.ts` - Removed openPaywall from interface
- `trade-flow-ui/src/features/subscription/contexts/SubscriptionProvider.tsx` - Removed noop and openPaywall property
- `trade-flow-ui/src/pages/CustomersPage.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/pages/JobsPage.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/pages/JobDetailPage.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/pages/QuotesPage.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/pages/ItemsPage.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/pages/DashboardPage.tsx` - Removed DashboardBanner, useSubscription, and openPaywall guards
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCard.tsx` - Removed useSubscription and openPaywall guards
- `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` - Removed useSubscription and openPaywall guards

## Decisions Made
- Removed openPaywall from SubscriptionContextValue entirely rather than keeping deprecated -- all consumers were cleaned in the same task, so no backward compatibility needed.
- Removed DashboardBanner import and render from DashboardPage -- it was the soft paywall upsell banner, not needed with the hard paywall blocking screen.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing Critical] Cleaned SubscriptionContext and SubscriptionProvider**
- **Found during:** Task 2 (openPaywall removal sweep)
- **Issue:** Plan listed feature page files but did not explicitly list SubscriptionContext.ts and SubscriptionProvider.tsx for openPaywall property removal
- **Fix:** Removed openPaywall from the SubscriptionContextValue interface and the noop/openPaywall from SubscriptionProvider
- **Files modified:** SubscriptionContext.ts, SubscriptionProvider.tsx
- **Verification:** `npx tsc --noEmit` passes clean
- **Committed in:** 22fa823 (Task 2 commit)

**2. [Rule 2 - Missing Critical] Cleaned feature barrel exports**
- **Found during:** Task 2 (grep sweep for remaining references)
- **Issue:** `src/features/subscription/index.ts` still re-exported DashboardBanner, PaywallModal, PersistentCta, PricingCard, SubscriptionGatedLayout from deleted files
- **Fix:** Removed all 5 deleted component re-exports from the barrel
- **Files modified:** src/features/subscription/index.ts
- **Verification:** `npx tsc --noEmit` passes clean
- **Committed in:** 22fa823 (Task 2 commit)

**3. [Rule 1 - Bug] Cleaned additional feature files not in plan**
- **Found during:** Task 2 (grep sweep for openPaywall)
- **Issue:** QuoteActionStrip.tsx, QuoteLineItemsCard.tsx, and BusinessDetails.tsx also contained openPaywall calls but were not listed in plan files
- **Fix:** Applied same openPaywall removal pattern to all three files
- **Files modified:** QuoteActionStrip.tsx, QuoteLineItemsCard.tsx, BusinessDetails.tsx
- **Verification:** `grep -rn "openPaywall" src/` returns zero matches
- **Committed in:** 22fa823 (Task 2 commit)

---

**Total deviations:** 3 auto-fixed (2 missing critical, 1 bug)
**Impact on plan:** All auto-fixes necessary for complete soft paywall removal. The plan's grep verification step caught the additional files. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Hard paywall (PaywallGuard + PaywallPage) is the sole subscription enforcement mechanism
- No soft paywall remnants exist in the codebase
- Phase 39 (welcome dashboard) can proceed without interference from paywall infrastructure

## Self-Check: PASSED

- 7 deleted files confirmed absent
- Commit 67cff43: FOUND
- Commit 22fa823: FOUND
- 38-02-SUMMARY.md: FOUND

---
*Phase: 38-hard-paywall-and-soft-paywall-removal*
*Completed: 2026-04-02*
