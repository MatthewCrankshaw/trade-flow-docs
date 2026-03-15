---
phase: quick
plan: 5
subsystem: ui
tags: [dinero.js, currency, math, rounding]

requires:
  - phase: none
    provides: n/a
provides:
  - "Defensive Math.round in createMoney() preventing Dinero.js crash"
  - "Integer-producing lineTotalIncTax helpers in quote components"
affects: [quotes, currency]

tech-stack:
  added: []
  patterns: ["Defensive rounding before Dinero.js dinero() calls"]

key-files:
  created: []
  modified:
    - trade-flow-ui/src/lib/currency.ts
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
    - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx

key-decisions:
  - "Belt-and-suspenders: round both at callsite (lineTotalIncTax) and in createMoney()"

patterns-established:
  - "Always Math.round amounts before passing to dinero() constructor"

requirements-completed: [QUICK-5]

duration: 2min
completed: 2026-03-15
---

# Quick Task 5: Fix Dinero.js Invalid Amount Summary

**Defensive Math.round in createMoney() and lineTotalIncTax helpers to prevent Dinero.js "Amount is invalid" crash on fractional minor-unit values**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-15T14:27:00Z
- **Completed:** 2026-03-15T14:29:00Z
- **Tasks:** 1
- **Files modified:** 3

## Accomplishments
- createMoney() now rounds amount before passing to dinero(), protecting all callers
- lineTotalIncTax helpers in both QuoteLineItemsTable and QuoteLineItemsCardList produce integers via Math.round
- TypeScript typecheck and ESLint both pass cleanly

## Task Commits

Each task was committed atomically:

1. **Task 1: Add defensive rounding to createMoney and lineTotalIncTax helpers** - `30247f4` (fix)

## Files Created/Modified
- `trade-flow-ui/src/lib/currency.ts` - Added Math.round(amount) in createMoney()
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` - Wrapped lineTotalIncTax with Math.round
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` - Wrapped lineTotalIncTax with Math.round

## Decisions Made
- Applied belt-and-suspenders approach: rounding both at the lineTotalIncTax callsite and in createMoney() to avoid masking logic bugs while still preventing crashes

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

---
*Quick Task: 5*
*Completed: 2026-03-15*
