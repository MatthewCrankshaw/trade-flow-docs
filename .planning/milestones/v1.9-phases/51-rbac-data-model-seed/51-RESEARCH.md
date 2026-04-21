# Phase 51: RBAC Data Model & Seed - Research

**Researched:** 2026-04-18
**Domain:** NestJS / MongoDB RBAC data model, seeding, permission hydration
**Confidence:** HIGH

## Summary

Phase 51 introduces a permissions-based RBAC system by extending the existing two-collection role infrastructure (`supportroles`, `businessroles`) with a new `permissions` collection and `permissionIds[]` arrays on each role document. The codebase already has a mature role pattern -- entities, DTOs, repositories, hydration in `UserRetriever`, assignment in `BusinessRoleAssigner` -- so this phase follows established conventions rather than inventing new patterns.

The primary work is: (1) create the `permissions` collection with entity/DTO/repository, (2) extend `SupportRoleEntity` and `BusinessRoleEntity` with `permissionIds[]`, (3) extend role DTOs with hydrated `permissions: DtoCollection<IPermissionDto>`, (4) build an idempotent seeder service using `onModuleInit` that upserts permissions, roles, and the super user assignment on every boot, and (5) update `UserRetriever` to hydrate permissions on roles during user resolution.

**Primary recommendation:** Follow the existing repository/entity/DTO patterns exactly. Use `updateOne` with `upsert: true` for idempotent seeding keyed on permission `name` and role `roleName`. Register the seeder in a new `RbacModule` (or within `UserModule`) using NestJS `OnModuleInit` lifecycle hook.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Extend the existing `supportroles` and `businessroles` collections by adding a `permissionIds[]` array to each. Do NOT replace them with a unified `roles` collection -- keep the two-collection split (support vs business roles) that already works.
- **D-02:** Add a new `permissions` collection with name, description, and category fields.
- **D-03:** Role DTOs (`ISupportRoleDto`, `IBusinessRoleDto`) must hydrate their permissions into a `permissions: DtoCollection<IPermissionDto>` field alongside `permissionIds[]`. This follows the same pattern as user entity (stores IDs, hydrates objects). Guards can check permissions without extra DB calls since `request.user` already carries hydrated roles.
- **D-04:** Seed comprehensive permissions covering ALL existing features (jobs, quotes, estimates, schedules, customers, items, tax rates, billing, etc.) plus v1.9 support-specific permissions (manage_users, impersonate_user, manage_roles, view_support_dashboard, bypass_subscription). Target ~15-20 permissions total.
- **D-05:** Permissions must be workflow-based (e.g., `send_quote`, `manage_schedules`, `view_financials`) -- NOT CRUD-based (not `create_job`, `read_customer`).
- **D-06:** Each permission has a `category` string field (e.g., 'jobs', 'quotes', 'support', 'billing') for grouping in future permission management UI.
- **D-07:** Idempotent upsert on every app boot via NestJS `onModuleInit`. If a permission/role exists, skip or update it. If missing, create it. Safe to run repeatedly.
- **D-08:** A `SUPER_USER_EMAIL` or `SUPER_USER_UID` env var identifies the initial super user. The seeder ensures this user has the Super User role on every boot, preventing lockout if the database is reset.
- **D-09:** Keep the existing `supportRoleIds[]` and `businessRoleIds[]` arrays on the user entity. Scope is implicit -- support roles are global, business roles are business-scoped via the business-user relationship. No separate assignments collection needed.
- **D-10:** Auto-assign the Business Administrator role when a user creates a business. This mirrors how default items/tax rates are auto-created during business creation. Solo operators never think about roles.

