# Phase 52: Permission Guard & Migration - Research

**Researched:** 2026-04-18
**Domain:** NestJS guard/decorator infrastructure, permission-based authorization, hardcoded role check migration
**Confidence:** HIGH

## Summary

Phase 52 builds two things: (1) a `@RequiresPermission()` decorator and `PermissionGuard` that validate user permissions at the endpoint level, and (2) a comprehensive migration of all hardcoded role checks (`isSupportUser()`, `isSuperUser()`, `supportRoles.length > 0`) to use the permission system established in Phase 51. The codebase already has an exact template for this work -- the `@SkipSubscriptionCheck` decorator + `SubscriptionGuard` pattern using `SetMetadata` and `Reflector` -- so the guard/decorator implementation follows an established pattern with zero new libraries required.

The migration scope is larger than the success criteria suggest. A full grep reveals **15 policy files** using `isSupportUser()` across 12 modules (business, customer, job, job-type, quote, quote-line-item, quote-settings, schedule, visit-type, tax-rate, item, estimate, estimate-line-item, estimate-settings) plus **1 policy file** using `isSuperUser()` (migration) and **1 guard** using `supportRoles.length > 0` (SubscriptionGuard). The `hasPermission()` utility replaces all of these, but policy-level checks use a different semantic than guard-level checks -- policies check "is this a support user who can bypass ownership?" while the guard checks "does the user have a specific named permission?". The `hasPermission` utility must handle both use cases.

**Primary recommendation:** Build `@RequiresPermission` decorator and `PermissionGuard` following the `@SkipSubscriptionCheck`/`SubscriptionGuard` pattern exactly. Create `hasPermission()` and `hasAnyPermission()` utilities in `src/auth/utilities/`. Migrate SubscriptionGuard first (proving the guard works), then migrate all policy files from `isSupportUser()`/`isSuperUser()` to permission-based checks. Delete the old utilities last.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `@RequiresPermission` is a guard-level capability check that complements the existing Policy/AccessControllerFactory pattern. Two distinct authorization layers: guard checks "can you do this type of thing?" (capability), policy checks "can you access THIS specific resource?" (ownership). No changes to policies in this phase.
- **D-02:** `@RequiresPermission` supports multiple permissions with OR logic -- `@RequiresPermission('manage_users', 'view_support_dashboard')` means user needs ANY of the listed permissions. Uses `SetMetadata` to store required permissions, `PermissionGuard` reads them via `Reflector`.
- **D-03:** Guard execution order: `JwtAuthGuard -> PermissionGuard -> SubscriptionGuard`. Permission check needs `request.user` (from JwtAuth). Support users with `bypass_subscription` permission skip SubscriptionGuard naturally through the RBAC system.
- **D-04:** Delete `isSupportUser()` and `isSuperUser()` utilities entirely. All callers switch to checking specific permissions via a new `hasPermission()` utility function. No more role-name-based checks anywhere in the codebase.
- **D-05:** New `hasPermission(user: IUserDto, permission: string): boolean` utility function -- standalone in `src/user/utilities/` or `src/auth/utilities/`. Follows existing utility function pattern. Reads permissions from the hydrated role DTOs on the user object (set up in Phase 51).
- **D-06:** Migrate ALL hardcoded role checks across the entire codebase, not just the ones listed in success criteria (SubscriptionGuard, PaywallGuard). Find every `isSupportUser()`/`isSuperUser()` call site and migrate them all.
- **D-07:** Permission guard rejection reveals the missing permission name in the 403 error: `"Missing required permission: manage_users"` with `details: { required: ["manage_users"], type: "permission_denied" }`. Safe because the existing error handler already obfuscates error messages and details in production.
- **D-08:** Use the existing `ForbiddenError` class with a `permission_denied` type in details. No new error class needed -- `createHttpError()` already maps `ForbiddenError` to 403.
- **D-09:** Only add `@RequiresPermission` to support-specific endpoints (Phases 53-57). Existing business endpoints (jobs, quotes, customers) keep using policies for resource ownership. Permission decorators on business endpoints deferred to when team roles arrive.
- **D-10:** Build the guard infrastructure AND apply it to the migrated checks in this phase. The guard is proven working by end of phase, not just theoretical infrastructure. SubscriptionGuard bypass becomes a permission check (`bypass_subscription`) instead of role array length check.

