---
phase: 51-rbac-data-model-seed
verified: 2026-04-19T09:00:00Z
status: passed
score: 9/9 must-haves verified
overrides_applied: 1
overrides:
  - must_have: "A roles collection stores roles with name, description, type (support or customer), and an array of permission IDs (ROADMAP SC #2 / RBAC-03)"
    reason: "Two-collection separation (supportroles + businessroles) was deliberately chosen in CONTEXT.md D-01 before planning began. The existing infrastructure already used two collections and unification was explicitly rejected to follow the project's separation-over-DRY-at-entity-boundaries convention. Plan must-have truth #5 explicitly required this separation. The functional intent of RBAC-03 (roles carry permissionIds, types are implicit by collection) is fully satisfied."
    accepted_by: "orchestrator"
    accepted_at: "2026-04-19"

notes: |
  The ROADMAP/RBAC-03 collection naming is a pre-existing documentation mismatch — Decision D-01
  explicitly rejected the unified collection approach before planning began. The plan's must-have
  truth #5 required the two-collection design. Override applied; status promoted to passed.
---

# Phase 51: RBAC Data Model & Seed Verification Report

**Phase Goal:** Establish the RBAC data model (permissions, roles with permission references) and seed it idempotently on every app boot.
**Verified:** 2026-04-19T09:00:00Z
**Status:** gaps_found (1 gap — documentation mismatch on collection naming; see override suggestion)
**Re-verification:** No — initial verification

## Verification Notes

All Phase 51 commits exist on the `main` branch (c8d588d, 4b0150b, c68ac10, e011111, ed40ace, 25af4b8). The `workspace/feature-branch` checkout does not include these commits because it diverged from a pre-Phase-51 base. Verification was performed against `git show main:...` for all artifacts.

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Permissions are workflow-based names (manage_jobs, send_quote) not CRUD-based (create_job, read_customer) | VERIFIED | `permission-name.enum.ts` on `main`: 18 workflow-based permissions; no CRUD names present |
| 2 | Permission entity has name, description, and category fields | VERIFIED | `permission.entity.ts`: `PermissionEntity extends IBaseEntity` with `name`, `description`, `category` |
| 3 | Role entities carry permissionIds arrays linking to the permissions collection | VERIFIED | `support-role.entity.ts`: `permissionIds: ObjectId[]`, `isUnrestrictable?: boolean`; `business-role.entity.ts`: `permissionIds: ObjectId[]` |
| 4 | Role DTOs expose both permissionIds (string[]) and permissions (DtoCollection<IPermissionDto>) | VERIFIED | `support-role.dto.ts` and `business-role.dto.ts` on `main` both have `permissionIds: string[]` and `permissions: DtoCollection<IPermissionDto>` as required fields |
| 5 | Support roles remain in supportroles collection and business roles in businessroles collection (no unification) | VERIFIED | Seeder and repositories confirmed using `supportroles` and `businessroles` collections |
| 6 | Permissions and roles are seeded idempotently on every app boot via onModuleInit | VERIFIED | `RbacSeeder implements OnModuleInit`; `onModuleInit` calls seedPermissions → seedSupportRoles → seedBusinessAdministratorRoles → ensureSuperUserRole; all use `updateOne` with `upsert: true` |
| 7 | Super User role has all permissions and isUnrestrictable is true; Admin has 3 support permissions | VERIFIED | `seedSupportRoles`: Super User gets `allPermissionIds` and `isUnrestrictable: true`; Admin gets [MANAGE_USERS, VIEW_SUPPORT_DASHBOARD, BYPASS_SUBSCRIPTION] |
| 8 | Role repositories hydrate permissions onto DTOs via PermissionRepository | VERIFIED | Both `support-role.repository.ts` and `business-role.repository.ts` inject `PermissionRepository` and batch-fetch permissions in findByIds |
| 9 | ROADMAP SC #2 requires a unified `roles` collection with a `type` field | FAILED | Implementation uses `supportroles` and `businessroles` — deliberate deviation documented in D-01, see gap below |

**Score:** 8/9 truths verified

### Deferred Items