### Claude's Discretion
- Permission entity schema details (indexes, unique constraints)
- Exact permission names and descriptions for each feature domain
- Order of operations in the seeder (permissions first, then roles, then super user assignment)
- Whether to use `updateOne` with `upsert: true` or `findOneAndUpdate` for the idempotent seed

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| RBAC-01 | System defines workflow-based permissions rather than CRUD-based | Permissions collection with workflow names; see Architecture Patterns > Permission Design |
| RBAC-02 | Permissions stored in `permissions` collection with name, description, category | New PermissionEntity/IPermissionDto/PermissionRepository following existing patterns |
| RBAC-03 | Roles stored with name, description, type, and permission IDs | Extend existing SupportRoleEntity/BusinessRoleEntity with `permissionIds[]` |
| RBAC-04 | Super User support role seeded with all permissions, cannot be restricted | Seeder creates Super User with all permission IDs; `isUnrestrictable: true` flag |
| RBAC-05 | Admin support role seeded with default permissions, configurable | Seeder creates Admin with subset of permissions; no `isUnrestrictable` flag |
| RBAC-06 | Business Administrator customer role seeded with full business-scoped permissions | Seeder creates via BusinessRoleAssigner pattern; existing `ensureBusinessAdministratorRole` already handles this |
| RBAC-07 | User-role assignments scoped to support (global) or business (business-specific) | Existing `supportRoleIds[]` (global) and `businessRoleIds[]` (business-scoped) on user entity -- no changes needed |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Permission data model | Database / Storage | -- | MongoDB collection with entity/DTO/repository pattern |
| Role-permission linking | Database / Storage | -- | `permissionIds[]` arrays stored on role documents |
| Permission hydration | API / Backend | -- | UserRetriever hydrates permissions onto roles during JWT resolution |
| Idempotent seeding | API / Backend | Database / Storage | NestJS `onModuleInit` lifecycle hook performs upserts on boot |
| Super user assignment | API / Backend | -- | Seeder reads env var and assigns role via UserRepository |
| Business admin auto-assignment | API / Backend | -- | BusinessCreator already calls BusinessRoleAssigner -- extend with permissions |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/core | 11.1.19 | Framework core, DI, lifecycle hooks | Already installed, provides `OnModuleInit` [VERIFIED: npm registry] |
| mongodb | 7.1.1 | Native MongoDB driver, `updateOne` with `upsert` | Already installed, used throughout codebase [VERIFIED: npm registry] |
| @nestjs/config | 4.0.4 | `ConfigService` for `SUPER_USER_EMAIL` env var | Already installed [VERIFIED: npm registry] |

### Supporting
No new dependencies required. This phase uses only existing packages.

**Installation:**
```bash
# No new packages needed
```

## Architecture Patterns

### System Architecture Diagram

```
App Boot
  |
  v
RbacSeeder.onModuleInit()
  |
  +--> 1. Upsert permissions (permissions collection)
  |         |
  |         v
  +--> 2. Upsert support roles with permissionIds[] (supportroles collection)
  |         |
  |         v
  +--> 3. Upsert business role template (businessroles collection -- for new businesses)
  |         |
  |         v
  +--> 4. Ensure super user has Super User role (users collection)
  |
  v
App Running
  |
  v
HTTP Request --> JwtAuthGuard --> UserRetriever.getByExternalAuthUserId()
                                    |
                                    +--> SupportRoleRepository.findByIds()
                                    |       |
                                    |       +--> PermissionRepository.findByIds() (hydrate onto role DTO)
                                    |
                                    +--> BusinessRoleRepository.findByIds()
                                    |       |
                                    |       +--> PermissionRepository.findByIds() (hydrate onto role DTO)
                                    |
                                    v
                                  request.user: IUserDto
                                    .supportRoles[].permissions[]  <-- permissions hydrated
                                    .businessRoles[].permissions[]
```

### Recommended Project Structure

New files within existing module structure:

```
src/
├── user/
│   ├── entities/
│   │   ├── permission.entity.ts          # NEW
│   │   ├── support-role.entity.ts        # MODIFY: add permissionIds[]
│   │   └── business-role.entity.ts       # MODIFY: add permissionIds[]
│   ├── data-transfer-objects/
│   │   ├── permission.dto.ts             # NEW
│   │   ├── support-role.dto.ts           # MODIFY: add permissionIds[], permissions
│   │   └── business-role.dto.ts          # MODIFY: add permissionIds[], permissions
│   ├── repositories/
│   │   ├── permission.repository.ts      # NEW
│   │   ├── support-role.repository.ts    # MODIFY: hydrate permissions
│   │   └── business-role.repository.ts   # MODIFY: hydrate permissions
│   ├── services/
│   │   ├── rbac-seeder.service.ts        # NEW: onModuleInit seeder
│   │   └── user-retriever.service.ts     # No change needed (roles already hydrated in repos)
│   ├── enums/
│   │   ├── permission-name.enum.ts       # NEW
│   │   └── permission-category.enum.ts   # NEW
│   ├── responses/
│   │   └── permission.response.ts        # NEW (for future endpoints)
│   ├── test/
│   │   ├── services/
│   │   │   └── rbac-seeder.service.spec.ts    # NEW
│   │   ├── repositories/
│   │   │   └── permission.repository.spec.ts  # NEW
│   │   └── mocks/
│   │       └── permission-mock-generator.ts   # NEW
│   └── user.module.ts                    # MODIFY: register new providers
```

