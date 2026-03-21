---
phase: 17-customer-quote-page
plan: 02
subsystem: ui
tags: [react, rtk-query, public-quote, responsive, shadcn]

requires:
  - phase: 17-customer-quote-page
    provides: firstViewedAt tracking, enriched error responses, viewedAt on Quote type

provides:
  - Public quote page at /quote/view/:token (no auth required)
  - Separate publicQuoteApi RTK Query slice (no auth headers)
  - Responsive line items (table on desktop, stacked cards on mobile)
  - Error states for not found, expired, revoked, and network errors
  - Loading skeleton matching card layout
  - Disabled PDF download button placeholder
  - Status banners for accepted/rejected quotes
  - Viewed badge on QuoteActionStrip for tradesperson

affects: [20-pdf-generation, customer-quote-actions]

tech-stack:
  added: []
  patterns:
    - "Separate RTK Query API slice for unauthenticated public endpoints"
    - "formatDecimal for major-unit currency display (not formatAmount)"

key-files:
  created:
    - trade-flow-ui/src/features/public-quote/api/publicQuoteApi.ts
    - trade-flow-ui/src/features/public-quote/types/public-quote.types.ts
    - trade-flow-ui/src/features/public-quote/components/PublicQuoteCard.tsx
    - trade-flow-ui/src/features/public-quote/components/PublicQuoteLineItems.tsx
    - trade-flow-ui/src/features/public-quote/components/PublicQuoteTotals.tsx
    - trade-flow-ui/src/features/public-quote/components/PublicQuoteSkeleton.tsx
    - trade-flow-ui/src/features/public-quote/components/PublicQuoteError.tsx
    - trade-flow-ui/src/features/public-quote/index.ts
    - trade-flow-ui/src/pages/PublicQuotePage.tsx
  modified:
    - trade-flow-ui/src/store/index.ts
    - trade-flow-ui/src/App.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx

key-decisions:
  - "Separate publicQuoteApi slice instead of injecting into authenticated apiSlice to avoid auth header leakage"
  - "Used formatDecimal for public quote amounts since API returns major units (decimals)"
  - "Viewed badge placed outside status-conditional blocks to show regardless of quote status"

patterns-established:
  - "Public API slice pattern: separate createApi with no prepareHeaders for unauthenticated endpoints"

requirements-completed: [RESP-01, RESP-05, DLVR-05]

duration: 3min
completed: 2026-03-20
---

# Phase 17 Plan 02: Customer Quote Page UI Summary

**Public quote page with responsive line items, error states, loading skeleton, disabled PDF button, and tradesperson viewed badge using separate unauthenticated RTK Query API slice**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-20T19:21:35Z
- **Completed:** 2026-03-20T19:24:47Z
- **Tasks:** 2
- **Files modified:** 12

## Accomplishments
- Complete public-quote feature module with separate RTK Query API slice (no auth headers)
- Responsive line items display: table on desktop (md+), stacked cards on mobile
- Four error states with exact UI-SPEC copy: not found, expired, revoked, network
- Loading skeleton matching card layout sections
- Disabled PDF download button with "coming soon" tooltip
- Accepted/rejected status banners with success/warning colors
- Viewed badge with Eye icon and formatted timestamp on QuoteActionStrip

## Task Commits

Each task was committed atomically:

1. **Task 1: Create public-quote feature module with API slice, types, and Redux store registration** - `b284bc3` (feat)
2. **Task 2: Build all public quote page components, the page, route registration, and viewed badge** - `727b1aa` (feat)

## Files Created/Modified
- `trade-flow-ui/src/features/public-quote/types/public-quote.types.ts` - Frontend mirror types for PublicQuoteResponse and PublicQuoteLineItem
- `trade-flow-ui/src/features/public-quote/api/publicQuoteApi.ts` - Separate RTK Query API slice with no auth headers
- `trade-flow-ui/src/features/public-quote/index.ts` - Barrel exports for public-quote feature
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteCard.tsx` - Main card layout with all sections
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteLineItems.tsx` - Responsive line items (table/cards)
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteTotals.tsx` - Right-aligned totals with separator
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteSkeleton.tsx` - Loading skeleton matching card layout
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteError.tsx` - Error states with copy per UI-SPEC
- `trade-flow-ui/src/pages/PublicQuotePage.tsx` - Route page orchestrating loading/error/success
- `trade-flow-ui/src/store/index.ts` - publicQuoteApi reducer and middleware registered
- `trade-flow-ui/src/App.tsx` - Public route at /quote/view/:token outside AuthenticatedLayout
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` - Viewed badge with Eye icon and timestamp

## Decisions Made
- Separate publicQuoteApi slice instead of injecting into authenticated apiSlice to prevent auth header leakage on public endpoints
- Used formatDecimal for public quote amounts since the public API returns major units (decimals), not minor units
- Viewed badge placed outside status-conditional blocks in QuoteActionStrip to display regardless of quote status

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Customer quote page complete and accessible at /quote/view/:token
- PDF download button is wired up but disabled -- Phase 20 will enable it
- View tracking end-to-end: backend records firstViewedAt, tradesperson sees Viewed badge
- All typecheck, lint, and production build pass

---
*Phase: 17-customer-quote-page*
*Completed: 2026-03-20*