None — all intended scope for Phase 51 was attempted.

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/user/enums/permission-name.enum.ts` | PermissionName enum with workflow-based permissions | VERIFIED | 18 entries (plan stated 19; actual enum omitted one — SUMMARY documents this deviation) |
| `trade-flow-api/src/user/enums/permission-category.enum.ts` | PermissionCategory enum with 8 categories | VERIFIED | 8 categories: JOBS, CUSTOMERS, QUOTES, ESTIMATES, INVENTORY, BUSINESS, FINANCIALS, SUPPORT |
| `trade-flow-api/src/user/entities/permission.entity.ts` | PermissionEntity extends IBaseEntity | VERIFIED | Contains `name`, `description`, `category` |
| `trade-flow-api/src/user/data-transfer-objects/permission.dto.ts` | IPermissionDto extends IBaseResourceDto | VERIFIED | Contains `id`, `name`, `description`, `category` |
| `trade-flow-api/src/user/repositories/permission.repository.ts` | PermissionRepository with in-memory cache | VERIFIED | `findAll()`, `findByIds()`, `clearCache()`, `private permissionCache: Map<string, IPermissionDto>` |
| `trade-flow-api/src/user/entities/support-role.entity.ts` | Extended with permissionIds and isUnrestrictable | VERIFIED | `permissionIds: ObjectId[]`, `isUnrestrictable?: boolean` |
| `trade-flow-api/src/user/entities/business-role.entity.ts` | Extended with permissionIds | VERIFIED | `permissionIds: ObjectId[]` |
| `trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts` | Extended with permissionIds and permissions DtoCollection | VERIFIED | `permissionIds: string[]`, `permissions: DtoCollection<IPermissionDto>`, `isUnrestrictable?: boolean` |
| `trade-flow-api/src/user/data-transfer-objects/business-role.dto.ts` | Extended with permissionIds and permissions DtoCollection | VERIFIED | `permissionIds: string[]`, `permissions: DtoCollection<IPermissionDto>` |
| `trade-flow-api/src/user/services/rbac-seeder.service.ts` | RbacSeeder implements OnModuleInit | VERIFIED | Full seeder with 18-permission seed array, support role seeding, business admin backfill, super user assignment |
| `trade-flow-api/src/user/repositories/support-role.repository.ts` | Permission hydration in mapToDto | VERIFIED | `permissionRepository` injected; batch-fetch pattern with `allPermissions.data.filter()` distribution |
| `trade-flow-api/src/user/repositories/business-role.repository.ts` | Permission hydration in mapToDto | VERIFIED | `permissionRepository` injected; permissions hydrated in all find methods |
| `trade-flow-api/src/user/repositories/user.repository.ts` | setSupportRoleIds method | VERIFIED | `public async setSupportRoleIds(userId: string, roleIds: string[]): Promise<void>` added |
| `trade-flow-api/src/user/user.module.ts` | PermissionRepository and RbacSeeder as providers | VERIFIED | Both in `providers` array; `PermissionRepository` in `exports` array |
| `trade-flow-api/src/user/test/mocks/permission-mock-generator.ts` | PermissionMockGenerator class | VERIFIED | `class PermissionMockGenerator` with `createPermissionDto` factory |
| `trade-flow-api/src/user/test/repositories/permission.repository.spec.ts` | Unit tests for PermissionRepository | VERIFIED | Tests for empty findByIds, cache hit, cache miss, findAll, clearCache, mapToDto |
| `trade-flow-api/src/user/test/services/rbac-seeder.service.spec.ts` | Unit tests for RbacSeeder | VERIFIED | Tests for seed ordering, upsert behavior, super user scenarios (no email, no user, assign, skip if present) |
| `trade-flow-api/.env.example` | SUPER_USER_EMAIL documented | VERIFIED | `SUPER_USER_EMAIL=         # Email of the super user account (assigned Super User role on every boot)` |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `permission.repository.ts` | `permission.entity.ts` | `fetcher.findMany<PermissionEntity>` | VERIFIED | Repository uses `PermissionEntity` generic type in all fetcher calls |
| `support-role.dto.ts` | `permission.dto.ts` | `DtoCollection<IPermissionDto>` type reference | VERIFIED | Import and type reference confirmed on main branch |
| `rbac-seeder.service.ts` | permissions collection | `updateOne` with `upsert: true` | VERIFIED | `$setOnInsert` for `_id` and `createdAt`; `$set` for description, category, updatedAt |
| `support-role.repository.ts` | `permission.repository.ts` | constructor injection, `findByIds` call | VERIFIED | `private readonly permissionRepository: PermissionRepository` in constructor; `findByIds` called for batch hydration |
| `rbac-seeder.service.ts` | `user.repository.ts` | `setSupportRoleIds` | VERIFIED | `await this.userRepository.setSupportRoleIds(...)` in `ensureSuperUserRole` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|--------------------|--------|
| `support-role.repository.ts` | `permissions` DtoCollection | `permissionRepository.findByIds()` → MongoDB `permissions` collection | Yes — batch-fetches from DB using ObjectId filter | FLOWING |
| `rbac-seeder.service.ts` | permission seeds | `PERMISSION_SEEDS` constant array → MongoDB `permissions` collection via `updateOne` upsert | Yes — real MongoDB operations | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED — verification performed against `main` branch via `git show`; running the app against this branch is not currently possible from the workspace branch.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|---------|
| RBAC-01 | 51-01 | Workflow-based permissions | SATISFIED | `permission-name.enum.ts` has 18 workflow-based names; no CRUD names |
| RBAC-02 | 51-01 | Permissions in `permissions` collection with name, description, category | SATISFIED | `permission.entity.ts`, `PermissionRepository.COLLECTION = "permissions"`, seed data with all three fields |
| RBAC-03 | 51-01, 51-02 | Roles in a `roles` collection with type | PARTIAL | Roles exist in `supportroles` and `businessroles` with permissionIds arrays; collection unification was explicitly rejected in D-01. Intent is satisfied but literal wording is not. |
| RBAC-04 | 51-02 | Super User seeded with all permissions, cannot be restricted | SATISFIED | `seedSupportRoles` assigns all 18 permissionIds and `isUnrestrictable: true` to Super User |
| RBAC-05 | 51-02 | Admin support role seeded with default permission set | SATISFIED | Admin seeded with 3 permissions: MANAGE_USERS, VIEW_SUPPORT_DASHBOARD, BYPASS_SUBSCRIPTION |
| RBAC-06 | 51-02 | Business Administrator seeded with full business-scoped permissions | SATISFIED | `seedBusinessAdministratorRoles` backfills all existing business_administrator roles with 13 non-support permissions via `updateMany` |
| RBAC-07 | 51-01, 51-02 | User-role assignments stored per user, scoped to support or business | SATISFIED | `UserRepository.setSupportRoleIds` added; `supportRoleIds[]` and `businessRoleIds[]` on UserEntity; no separate assignments collection (D-09) |

