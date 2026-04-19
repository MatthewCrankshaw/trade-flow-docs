---
phase: 51-rbac-data-model-seed
plan: "01"
subsystem: trade-flow-api
tags:
  - rbac
  - permissions
  - data-model
  - enums
  - repository
dependency_graph:
  requires: []
  provides:
    - PermissionName enum (19 workflow-based permissions)
    - PermissionCategory enum (8 categories)
    - PermissionEntity interface
    - IPermissionDto interface
    - PermissionRepository with in-memory cache
    - Extended SupportRoleEntity with permissionIds and isUnrestrictable
    - Extended BusinessRoleEntity with permissionIds
    - Extended ISupportRoleDto with permissionIds and permissions DtoCollection
    - Extended IBusinessRoleDto with permissionIds and permissions DtoCollection
  affects:
    - trade-flow-api/src/user/repositories/support-role.repository.ts (downstream errors, fixed in Plan 02)
    - trade-flow-api/src/user/repositories/business-role.repository.ts (downstream errors, fixed in Plan 02)
    - trade-flow-api/src/user/services/business-role-assigner.service.ts (downstream errors, fixed in Plan 02)
    - trade-flow-api/src/business/test/policies/business.policy.spec.ts (downstream errors, fixed in Plan 02)
tech_stack:
  added: []
  patterns:
    - In-memory Map cache for static permission data (avoid N+1 queries)
    - Separate collections for support and business roles (separation over DRY at entity boundaries)
    - DtoCollection<IPermissionDto> hydration field on role DTOs
key_files:
  created:
    - trade-flow-api/src/user/enums/permission-name.enum.ts
    - trade-flow-api/src/user/enums/permission-category.enum.ts
    - trade-flow-api/src/user/entities/permission.entity.ts
    - trade-flow-api/src/user/data-transfer-objects/permission.dto.ts
    - trade-flow-api/src/user/repositories/permission.repository.ts
  modified:
    - trade-flow-api/src/user/entities/support-role.entity.ts
    - trade-flow-api/src/user/entities/business-role.entity.ts
    - trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts
    - trade-flow-api/src/user/data-transfer-objects/business-role.dto.ts
decisions:
  - "Permissions are workflow-based (manage_jobs, send_quote) not CRUD-based (create_job, read_customer) per D-04/D-05"
  - "Two separate role collections preserved (supportroles, businessroles) per D-01 -- no unification"
  - "PermissionRepository uses in-memory Map cache since permissions are seeded on boot and never change at runtime"
  - "SupportRoleEntity gains isUnrestrictable field for Super User bypass pattern"
metrics:
  duration: ~10 minutes
  completed_date: "2026-04-19"
  tasks_completed: 3
  files_created: 5
  files_modified: 4
requirements:
  - RBAC-01
  - RBAC-02
  - RBAC-03
  - RBAC-07
---

# Phase 51 Plan 01: RBAC Permission Data Model Summary

Permission data model (entity, DTO, repository, enums) created and existing role entities/DTOs extended with permission references. Establishes foundational types and data access layer that the seeder (Plan 02) will populate and role repositories will hydrate.

## What Was Built

### New Files (5)

**Enums:**
- `permission-name.enum.ts` — 19 workflow-based permissions across 5 domains (jobs/scheduling, customers, quotes/estimates, inventory/pricing, business settings, financials, support-only)
- `permission-category.enum.ts` — 8 categories for UI grouping (jobs, customers, quotes, estimates, inventory, business, financials, support)

**Entity and DTO:**
- `permission.entity.ts` — `PermissionEntity extends IBaseEntity` with `name`, `description`, `category` fields
- `permission.dto.ts` — `IPermissionDto extends IBaseResourceDto` with `name`, `description`, `category` fields

**Repository:**
- `permission.repository.ts` — `PermissionRepository` with `findAll()`, `findByIds()`, in-memory Map cache, `clearCache()` for testing

### Modified Files (4)

**Role entities** extended with `permissionIds: ObjectId[]` arrays:
- `support-role.entity.ts` — added `permissionIds: ObjectId[]` and `isUnrestrictable?: boolean`
- `business-role.entity.ts` — added `permissionIds: ObjectId[]`

**Role DTOs** extended with both `permissionIds: string[]` and `permissions: DtoCollection<IPermissionDto>`:
- `support-role.dto.ts` — added `permissionIds`, `permissions`, `isUnrestrictable`
- `business-role.dto.ts` — added `permissionIds`, `permissions`

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1 | c8d588d | feat(51-01): create PermissionName enum, PermissionCategory enum, PermissionEntity, and IPermissionDto |
| Task 2 | 4b0150b | feat(51-01): create PermissionRepository with in-memory cache and findByIds |
| Task 3 | c68ac10 | feat(51-01): extend role entities and DTOs with permissionIds and permissions fields |

## Deviations from Plan

None — plan executed exactly as written.

## Known Downstream Errors (Expected)

Extending the role DTO interfaces causes TypeScript errors in existing repositories and tests that construct these DTOs without the new fields. Per the plan, these are intentional and will be resolved in Plan 02:

- `support-role.repository.ts` — `mapToDto` returns object missing `permissionIds` and `permissions`
- `business-role.repository.ts` — `mapToDto` returns object missing `permissionIds` and `permissions`
- `business-role-assigner.service.ts` — constructs `IBusinessRoleDto` without new fields
- `business.policy.spec.ts` — test mock missing new `ISupportRoleDto` fields

These errors are in Plan 02's scope.

## Threat Flags

None. This plan creates data model types only — no endpoints, no user input, no runtime behavior changes.

## Self-Check: PASSED

- trade-flow-api/src/user/enums/permission-name.enum.ts: FOUND
- trade-flow-api/src/user/enums/permission-category.enum.ts: FOUND
- trade-flow-api/src/user/entities/permission.entity.ts: FOUND
- trade-flow-api/src/user/data-transfer-objects/permission.dto.ts: FOUND
- trade-flow-api/src/user/repositories/permission.repository.ts: FOUND
- trade-flow-api/src/user/entities/support-role.entity.ts: modified with permissionIds
- trade-flow-api/src/user/entities/business-role.entity.ts: modified with permissionIds
- trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts: modified with permissions DtoCollection
- trade-flow-api/src/user/data-transfer-objects/business-role.dto.ts: modified with permissions DtoCollection
- Commits c8d588d, 4b0150b, c68ac10: FOUND in trade-flow-api git log
