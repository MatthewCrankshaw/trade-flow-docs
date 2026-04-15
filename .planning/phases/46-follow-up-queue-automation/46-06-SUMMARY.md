---
phase: 46-follow-up-queue-automation
plan: 06
subsystem: estimate-followup-processor
tags: [bugfix, email-renderer, follow-up, processor]
dependency_graph:
  requires: [46-04]
  provides: [correct-followup-email-rendering]
  affects: [estimate-followup-processor]
tech_stack:
  added: []
  patterns: []
key_files:
  created: []
  modified:
    - trade-flow-api/src/worker/processors/estimate-followup.processor.ts
    - trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts
    - trade-flow-api/src/worker/test/processors/estimate-expiry.processor.spec.ts
decisions:
  - Used business.name for traderFirstName since individual trader name is not available on BusinessDto
metrics:
  duration: 5min
  completed: 2026-04-15
  tasks: 1
  files: 3
---

# Phase 46 Plan 06: Fix Follow-up Email Renderer Wiring Summary

Replace wrong EstimateEmailRenderer with purpose-built EstimateFollowupEmailRenderer in processor so follow-up emails use step-specific copy (3d/10d/21d) and three CTAs (Go Ahead, Message [FirstName], Not right for me).

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 (RED) | Failing tests for EstimateFollowupEmailRenderer wiring | api@0167909 | estimate-followup.processor.spec.ts |
| 1 (GREEN) | Wire EstimateFollowupEmailRenderer in processor | api@8620dc6 | estimate-followup.processor.ts, estimate-followup.processor.spec.ts, estimate-expiry.processor.spec.ts |

## Implementation Details

### Processor Changes (estimate-followup.processor.ts)

- Replaced `import { EstimateEmailRenderer }` with `import { EstimateFollowupEmailRenderer }`
- Changed constructor parameter from `estimateEmailRenderer: EstimateEmailRenderer` to `estimateFollowupEmailRenderer: EstimateFollowupEmailRenderer`
- Replaced render call: removed hardcoded `message` and `estimateDate`/`contingencyPercent` params; now passes `{ businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay }`
- `followupStep` is cast to `"3d" | "10d" | "21d"` FollowupStep type from STEP_TO_LABEL map
- Removed the hardcoded `"Just following up on your estimate..."` message string

### Test Changes

- Follow-up processor spec: replaced all EstimateEmailRenderer references with EstimateFollowupEmailRenderer
- Added 3 new test cases:
  - Asserts `estimateFollowupEmailRenderer.render` called with `followupStep` and all required data fields
  - Asserts correct `followupStep` mapping: step 2 -> "10d", step 3 -> "21d"
  - Asserts rendered HTML from EstimateFollowupEmailRenderer is passed to `emailSenderService.sendEmail`
- Expiry processor spec: updated DI provider from EstimateEmailRenderer to EstimateFollowupEmailRenderer

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Updated estimate-expiry.processor.spec.ts DI token**
- **Found during:** Task 1 GREEN phase
- **Issue:** Expiry processor tests also referenced EstimateEmailRenderer for DI, causing test compilation failure after processor change
- **Fix:** Replaced EstimateEmailRenderer with EstimateFollowupEmailRenderer in the expiry test module providers
- **Files modified:** estimate-expiry.processor.spec.ts
- **Commit:** api@8620dc6

## Verification Results

- Follow-up processor tests: 11/11 passed
- Expiry processor tests: 6/6 passed
- `grep "EstimateFollowupEmailRenderer" estimate-followup.processor.ts`: 2 matches (import + constructor)
- `grep "EstimateEmailRenderer" estimate-followup.processor.ts | grep -v Followup`: 0 matches
- `grep "followupStep" estimate-followup.processor.ts`: present in render call (not just log)
- ESLint on modified files: 0 errors
- Pre-existing lint errors in unrelated files (estimate-to-quote-converter.service.ts): 2 prettier errors, out of scope

## Self-Check: PASSED

- All 3 modified files exist on disk
- Commit api@0167909 (RED): confirmed in git log
- Commit api@8620dc6 (GREEN): confirmed in git log
