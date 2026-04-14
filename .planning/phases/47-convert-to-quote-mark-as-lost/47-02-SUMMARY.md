---
phase: 47-convert-to-quote-mark-as-lost
plan: 02
subsystem: estimate
tags: [backend, controller, endpoints, openapi, module-wiring, error-codes]
dependency_graph:
  requires: [47-01]
  provides: [convert-endpoint, mark-lost-endpoint, openapi-docs]
  affects: [estimate controller, estimate module, error-codes enum, errors-map, openapi.yaml]
tech_stack:
  added: []
  patterns: [idempotency-header, optional-body-validation]
key_files:
  created: []
  modified:
    - trade-flow-api/src/estimate/controllers/estimate.controller.ts
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
    - trade-flow-api/openapi.yaml
    - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
decisions:
  - EstimateModule wiring already complete from Plan 01 -- no additional module changes needed
  - Error codes ESTIMATE_NOT_CONVERTIBLE and ESTIMATE_NOT_LOSABLE added for future use; current services use ESTIMATE_INVALID_TRANSITION
metrics:
  duration: 5min
  completed: "2026-04-14T07:25:00Z"
  tasks: 2
  files: 5
---

# Phase 47 Plan 02: Controller Endpoints and OpenAPI Documentation for Convert and Mark as Lost Summary

Two POST endpoints wired on EstimateController -- convert reads Idempotency-Key from header and delegates to EstimateToQuoteConverter returning quoteId; mark-lost accepts optional MarkEstimateLostRequest body and delegates to EstimateLostMarker returning the updated estimate response. OpenAPI spec documents both endpoints with all parameters, response codes, and LostReason enum values.

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Controller endpoints and module wiring | 7dacc44 | 3 files (controller, error-codes enum, errors-map constant) |
| 2 | OpenAPI documentation and CI gate | 1046b92 | 2 files (openapi.yaml, controller spec) |

## Decisions Made

1. **EstimateModule wiring already complete**: Plan 01 already added EstimateToQuoteConverter and EstimateLostMarker to providers, exports, and QuoteModule forwardRef. No additional module changes required for Plan 02.

2. **New error codes for future use**: Added ESTIMATE_NOT_CONVERTIBLE and ESTIMATE_NOT_LOSABLE error codes and map entries. The current services use ESTIMATE_INVALID_TRANSITION for status validation errors, but these dedicated codes provide more specific error identification for future consumers.

## Deviations from Plan

None -- plan executed exactly as written.

## Verification

- TypeScript compilation: zero new errors in modified files
- Unit tests: 8 new controller tests added (3 convert, 3 mark-lost, 2 endpoint existence checks)
- Full CI gate: all tests pass, lint clean, format clean, typecheck clean (exit code 0)
- OpenAPI: both endpoints documented with parameters, request bodies, response codes, and security

## Self-Check: PASSED
