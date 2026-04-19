---
phase: 52-permission-guard-migration
reviewed: 2026-04-19T00:00:00Z
depth: standard
files_reviewed: 31
files_reviewed_list:
  - trade-flow-api/src/auth/auth.module.ts
  - trade-flow-api/src/auth/decorators/requires-permission.decorator.ts
  - trade-flow-api/src/auth/guards/permission.guard.ts
  - trade-flow-api/src/auth/test/guards/permission.guard.spec.ts
  - trade-flow-api/src/auth/test/utilities/has-permission.utility.spec.ts
  - trade-flow-api/src/auth/utilities/has-any-permission.utility.ts
  - trade-flow-api/src/auth/utilities/has-permission.utility.ts
  - trade-flow-api/src/business/policies/business.policy.ts
  - trade-flow-api/src/business/test/policies/business.policy.spec.ts
  - trade-flow-api/src/customer/policies/customer.policy.ts
  - trade-flow-api/src/estimate-settings/policies/estimate-settings.policy.ts
  - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
  - trade-flow-api/src/estimate/policies/estimate.policy.ts
  - trade-flow-api/src/item/policies/item.policy.ts
  - trade-flow-api/src/job/policies/job-type.policy.ts
  - trade-flow-api/src/job/policies/job.policy.ts
  - trade-flow-api/src/migration/policies/migration.policy.ts
  - trade-flow-api/src/quote-settings/policies/quote-settings.policy.ts
  - trade-flow-api/src/quote/policies/quote-line-item.policy.ts
  - trade-flow-api/src/quote/policies/quote.policy.ts
  - trade-flow-api/src/schedule/policies/schedule.policy.ts
  - trade-flow-api/src/schedule/test/policies/schedule.policy.spec.ts
  - trade-flow-api/src/subscription/guards/subscription.guard.ts
  - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
  - trade-flow-api/src/tax-rate/policies/tax-rate.policy.ts
  - trade-flow-api/src/user/data-transfer-objects/business-role.dto.ts
  - trade-flow-api/src/user/data-transfer-objects/permission.dto.ts
  - trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts
  - trade-flow-api/src/user/policies/support-role.policy.ts
  - trade-flow-api/src/visit-type/policies/visit-type.policy.ts
  - trade-flow-api/src/visit-type/test/policies/visit-type.policy.spec.ts
findings:
  critical: 1
  warning: 4
  info: 3
  total: 8
status: issues_found
---

# Phase 52: Code Review Report

**Reviewed:** 2026-04-19T00:00:00Z
**Depth:** standard
**Files Reviewed:** 31
**Status:** issues_found

## Summary

This phase migrated hardcoded `isSupportUser()`/`isSuperUser()` role checks across 12+ policy classes and 2 guards to a generic `hasPermission()` system. The core permission infrastructure (`PermissionGuard`, `hasPermission`, `hasAnyPermission`, `RequiresPermission` decorator) is well-constructed and the migration is complete — no remnants of the old role-check pattern were found.

One critical security issue was found in `PermissionGuard`: when no `@RequiresPermission` metadata is present on a route, the guard silently passes all authenticated users through. This is the correct NestJS pattern for backwards compatibility, but it means `PermissionGuard` alone provides no protection on unannotated routes. This is a design risk that needs to be documented or addressed.

Four warning-level issues were identified: a privilege escalation risk in `BusinessPolicy.canUpdate` (support users can mutate any business via `view_support_dashboard`, which is a read permission), a logic anomaly in `TaxRatePolicy.canCreate` that silently grants access when no resource is provided, a gap in the `PermissionGuard` test suite (no test for businessRoles path), and the `SubscriptionGuard` silently passes unauthenticated requests to non-GET/non-HEAD routes.

---

## Critical Issues

### CR-01: PermissionGuard passes all requests when no @RequiresPermission is set — routes are unprotected by default

**File:** `trade-flow-api/src/auth/guards/permission.guard.ts:19`
**Issue:** When `requiredPermissions` is `undefined` or empty, `canActivate` returns `true` unconditionally. This is an open-by-default posture. Any route that a developer forgets to annotate with `@RequiresPermission` will be accessible to any authenticated user. Because `PermissionGuard` is exported from `AuthModule` and intended for use across the application, a single missed annotation silently removes the permission check rather than failing closed.

This is especially concerning for privilege-sensitive routes like those in `MigrationPolicy` and `SupportRolePolicy`, which depend on `@RequiresPermission` being correctly applied in the controller. If a controller imports `PermissionGuard` but a developer forgets the decorator, the route is open.

