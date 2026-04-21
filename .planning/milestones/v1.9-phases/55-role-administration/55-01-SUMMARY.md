---
phase: 55-role-administration
plan: "01"
subsystem: trade-flow-api
tags: [rbac, role-administration, support-role, permission-guard, nestjs, mongodb]
dependency_graph:
  requires: [52-permission-guard-migration, 51-rbac-data-model-seed]
  provides:
    - SupportRoleAssigner service with grant() and revoke() methods
    - POST /v1/users/:id/support-role endpoint (grant admin role)
    - DELETE /v1/users/:id/support-role endpoint (revoke admin role)
    - findByRoleName() on SupportRoleRepository
    - addSupportRoleId() / removeSupportRoleId() / countBySupportRoleId() on UserRepository
    - PermissionGuard infrastructure (Phase 52 prerequisite, Rule 3)
  affects:
    - trade-flow-api/src/user/repositories/support-role.repository.ts
    - trade-flow-api/src/user/repositories/user.repository.ts
    - trade-flow-api/src/user/services/support-role-assigner.service.ts
    - trade-flow-api/src/user/controllers/support-role-admin.controller.ts
    - trade-flow-api/src/user/user.module.ts
    - trade-flow-api/src/auth/ (PermissionGuard, RequiresPermission, hasPermission)
tech_stack:
  added: []
  patterns:
    - SupportRoleAssigner mirrors BusinessRoleAssigner pattern
    - Atomic $addToSet/$pull via as unknown as UpdateFilter<UserEntity> cast (MongoDB driver type limitation)
    - PermissionGuard via @UseGuards(JwtAuthGuard, PermissionGuard) + @RequiresPermission()
    - Self-revocation guard + super-user protection + last-admin count check in revoke()
key_files:
  created:
    - trade-flow-api/src/user/services/support-role-assigner.service.ts
    - trade-flow-api/src/user/controllers/support-role-admin.controller.ts
    - trade-flow-api/src/user/test/services/support-role-assigner.service.spec.ts
    - trade-flow-api/src/user/test/controllers/support-role-admin.controller.spec.ts
    - trade-flow-api/src/auth/guards/permission.guard.ts
    - trade-flow-api/src/auth/decorators/requires-permission.decorator.ts
    - trade-flow-api/src/auth/utilities/has-permission.utility.ts
    - trade-flow-api/src/auth/utilities/has-any-permission.utility.ts
  modified:
    - trade-flow-api/src/user/repositories/support-role.repository.ts
    - trade-flow-api/src/user/repositories/user.repository.ts
    - trade-flow-api/src/user/user.module.ts
    - trade-flow-api/src/auth/auth.module.ts
decisions:
  - Used as unknown as UpdateFilter<UserEntity> cast for $addToSet and $pull operators due to MongoDB driver's strict SetFields<T> typing (acceptable per CLAUDE.md type assertion guidance for third-party SDK limitations)
  - countBySupportRoleId uses findMany().length since MongoDbFetcher has no count() method
  - UserRetriever.getById() used (not findByIdOrFail) with manual null check since findByIdOrFail does not exist on UserRetriever
  - SupportRoleAssigner uses UserRetriever rather than UserRepository for hydrated role access (needed for roleName checks)
  - Phase 52 PermissionGuard infrastructure created here as Rule 3 (blocking issue) since code was not previously applied to the api codebase
metrics:
  duration: "~25 minutes"
  completed: "2026-04-19T07:20:00Z"
  tasks_completed: 2
  files_created: 8
  files_modified: 4
---

# Phase 55 Plan 01: Role Administration Backend Summary

**One-liner:** SupportRoleAssigner service with grant/revoke methods, atomic repository operations, dedicated controller at POST/DELETE v1/users/:id/support-role protected by manage_roles permission, with self-revocation, super-user protection, and last-admin count guardrails.

## What Was Built

### Task 1: Repository methods and SupportRoleAssigner service

**`src/user/repositories/support-role.repository.ts`**
- Added `findByRoleName(name: SupportRoleName): Promise<ISupportRoleDto | null>` — looks up a support role by its enum name (used to find the seeded Admin role for grant operations)

**`src/user/repositories/user.repository.ts`**
- Added `addSupportRoleId(userId, roleId): Promise<void>` — atomic `$addToSet` to prevent duplicate role IDs
- Added `removeSupportRoleId(userId, roleId): Promise<void>` — atomic `$pull` for clean removal
- Added `countBySupportRoleId(roleId): Promise<number>` — counts users holding a given role (used for last-admin protection)
- MongoDB driver strict `SetFields<T>` typing required `as unknown as UpdateFilter<UserEntity>` cast (acceptable SDK limitation per CLAUDE.md)

**`src/user/services/support-role-assigner.service.ts`**
- `grant(authUser, targetUserId)`: fetches target user, rejects if already has support role, finds Admin role by name, adds role ID atomically, returns hydrated user
- `revoke(authUser, targetUserId)`: self-revocation guard (ForbiddenError "self_revocation"), super-user guard (ForbiddenError "system_managed_role"), last-admin count check (ForbiddenError "last_admin_protection"), removes Admin role ID atomically
- 11 unit tests covering all guardrails and success paths

### Task 2: SupportRoleAdminController and module registration

