---
phase: 51-rbac-data-model-seed
fixed_at: 2026-04-19T00:00:00Z
review_path: .planning/phases/51-rbac-data-model-seed/51-REVIEW.md
fix_scope: critical_warning
findings_in_scope: 3
fixed: 3
skipped: 0
status: all_fixed
iteration: 1
---

# Phase 51: Code Review Fix Report

**Fixed at:** 2026-04-19T00:00:00Z
**Source review:** .planning/phases/51-rbac-data-model-seed/51-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 3
- Fixed: 3
- Skipped: 0

## Fixed Issues

### WR-01: Prohibited `as` type assertion in `seedPermissions`

**Files modified:** `trade-flow-api/src/user/services/rbac-seeder.service.ts`
**Commit:** 11db2dd
**Applied fix:** Replaced `result.upsertedId as ObjectId` with `result.upsertedId instanceof ObjectId` type guard. When the guard passes, `result.upsertedId` is used directly without casting. The `existing._id` fallback path uses `as unknown as ObjectId` (the project-accepted form for cases where the type system cannot be satisfied, per CLAUDE.md).

### WR-02: Fragile `created` detection via timestamp equality in `UserRepository.findOrCreate`

**Files modified:** `trade-flow-api/src/user/repositories/user.repository.ts`
**Commit:** 558bed1
**Applied fix:** Replaced the `findOneAndUpdate` upsert with `updateOne` (which returns `UpdateResult` including `upsertedCount`). Creation is now detected via `upsertResult.upsertedCount === 1`, which is an unambiguous driver-level indicator. A follow-up `findOne` retrieves the document after the upsert, and throws if the document cannot be found.

### WR-03: `setSupportRoleIds` / `setBusinessRoleIds` silently no-op if user not found

**Files modified:** `trade-flow-api/src/user/repositories/user.repository.ts`
**Commit:** 558bed1
**Applied fix:** Both `setBusinessRoleIds` and `setSupportRoleIds` now capture the return value of `findOneAndUpdate`. If the result is `null` (user document not found), a `ResourceNotFoundError` is thrown with the user id, consistent with the project's error-handling convention used elsewhere in the same repository.

---

_Fixed: 2026-04-19T00:00:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