**Fix:** The guard cannot unilaterally break all undecorated routes without auditing every controller. The safest immediate fix is to document this explicitly in a comment on the guard and ensure every controller that uses `PermissionGuard` is audited to confirm all protected routes have `@RequiresPermission`. Longer-term, consider a stricter variant that fails closed by default:

```typescript
// Option A: explicit opt-out for unprotected routes via @Public() decorator
if (!requiredPermissions || requiredPermissions.length === 0) {
  // Fail closed: require explicit @Public() or @RequiresPermission() on every route
  throw new ForbiddenError(
    ErrorCodes.ACTION_FORBIDDEN,
    "No permissions declared on this route — access denied by default",
  );
}

// Option B (current, open-by-default): document the assumption explicitly
// NOTE: This guard is open-by-default. All routes protected by PermissionGuard
// MUST be annotated with @RequiresPermission or they will be accessible to all authenticated users.
```

---

## Warnings

### WR-01: BusinessPolicy grants write access via a read-only permission ("view_support_dashboard")

**File:** `trade-flow-api/src/business/policies/business.policy.ts:24-32`
**Issue:** `canUpdate` uses `hasPermission(authUser, "view_support_dashboard")` as the support-user gate, which is the same permission checked in `canRead`. The permission name "view_support_dashboard" semantically describes read access. A support user with only this permission can also mutate any business record — this is a privilege escalation from read to write.

The same pattern appears across every policy that has both `canRead` and `canUpdate` methods gated on `"view_support_dashboard"` (customer, estimate-settings, estimate, item, job-type, job, quote-settings, quote, schedule, tax-rate, visit-type). It is a systematic pattern from the migration.

If this is intentional (support roles that can view can also update), it should be expressed as a dedicated write permission (e.g., `"manage_support_resources"`). Using a read-named permission for write operations is fragile — future permission assignment will be unintuitive.

**Fix:** Either rename the permission to reflect combined read/write access, or introduce separate read (`view_support_dashboard`) and write (`manage_support_resources`) permissions and gate `canUpdate` on the write permission:

```typescript
// business.policy.ts
public canUpdate(authUser: IUserDto, _resource: IBusinessDto): boolean {
  if (hasPermission(authUser, "manage_support_resources")) {
    return true;
  }
  if (authUser.businessIds.includes(_resource.id)) {
    return true;
  }
  return false;
}
```

### WR-02: TaxRatePolicy.canCreate logic anomaly — silently grants access when resource is undefined

**File:** `trade-flow-api/src/tax-rate/policies/tax-rate.policy.ts:9-18`
**Issue:** The `canCreate` method has this logic when `resource` is undefined:
1. `hasPermission(authUser, "view_support_dashboard")` — checked first (correct)
2. The `if (resource && !authUser.businessIds.includes(resource.businessId))` block is skipped
3. Falls through to `return authUser.businessIds.length > 0`

When `resource` is `undefined`, any user with at least one `businessId` is granted create access regardless of which business they are creating a tax rate for. This bypasses the intent of the businessId ownership check. The method signature makes `resource` optional (`resource?: ITaxRateDto`) but the policy's protection depends on the resource being present.

**Fix:** Either require the resource for `canCreate` (it should always be available) or explicitly fail closed when resource is absent:

```typescript
public canCreate(authUser: IUserDto, resource: ITaxRateDto): boolean {
  if (hasPermission(authUser, "view_support_dashboard")) {
    return true;
  }
  return authUser.businessIds.includes(resource.businessId);
}
```

### WR-03: PermissionGuard test suite only exercises supportRoles path — businessRoles path is untested

**File:** `trade-flow-api/src/auth/test/guards/permission.guard.spec.ts:25-44`
**Issue:** `buildUserWithPermissions` builds a user with permissions exclusively in `supportRoles`. There is no test case where a user with a `businessRole` containing the required permission is expected to pass the guard. Since `hasPermission` checks both `supportRoles` and `businessRoles`, the guard's behavior for business-role users is not covered.

If a future change accidentally broke `businessRoles` traversal in `hasPermission`, all guard tests would still pass. The `has-permission.utility.spec.ts` does test the businessRoles path at the utility level, but the guard integration test does not.

**Fix:** Add a `buildUserWithBusinessPermissions` helper and a corresponding test case:

```typescript
function buildUserWithBusinessPermissions(permissionNames: string[]): IUserDto {
  const permissions = permissionNames.map(buildPermission);
  const businessRole: IBusinessRoleDto = {
    id: "business_role_1",
    roleName: "owner",
    businessId: "biz_1",
    permissions: DtoCollection.create<IPermissionDto>(permissions),
  };
  return {
    id: "user_3",
    externalAuthUserId: "ext_3",
    nickname: null,
    name: null,
    email: "user3@test.com",
    businessIds: ["biz_1"],
    supportRoleIds: [],
    businessRoleIds: [businessRole.id],
    supportRoles: DtoCollection.empty<ISupportRoleDto>(),
    businessRoles: DtoCollection.create<IBusinessRoleDto>([businessRole]),
  };
}

it("should return true when user has the required permission via a businessRole", () => {
  reflector.getAllAndOverride.mockReturnValue(["manage_jobs"]);
  const user = buildUserWithBusinessPermissions(["manage_jobs"]);
  const context = createMockContext(user);
  expect(guard.canActivate(context)).toBe(true);
});
```

### WR-04: SubscriptionGuard silently passes unauthenticated requests on non-GET/HEAD mutation routes

**File:** `trade-flow-api/src/subscription/guards/subscription.guard.ts:43-46`
**Issue:** When `user` is undefined (no authenticated user on the request) and the method is POST/PUT/PATCH/DELETE, the guard returns `true` rather than failing closed. The comment says "let auth guard handle it," but this assumes the subscription guard always runs after the auth guard in the chain. If the guard order is misconfigured, unauthenticated mutation requests bypass the subscription check entirely.

```typescript
// current: open pass-through for unauthenticated users on mutations
if (!user) {
  return true;
}
```

**Fix:** The guard should fail closed on unauthenticated mutation requests. If the intent is that `JwtAuthGuard` runs first, that ordering should be enforced at the module level and documented. Alternatively:

```typescript
if (!user) {
  // If no user, this is an unauthenticated request on a non-public, non-GET route.
  // JwtAuthGuard should have already rejected this — but fail closed defensively.
  throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Authentication required");
}
```

---

## Info

### IN-01: RequiresPermission decorator accepts untyped string permissions — no compile-time safety

**File:** `trade-flow-api/src/auth/decorators/requires-permission.decorator.ts:5`
**Issue:** `RequiresPermission(...permissions: string[])` accepts any string. There is no type-safe permission enum or constant set. A typo in a permission name (e.g., `@RequiresPermission("managge_users")`) will silently fail at runtime — the guard will deny all users rather than raising an error during compilation or startup.

**Fix:** Define a `Permission` enum or const object in the permission DTO module and use it as the parameter type:

```typescript
// permission.dto.ts or a new permission-names.enum.ts
export const Permission = {
  MANAGE_USERS: "manage_users",
  VIEW_SUPPORT_DASHBOARD: "view_support_dashboard",
  MANAGE_MIGRATIONS: "manage_migrations",
  BYPASS_SUBSCRIPTION: "bypass_subscription",
} as const;

export type PermissionName = (typeof Permission)[keyof typeof Permission];

// decorator
export const RequiresPermission = (...permissions: PermissionName[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

### IN-02: BusinessPolicy.canUpdate test asserts "always returns false" — but the implementation does not

**File:** `trade-flow-api/src/business/test/policies/business.policy.spec.ts:126-138`
**Issue:** The `canUpdate` describe block contains a single test titled "should always return false" but `BusinessPolicy.canUpdate` actually returns `true` for support users and for users whose businessIds include the resource ID. The test passes a default user (with no businessIds and no support role), so the assertion happens to be correct, but the test description is wrong and the cases for `true` are not covered.

**Fix:** Rename the test and add the missing coverage cases:

```typescript
describe("canUpdate", () => {
  it("should return true when user is a support user", () => { ... });
  it("should return true when user owns the business", () => { ... });
  it("should return false when user does not own the business and is not support", () => { ... });
});
```

### IN-03: Redundant log messages in policy canUpdate methods reference "support user" in support-user log but use hasPermission

**File:** `trade-flow-api/src/item/policies/item.policy.ts:44`, `trade-flow-api/src/job/policies/job.policy.ts:44`, `trade-flow-api/src/job/policies/job-type.policy.ts:44`, `trade-flow-api/src/schedule/policies/schedule.policy.ts:44`, `trade-flow-api/src/visit-type/policies/visit-type.policy.ts:44`
**Issue:** The log messages on the permission-granted branch say `"Access granted: user is a support user"` but the actual check is `hasPermission(authUser, "view_support_dashboard")`, which could be satisfied by a business role. The message is misleading for non-support users who happen to hold this permission.

**Fix:** Replace the misleading log message with a permission-based description:

```typescript
this.logger.log("Access granted: user has view_support_dashboard permission");
```

---

_Reviewed: 2026-04-19T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
