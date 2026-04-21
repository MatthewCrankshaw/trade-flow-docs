---
phase: 52-permission-guard-migration
plan: "01"
subsystem: trade-flow-api
tags: [auth, rbac, permissions, guard, decorator]
dependency_graph:
  requires: [51-rbac-data-model-seed]
  provides: [hasPermission utility, hasAnyPermission utility, RequiresPermission decorator, PermissionGuard]
  affects: [trade-flow-api/src/auth]
tech_stack:
  added: []
  patterns: [NestJS CanActivate guard, SetMetadata decorator pattern, DtoCollection permission iteration]
key_files:
  created:
    - trade-flow-api/src/auth/utilities/has-permission.utility.ts
    - trade-flow-api/src/auth/utilities/has-any-permission.utility.ts
    - trade-flow-api/src/auth/decorators/requires-permission.decorator.ts
    - trade-flow-api/src/auth/guards/permission.guard.ts
    - trade-flow-api/src/auth/test/utilities/has-permission.utility.spec.ts
    - trade-flow-api/src/auth/test/guards/permission.guard.spec.ts
    - trade-flow-api/src/user/data-transfer-objects/permission.dto.ts
  modified:
    - trade-flow-api/src/auth/auth.module.ts
    - trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts
    - trade-flow-api/src/user/data-transfer-objects/business-role.dto.ts
decisions:
  - PermissionGuard applied per-endpoint via @UseGuards, not registered as APP_GUARD (avoids ordering issues with JwtAuthGuard)
  - hasPermission uses optional chaining on role.permissions to handle current state before Phase 51 adds the field as required
  - IPermissionDto created in this plan as a parallel wave-1 deviation (Phase 51 will supersede with full implementation)
  - permissions field added as optional to ISupportRoleDto and IBusinessRoleDto to allow compilation before Phase 51 merges
metrics:
  duration: 7 minutes
  completed: "2026-04-19"
  tasks_completed: 2
  files_changed: 10
---

# Phase 52 Plan 01: Permission Guard Infrastructure Summary

Permission guard and decorator infrastructure built: `RequiresPermission` decorator using SetMetadata, `PermissionGuard` implementing CanActivate with synchronous default-deny logic, `hasPermission`/`hasAnyPermission` utilities reading hydrated DtoCollection permissions from both supportRoles and businessRoles.

## What Was Built

### Task 1: hasPermission and hasAnyPermission utilities with tests

**`src/auth/utilities/has-permission.utility.ts`**
- Exports `hasPermission(user: IUserDto, permission: string): boolean`
- Iterates `user.supportRoles` then `user.businessRoles` using DtoCollection `.some()`
- Returns `false` by default (default-deny per T-52-02)
- Handles missing `permissions` field with `!!role.permissions &&` guard

**`src/auth/utilities/has-any-permission.utility.ts`**
- Exports `hasAnyPermission(user: IUserDto, permissions: string[]): boolean`
- OR logic: returns true if user has ANY of the listed permissions
- Delegates to `hasPermission` for each permission check

**`src/auth/test/utilities/has-permission.utility.spec.ts`**
- 9 tests covering: no roles, permission in supportRoles, permission in businessRoles, permission not found, empty DtoCollection, missing permissions field, OR logic (any), OR logic (none), empty permissions array

### Task 2: RequiresPermission decorator, PermissionGuard, and AuthModule registration

**`src/auth/decorators/requires-permission.decorator.ts`**
- `PERMISSIONS_KEY = "requiredPermissions"` constant
- `RequiresPermission(...permissions: string[])` variadic decorator using `SetMetadata`

**`src/auth/guards/permission.guard.ts`**
- `@Injectable()` class implementing `CanActivate`
- Synchronous `canActivate(context: ExecutionContext): boolean`
- Reads metadata via `Reflector.getAllAndOverride(PERMISSIONS_KEY, [handler, class])`
- Returns `true` when no `@RequiresPermission` metadata present
- Throws `ForbiddenError(ACTION_FORBIDDEN)` when user missing or lacks permissions
- Error details include the required permission name(s)

**`src/auth/auth.module.ts`**
- Added `PermissionGuard` to `providers` and `exports` arrays

**`src/auth/test/guards/permission.guard.spec.ts`**
- 6 tests: no metadata allows, valid permission allows, OR logic allows, missing permission rejects, no user rejects, error message contains permission name

## Deviations from Plan

### Auto-added Missing Critical Functionality

**1. [Rule 2 - Missing critical type] Created IPermissionDto interface**
- **Found during:** Task 1 — TypeScript compilation failure
- **Issue:** `ISupportRoleDto` and `IBusinessRoleDto` needed a `permissions` field typed as `DtoCollection<IPermissionDto>`, but `IPermissionDto` is a Phase 51 artifact that hadn't been created yet
- **Fix:** Created `src/user/data-transfer-objects/permission.dto.ts` with the interface `IPermissionDto extends IBaseResourceDto` (name, description, category fields) — matches exactly what Phase 51 will produce
- **Files modified:** `src/user/data-transfer-objects/permission.dto.ts` (new)
- **Commit:** e5b453f

**2. [Rule 2 - Missing critical type] Added optional permissions field to role DTOs**
- **Found during:** Task 1 — TypeScript compilation failure
- **Issue:** `ISupportRoleDto` and `IBusinessRoleDto` didn't have a `permissions` field, causing TypeScript errors in the utility and test files
- **Fix:** Added optional `permissions?: DtoCollection<IPermissionDto>` to both role DTOs. Phase 51 will make this required and add `permissionIds`/`isUnrestrictable` fields
- **Files modified:** `src/user/data-transfer-objects/support-role.dto.ts`, `src/user/data-transfer-objects/business-role.dto.ts`
- **Commit:** e5b453f

**3. [Rule 1 - Bug] Fixed Prettier formatting**
- **Found during:** CI gate lint check after Task 2
- **Issue:** `has-permission.utility.ts` line 12-14 had multi-line formatting for a callback that fit within 125 chars on one line
- **Fix:** Collapsed to single line `return user.businessRoles.some((role) => !!role.permissions && role.permissions.some((p) => p.name === permission));`
- **Files modified:** `src/auth/utilities/has-permission.utility.ts`
- **Commit:** 38c6f69

## Verification

All target tests pass:
- `npm run test -- --testPathPatterns="(permission.guard|has-permission)"` → 15 tests, 15 passed
- `npm run ci` → 854 tests passed, 0 errors, exit code 0

## Threat Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| T-52-01 Elevation of Privilege | PermissionGuard is default-deny: when `@RequiresPermission` metadata exists, user MUST have at least one required permission |
| T-52-02 Elevation of Privilege | `hasPermission` returns `false` for empty DtoCollection; `!!role.permissions &&` guard handles missing field; no user throws ForbiddenError |

## Self-Check: PASSED

Files exist:
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/utilities/has-permission.utility.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/utilities/has-any-permission.utility.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/decorators/requires-permission.decorator.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/guards/permission.guard.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/test/utilities/has-permission.utility.spec.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/auth/test/guards/permission.guard.spec.ts` — FOUND
- `/Users/mattc/gsd-workspaces/feature-branch/trade-flow-api/src/user/data-transfer-objects/permission.dto.ts` — FOUND

Commits exist:
- e5b453f: feat(52-01): add hasPermission and hasAnyPermission utilities with tests — FOUND
- 38c6f69: feat(52-01): add RequiresPermission decorator, PermissionGuard, and register in AuthModule — FOUND
