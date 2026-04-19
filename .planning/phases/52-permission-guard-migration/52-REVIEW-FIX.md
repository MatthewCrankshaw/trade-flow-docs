---
phase: 52-permission-guard-migration
fixed_at: 2026-04-19T00:00:00Z
review_path: .planning/phases/52-permission-guard-migration/52-REVIEW.md
iteration: 1
findings_in_scope: 5
fixed: 5
skipped: 0
status: all_fixed
---

# Phase 52: Code Review Fix Report

**Fixed at:** 2026-04-19T00:00:00Z
**Source review:** .planning/phases/52-permission-guard-migration/52-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 5
- Fixed: 5
- Skipped: 0

## Fixed Issues

### CR-01: PermissionGuard passes all requests when no @RequiresPermission is set

**Files modified:** `trade-flow-api/src/auth/guards/permission.guard.ts`
**Commit:** d2bba92
**Applied fix:** Added a block comment above the open-by-default early-return explaining that all routes protected by PermissionGuard must be annotated with `@RequiresPermission`, and that unannotated routes are accessible to all authenticated users. Notes a stricter `@Public()` opt-out variant as a future improvement.

### WR-01: BusinessPolicy grants write access via a read-only permission ("view_support_dashboard")

**Files modified:** `trade-flow-api/src/business/policies/business.policy.ts`
**Commit:** 3179df2
**Applied fix:** Added a TODO comment in `canUpdate` documenting that `"view_support_dashboard"` is a read-named permission being used to gate write access. This is a known design issue across all policies; the comment flags that a dedicated write permission (e.g. `"manage_support_resources"`) should be introduced in a future change rather than applying a premature rename that would require a database migration.

### WR-02: TaxRatePolicy.canCreate logic anomaly — silently grants access when resource is undefined

**Files modified:** `trade-flow-api/src/tax-rate/policies/tax-rate.policy.ts`
**Commit:** dda0a0d
**Applied fix:** Changed `resource?: ITaxRateDto` to `resource: ITaxRateDto` (required, not optional) in the `canCreate` signature. Removed the conditional guard and `authUser.businessIds.length > 0` fallback; the method now delegates directly to `authUser.businessIds.includes(resource.businessId)`, which is consistent with `canRead` and `canUpdate`. The base class `canCreate` accepts an optional resource, so narrowing the subclass signature is type-safe.

### WR-03: PermissionGuard test suite only exercises supportRoles path — businessRoles path is untested

**Files modified:** `trade-flow-api/src/auth/test/guards/permission.guard.spec.ts`
**Commit:** a186521
**Applied fix:** Added `buildUserWithBusinessPermissions` helper function that creates a user with permissions nested inside a `businessRole` (no support roles). Added a corresponding test case `"should return true when user has the required permission via a businessRole"` that exercises the `businessRoles` traversal path through `hasPermission`.

### WR-04: SubscriptionGuard silently passes unauthenticated requests on non-GET/HEAD mutation routes

**Files modified:** `trade-flow-api/src/subscription/guards/subscription.guard.ts`, `trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts`
**Commit:** 603dd0d
**Applied fix:** Changed the `!user` branch from `return true` to `throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Authentication required")`. Updated the comment to explain this is a defence-in-depth measure since `JwtAuthGuard` should already reject these requests. Added a new test group `"unauthenticated mutation requests"` with a test case verifying that a POST with no user throws `ForbiddenError` and does not reach the repository.

## Skipped Issues

None — all in-scope findings were fixed.

## Not In Scope (Info findings)

- IN-01: RequiresPermission decorator accepts untyped string permissions — no compile-time safety
- IN-02: BusinessPolicy.canUpdate test asserts "always returns false" — but the implementation does not
- IN-03: Redundant log messages in policy canUpdate methods reference "support user" but use hasPermission

---

_Fixed: 2026-04-19T00:00:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