### Pattern 1: Permission Entity and DTO

**What:** New `permissions` collection following the exact same entity/DTO/repository pattern as `supportroles`
**When to use:** For all permission data access

```typescript
// Source: Existing SupportRoleEntity pattern (trade-flow-api/src/user/entities/support-role.entity.ts) [VERIFIED: codebase]
export interface PermissionEntity extends IBaseEntity {
  name: string;
  description: string;
  category: string;
}

export interface IPermissionDto extends IBaseResourceDto {
  id: string;
  name: string;
  description: string;
  category: string;
}
```

### Pattern 2: Extended Role Entities with Permission IDs

**What:** Add `permissionIds: ObjectId[]` to both role entities and `permissionIds: string[]` + `permissions: DtoCollection<IPermissionDto>` to role DTOs
**When to use:** Every role now carries its permission set

```typescript
// Source: Existing UserEntity pattern (stores IDs, hydrates objects) [VERIFIED: codebase]

// Entity layer (stores ObjectId references)
export interface SupportRoleEntity extends IBaseEntity {
  roleName: string;
  permissionIds: ObjectId[];
  isUnrestrictable?: boolean;  // true for Super User only
}

// DTO layer (hydrated permissions)
export interface ISupportRoleDto extends IBaseResourceDto {
  id: string;
  roleName: string;
  permissionIds: string[];
  permissions: DtoCollection<IPermissionDto>;
  isUnrestrictable?: boolean;
}
```

### Pattern 3: Idempotent Seeder with onModuleInit

**What:** A service that implements `OnModuleInit` to upsert seed data on every app boot
**When to use:** For permissions, roles, and super user assignment

```typescript
// Source: Existing onModuleInit pattern in SubscriptionRepository [VERIFIED: codebase]
// Source: MongoDbWriter.updateOne supports options including upsert [VERIFIED: codebase]

@Injectable()
export class RbacSeeder implements OnModuleInit {
  constructor(
    private readonly writer: MongoDbWriter,
    private readonly fetcher: MongoDbFetcher,
    private readonly connection: MongoConnectionService,
    private readonly configService: ConfigService,
    private readonly userRepository: UserRepository,
  ) {}

  async onModuleInit(): Promise<void> {
    const permissionIds = await this.seedPermissions();
    await this.seedSupportRoles(permissionIds);
    await this.ensureSuperUserRole();
  }

  private async seedPermissions(): Promise<Map<string, ObjectId>> {
    // For each permission definition:
    // updateOne({ name: "send_quote" }, { $setOnInsert: { _id: new ObjectId(), createdAt }, $set: { description, category, updatedAt } }, { upsert: true })
    // Return map of name -> ObjectId for role seeding
  }
}
```

### Pattern 4: Permission Hydration in Role Repositories

**What:** Role repositories hydrate `permissionIds[]` into `permissions: DtoCollection<IPermissionDto>` during DTO mapping
**When to use:** Whenever roles are fetched (which happens on every authenticated request via UserRetriever)

There are two approaches to hydration:

**Option A -- Hydrate in role repositories (recommended):** Each role repository takes a dependency on `PermissionRepository` and hydrates permissions during `mapToDto`. This means `SupportRoleRepository.findByIds()` returns DTOs that already have `permissions` populated.

**Option B -- Hydrate in UserRetriever:** Keep role repositories simple and add a hydration step in `UserRetriever` after fetching roles. This adds complexity to UserRetriever but keeps repositories as simple data access.

**Recommendation: Option A.** It follows the same pattern as how `UserRetriever` already hydrates `supportRoles` and `businessRoles` on the user DTO -- each layer hydrates its own children. The role repository is the natural owner of "what a complete role looks like."

However, this introduces a performance consideration: every user resolution now triggers N+1 queries (fetch roles, then fetch permissions for each role). Since permissions are a small, finite set (~15-20 total), a practical optimization is for `PermissionRepository` to cache all permissions in memory on first load. The seeder already loads them all on boot.

