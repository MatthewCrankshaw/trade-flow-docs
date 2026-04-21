---
phase: 51-rbac-data-model-seed
plan: "02"
subsystem: trade-flow-api
tags:
  - rbac
  - permissions
  - seeder
  - onModuleInit
  - repository
  - testing

dependency_graph:
  requires:
    - phase: 51-01
      provides: PermissionName enum, IPermissionDto, PermissionRepository, extended ISupportRoleDto/IBusinessRoleDto with permissionIds and permissions fields
  provides:
    - RbacSeeder service with OnModuleInit lifecycle (idempotent permissions/roles/super user seeding on every boot)
    - Permission hydration in SupportRoleRepository and BusinessRoleRepository via PermissionRepository injection
    - UserRepository.setSupportRoleIds method
    - PermissionMockGenerator and unit tests for PermissionRepository and RbacSeeder
    - PermissionRepository and RbacSeeder registered in UserModule providers and exports
  affects:
    - Phase 52 (permission guard enforcement) — role DTOs now carry hydrated permissions ready for guard checks
    - Any module importing UserModule — PermissionRepository is now exported

tech-stack:
  added: []
  patterns:
    - Batch-fetch permissions once per findByIds call then distribute to entity DTOs (avoid N+1 queries)
    - OnModuleInit lifecycle hook for idempotent startup seeding via MongoDB updateOne with upsert: true
    - $setOnInsert for immutable fields (_id, createdAt), $set for mutable fields (description, category, updatedAt)
    - updateMany to backfill existing documents missing new required fields

key-files:
  created:
    - trade-flow-api/src/user/services/rbac-seeder.service.ts
    - trade-flow-api/src/user/test/mocks/permission-mock-generator.ts
    - trade-flow-api/src/user/test/repositories/permission.repository.spec.ts
    - trade-flow-api/src/user/test/services/rbac-seeder.service.spec.ts
  modified:
    - trade-flow-api/src/user/repositories/support-role.repository.ts
    - trade-flow-api/src/user/repositories/business-role.repository.ts
    - trade-flow-api/src/user/repositories/user.repository.ts
    - trade-flow-api/src/user/services/business-role-assigner.service.ts
    - trade-flow-api/src/user/user.module.ts
    - trade-flow-api/.env.example
    - trade-flow-api/src/business/test/policies/business.policy.spec.ts
    - trade-flow-api/src/schedule/test/policies/schedule.policy.spec.ts
    - trade-flow-api/src/visit-type/test/policies/visit-type.policy.spec.ts
    - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts

key-decisions:
  - "BusinessRoleAssigner creates per-business roles (with businessId) not template roles — seeder only backfills existing ones, does NOT seed a global template"
  - "Actual PermissionName enum has 18 entries (not 19 as stated in plan); business administrator gets 13 non-support permissions; tests corrected to match"
  - "DtoCollection has no filterByIds method — used allPermissions.data.filter() to distribute permissions per entity"
  - "SUPER_USER_EMAIL env var assigns role on every boot; seeder is idempotent so repeated assignment is skipped if already present"

patterns-established:
  - "Permission hydration pattern: batch-fetch all unique permissionIds from entity collection, pass full permissions DtoCollection to mapToDto, filter per-entity inline"
  - "RBAC seeder ordering: permissions first, then support roles (depend on permission IDs), then business administrator backfill, then super user assignment"

requirements-completed:
  - RBAC-02
  - RBAC-03
  - RBAC-04
  - RBAC-05
  - RBAC-06
  - RBAC-07

duration: ~25min
completed: "2026-04-19"
---

# Phase 51 Plan 02: RBAC Seeder Summary

**RbacSeeder seeds 18 permissions and support roles on every boot via OnModuleInit; role repositories now hydrate permissions onto DTOs via PermissionRepository injection; UserRepository gains setSupportRoleIds.**

## Performance

- **Duration:** ~25 min
- **Started:** 2026-04-19T07:30:00Z
- **Completed:** 2026-04-19T07:55:00Z
- **Tasks:** 3
- **Files modified:** 14 (6 created, 8 modified)

## Accomplishments

- `RbacSeeder` implements `OnModuleInit` and idempotently seeds 18 permissions, Super User role (all permissions, `isUnrestrictable: true`), Support Administrator role (3 support permissions), and backfills existing Business Administrator roles with 13 non-support permissions
- `SupportRoleRepository` and `BusinessRoleRepository` now inject `PermissionRepository` and hydrate permissions onto DTOs in a single batch fetch per request
- `UserRepository.setSupportRoleIds` added following the existing `setBusinessRoleIds` pattern
- 25 unit tests across `PermissionRepository` and `RbacSeeder` — all passing; full CI gate green

## Task Commits

Each task was committed atomically:

1. **Task 1: Add permission hydration to role repositories and setSupportRoleIds** - `e011111` (feat)
2. **Task 2: Create RbacSeeder service and register providers in UserModule** - `ed40ace` (feat)
3. **Task 3: Create unit tests for PermissionRepository and RbacSeeder** - `25af4b8` (test)

## Files Created/Modified

**Created:**
- `src/user/services/rbac-seeder.service.ts` - OnModuleInit seeder for 18 permissions, 2 support roles, business admin backfill, super user assignment
- `src/user/test/mocks/permission-mock-generator.ts` - PermissionMockGenerator with createPermissionDto factory
- `src/user/test/repositories/permission.repository.spec.ts` - Unit tests: findByIds empty/cache/partial-cache, findAll, clearCache, mapToDto
- `src/user/test/services/rbac-seeder.service.spec.ts` - Unit tests: seed ordering, updateOne/upsert/field shapes, role permission counts, backfill filter, super user scenarios