**`src/user/controllers/support-role-admin.controller.ts`**
- `POST :id/support-role` → `grant()` — returns `IResponse<IUserResponse>` with hydrated user
- `DELETE :id/support-role` → `revoke()` — returns `IResponse<IUserResponse>` with hydrated user
- `@UseGuards(JwtAuthGuard, PermissionGuard)` + `@RequiresPermission("manage_roles")` on both endpoints
- `mapToResponse()` replicates UserController pattern — maps supportRoles and businessRoles to response shape
- 5 unit tests covering success paths, error delegation, and response mapping

**`src/user/user.module.ts`**
- Added `SupportRoleAdminController` to controllers array
- Added `SupportRoleAssigner` to providers and exports arrays

### Phase 52 Prerequisite (Rule 3 — Blocking Issue)

**`src/auth/utilities/has-permission.utility.ts`**
- `hasPermission(user, permission)` — iterates supportRoles and businessRoles DtoCollection

**`src/auth/utilities/has-any-permission.utility.ts`**
- `hasAnyPermission(user, permissions[])` — OR logic over hasPermission

**`src/auth/decorators/requires-permission.decorator.ts`**
- `RequiresPermission(...permissions)` decorator using SetMetadata with PERMISSIONS_KEY

**`src/auth/guards/permission.guard.ts`**
- `PermissionGuard` implementing CanActivate — default-deny, reads metadata via Reflector, throws ForbiddenError when permission missing

**`src/auth/auth.module.ts`**
- Added `PermissionGuard` to providers and exports

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking Issue] Created Phase 52 PermissionGuard infrastructure**
- **Found during:** Task 2 — SupportRoleAdminController imports PermissionGuard which did not exist
- **Issue:** Phase 52 SUMMARY was documented but the actual code files (PermissionGuard, RequiresPermission, hasPermission utilities) had not been applied to the api codebase. The auth directory only contained JwtAuthGuard.
- **Fix:** Created all Phase 52 auth infrastructure files: `has-permission.utility.ts`, `has-any-permission.utility.ts`, `requires-permission.decorator.ts`, `permission.guard.ts`, updated `auth.module.ts`
- **Files modified:** 5 files created + auth.module.ts updated

**2. [Rule 1 - Bug] Fixed MongoDB driver strict typing for $addToSet/$pull**
- **Found during:** Task 1 — TypeScript compilation failure
- **Issue:** MongoDB driver's `SetFields<UserEntity>` type rejects direct `ObjectId` assignment in `$addToSet` and `PullOperator<UserEntity>` rejects `ObjectId` in `$pull` due to strict type constraints on array field updates
- **Fix:** Used `as unknown as UpdateFilter<UserEntity>` cast — acceptable per CLAUDE.md "acceptable uses: third-party SDK type limitations"
- **Files modified:** `user.repository.ts`

**3. [Rule 1 - Bug] Fixed two pre-existing prettier formatting errors**
- **Found during:** Task 2 CI gate
- **Issue:** `support-user.repository.ts` and `support-dashboard-metrics.service.ts` had pre-existing prettier formatting errors that blocked CI
- **Fix:** Ran `prettier --write` on both files
- **Files modified:** `src/support/repositories/support-user.repository.ts`, `src/support/services/support-dashboard-metrics.service.ts`

**4. [Rule 2 - Design adaptation] Used UserRetriever.getById() instead of findByIdOrFail()**
- **Found during:** Task 1 — UserRetriever has no findByIdOrFail method
- **Issue:** Plan referenced `userRetriever.findByIdOrFail()` but UserRetriever only has `getById()` returning `IUserDto | null`
- **Fix:** Used `getById()` with explicit null check throwing `ResourceNotFoundError`

## Threat Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| T-55-01 Elevation of Privilege (grant) | `@RequiresPermission("manage_roles")` via PermissionGuard enforced at controller level |
| T-55-02 Elevation of Privilege (revoke) | `@RequiresPermission("manage_roles")` + self_revocation guard + system_managed_role guard + last_admin_protection count check |
| T-55-03 Tampering (param :id) | UserRetriever.getById() throws ResourceNotFoundError for non-existent users; MongoDB ObjectId constructor throws on invalid strings |
| T-55-04 Denial of Service (last admin) | Self-revocation blocked by ForbiddenError; last-admin count check blocks revocation when ≤1 super user remains; super user role cannot be revoked via API |

## Verification

- `npm run test -- --testPathPatterns=support-role-assigner`: 11 tests, 11 passed
- `npm run test -- --testPathPatterns=support-role-admin.controller`: 5 tests, 5 passed
- `npm run test -- --no-coverage`: 107 suites, 909 tests, all passed
- `npm run lint:check`: 0 errors, 33 warnings (pre-existing)
- `npm run format:check`: all files pass
- `npm run typecheck`: 0 errors

## Self-Check: PASSED

Files exist:
- `trade-flow-api/src/user/services/support-role-assigner.service.ts` — FOUND
- `trade-flow-api/src/user/controllers/support-role-admin.controller.ts` — FOUND
- `trade-flow-api/src/user/test/services/support-role-assigner.service.spec.ts` — FOUND
- `trade-flow-api/src/user/test/controllers/support-role-admin.controller.spec.ts` — FOUND
- `trade-flow-api/src/auth/guards/permission.guard.ts` — FOUND
- `trade-flow-api/src/auth/decorators/requires-permission.decorator.ts` — FOUND
- `.planning/phases/55-role-administration/55-01-SUMMARY.md` — FOUND