### Anti-Patterns to Avoid
- **Unified roles collection:** D-01 explicitly prohibits this. Keep `supportroles` and `businessroles` separate.
- **CRUD-based permissions:** D-05 explicitly prohibits `create_job`, `read_customer` style permissions. Use workflow-based names.
- **Lazy seeding / migration-based seeding:** D-07 requires seeding on every boot via `onModuleInit`, not as a one-time migration.
- **Hardcoded permission checks:** After this phase, the system has permissions data but Phase 52 adds guard/decorator enforcement. Do not add `@RequiresPermission()` decorators in this phase.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Idempotent upsert | Custom find-then-insert logic | `MongoDbWriter.updateOne` with `{ upsert: true }` | Atomic operation, no race conditions [VERIFIED: codebase -- MongoDbWriter accepts UpdateOptions] |
| Permission caching | Custom cache layer | In-memory `Map<string, IPermissionDto>` populated on boot | Permissions are finite and immutable at runtime; no cache invalidation needed |
| Role-permission mapping | Custom join logic | `PermissionRepository.findByIds()` pattern | Same pattern as `SupportRoleRepository.findByIds()` already used [VERIFIED: codebase] |

## Common Pitfalls

### Pitfall 1: Non-Idempotent Seeding
**What goes wrong:** Duplicate permissions/roles created on each app restart, or `insertOne` fails with duplicate key error on second boot.
**Why it happens:** Using `insertOne` instead of `updateOne` with `upsert: true`, or not keying on a unique field.
**How to avoid:** Use `updateOne` with filter on `{ name: permissionName }` and `{ upsert: true }`. Use `$setOnInsert` for fields that should only be set on creation (like `_id`, `createdAt`) and `$set` for fields that can be updated (description, category, updatedAt).
**Warning signs:** App crashes on second startup with duplicate key error.

### Pitfall 2: Circular Module Dependencies
**What goes wrong:** `RbacSeeder` needs `UserRepository` (to assign super user role) and `PermissionRepository`, but `UserModule` already imports other modules.
**Why it happens:** NestJS DI container can't resolve circular `forwardRef` chains.
**How to avoid:** Keep `RbacSeeder` in `UserModule` where all role/user repositories already live. No new module needed since all dependencies are already within `UserModule`.
**Warning signs:** NestJS throws "Nest can't resolve dependencies" error on startup.

### Pitfall 3: N+1 Query on Every Request
**What goes wrong:** Every authenticated request triggers: fetch user -> fetch roles -> fetch permissions per role = 4+ DB queries minimum.
**Why it happens:** Permission hydration adds a query per role type.
**How to avoid:** Cache all permissions in memory. There are only ~15-20 permissions, they are seeded on boot and don't change at runtime. `PermissionRepository` can hold a `Map<string, IPermissionDto>` populated during `onModuleInit` or on first access.
**Warning signs:** Increased latency on every authenticated request; high query count in MongoDB logs.

### Pitfall 4: Super User Email Not Configured
**What goes wrong:** App boots without `SUPER_USER_EMAIL` env var, seeder silently skips super user assignment, no one has admin access.
**Why it happens:** Env var not added to `.env` or deployment config.
**How to avoid:** Log a warning (not error) if `SUPER_USER_EMAIL` is not configured. The seeder should still create permissions and roles -- just skip the user assignment step. This allows the app to boot in development without the env var.
**Warning signs:** No user has Super User role after fresh deployment.

### Pitfall 5: Business Administrator Permissions Not Updated for Existing Businesses
**What goes wrong:** Existing Business Administrator roles (created before this phase) have no `permissionIds[]` array.
**Why it happens:** The seeder creates roles for new businesses but existing business roles in the DB lack the new field.
**How to avoid:** The seeder should also update existing Business Administrator roles in `businessroles` collection to add `permissionIds[]` if missing. Use `updateMany` with filter `{ roleName: "business_administrator", permissionIds: { $exists: false } }`.
**Warning signs:** Existing users lose access when Phase 52 enables permission checking.

## Code Examples

### Permission Names and Categories (Recommended)