**Modified:**
- `src/user/repositories/support-role.repository.ts` - PermissionRepository injected; mapToDto hydrates permissions from batch-fetched DtoCollection
- `src/user/repositories/business-role.repository.ts` - PermissionRepository injected; all find methods hydrate permissions
- `src/user/repositories/user.repository.ts` - Added setSupportRoleIds method
- `src/user/services/business-role-assigner.service.ts` - Fixed to pass permissionIds/permissions to satisfy updated IBusinessRoleDto
- `src/user/user.module.ts` - PermissionRepository and RbacSeeder added to providers and exports
- `trade-flow-api/.env.example` - SUPER_USER_EMAIL documented
- `src/business/test/policies/business.policy.spec.ts` - Fixed ISupportRoleDto mock (added permissionIds, permissions)
- `src/schedule/test/policies/schedule.policy.spec.ts` - Fixed ISupportRoleDto mocks (3 occurrences)
- `src/visit-type/test/policies/visit-type.policy.spec.ts` - Fixed ISupportRoleDto mocks (3 occurrences)
- `src/subscription/test/guards/subscription.guard.spec.ts` - Fixed ISupportRoleDto object literal

## Decisions Made

- `BusinessRoleAssigner` creates per-business roles with a `businessId` — there is no "template" role in the `businessroles` collection. The seeder therefore does NOT upsert a global template; it only backfills existing business administrator roles that lack `permissionIds`.
- Actual enum has 18 `PermissionName` entries (plan stated 19). Business administrator receives 13 non-support permissions (18 minus 5 support-only). Tests corrected to reflect actual counts.
- `DtoCollection` has no `filterByIds` method — permission distribution uses `allPermissions.data.filter()` inline in `mapToDto`.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed ISupportRoleDto/IBusinessRoleDto mock objects in downstream test files**
- **Found during:** Task 1 (verifying TypeScript after repository changes)
- **Issue:** Plan 01 extended the role DTO interfaces with required `permissionIds` and `permissions` fields. Four existing spec files constructed these DTOs inline without the new fields, causing TypeScript errors.
- **Fix:** Added `permissionIds: []` and `permissions: DtoCollection.empty()` to each occurrence. Added `import type { ISupportRoleDto }` where needed.
- **Files modified:** business.policy.spec.ts, schedule.policy.spec.ts, visit-type.policy.spec.ts, subscription.guard.spec.ts
- **Committed in:** e011111 and ed40ace (Task 1 and 2 commits)

**2. [Rule 1 - Bug] Fixed BusinessRoleAssigner.create() call missing required DTO fields**
- **Found during:** Task 1 (TypeScript check)
- **Issue:** `business-role-assigner.service.ts` passed `{ id, businessId, roleName }` to `businessRoleRepository.create()` — missing `permissionIds` and `permissions` required by updated `IBusinessRoleDto`.
- **Fix:** Added `permissionIds: []` and `permissions: DtoCollection.empty()` to the create call. The seeder's backfill will populate real permissionIds for new roles.
- **Files modified:** business-role-assigner.service.ts
- **Committed in:** e011111 (Task 1 commit)

**3. [Rule 1 - Bug] Corrected permission and business-admin counts from 19/14 to 18/13**
- **Found during:** Task 3 (running tests — 3 failures due to count mismatch)
- **Issue:** Plan stated 19 permissions and 14 business-admin permissions. Actual `PermissionName` enum has 18 entries. SUPPORT_ONLY list has 5 permissions, leaving 13 for business admin.
- **Fix:** Updated test assertions from 19→18 (permission count, super user permissions) and 14→13 (business admin permissions).
- **Files modified:** rbac-seeder.service.spec.ts
- **Committed in:** 25af4b8 (Task 3 commit)

**4. [Rule 3 - Blocking] Fixed pre-existing Prettier error in query-filter-parser.utility.ts**
- **Found during:** Task 3 (npm run ci)
- **Issue:** `src/core/utilities/query-filter-parser.utility.ts` had a Prettier formatting error introduced by another wave 2 agent (commit b92c64c from Phase 54). This caused CI lint check to exit non-zero.
- **Fix:** Ran `npx prettier --write` on the file to fix formatting.
- **Files modified:** src/core/utilities/query-filter-parser.utility.ts
- **Committed in:** 25af4b8 (Task 3 commit)

---

**Total deviations:** 4 auto-fixed (2 Rule 1 bugs, 1 Rule 1 count correction, 1 Rule 3 blocking)
**Impact on plan:** All auto-fixes necessary for correctness and CI. No scope creep.

## Issues Encountered

- Ran into "test suite failed to run" intermittently when running all tests in parallel under high load — individual tests passed; CI passed cleanly on a fresh run. Pre-existing infrastructure issue, not caused by Plan 02 changes.

## Known Stubs

None. All repositories hydrate real permission data. Seeder wires to real collections.

## Threat Flags

None. No new network endpoints or trust boundaries introduced. SUPER_USER_EMAIL is a server-side-only environment variable. Permission data is non-secret.

## Next Phase Readiness

- Phase 52 permission guard enforcement can now check `request.user.supportRoles[*].permissions` and `request.user.businessRoles[*].permissions` — data is fully hydrated on every authenticated request
- `PermissionRepository` is exported from `UserModule` for use in Phase 52 guards
- `isUnrestrictable: true` on Super User role is available for bypass logic

---
*Phase: 51-rbac-data-model-seed*
*Completed: 2026-04-19*
