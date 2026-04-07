---
phase: 40-subscription-guard-onboarding-bypass
plan: 01
subsystem: api/subscription
tags: [guard, onboarding, decorator, bypass]
dependency_graph:
  requires: []
  provides:
    - "SkipSubscriptionCheck on UserController.patch()"
    - "SkipSubscriptionCheck on BusinessController.create()"
    - "Guard tests verifying onboarding bypass"
  affects:
    - "trade-flow-api/src/user/controllers/user.controller.ts"
    - "trade-flow-api/src/business/controllers/business.controller.ts"
    - "trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts"
tech_stack:
  added: []
  patterns:
    - "@SkipSubscriptionCheck decorator on method-level for onboarding bypass"
key_files:
  modified:
    - trade-flow-api/src/user/controllers/user.controller.ts
    - trade-flow-api/src/business/controllers/business.controller.ts
    - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
decisions:
  - "Decorator placed at method level (not class level) to limit bypass scope"
  - "Three new test cases added to existing guard spec covering PATCH bypass, POST bypass, and non-regression"
metrics:
  duration: "3min"
  completed: "2026-04-07"
  tasks: 2
  files: 3
---

# Phase 40 Plan 01: SubscriptionGuard Onboarding Bypass Summary

@SkipSubscriptionCheck decorator applied to PATCH /v1/user/me and POST /v1/business so new users without a subscription can complete the mandatory onboarding wizard; three guard tests verify bypass and non-regression.

## What Was Done

### Task 1: Add @SkipSubscriptionCheck to onboarding controller methods

- Added `import { SkipSubscriptionCheck }` to `user.controller.ts` and `business.controller.ts`
- Applied `@SkipSubscriptionCheck()` decorator to `UserController.patch()` method (PATCH /v1/user/me)
- Applied `@SkipSubscriptionCheck()` decorator to `BusinessController.create()` method (POST /v1/business)
- TypeScript compilation verified (no new errors in modified files)
- Commit: `7bcf3c3` (trade-flow-api)

### Task 2: Add guard spec tests for onboarding bypass

- Added `describe("onboarding bypass routes")` block with three test cases:
  1. PATCH request allowed when `skipSubscriptionCheck` metadata is true (simulates user profile update during onboarding)
  2. POST request allowed when `skipSubscriptionCheck` metadata is true (simulates business creation during onboarding)
  3. POST without subscription throws ForbiddenError when `skipSubscriptionCheck` is absent (non-regression)
- All 13 tests pass (10 existing + 3 new)
- Full test suite: 380 passed, 6 failed (pre-existing failures in quote-email-sender, unrelated)
- Commit: `0f8accc` (trade-flow-api)

## Deviations from Plan

None - plan executed exactly as written.

## Verification Results

1. TypeScript compilation: No errors in modified files (pre-existing errors in unrelated files)
2. Guard unit tests: 13/13 passed
3. Full test suite: 380/386 passed (6 pre-existing failures unrelated to changes)
4. Decorator placement confirmed via grep - only on target methods, not class level

## Self-Check: PASSED