### Claude's Discretion
- Whether `hasPermission()` lives in `src/user/utilities/` or `src/auth/utilities/`
- Whether to add a `hasAnyPermission(user, permissions[])` variant alongside `hasPermission()`
- Exact `SetMetadata` key name for the permission decorator
- Test structure and mock strategy for the PermissionGuard

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| RBAC-08 | Permission-checking guard/decorator infrastructure validates user permissions on API endpoints | `@RequiresPermission` decorator + `PermissionGuard` following `@SkipSubscriptionCheck`/`SubscriptionGuard` pattern; see Architecture Patterns |
| RBAC-09 | Existing hardcoded role checks (SubscriptionGuard support bypass, PaywallGuard support bypass) migrated to permission system | SubscriptionGuard `supportRoles.length > 0` replaced with `hasPermission(user, "bypass_subscription")`; no PaywallGuard exists -- only SubscriptionGuard; 15 policy files migrated from `isSupportUser()`/`isSuperUser()` to `hasPermission()` |
| RBAC-10 | Solo business users never see role management UI; permission infrastructure is entirely backend-enforced | No frontend changes in this phase; all permission enforcement is guard/policy level on API |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Permission guard | API / Backend | -- | NestJS guard intercepting HTTP requests before handler execution |
| Permission decorator | API / Backend | -- | NestJS decorator attaching metadata to route handlers |
| hasPermission utility | API / Backend | -- | Utility function reading hydrated permissions from user DTO |
| SubscriptionGuard migration | API / Backend | -- | Modifying existing global guard to use permission check |
| Policy migration | API / Backend | -- | Replacing `isSupportUser()`/`isSuperUser()` calls in policy classes |
| Error response | API / Backend | -- | ForbiddenError with permission_denied details |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/core | 11.1.19 | `Reflector`, `APP_GUARD`, `CanActivate`, `SetMetadata` | Already installed; provides all guard/decorator infrastructure [VERIFIED: codebase] |
| @nestjs/common | 11.1.12 | `Injectable`, `ExecutionContext`, `SetMetadata` | Already installed [VERIFIED: codebase] |

### Supporting
No new dependencies required. This phase uses only existing packages.

**Installation:**
```bash
# No new packages needed
```

## Architecture Patterns

### System Architecture Diagram

```
HTTP Request
  |
  v
JwtAuthGuard (existing, global via Passport)
  |  attaches request.user: IUserDto (with hydrated roles + permissions)
  v
PermissionGuard (NEW, applied per-endpoint via @UseGuards or globally via APP_GUARD)
  |  reads @RequiresPermission metadata via Reflector
  |  if no metadata -> allow (no permission required)
  |  if metadata -> check hasAnyPermission(user, requiredPermissions)
  |  if missing -> throw ForbiddenError with permission_denied details
  v
SubscriptionGuard (existing, global via APP_GUARD)
  |  MODIFIED: supportRoles.length > 0  -->  hasPermission(user, "bypass_subscription")
  v
Controller Handler
  |  calls Service
  v
Service (with Policy check via AccessControllerFactory)
  |  Policy.canRead/canWrite checks use hasPermission()
  |  instead of isSupportUser()/isSuperUser()
  v
Repository -> Database
```

### Recommended Project Structure

New and modified files:

```
src/
├── auth/
│   ├── decorators/
│   │   └── requires-permission.decorator.ts   # NEW
│   ├── guards/
│   │   └── permission.guard.ts                # NEW
│   ├── utilities/
│   │   ├── has-permission.utility.ts          # NEW
│   │   └── has-any-permission.utility.ts      # NEW (optional)
│   └── auth.module.ts                         # MODIFY: register PermissionGuard
├── subscription/
│   └── guards/
│       └── subscription.guard.ts              # MODIFY: use hasPermission
├── business/policies/business.policy.ts       # MODIFY: use hasPermission
├── customer/policies/customer.policy.ts       # MODIFY: use hasPermission
├── job/policies/job.policy.ts                 # MODIFY: use hasPermission
├── job/policies/job-type.policy.ts            # MODIFY: use hasPermission
├── quote/policies/quote.policy.ts             # MODIFY: use hasPermission
├── quote/policies/quote-line-item.policy.ts   # MODIFY: use hasPermission
├── quote-settings/policies/quote-settings.policy.ts   # MODIFY
├── schedule/policies/schedule.policy.ts       # MODIFY
├── visit-type/policies/visit-type.policy.ts   # MODIFY
├── tax-rate/policies/tax-rate.policy.ts       # MODIFY
├── item/policies/item.policy.ts               # MODIFY
├── estimate/policies/estimate.policy.ts       # MODIFY
├── estimate/policies/estimate-line-item.policy.ts   # MODIFY
├── estimate-settings/policies/estimate-settings.policy.ts   # MODIFY
├── migration/policies/migration.policy.ts     # MODIFY: use hasPermission
├── user/
│   ├── policies/support-role.policy.ts        # MODIFY: use hasPermission
│   └── utilities/
│       ├── is-support-user.utility.ts         # DELETE
│       └── is-super-user.utility.ts           # DELETE
└── auth/
    └── test/
        ├── guards/
        │   └── permission.guard.spec.ts       # NEW
        └── utilities/
            └── has-permission.utility.spec.ts # NEW
```

### Pattern 1: RequiresPermission Decorator

**What:** Custom decorator that stores required permission names as metadata on the route handler
**When to use:** On controller endpoints that require specific permissions (Phases 53-57 will use this)

```typescript
// Source: @SkipSubscriptionCheck decorator pattern [VERIFIED: codebase]
// File: src/auth/decorators/requires-permission.decorator.ts

import { SetMetadata } from "@nestjs/common";

export const PERMISSIONS_KEY = "requiredPermissions";

export const RequiresPermission = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

### Pattern 2: PermissionGuard

**What:** Guard that reads required permissions from metadata and checks user has at least one
**When to use:** Applied alongside `@RequiresPermission` decorator

```typescript
// Source: SubscriptionGuard pattern [VERIFIED: codebase]
// File: src/auth/guards/permission.guard.ts

import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { PERMISSIONS_KEY } from "@auth/decorators/requires-permission.decorator";
import { ForbiddenError } from "@core/errors/forbidden-error.error";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { hasAnyPermission } from "@auth/utilities/has-any-permission.utility";

@Injectable()
export class PermissionGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions || requiredPermissions.length === 0) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user as IUserDto | undefined;

    if (!user) {
      throw new ForbiddenError(
        ErrorCodes.ACTION_FORBIDDEN,
        `Missing required permission: ${requiredPermissions.join(", ")}`,
      );
    }

    if (!hasAnyPermission(user, requiredPermissions)) {
      throw new ForbiddenError(
        ErrorCodes.ACTION_FORBIDDEN,
        `Missing required permission: ${requiredPermissions.join(", ")}`,
      );
    }

    return true;
  }
}
```

### Pattern 3: hasPermission / hasAnyPermission Utilities

**What:** Standalone utility functions that check if a user has specific permissions via hydrated role DTOs
**When to use:** In policies, guards, and service-level authorization checks

```typescript
// Source: isSupportUser() utility pattern [VERIFIED: codebase]
// File: src/auth/utilities/has-permission.utility.ts

import { IUserDto } from "@user/data-transfer-objects/user.dto";

export function hasPermission(user: IUserDto, permission: string): boolean {
  for (const role of user.supportRoles) {
    if (role.permissions && role.permissions.some((p) => p.name === permission)) {
      return true;
    }
  }

  for (const role of user.businessRoles) {
    if (role.permissions && role.permissions.some((p) => p.name === permission)) {
      return true;
    }
  }

  return false;
}
```

```typescript
// File: src/auth/utilities/has-any-permission.utility.ts

import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { hasPermission } from "@auth/utilities/has-permission.utility";