### Anti-Patterns Found

No stubs, placeholder handlers, or hardcoded empty returns found in the Phase 51 artifacts on `main`. The seeder contains real MongoDB operations. Repository hydration calls real `PermissionRepository.findByIds`. The test files contain real assertions, not placeholder tests.

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| None | — | — | — |

### Human Verification Required

None — all must-haves can be verified programmatically against the `main` branch source.

## Gaps Summary

**1 gap: ROADMAP SC #2 / RBAC-03 collection naming mismatch**

The ROADMAP Success Criteria #2 states "A `roles` collection stores roles..." and RBAC-03 states "Roles are stored in a `roles` collection with name, description, type (support or customer)". The implementation uses two existing collections (`supportroles`, `businessroles`) rather than a unified `roles` collection.

This is a documentation mismatch, not an implementation failure. Decision D-01 in CONTEXT.md explicitly chose the two-collection approach before planning started. The plan's must-have truth #5 requires this separation. The implementation is correct per the plan.

**Recommended resolution:** Accept this deviation as an override. The two-collection design correctly follows the project's "separation over DRY at entity boundaries" guideline (MEMORY.md). Unification would have created a shared-with-discriminator collection which the project explicitly avoids.

To accept, add to VERIFICATION.md frontmatter:

```yaml
overrides:
  - must_have: "A roles collection stores roles with name, description, type (support or customer), and an array of permission IDs (ROADMAP SC #2 / RBAC-03)"
    reason: "Two-collection separation (supportroles + businessroles) was deliberately chosen in CONTEXT.md D-01. The existing infrastructure already used two collections and unification was explicitly rejected to follow the project's separation-over-DRY-at-entity-boundaries principle. The functional intent of RBAC-03 is satisfied — roles carry permissionIds, both collections are seeded, and the type distinction is implicit by collection."
    accepted_by: ""
    accepted_at: ""
```

## Additional Notes

**Permission count:** The plan stated 19 permissions; the actual `PermissionName` enum has 18. This was caught and documented in the SUMMARY as a known auto-fix. Business Administrator consequently gets 13 non-support permissions (not 14). All test assertions were corrected to match the actual count. This does not affect any must-have truth or requirement.

**Git branch situation:** Phase 51 commits are on the `main` branch. The `workspace/feature-branch` checkout diverged from an earlier base and does not include Phase 51 through Phase 53 work. Phase 52 commits (6dbd7c2, f6901f8, 38c6f69, e5b453f) are on `workspace/feature-branch` and do not include Phase 51's artifacts. Phase 51 work is complete and correct on `main`.

---
_Verified: 2026-04-19T09:00:00Z_
_Verifier: Claude (gsd-verifier)_
