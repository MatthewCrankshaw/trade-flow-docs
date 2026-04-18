---
phase: 41-estimate-module-crud-backend
plan: 02
subsystem: item-module
tags: [refactor, bundle-helpers, shared-services]
dependency_graph:
  requires: [41-01]
  provides: [shared-bundle-helpers-in-item-module]
  affects: [quote-module, future-estimate-module]
tech_stack:
  added: []
  patterns: [structural-interface-for-cross-module-sharing]
key_files:
  created:
    - trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts
    - trade-flow-api/src/item/test/services/bundle-config-validator.service.spec.ts
    - trade-flow-api/src/item/test/services/bundle-pricing-planner.service.spec.ts
    - trade-flow-api/src/item/test/services/bundle-tax-rate-calculator.service.spec.ts
  modified:
    - trade-flow-api/src/item/services/bundle-config-validator.service.ts (moved from quote)
    - trade-flow-api/src/item/services/bundle-pricing-planner.service.ts (moved from quote)
    - trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts (moved from quote, generalised)
    - trade-flow-api/src/item/item.module.ts
    - trade-flow-api/src/quote/quote.module.ts
    - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
decisions:
  - ILineItemTaxInput structural interface replaces IQuoteLineItemDto coupling in BundleTaxRateCalculator
  - Three bundle helpers exported from ItemModule for cross-module consumption
metrics:
  duration: ~8min
  completed: 2026-04-12
  tasks_completed: 3
  tasks_total: 3
---

# Phase 41 Plan 02: Lift Bundle Helpers Summary

Relocated BundleConfigValidator, BundlePricingPlanner, and BundleTaxRateCalculator from @quote/services/ to @item/services/ with a new ILineItemTaxInput structural interface, enabling cross-module sharing without quote-estimate coupling.

## What Changed

### Files Moved (git mv, preserves history)

| Before | After |
|--------|-------|
| `src/quote/services/bundle-config-validator.service.ts` | `src/item/services/bundle-config-validator.service.ts` |
| `src/quote/services/bundle-pricing-planner.service.ts` | `src/item/services/bundle-pricing-planner.service.ts` |
| `src/quote/services/bundle-tax-rate-calculator.service.ts` | `src/item/services/bundle-tax-rate-calculator.service.ts` |

### New Files Created

- `src/item/interfaces/line-item-tax-input.interface.ts` -- structural interface `{ unitPrice: Money; quantity: number; taxRate: number }` replacing the `IQuoteLineItemDto[]` coupling in BundleTaxRateCalculator
- `src/item/test/services/bundle-config-validator.service.spec.ts` -- 5 tests covering valid config, missing config, empty components
- `src/item/test/services/bundle-pricing-planner.service.spec.ts` -- 4 tests covering component-based pricing, fixed pricing, missing bundle price, quantity multiplication
- `src/item/test/services/bundle-tax-rate-calculator.service.spec.ts` -- 4 tests covering empty input, single component, multiple components, zero tax rate

### Module Wiring

- **ItemModule** (`item.module.ts`): Added BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator to both `providers` and `exports` arrays
- **QuoteModule** (`quote.module.ts`): Removed the three bundle helpers from `providers` and their imports. QuoteModule already imports ItemModule, so DI resolution is automatic.
- **QuoteBundleLineItemFactory**: Updated three import paths from `@quote/services/bundle-*` to `@item/services/bundle-*`

### Type Generalisation

`BundleTaxRateCalculator.calculate()` parameter changed from `IQuoteLineItemDto[]` to `ILineItemTaxInput[]`. No callsite changes were needed -- TypeScript structural typing accepts `IQuoteLineItemDto[]` as `ILineItemTaxInput[]` because `IQuoteLineItemDto` already contains `{ unitPrice: Money, quantity: number, taxRate: number }`.

The stale JSDoc comment on the calculate method was removed (it referenced a `parentLineTotal` parameter that did not exist in the signature).

## Verification Results

- **Bundle helper specs**: 13/13 passed (3 new spec files)
- **Quote test suite**: 62/62 passed across 11 suites (zero regression)
- **Full test suite**: 439/439 passed across 67 suites
- **TypeScript typecheck**: 0 errors
- **Prettier format check**: All plan-modified files pass
- **ESLint**: 0 new errors (6 pre-existing prettier errors in unrelated `document-token` module)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing] Created three spec files (no existing specs to relocate)**
- **Found during:** Task 1
- **Issue:** Plan Step 3 said to relocate existing spec files from `src/quote/test/services/bundle-*.spec.ts`, but none existed
- **Fix:** Created new spec files from scratch under `src/item/test/services/` covering happy-path and error cases per public method, following existing item test patterns (TestingModule setup, mock generators)
- **Files created:** 3 spec files (13 total tests)

**2. [Rule 1 - Bug] Removed stale JSDoc from BundleTaxRateCalculator**
- **Found during:** Task 1
- **Issue:** JSDoc on `calculate()` referenced a non-existent `@param parentLineTotal` parameter
- **Fix:** Removed the entire JSDoc block per CLAUDE.md conventions (prefer self-documenting code)
- **Files modified:** `src/item/services/bundle-tax-rate-calculator.service.ts`

**3. [Rule 3 - Blocking] Jest CLI flag change**
- **Found during:** Task 1 verification
- **Issue:** `--testPathPattern` was replaced by `--testPathPatterns` in Jest 30.x
- **Fix:** Used `npx jest --testPathPatterns=` directly instead of `npm run test --`

### Commit Limitation

Git write operations (add/commit) to the trade-flow-api sub-repository were blocked by the sandbox. All code changes are applied to disk, all tests pass, but the per-task atomic commits could not be created. The orchestrator or user must commit the changes manually.

## Known Stubs

None.

## Self-Check: PASSED

- All 7 new/moved files verified present at target locations
- All 3 old file locations verified removed
- 439/439 tests pass
- TypeScript typecheck: 0 errors
- Commits: not created (sandbox limitation on trade-flow-api sub-repo git write operations)