export function hasAnyPermission(user: IUserDto, permissions: string[]): boolean {
  return permissions.some((permission) => hasPermission(user, permission));
}
```

### Pattern 4: Policy Migration (isSupportUser -> hasPermission)

**What:** Replacing role-name-based checks with permission-based checks in all policy files
**When to use:** All 15 policy files that currently use `isSupportUser()` or `isSuperUser()`

The key insight: in the current codebase, `isSupportUser()` means "this user has a support role and can bypass resource-ownership checks." In the new system, support roles carry permissions. The policy check becomes: "does this user have ANY permission from a support role?" which is equivalent to checking if they have a support-scoped permission.

**Migration approach for policies:**

Per D-01, policies still handle resource-ownership authorization. The `isSupportUser()` call in policies means "support users can access all resources regardless of ownership." This translates to checking if the user has any support-level permission (e.g., `view_support_dashboard`) since all support roles carry this permission by Phase 51's seeding.

```typescript
// BEFORE (current)
import { isSupportUser } from "@user/utilities/is-support-user.utility";

export class BusinessPolicy extends BasePolicy<IBusinessDto> {
  public canRead(authUser: IUserDto, resource: IBusinessDto): boolean {
    if (isSupportUser(authUser)) {
      return true;
    }
    if (authUser.businessIds.includes(resource.id)) {
      return true;
    }
    return false;
  }
}

// AFTER (migrated)
import { hasPermission } from "@auth/utilities/has-permission.utility";

export class BusinessPolicy extends BasePolicy<IBusinessDto> {
  public canRead(authUser: IUserDto, resource: IBusinessDto): boolean {
    if (hasPermission(authUser, "view_support_dashboard")) {
      return true;
    }
    if (authUser.businessIds.includes(resource.id)) {
      return true;
    }
    return false;
  }
}
```

**Migration for `isSuperUser()` in MigrationPolicy:**

```typescript
// BEFORE
import { isSuperUser } from "@user/utilities/is-super-user.utility";

public canCreate(authUser: IUserDto): boolean {
  return isSuperUser(authUser);
}

// AFTER
import { hasPermission } from "@auth/utilities/has-permission.utility";

public canCreate(authUser: IUserDto): boolean {
  return hasPermission(authUser, "manage_migrations");
}
```

Note: `manage_migrations` may need to be added to the Phase 51 permission seeds if not already present. The MigrationPolicy restricts access to super users only, so this permission should only be on the Super User role. [ASSUMED]

### Pattern 5: SubscriptionGuard Migration

**What:** Replacing the `supportRoles.length > 0` check with a permission check
**When to use:** In SubscriptionGuard, the global guard that enforces subscription requirements

```typescript
// BEFORE (current)
if (user.supportRoles && user.supportRoles.length > 0) {
  return true;
}

// AFTER (migrated)
import { hasPermission } from "@auth/utilities/has-permission.utility";

