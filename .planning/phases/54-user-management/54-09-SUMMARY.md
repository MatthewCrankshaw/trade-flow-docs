---
plan: 54-09
phase: 54-user-management
status: complete
---

## Summary

Threaded the `createdAt` timestamp from MongoDB `users` collection through the support user aggregation pipeline, DTO, and controller response.

## What Was Built

**Gap closed:** UAT Gap 1 — Created column in UserListTable always showed "---".

**Root cause:** `mapToResponse` in `SupportUserController` hardcoded `createdAt: null`.

**Fix applied across 4 files:**

1. `support-user.dto.ts` — Added `createdAt: string | null` to `ISupportUserDto`
2. `support-user.repository.ts` — Added `createdAt?: Date | null` to `SupportUserAggregateResult` interface; added `createdAt: doc.createdAt ? doc.createdAt.toISOString() : null` mapping in `mapToDto`
3. `support-user.controller.ts` — Changed `createdAt: null` → `createdAt: dto.createdAt` in `mapToResponse`
4. `support-user.mock.ts` — Added `createdAt` default value to `createSupportUserDto` factory

No aggregation pipeline changes were needed — the `users` collection already stores `createdAt` and the pipeline has no restrictive `$project` on the users stage, so the field flows through automatically.

## Verification

- `npm run ci` passed: 108 test suites, 924 tests, 0 errors
- `grep "createdAt: dto.createdAt"` matches in controller
- `grep "createdAt: string | null"` matches in DTO
- `grep "doc.createdAt"` matches in repository

## Self-Check: PASSED

## key-files.created
- trade-flow-api/src/support/data-transfer-objects/support-user.dto.ts (modified)
- trade-flow-api/src/support/repositories/support-user.repository.ts (modified)
- trade-flow-api/src/support/controllers/support-user.controller.ts (modified)
- trade-flow-api/src/support/test/mocks/support-user.mock.ts (modified)