```typescript
// Source: D-04, D-05, D-06 from CONTEXT.md [VERIFIED: user decisions]
// Source: Feature modules in codebase (src/ directory listing) [VERIFIED: codebase]

export enum PermissionName {
  // Jobs & Scheduling
  MANAGE_JOBS = "manage_jobs",
  MANAGE_SCHEDULES = "manage_schedules",

  // Customers
  MANAGE_CUSTOMERS = "manage_customers",

  // Quotes & Estimates
  MANAGE_QUOTES = "manage_quotes",
  SEND_QUOTE = "send_quote",
  MANAGE_ESTIMATES = "manage_estimates",
  SEND_ESTIMATE = "send_estimate",

  // Inventory & Pricing
  MANAGE_ITEMS = "manage_items",
  MANAGE_TAX_RATES = "manage_tax_rates",

  // Business Settings
  MANAGE_BUSINESS_SETTINGS = "manage_business_settings",
  MANAGE_JOB_TYPES = "manage_job_types",
  MANAGE_VISIT_TYPES = "manage_visit_types",

  // Financials
  VIEW_FINANCIALS = "view_financials",

  // Support-only
  MANAGE_USERS = "manage_users",
  IMPERSONATE_USER = "impersonate_user",
  MANAGE_ROLES = "manage_roles",
  VIEW_SUPPORT_DASHBOARD = "view_support_dashboard",
  BYPASS_SUBSCRIPTION = "bypass_subscription",
}

export enum PermissionCategory {
  JOBS = "jobs",
  CUSTOMERS = "customers",
  QUOTES = "quotes",
  ESTIMATES = "estimates",
  INVENTORY = "inventory",
  BUSINESS = "business",
  FINANCIALS = "financials",
  SUPPORT = "support",
}
```

This gives ~19 permissions total, within the D-04 target of 15-20.

### Permission Seed Data Structure

```typescript
// [ASSUMED] -- exact permission descriptions are Claude's discretion per CONTEXT.md
interface PermissionSeedData {
  name: PermissionName;
  description: string;
  category: PermissionCategory;
}

const PERMISSION_SEEDS: PermissionSeedData[] = [
  { name: PermissionName.MANAGE_JOBS, description: "Create, edit, and close jobs", category: PermissionCategory.JOBS },
  { name: PermissionName.MANAGE_SCHEDULES, description: "Create and manage schedule entries for jobs", category: PermissionCategory.JOBS },
  { name: PermissionName.MANAGE_CUSTOMERS, description: "Create, edit, and manage customer records", category: PermissionCategory.CUSTOMERS },
  // ... etc
];
```

### Idempotent Upsert Pattern

```typescript
// Source: MongoDbWriter.updateOne signature [VERIFIED: codebase]
// Source: MongoDB updateOne with upsert [VERIFIED: mongodb 7.x driver docs]

private async upsertPermission(seed: PermissionSeedData): Promise<ObjectId> {
  const db = await this.connection.getDb();
  const collection = db.collection("permissions");
  const now = new Date();

  const result = await collection.updateOne(
    { name: seed.name },
    {
      $setOnInsert: {
        _id: new ObjectId(),
        createdAt: now,
      },
      $set: {
        description: seed.description,
        category: seed.category,
        updatedAt: now,
      },
    },
    { upsert: true },
  );

  if (result.upsertedId) {
    return result.upsertedId;
  }

  // Existing document -- fetch its ID
  const existing = await collection.findOne({ name: seed.name });
  return existing!._id;
}
```

**Note:** This example uses the raw collection directly (via `MongoConnectionService.getDb()`) rather than `MongoDbWriter.updateOne()` because `MongoDbWriter.updateOne` returns `UpdateResult` which does not include the `_id` of the existing document when `upsertedId` is null. The seeder needs the `_id` to build the permission-to-role mapping. This matches the pattern used in `SubscriptionRepository.ensureIndexes()` which also accesses the raw collection. [VERIFIED: codebase]

### Role Seeding with Permission References