if (hasPermission(user, "bypass_subscription")) {
  return true;
}
```

### Anti-Patterns to Avoid
- **Registering PermissionGuard as APP_GUARD:** D-03 specifies guard ordering `JwtAuthGuard -> PermissionGuard -> SubscriptionGuard`. Since SubscriptionGuard is already a global `APP_GUARD`, adding PermissionGuard as another `APP_GUARD` requires careful provider ordering in `AppModule`. However, per D-09, `@RequiresPermission` is only applied to support-specific endpoints (Phases 53-57), so the guard should run per-endpoint via `@UseGuards(PermissionGuard)` rather than globally. This avoids ordering concerns entirely.
- **Modifying the Policy/AccessControllerFactory pattern:** D-01 explicitly says no changes to the policy mechanism itself. The migration only changes what the policies call internally (permission checks instead of role name checks).
- **Adding @RequiresPermission to business endpoints:** D-09 defers this to team roles. Only infrastructure + migration in this phase.
- **Creating a new error class:** D-08 specifies using the existing `ForbiddenError` with `ErrorCodes.ACTION_FORBIDDEN`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Metadata storage for decorators | Custom metadata solution | `SetMetadata` from `@nestjs/common` | Standard NestJS pattern, used by `@SkipSubscriptionCheck` already [VERIFIED: codebase] |
| Metadata retrieval in guards | Manual metadata parsing | `Reflector.getAllAndOverride()` from `@nestjs/core` | Standard NestJS pattern, used by `SubscriptionGuard` already [VERIFIED: codebase] |
| Permission error responses | New error class | `ForbiddenError` with `ErrorCodes.ACTION_FORBIDDEN` | Already maps to 403 via `createHttpError()` [VERIFIED: codebase] |
| Guard infrastructure | Custom interceptor/middleware | `CanActivate` interface | Standard NestJS guard contract [VERIFIED: codebase] |

## Complete Migration Inventory

### Files Using `isSupportUser()` (15 files -- all are policies)

| File | Module | Call Count | Migration Target Permission |
|------|--------|------------|----------------------------|
| `src/business/policies/business.policy.ts` | Business | 2 (canRead, canUpdate) | `view_support_dashboard` |
| `src/customer/policies/customer.policy.ts` | Customer | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/job/policies/job.policy.ts` | Job | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/job/policies/job-type.policy.ts` | Job | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/quote/policies/quote.policy.ts` | Quote | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/quote/policies/quote-line-item.policy.ts` | Quote | 2 (canCreate, canRead) | `view_support_dashboard` |
| `src/quote-settings/policies/quote-settings.policy.ts` | QuoteSettings | 2 (canRead, canUpdate) | `view_support_dashboard` |
| `src/schedule/policies/schedule.policy.ts` | Schedule | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/visit-type/policies/visit-type.policy.ts` | VisitType | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/tax-rate/policies/tax-rate.policy.ts` | TaxRate | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/item/policies/item.policy.ts` | Item | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/estimate/policies/estimate.policy.ts` | Estimate | 3 (canCreate, canRead, canUpdate) | `view_support_dashboard` |
| `src/estimate/policies/estimate-line-item.policy.ts` | EstimateLineItem | 2 (canCreate, canRead) | `view_support_dashboard` |
| `src/estimate-settings/policies/estimate-settings.policy.ts` | EstimateSettings | 2 (canRead, canUpdate) | `view_support_dashboard` |
| `src/user/policies/support-role.policy.ts` | User | 1 (canRead) | `view_support_dashboard` |

### Files Using `isSuperUser()` (1 file)

| File | Module | Call Count | Migration Target Permission |
|------|--------|------------|----------------------------|
| `src/migration/policies/migration.policy.ts` | Migration | 4 (all CRUD) | `manage_migrations` (super user only) |

### Files Using `supportRoles.length > 0` (1 file)

| File | Module | Call Count | Migration Target Permission |
|------|--------|------------|----------------------------|
| `src/subscription/guards/subscription.guard.ts` | Subscription | 1 | `bypass_subscription` |

### Files to DELETE (2 files)

| File | Replaced By |
|------|-------------|
| `src/user/utilities/is-support-user.utility.ts` | `hasPermission()` in `src/auth/utilities/` |
| `src/user/utilities/is-super-user.utility.ts` | `hasPermission()` in `src/auth/utilities/` |

### Tests to UPDATE

| File | What Changes |
|------|-------------|
| `src/subscription/test/guards/subscription.guard.spec.ts` | Support user test must use user with `bypass_subscription` permission instead of just `supportRoles.length > 0` |
| All policy spec files referencing `isSupportUser`/`isSuperUser` | Update mocks to use users with permissions |

### PaywallGuard Note

The CONTEXT.md references PaywallGuard but no `PaywallGuard` exists in the codebase. The `SubscriptionGuard` is the only guard performing subscription/paywall checks. This was confirmed via codebase grep. [VERIFIED: codebase]

## Common Pitfalls

### Pitfall 1: Guard Ordering When Using APP_GUARD
**What goes wrong:** `PermissionGuard` executes before `JwtAuthGuard`, so `request.user` is undefined and guard throws 403 instead of 401.
**Why it happens:** `APP_GUARD` providers execute in registration order in `AppModule.providers`. If `PermissionGuard` is registered before `JwtAuthGuard` resolves, it runs first.
**How to avoid:** Do NOT register `PermissionGuard` as `APP_GUARD`. Apply it per-endpoint via `@UseGuards(PermissionGuard)` alongside `@RequiresPermission`. This matches D-09 (only support endpoints get permission decorators). The guard should handle the case where no `@RequiresPermission` metadata exists by returning `true` (no restriction).
**Warning signs:** 403 errors on endpoints that should return 401 for unauthenticated requests.

### Pitfall 2: DtoCollection Iteration for Permission Checking
**What goes wrong:** `role.permissions.some(...)` fails because `permissions` is a `DtoCollection<IPermissionDto>`, not an array.
**Why it happens:** Phase 51 adds `permissions: DtoCollection<IPermissionDto>` to role DTOs. `DtoCollection` has a `.some()` method but it delegates to `this._data.some()` internally.
**How to avoid:** `DtoCollection` already implements `.some()` [VERIFIED: codebase]. The `hasPermission()` utility can safely call `role.permissions.some((p) => p.name === permission)`. However, be aware that `DtoCollection` might be empty (`DtoCollection.empty()`) for roles without permissions -- `.some()` on an empty collection correctly returns `false`.
**Warning signs:** TypeScript errors about `some` not existing on type.

### Pitfall 3: Missing Permission Seeds for Migration
**What goes wrong:** Policy migration changes `isSuperUser()` to `hasPermission(user, "manage_migrations")` but `manage_migrations` is not in the Phase 51 permission seeds, so the check always returns `false` and migration endpoints become inaccessible.
**Why it happens:** Phase 51's permission seed list was designed before the full migration scope was known.
**How to avoid:** Audit the Phase 51 permission seeds against all migration targets before implementation. If `manage_migrations` is not seeded, add it in this phase's seeder update. All support roles used in the `isSupportUser()` context need `view_support_dashboard` permission seeded on both Super User and Admin roles.
**Warning signs:** Endpoints that previously worked for support/super users return 403 after migration.

### Pitfall 4: Test Mocks Missing Permission Structure
**What goes wrong:** Existing policy tests create mock users with `supportRoles` containing simple role DTOs without `permissions` field. After migration, `hasPermission()` reads `role.permissions.some(...)` which fails or returns `false` on mocks without permissions.
**Why it happens:** Mock generators were created before Phase 51 added permissions to role DTOs.
**How to avoid:** Update mock generators to include `permissions: DtoCollection<IPermissionDto>` on role DTOs. Create a helper that builds a support user mock with standard support permissions.
**Warning signs:** Tests that previously tested support user access now fail.

### Pitfall 5: Path Alias for New Auth Utilities
**What goes wrong:** `import { hasPermission } from "@auth/utilities/has-permission.utility"` fails because `@auth/*` path alias only maps to `src/auth/*`.
**Why it happens:** The `@auth/*` alias is already configured in `tsconfig.json` and `jest` moduleNameMapper.
**How to avoid:** Verify that `src/auth/utilities/` directory is within the `@auth/*` alias scope. It should be, since the alias maps to `src/auth/*`. [VERIFIED: codebase -- `@auth/*` maps to `src/auth/*`]
**Warning signs:** Module resolution errors during compilation or test runs.

## Code Examples

### Error Response Structure

```typescript
// Source: D-07, D-08 from CONTEXT.md [VERIFIED: user decisions]
// Source: ForbiddenError class [VERIFIED: codebase]

// When PermissionGuard rejects:
throw new ForbiddenError(
  ErrorCodes.ACTION_FORBIDDEN,
  `Missing required permission: ${requiredPermissions.join(", ")}`,
);

// This produces via createHttpError():
// HTTP 403
// {
//   "errors": [{
//     "code": "CORE_7",
//     "message": "Action forbidden",
//     "details": "Missing required permission: manage_users"
//   }]
// }
```

Note: D-07 mentions `details: { required: ["manage_users"], type: "permission_denied" }` as an object, but `ForbiddenError.getDetails()` returns a `string` type [VERIFIED: codebase]. Either the details field needs to remain a string (matching current pattern) or `ForbiddenError` needs modification. Recommendation: keep it as a string `"Missing required permission: manage_users"` to match the existing error pattern. The structured object format would require changing the error class interface. [ASSUMED -- needs user confirmation]

### Complete Guard Registration

```typescript
// Source: AppModule APP_GUARD pattern [VERIFIED: codebase]
// File: src/app.module.ts -- NO CHANGES needed if PermissionGuard is per-endpoint

// PermissionGuard is NOT registered as APP_GUARD.
// Instead, it is applied per-endpoint:

@UseGuards(JwtAuthGuard, PermissionGuard)
@RequiresPermission("manage_users")
@Get("support/users")
public async listUsers() { ... }
```

### Mock User with Permissions for Tests

```typescript
// Source: SubscriptionGuard test pattern [VERIFIED: codebase]
// Source: Phase 51 permission/role DTO structure [VERIFIED: Phase 51 research]

import { DtoCollection } from "@core/collections/dto.collection";
import { IPermissionDto } from "@user/data-transfer-objects/permission.dto";
import { ISupportRoleDto } from "@user/data-transfer-objects/support-role.dto";

function createSupportUserWithPermissions(permissions: string[]): IUserDto {
  const permissionDtos: IPermissionDto[] = permissions.map((name, index) => ({
    id: `perm_${index}`,
    name,
    description: `Test permission: ${name}`,
    category: "support",
  }));

  const supportRole: ISupportRoleDto = {
    id: "role_super_user",
    roleName: "super_user",
    permissionIds: permissionDtos.map((p) => p.id),
    permissions: DtoCollection.create<IPermissionDto>(permissionDtos),
  };

  return {
    id: "user_1",
    externalAuthUserId: "ext_1",
    nickname: "Test Support",
    name: "Test Support User",
    email: "support@test.com",
    businessIds: [],
    supportRoleIds: [supportRole.id],
    businessRoleIds: [],
    supportRoles: DtoCollection.create<ISupportRoleDto>([supportRole]),
    businessRoles: DtoCollection.empty(),
  };
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `isSupportUser()` checking role names | `hasPermission(user, "permission_name")` checking hydrated permissions | This phase (v1.9) | Granular, permission-based authorization |
| `isSuperUser()` checking role name | `hasPermission(user, "manage_migrations")` | This phase (v1.9) | Super user identified by permissions, not role name |
| `supportRoles.length > 0` in SubscriptionGuard | `hasPermission(user, "bypass_subscription")` | This phase (v1.9) | Subscription bypass is a specific permission, not implicit to having any role |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `manage_migrations` permission may need to be added to Phase 51 seeds | Pattern 4 (Policy Migration) | HIGH -- migration endpoints would be locked out if permission doesn't exist |
| A2 | `view_support_dashboard` is the right permission for policy support-user bypass | Complete Migration Inventory | MEDIUM -- could be a different permission; needs to exist on both Super User and Admin roles |
| A3 | ForbiddenError details should remain a string (not structured object) to match existing pattern | Code Examples > Error Response | LOW -- D-07 mentions structured object but existing code uses strings; easily adjusted |
| A4 | PermissionGuard should be per-endpoint not global APP_GUARD | Anti-Patterns | LOW -- D-09 confirms only support endpoints get decorators; global would add overhead to all endpoints |

## Open Questions

1. **Should the `details` field in the ForbiddenError be a string or structured object?**
   - What we know: D-07 specifies `details: { required: ["manage_users"], type: "permission_denied" }` but `ForbiddenError.getDetails()` returns `string` and all existing usages pass strings.
   - What's unclear: Whether to change `ForbiddenError` to accept objects or keep string format.
   - Recommendation: Keep string format `"Missing required permission: manage_users"` to avoid breaking existing error handling. The permission name is included in the message either way.

2. **Does Phase 51 seed a `manage_migrations` permission?**
   - What we know: Phase 51 research lists ~19 permissions. It does NOT include `manage_migrations`.
   - What's unclear: Whether this should be added in Phase 51 or Phase 52.
   - Recommendation: Add it in Phase 52 as a seeder update. The seeder is idempotent, so adding new permissions is safe. Alternatively, the MigrationPolicy could use an existing permission like `manage_roles` since only super users access migrations.

3. **Which specific permission should replace `isSupportUser()` in policies?**
   - What we know: All support roles (Super User and Admin) have `view_support_dashboard`. Using this as the "is support user" proxy works if all support roles carry it.
   - What's unclear: Whether a dedicated `access_all_resources` or similar meta-permission would be cleaner.
   - Recommendation: Use `view_support_dashboard` -- it is seeded on both support roles and semantically means "this user has support-level access." A meta-permission adds complexity without value.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | `jest` config in `package.json` (trade-flow-api) |
| Quick run command | `npm run test -- --testPathPattern=permission` |
| Full suite command | `npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RBAC-08 | PermissionGuard allows when user has required permission | unit | `npm run test -- --testPathPattern=permission.guard` | No -- Wave 0 |
| RBAC-08 | PermissionGuard rejects with 403 when user lacks permission | unit | `npm run test -- --testPathPattern=permission.guard` | No -- Wave 0 |
| RBAC-08 | PermissionGuard allows when no @RequiresPermission metadata | unit | `npm run test -- --testPathPattern=permission.guard` | No -- Wave 0 |
| RBAC-08 | hasPermission checks both support and business roles | unit | `npm run test -- --testPathPattern=has-permission` | No -- Wave 0 |
| RBAC-09 | SubscriptionGuard uses bypass_subscription permission | unit | `npm run test -- --testPathPattern=subscription.guard` | Yes -- needs update |
| RBAC-09 | Policy files use hasPermission instead of isSupportUser | unit | `npm run test -- --testPathPattern=policy` | Partial -- existing policy tests need update |
| RBAC-10 | No frontend changes | manual-only | N/A | N/A -- verification by absence |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=<changed-file>`
- **Per wave merge:** `npm run ci`
- **Phase gate:** Full `npm run ci` green before verification

### Wave 0 Gaps
- [ ] `src/auth/test/guards/permission.guard.spec.ts` -- covers RBAC-08 guard behavior
- [ ] `src/auth/test/utilities/has-permission.utility.spec.ts` -- covers RBAC-08 utility
- [ ] Update `src/subscription/test/guards/subscription.guard.spec.ts` -- covers RBAC-09

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Existing Firebase JWT -- unchanged |
| V3 Session Management | No | Existing guard pattern -- unchanged |
| V4 Access Control | Yes | PermissionGuard + hasPermission utility; policy migration maintains existing authorization |
| V5 Input Validation | No | No new user input endpoints |
| V6 Cryptography | No | No cryptographic operations |

### Known Threat Patterns for This Phase

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Permission bypass if guard not applied | Elevation of Privilege | Guard runs per-endpoint via `@UseGuards`; endpoints without `@RequiresPermission` are unaffected (guard returns `true`) |
| Privilege escalation if permission check reads empty collection as "all access" | Elevation of Privilege | `hasPermission()` returns `false` for empty/missing permission collections; default-deny |
| Regression in existing authorization after migration | Elevation of Privilege | Every `isSupportUser()` call site migrated to equivalent permission check; tests updated to verify |
| Broken support access after deleting utilities | Denial of Service | Delete utilities only after all call sites migrated; CI will catch missing imports |

## Sources

### Primary (HIGH confidence)
- Codebase: `trade-flow-api/src/subscription/guards/subscription.guard.ts` -- guard pattern, `Reflector` usage, support bypass logic [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts` -- `SetMetadata` decorator pattern [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/user/utilities/is-support-user.utility.ts` -- current role-name check to replace [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/user/utilities/is-super-user.utility.ts` -- current role-name check to replace [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/core/errors/forbidden-error.error.ts` -- error class for 403 responses [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/core/errors/error-codes.enum.ts` -- `ACTION_FORBIDDEN = "CORE_7"` [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/app.module.ts` -- `APP_GUARD` registration for SubscriptionGuard [VERIFIED: codebase]
- Codebase: `trade-flow-api/src/core/collections/dto.collection.ts` -- `DtoCollection.some()` method exists [VERIFIED: codebase]
- Codebase: Full grep of `isSupportUser`/`isSuperUser` call sites -- 17 files total [VERIFIED: codebase]

### Secondary (MEDIUM confidence)
- Phase 51 Research: Permission/role DTO structure, hydration strategy, permission names [VERIFIED: planning docs]

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new packages, all patterns verified in codebase
- Architecture: HIGH -- exact template exists in `@SkipSubscriptionCheck`/`SubscriptionGuard` pattern
- Pitfalls: HIGH -- all identified from direct codebase analysis and migration inventory
- Migration scope: HIGH -- complete grep of all call sites, every file identified

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable -- no external dependency changes expected)
