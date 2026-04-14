---
phase: 47-convert-to-quote-mark-as-lost
plan: 01
subsystem: estimate, quote
tags: [backend, services, schema, convert-to-quote, mark-as-lost, unit-tests]
dependency_graph:
  requires: [41-07, 45-03, 46-05]
  provides: [EstimateToQuoteConverter, EstimateLostMarker, LostReason enum, schema extensions]
  affects: [estimate entity/dto/response, quote entity/dto/response, EstimateModule, QuoteModule]
tech_stack:
  added: []
  patterns: [idempotent-conversion, followup-cancellation, cross-module-service]
key_files:
  created:
    - trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts
    - trade-flow-api/src/estimate/services/estimate-lost-marker.service.ts
    - trade-flow-api/src/estimate/enums/lost-reason.enum.ts
    - trade-flow-api/src/estimate/requests/mark-estimate-lost.request.ts
    - trade-flow-api/src/estimate/requests/convert-estimate.request.ts
    - trade-flow-api/src/estimate/test/services/estimate-to-quote-converter.service.spec.ts
    - trade-flow-api/src/estimate/test/services/estimate-lost-marker.service.spec.ts
  modified:
    - trade-flow-api/src/estimate/entities/estimate.entity.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
    - trade-flow-api/src/estimate/responses/estimate.responses.ts
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts
    - trade-flow-api/src/estimate/estimate.module.ts
    - trade-flow-api/src/quote/entities/quote.entity.ts
    - trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts
    - trade-flow-api/src/quote/responses/quote.responses.ts
    - trade-flow-api/src/quote/repositories/quote.repository.ts
    - trade-flow-api/src/quote/controllers/quote.controller.ts
    - trade-flow-api/src/quote/quote.module.ts
decisions:
  - Used QuoteLineItemCreator for literal snapshot line-item copy instead of QuoteUpdater.addLineItem (which re-resolves prices from item catalog)
  - Exported QuoteCreator and QuoteLineItemCreator from QuoteModule for cross-module access by EstimateToQuoteConverter
  - Used forwardRef for QuoteModule import in EstimateModule to avoid circular dependency
metrics:
  duration: 10min
  completed: "2026-04-14T07:16:11Z"
  tasks: 2
  files: 19
---

# Phase 47 Plan 01: Backend Services and Schema Changes for Convert to Quote and Mark as Lost Summary

EstimateToQuoteConverter creates draft quotes from SENT/RESPONDED estimates with idempotent re-call guard, literal line-item snapshot copy, sourceEstimateId/sourceEstimateNumber back-link on quote, and automatic follow-up cancellation. EstimateLostMarker transitions estimates to LOST with optional structured reason (LostReason enum) and freeform notes, also cancelling follow-ups.

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Entity/DTO/Response schema extensions and request classes | 240f6d0 | 13 files (entity/DTO/response on both modules, enum, request DTOs, repository mappings, controller mapToResponse) |
| 2 | EstimateToQuoteConverter and EstimateLostMarker services with unit tests | d54f84f | 6 files (2 services, 2 test files, estimate module, quote module) |

## Decisions Made

1. **Literal line-item copy via QuoteLineItemCreator**: The converter copies estimate line items directly to quote line items using QuoteLineItemCreator.create() with snapshot values (unitPrice, taxRate, quantity) rather than going through QuoteUpdater.addLineItem which re-resolves prices from the item catalog. This preserves the exact values the customer saw on the estimate.

2. **Cross-module exports**: Exported QuoteCreator and QuoteLineItemCreator from QuoteModule so EstimateToQuoteConverter can create quotes and line items. Previously only QuoteRepository and QuoteTransitionService were exported.

3. **forwardRef for QuoteModule**: Used forwardRef(() => QuoteModule) in EstimateModule imports to handle the potential circular dependency between Estimate and Quote modules.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] QuoteModule exports insufficient for cross-module converter**
- **Found during:** Task 2
- **Issue:** QuoteModule only exported QuoteRepository, QuoteTransitionService, and QuoteTotalsCalculator. EstimateToQuoteConverter needs QuoteCreator and QuoteLineItemCreator.
- **Fix:** Added QuoteCreator and QuoteLineItemCreator to QuoteModule exports; added forwardRef(() => QuoteModule) to EstimateModule imports.
- **Files modified:** quote.module.ts, estimate.module.ts
- **Commit:** d54f84f

## Verification

- TypeScript compilation: zero new errors (pre-existing errors in unrelated files only)
- Unit tests: 16 tests pass (8 converter + 7 lost marker + 1 additional)
- Full CI: All 821 tests pass, lint warnings only (no errors in new files after prettier fix)

## Self-Check: PASSED