```typescript
// Source: SupportRoleName enum [VERIFIED: codebase]

private async seedSupportRoles(permissionIdMap: Map<string, ObjectId>): Promise<void> {
  const allPermissionIds = Array.from(permissionIdMap.values());

  // Support-only permissions for Admin
  const adminPermissionNames = [
    PermissionName.MANAGE_USERS,
    PermissionName.VIEW_SUPPORT_DASHBOARD,
    PermissionName.BYPASS_SUBSCRIPTION,
  ];
  const adminPermissionIds = adminPermissionNames
    .map((name) => permissionIdMap.get(name))
    .filter((id): id is ObjectId => id !== undefined);

  const db = await this.connection.getDb();
  const collection = db.collection("supportroles");
  const now = new Date();

  // Super User: all permissions, unrestrictable
  await collection.updateOne(
    { roleName: SupportRoleName.SUPER_USER },
    {
      $setOnInsert: { _id: new ObjectId(), createdAt: now },
      $set: { permissionIds: allPermissionIds, isUnrestrictable: true, updatedAt: now },
    },
    { upsert: true },
  );

  // Admin: configurable default set
  await collection.updateOne(
    { roleName: SupportRoleName.SUPPORT_ADMINISTRATOR },
    {
      $setOnInsert: { _id: new ObjectId(), createdAt: now },
      $set: { permissionIds: adminPermissionIds, updatedAt: now },
    },
    { upsert: true },
  );
}
```

### Ensure Super User Assignment

```typescript
// Source: D-08 from CONTEXT.md [VERIFIED: user decisions]
// Source: UserRepository.findByEmail() [VERIFIED: codebase]

private async ensureSuperUserRole(): Promise<void> {
  const superUserEmail = this.configService.get<string>("SUPER_USER_EMAIL");
  if (!superUserEmail) {
    this.logger.warn("SUPER_USER_EMAIL not configured -- skipping super user role assignment");
    return;
  }

  const user = await this.userRepository.findByEmail(superUserEmail);
  if (!user) {
    this.logger.warn("Super user not found in database", { email: superUserEmail });
    return;
  }

  const db = await this.connection.getDb();
  const superUserRole = await db.collection("supportroles").findOne({ roleName: SupportRoleName.SUPER_USER });
  if (!superUserRole) {
    this.logger.error("Super User role not found after seeding");
    return;
  }

  const roleId = superUserRole._id.toString();
  if (!user.supportRoleIds.includes(roleId)) {
    await this.userRepository.setSupportRoleIds(user.id, [...user.supportRoleIds, roleId]);
    this.logger.log("Assigned Super User role", { email: superUserEmail });
  }
}
```

**Note:** `UserRepository` does not currently have a `setSupportRoleIds` method -- only `setBusinessRoleIds`. A new `setSupportRoleIds` method will need to be added following the same pattern. [VERIFIED: codebase]

### Backfill Existing Business Administrator Roles

