# Phase 51: RBAC Data Model & Seed - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the permissions and roles data model with seeded support roles (Super User, Admin) and a default Business Administrator customer role. It extends the existing `supportroles` and `businessroles` collections with permission references, adds a new `permissions` collection, and ensures the data model is ready for Phase 52's permission guard/decorator enforcement.

</domain>

<decisions>
## Implementation Decisions

### Existing Role Infrastructure
- **D-01:** Extend the existing `supportroles` and `businessroles` collections by adding a `permissionIds[]` array to each. Do NOT replace them with a unified `roles` collection — keep the two-collection split (support vs business roles) that already works.
- **D-02:** Add a new `permissions` collection with name, description, and category fields.
- **D-03:** Role DTOs (`ISupportRoleDto`, `IBusinessRoleDto`) must hydrate their permissions into a `permissions: DtoCollection<IPermissionDto>` field alongside `permissionIds[]`. This follows the same pattern as user entity (stores IDs, hydrates objects). Guards can check permissions without extra DB calls since `request.user` already carries hydrated roles.

### Permission Granularity
- **D-04:** Seed comprehensive permissions covering ALL existing features (jobs, quotes, estimates, schedules, customers, items, tax rates, billing, etc.) plus v1.9 support-specific permissions (manage_users, impersonate_user, manage_roles, view_support_dashboard, bypass_subscription). Target ~15-20 permissions total.
- **D-05:** Permissions must be workflow-based (e.g., `send_quote`, `manage_schedules`, `view_financials`) — NOT CRUD-based (not `create_job`, `read_customer`).
- **D-06:** Each permission has a `category` string field (e.g., 'jobs', 'quotes', 'support', 'billing') for grouping in future permission management UI.

### Seeding Strategy
- **D-07:** Idempotent upsert on every app boot via NestJS `onModuleInit`. If a permission/role exists, skip or update it. If missing, create it. Safe to run repeatedly.
- **D-08:** A `SUPER_USER_EMAIL` or `SUPER_USER_UID` env var identifies the initial super user. The seeder ensures this user has the Super User role on every boot, preventing lockout if the database is reset.

### User-Role Assignment
- **D-09:** Keep the existing `supportRoleIds[]` and `businessRoleIds[]` arrays on the user entity. Scope is implicit — support roles are global, business roles are business-scoped via the business-user relationship. No separate assignments collection needed.
- **D-10:** Auto-assign the Business Administrator role when a user creates a business. This mirrors how default items/tax rates are auto-created during business creation. Solo operators never think about roles.

### Claude's Discretion
- Permission entity schema details (indexes, unique constraints)
- Exact permission names and descriptions for each feature domain
- Order of operations in the seeder (permissions first, then roles, then super user assignment)
- Whether to use `updateOne` with `upsert: true` or `findOneAndUpdate` for the idempotent seed

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Role Infrastructure
- `trade-flow-api/src/user/entities/user.entity.ts` — User entity with `supportRoleIds[]` and `businessRoleIds[]` arrays
- `trade-flow-api/src/user/data-transfer-objects/user.dto.ts` — IUserDto with hydrated `supportRoles` and `businessRoles` DtoCollections
- `trade-flow-api/src/user/entities/support-role.entity.ts` — Existing SupportRoleEntity structure
- `trade-flow-api/src/user/data-transfer-objects/support-role.dto.ts` — Existing ISupportRoleDto
- `trade-flow-api/src/user/repositories/support-role.repository.ts` — SupportRoleRepository with findByIds pattern
- `trade-flow-api/src/user/enums/support-role-name.enum.ts` — SupportRoleName enum (SUPER_USER, SUPPORT_ADMINISTRATOR)

### Auth & Guard Patterns
- `trade-flow-api/src/auth/auth.guard.ts` — JwtAuthGuard that hydrates full IUserDto on request.user
- `trade-flow-api/src/user/services/user-retriever.service.ts` — UserRetriever.getByExternalAuthUserId() hydration flow (fetches roles, businesses)
- `trade-flow-api/src/subscription/guards/subscription.guard.ts` — Existing hardcoded support bypass (user.supportRoles.length > 0)
- `trade-flow-api/src/user/utilities/is-support-user.utility.ts` — isSupportUser() utility
- `trade-flow-api/src/user/utilities/is-super-user.utility.ts` — isSuperUser() utility

### Business Creation & Defaults
- `trade-flow-api/src/business/services/business-creator.service.ts` — Where Business Administrator auto-assignment should happen
- `trade-flow-api/src/business/services/default-business-items-creator.service.ts` — Pattern for auto-creating defaults on business creation
- `trade-flow-api/src/user/utilities/business-role-assigner.service.ts` — Existing BusinessRoleAssigner service

### Core Patterns
- `trade-flow-api/src/core/collections/dto.collection.ts` — DtoCollection<T> wrapper used for role hydration
- `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` — MongoDbWriter for insert/update operations
- `.planning/codebase/CONVENTIONS.md` — File naming, class naming, module structure conventions

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `DtoCollection<T>` — Already used for role hydration, will be used for permission hydration on role DTOs
- `SupportRoleRepository.findByIds()` — Pattern for fetching roles by ID array, can be replicated for permissions
- `BusinessRoleAssigner` service — Existing service for assigning business roles to users during business creation
- `MongoDbWriter` / `MongoDbFetcher` — Core database services following established repository pattern

### Established Patterns
- Role entities store in separate collections (`supportroles`, `businessroles`) — convention of separation over DRY at entity boundaries
- Role hydration happens in `UserRetriever.getByExternalAuthUserId()` — permission hydration should follow same flow
- Default data creation uses service classes called during business creation (DefaultBusinessItemsCreator, DefaultTaxRatesCreator, DefaultJobTypesCreator)
- `onModuleInit` lifecycle hook available in NestJS for startup seeding

### Integration Points
- `UserRetriever` — Must be updated to hydrate permissions on roles
- `BusinessCreator` — Must be updated to auto-assign Business Administrator role
- `AppModule` or new `RbacModule` — Seeder service registered here
- `user.entity.ts` — No changes needed (arrays already exist)

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 51-rbac-data-model-seed*
*Context gathered: 2026-04-18*