```typescript
// Source: D-10 from CONTEXT.md [VERIFIED: user decisions]

private async backfillBusinessAdministratorPermissions(businessPermissionIds: ObjectId[]): Promise<void> {
  const db = await this.connection.getDb();
  const collection = db.collection("businessroles");
  const now = new Date();

  const result = await collection.updateMany(
    { roleName: BusinessRoleName.BUSINESS_ADMINISTRATOR, permissionIds: { $exists: false } },
    { $set: { permissionIds: businessPermissionIds, updatedAt: now } },
  );

  if (result.modifiedCount > 0) {
    this.logger.log("Backfilled permissions on existing business administrator roles", {
      count: result.modifiedCount,
    });
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hardcoded `user.supportRoles.length > 0` checks | Permission-based checks via `permissionIds[]` on roles | This phase (v1.9) | Guards can check specific permissions instead of role names |
| No permissions concept | Workflow-based permissions collection | This phase (v1.9) | Granular access control without CRUD explosion |
| Roles without permissions | Roles carry `permissionIds[]` with hydrated `permissions` on DTO | This phase (v1.9) | Single user resolution fetches complete permission tree |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Permission descriptions are Claude's discretion | Code Examples > Permission Names | Low -- descriptions are documentation only, easily changed |
| A2 | Admin support role default permissions should be manage_users, view_support_dashboard, bypass_subscription | Code Examples > Role Seeding | Medium -- admin may need more/fewer defaults, but seed is idempotent so easy to update |
| A3 | In-memory permission caching is sufficient (no Redis/distributed cache needed) | Common Pitfalls > N+1 Query | Low -- single-instance deployment confirmed by Railway config |

## Open Questions

1. **Should `PermissionRepository` use in-memory caching or make DB calls on every request?**
   - What we know: Permissions are finite (~19), immutable at runtime, and seeded on boot.
   - What's unclear: Whether the overhead of DB calls per request matters at current scale.
   - Recommendation: Use in-memory cache. Populate on first access or during `onModuleInit`. This is safe because permissions only change when the app restarts (via seeder).

2. **Should the seeder use `MongoDbWriter` or raw collection access?**
   - What we know: `MongoDbWriter.updateOne` returns `UpdateResult`, not the document. The seeder needs document `_id` values to build role-permission mappings.
   - What's unclear: Whether to use `findOneAndUpdate` (returns document) or `updateOne` + separate `findOne`.
   - Recommendation: Use raw collection via `MongoConnectionService.getDb()` for the seeder, matching the `ensureIndexes` pattern already used in `SubscriptionRepository`. This gives full access to `updateOne` + `findOne` without wrapper limitations.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | `jest` config in `package.json` (trade-flow-api) |
| Quick run command | `npm run test -- --testPathPattern=rbac-seeder` |
| Full suite command | `npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RBAC-01 | Permission names are workflow-based (enum values) | unit | `npm run test -- --testPathPattern=permission` | No -- Wave 0 |
| RBAC-02 | PermissionRepository CRUD operations | unit | `npm run test -- --testPathPattern=permission.repository` | No -- Wave 0 |
| RBAC-03 | Role entities/DTOs include permissionIds and hydrated permissions | unit | `npm run test -- --testPathPattern=support-role.repository` | Partial -- existing tests need update |
| RBAC-04 | Seeder creates Super User with all permissions + isUnrestrictable | unit | `npm run test -- --testPathPattern=rbac-seeder` | No -- Wave 0 |
| RBAC-05 | Seeder creates Admin with default permission subset | unit | `npm run test -- --testPathPattern=rbac-seeder` | No -- Wave 0 |
| RBAC-06 | Business Administrator role gets business-scoped permissions | unit | `npm run test -- --testPathPattern=rbac-seeder` | No -- Wave 0 |
| RBAC-07 | User-role assignment scope is implicit (support=global, business=scoped) | unit | `npm run test -- --testPathPattern=user-retriever` | Partial -- existing tests need update |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=<changed-file>`
- **Per wave merge:** `npm run ci`
- **Phase gate:** Full `npm run ci` green before verification

### Wave 0 Gaps
- [ ] `src/user/test/services/rbac-seeder.service.spec.ts` -- covers RBAC-04, RBAC-05, RBAC-06
- [ ] `src/user/test/repositories/permission.repository.spec.ts` -- covers RBAC-02
- [ ] `src/user/test/mocks/permission-mock-generator.ts` -- shared mock factory for permission DTOs

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Existing Firebase JWT -- unchanged |
| V3 Session Management | No | Existing guard pattern -- unchanged |
| V4 Access Control | Yes | Permission-based role model; enforcement in Phase 52 |
| V5 Input Validation | No | No new user input endpoints in this phase |
| V6 Cryptography | No | No cryptographic operations |

### Known Threat Patterns for This Phase

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Privilege escalation via missing permissions on existing roles | Elevation of Privilege | Seeder backfills permissions on ALL existing business administrator roles, not just new ones |
| Super user lockout after DB reset | Denial of Service | `SUPER_USER_EMAIL` env var ensures super user role is reassigned on every boot |
| Permission bypass if hydration fails silently | Elevation of Privilege | Empty `DtoCollection` (no permissions) should be treated as "no access" by Phase 52 guards, never as "all access" |

## Sources

### Primary (HIGH confidence)
- Codebase: `trade-flow-api/src/user/` -- all entity, DTO, repository, service patterns verified
- Codebase: `trade-flow-api/src/subscription/repositories/subscription.repository.ts` -- `onModuleInit` + `ensureIndexes` pattern verified
- Codebase: `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` -- `updateOne` with `UpdateOptions` (supports upsert) verified
- Codebase: `trade-flow-api/src/user/services/business-role-assigner.service.ts` -- role assignment pattern verified
- npm registry: mongodb 7.1.1, @nestjs/core 11.1.19, @nestjs/config 4.0.4 verified

### Secondary (MEDIUM confidence)
- NestJS `OnModuleInit` lifecycle hook documentation -- well-established pattern, multiple codebase examples confirm [CITED: https://docs.nestjs.com/fundamentals/lifecycle-events]

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new packages, all patterns verified in codebase
- Architecture: HIGH -- extending existing entity/DTO/repository/service patterns with minimal deviation
- Pitfalls: HIGH -- identified from direct codebase analysis (N+1 queries, missing backfill, circular deps)

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable -- no external dependency changes expected)
