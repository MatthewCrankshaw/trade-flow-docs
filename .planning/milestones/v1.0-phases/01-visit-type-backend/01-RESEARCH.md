# Phase 1: Visit Type Backend - Research

**Researched:** 2026-02-21
**Domain:** NestJS backend module, native MongoDB driver, CRUD API with default generation
**Confidence:** HIGH

## Summary

Phase 1 introduces visit types as a managed data type that mirrors the existing JobType pattern in the codebase. The implementation is straightforward because the codebase already has an established, well-documented module structure (Controller -> Service -> Repository -> Database) with concrete examples to follow. The JobType module is the closest analog and serves as the primary template.

The codebase uses NestJS 11.x with the **native MongoDB driver** (not Mongoose/ODM) via custom `MongoDbFetcher` and `MongoDbWriter` services. Visit types will need a new `visit-type` module under `src/visit-type/` following the identical layered architecture, a `DefaultVisitTypesCreatorService` in the business module wired into `BusinessCreator`, and a migration to create a unique compound index on `(businessId, name)` for the `visitTypes` collection.

**Primary recommendation:** Clone the JobType module structure exactly, adding `color`, `icon`, and `status` fields. Wire default generation into the existing `BusinessCreator` service following the same pattern as `DefaultJobTypesCreatorService`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- 4 generic defaults for ALL trades (except "Other"): **Quote, Job, Follow-up, Inspection**
- "Other" trade gets no default visit types
- Defaults are not trade-specific -- same 4 types regardless of trade
- Each default has a predefined, curated color + icon pairing (e.g., Quote = specific color + icon)
- Fields: name, description, business ID, color, icon
- No duration hint field -- default schedule duration is hardcoded at 30 minutes for v1 (duration on visit type is a future enhancement)
- No isDefault/origin flag -- defaults and custom types are treated identically once created
- No sort order field -- no explicit ordering
- **Soft delete** -- mark as inactive/archived; existing schedules keep the reference, type hidden from new schedule creation
- **Name locked after creation** -- description, color, and icon can be edited, but name is permanent
- **Unique names per business** -- enforced; no duplicate visit type names within a business
- **No limit** on custom visit types per business
- **Must have at least one** -- prevent deletion of the last active visit type for a business
- Defaults generated **during onboarding** (business creation) -- they exist from the start
- No migration needed for existing businesses -- not in production yet
- Pre-production: no backfill or lazy generation needed

### Claude's Discretion
- Exact color + icon assignments for the 4 defaults (should be visually distinct and intuitive)
- API endpoint structure and naming conventions (follow existing codebase patterns)
- Validation error message wording
- How soft-deleted types are stored/flagged in MongoDB

### Deferred Ideas (OUT OF SCOPE)
- Default duration per visit type (configurable on the type itself) -- future enhancement
- Trade-specific visit type defaults (different types per trade) -- future enhancement if needed
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| VTYPE-02 | Default visit types are generated per trade when a business is created | Existing pattern in `DefaultJobTypesCreatorService` and `DefaultTaxRatesCreatorService` -- wire a new `DefaultVisitTypesCreatorService` into `BusinessCreator.create()` following the identical pattern. Skip for `BusinessTrade.OTHER`. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Application framework | Already in use -- `@nestjs/common`, `@nestjs/core` |
| mongodb (native driver) | 7.x | Database operations | Already in use via `MongoDbFetcher` and `MongoDbWriter` -- NOT Mongoose |
| class-validator | 0.14.x | Request body validation | Already in use for all request classes |
| class-transformer | 0.5.x | Request body transformation | Already in use with `ValidationPipe` |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Jest | 30.x | Unit testing | All service, repository, and controller tests |
| @nestjs/testing | 11.x | Test module setup | Dependency injection in tests |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Native MongoDB driver | Mongoose ODM | Codebase deliberately uses native driver; Mongoose would break consistency and require refactoring all existing patterns |

**Installation:**
No new packages needed. All required libraries are already installed.

## Architecture Patterns

### Recommended Project Structure

New module:
```
src/visit-type/
  visit-type.module.ts
  controllers/
    visit-type.controller.ts
    mappers/
      map-create-visit-type-request-to-dto.utility.ts
      map-visit-type-to-response.utility.ts
      merge-existing-visit-type-with-changes.utility.ts
  services/
    visit-type-creator.service.ts
    visit-type-retriever.service.ts
    visit-type-updater.service.ts
  repositories/
    visit-type.repository.ts
  entities/
    visit-type.entity.ts
  data-transfer-objects/
    visit-type.dto.ts
  requests/
    create-visit-type.request.ts
    update-visit-type.request.ts
  responses/
    visit-type.response.ts
  policies/
    visit-type.policy.ts
  enums/
    visit-type-status.enum.ts
  test/
    services/
    repositories/
    controllers/
    policies/
    mocks/
      visit-type-mock-generator.ts
```

Files modified in other modules:
```
src/business/
  services/
    default-visit-types-creator.service.ts  # NEW - default generation
    business-creator.service.ts              # MODIFIED - wire in defaults
  business.module.ts                         # MODIFIED - import VisitTypeModule, provide DefaultVisitTypesCreatorService

src/migration/migrations/
  YYYYMMDDHHMMSS-create-visit-types-indexes.migration.ts  # NEW

src/app.module.ts                            # MODIFIED - import VisitTypeModule

tsconfig.json                                # MODIFIED - add @visit-type/* path alias
package.json                                 # MODIFIED - add @visit-type/* jest moduleNameMapper
openapi.yaml                                 # MODIFIED - add visit-type endpoints
```

### Pattern 1: Layered Module Architecture (Mirroring JobType)
**What:** Controller -> Service -> Repository -> Database with strict layer boundaries. Entities stay in repository layer; DTOs are the contract between layers.
**When to use:** Every new domain resource in this codebase.
**Source:** Existing `src/job/` module -- verified by reading all files.
**Example:**
```typescript
// Entity (repository layer only)
export interface IVisitTypeEntity extends IBaseEntity {
  businessId: ObjectId;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}

// DTO (contract between all layers)
export interface IVisitTypeDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}

// Response (independent of DTO -- duplicates properties)
export interface IVisitTypeResponse {
  id: string;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}
```

### Pattern 2: Default Generation at Onboarding
**What:** When `BusinessCreator.create()` runs, it triggers default resource creation services sequentially. A new `DefaultVisitTypesCreatorService` follows the same pattern as `DefaultJobTypesCreatorService` and `DefaultTaxRatesCreatorService`.
**When to use:** Creating default visit types during business onboarding.
**Source:** `src/business/services/business-creator.service.ts` and `src/business/services/default-job-types-creator.service.ts` -- verified by reading both files.
**Example:**
```typescript
// In BusinessCreator.create():
await this.defaultVisitTypesCreator.createDefaultVisitTypes(userWithRole, createdBusiness);

// DefaultVisitTypesCreatorService follows DefaultTaxRatesCreatorService pattern:
// - Skip if trade === BusinessTrade.OTHER
// - Create 4 fixed visit types with curated color+icon
// - Use VisitTypeCreatorService.create() for each
```

### Pattern 3: Soft Delete via Status Enum
**What:** Use a `VisitTypeStatus` enum with `ACTIVE` and `INACTIVE` values. "Deleting" a visit type sets its status to `INACTIVE`. The list endpoint filters to active-only by default.
**When to use:** When soft-deleting visit types.
**Source:** `JobTypeStatus` enum in `src/job/enum/job-type-status.enum.ts` -- verified by reading the file. Uses same `ACTIVE`/`INACTIVE` pattern.
**Key point:** The update endpoint already handles status changes via `PATCH`. The "archive/soft-delete" action is just a PATCH that sets `status: "inactive"`. No separate DELETE endpoint is needed. The `findJobTypesByBusinessId` repository method does NOT currently filter by status -- adding a status filter to the visit type list endpoint is a new behavior specific to visit types.

### Pattern 4: Authorization via Policies and Factories
**What:** Each resource has a Policy class extending `BasePolicy<T>` that checks user business membership. `AccessControllerFactory` and `AuthorizedCreatorFactory` wrap policies for read/write operations.
**When to use:** Every controller/service operation.
**Source:** `src/job/policies/job-type.policy.ts` and `src/core/factories/` -- verified by reading files.

### Pattern 5: Unique Name Enforcement
**What:** Unique visit type names per business, enforced via a MongoDB unique compound index on `{ businessId: 1, name: 1 }` with a partial filter expression `{ status: "active" }` to allow re-creation of names from inactive types. Alternatively, enforce uniqueness across all statuses if inactive names should remain reserved.
**When to use:** When creating or updating visit types. The index is created via a migration.
**Source:** Migration pattern from `src/migration/migrations/20240101000000-example-create-users-index.migration.ts` -- verified by reading file.
**Important consideration:** Since names are locked after creation (cannot be changed via update), the uniqueness constraint only needs to be checked during creation. However, a database-level unique index is still the safest approach as it prevents race conditions. Because inactive visit types retain their names, the index should span ALL statuses (not partial) -- this prevents creating a new visit type with the same name as an archived one, which could cause confusion if the archived one is referenced by existing schedules.

### Anti-Patterns to Avoid
- **Using Mongoose:** The codebase uses native MongoDB driver. Do NOT introduce Mongoose schemas, models, or the `@nestjs/mongoose` package.
- **Skipping the repository layer:** Never access `MongoDbFetcher`/`MongoDbWriter` from services or controllers -- always go through the repository.
- **Returning entities from repositories:** Always convert entities to DTOs before returning from repository methods.
- **Extending response interfaces from DTOs:** Response interfaces must be independent, duplicating necessary fields.
- **Using relative imports:** Always use path aliases (`@visit-type/*`, `@core/*`, etc.).
- **Adding business logic to controllers:** Controllers handle HTTP concerns only; business logic belongs in services.
- **Checking unique names in service layer only:** Without a database-level index, race conditions can create duplicates. Always have both a service-level check and a database-level index.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Authorization checks | Custom if-statements in each method | `AccessControllerFactory` + `AuthorizedCreatorFactory` + Policy classes | Consistent pattern, already tested, handles support user bypass |
| Request validation | Manual field checking | `class-validator` decorators on request classes | Global `ValidationPipe` handles it automatically, returns proper 422 responses |
| Error handling | Custom try-catch with manual HTTP status mapping | `createHttpError()` utility + domain error classes (`InvalidRequestError`, `ResourceNotFoundError`) | Already maps all domain errors to proper HTTP responses |
| Pagination | Custom skip/limit logic | `MongoDbFetcher.findMany()` with `IBaseQueryOptionsDto` | Already handles pagination correctly |
| Collection mapping | Manual array iteration for entity-to-DTO | `EntityCollection` / `DtoCollection` with `.map()` | Preserves pagination metadata alongside data |

**Key insight:** This codebase has a mature set of abstractions. Nearly every infrastructure concern is already solved. The visit type module should use these abstractions rather than introducing new patterns.

## Common Pitfalls

### Pitfall 1: Introducing Mongoose
**What goes wrong:** Developer sees `mongoose` in `package.json` and assumes models/schemas should be used.
**Why it happens:** `mongoose` is listed as a dependency but the codebase exclusively uses the native `mongodb` driver via `MongoConnectionService`, `MongoDbFetcher`, and `MongoDbWriter`.
**How to avoid:** Use only `ObjectId`, `Filter`, `FindOptions`, etc. from the `mongodb` package. Follow the `JobTypeRepository` pattern exactly.
**Warning signs:** Importing from `mongoose`, defining `Schema` objects, calling `.model()`.

### Pitfall 2: Forgetting to Wire into BusinessCreator
**What goes wrong:** Default visit types are not generated during onboarding because the new service is not injected and called.
**Why it happens:** The visit type module works standalone, but the integration point in `BusinessCreator.create()` is easy to forget.
**How to avoid:** Follow the checklist: (1) Create `DefaultVisitTypesCreatorService` in `business/services/`, (2) Add to `BusinessModule` providers and imports, (3) Inject into `BusinessCreator` constructor, (4) Call in `create()` after the other default creators.
**Warning signs:** New businesses have job types and tax rates but no visit types.

### Pitfall 3: Allowing Name Changes on Update
**What goes wrong:** The update endpoint accepts a `name` field in the request body, allowing visit type names to be changed.
**Why it happens:** The JobType update request DOES allow name changes (`name` is optional in `UpdateJobTypeRequest`). Visit types deliberately differ here.
**How to avoid:** The `UpdateVisitTypeRequest` must NOT include `name`. The merge utility must NOT overwrite `name`. Only `description`, `color`, `icon`, and `status` are updatable.
**Warning signs:** `name` field appearing in the update request class or merge utility.

### Pitfall 4: Not Filtering Inactive Types on List Endpoint
**What goes wrong:** The list endpoint returns inactive (soft-deleted) visit types alongside active ones, confusing the UI.
**Why it happens:** The existing `JobTypeRepository.findJobTypesByBusinessId` does NOT filter by status. The visit type equivalent needs to add a status filter.
**How to avoid:** Add `status: VisitTypeStatus.ACTIVE` to the default filter in `findVisitTypesByBusinessId()`. Optionally support a query parameter to include inactive types for admin views.
**Warning signs:** Archived visit types appearing in the new schedule creation dropdown.

### Pitfall 5: Deleting the Last Active Visit Type
**What goes wrong:** All visit types for a business are archived, leaving no active types for new schedule creation.
**Why it happens:** No guard prevents the last active type from being archived.
**How to avoid:** In the updater service, when transitioning to `INACTIVE`, count the remaining active visit types. If there is only 1 active (the one being archived), reject the operation with an `InvalidRequestError`.
**Warning signs:** A business has zero active visit types.

### Pitfall 6: Missing Path Alias Registration
**What goes wrong:** TypeScript compiles but tests fail, or imports fail at runtime.
**Why it happens:** Path aliases must be registered in THREE places: `tsconfig.json` (compiler), `package.json` jest `moduleNameMapper` (test runner), and `nest-cli.json` may also need updates.
**How to avoid:** Add `@visit-type/*` to all three configuration files following the pattern of `@job/*`.
**Warning signs:** `Cannot find module '@visit-type/...'` errors in tests or at runtime.

### Pitfall 7: Unique Index Without Considering Inactive Types
**What goes wrong:** A user archives a visit type named "Quote", then tries to create a new visit type named "Quote", and gets a duplicate key error.
**Why it happens:** The unique compound index on `(businessId, name)` spans all documents including inactive ones.
**How to avoid:** This is actually the DESIRED behavior per the decisions. Names are permanent and unique across all statuses. If a user archives "Quote" and wants it back, they would need to reactivate the inactive one (which is a future consideration). The error message should be clear: "A visit type with this name already exists for your business."
**Warning signs:** Users confused about why they cannot create a type with the same name as an archived one. Consider whether the reactivation flow needs to be part of Phase 2 UI.

## Discretion Recommendations

### Color + Icon Assignments for Default Visit Types

**Confidence: MEDIUM** -- These are aesthetic choices; the following are recommendations based on common scheduling UI conventions.

The 4 defaults need visually distinct, intuitive pairings. Colors should be hex values (stored as strings). Icons should use a consistent icon library identifier (recommendation: use string identifiers that the frontend can map to its icon library).

| Visit Type | Color (Hex) | Color Name | Icon | Rationale |
|------------|-------------|------------|------|-----------|
| Quote | `#3B82F6` | Blue | `clipboard-list` | Blue = informational; clipboard = assessment/documentation |
| Job | `#10B981` | Green | `wrench` | Green = active/go; wrench = hands-on work |
| Follow-up | `#F59E0B` | Amber | `rotate-cw` | Amber = attention/return; rotate = revisit |
| Inspection | `#8B5CF6` | Purple | `search` | Purple = review/authority; search = examination |

**Alternative icon naming approach:** Use generic string identifiers like `quote`, `job`, `follow-up`, `inspection` and let the frontend decide rendering. This decouples backend from any specific icon library.

**Recommendation:** Store icon as a simple string identifier. The frontend maps it to the actual icon component. This avoids coupling the backend to a specific icon library (Lucide, Heroicons, etc.).

### API Endpoint Structure

**Confidence: HIGH** -- Directly derived from existing codebase patterns.

Following the exact pattern of job-type endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/business/:businessId/visit-types` | List active visit types for a business |
| POST | `/v1/business/:businessId/visit-type` | Create a visit type |
| GET | `/v1/visit-type/:visitTypeId` | Get a single visit type by ID |
| PATCH | `/v1/business/:businessId/visit-type/:visitTypeId` | Update a visit type (description, color, icon, status) |

Note: Singular `visit-type` for single-resource operations, plural `visit-types` for collection endpoint. This matches the existing `job-type` / `job-types` pattern.

No DELETE endpoint -- soft delete via PATCH with `status: "inactive"`.

### Validation Error Message Wording

**Confidence: HIGH** -- Following existing error message patterns from `errors-map.constant.ts`.

New error codes needed:

| Error Code | Enum Key | Message |
|------------|----------|---------|
| `VISIT_TYPE_0` | `VISIT_TYPE_NAME_ALREADY_EXISTS` | `A visit type with this name already exists` |
| `VISIT_TYPE_1` | `VISIT_TYPE_LAST_ACTIVE_CANNOT_ARCHIVE` | `Cannot archive the last active visit type` |

### Soft Delete Storage Approach

**Confidence: HIGH** -- Directly matches existing `JobTypeStatus` pattern.

Use the same `ACTIVE`/`INACTIVE` status enum pattern as JobType:

```typescript
export enum VisitTypeStatus {
  ACTIVE = "active",
  INACTIVE = "inactive",
}
```

The `status` field is stored directly on the document. No separate `deletedAt` timestamp or `isDeleted` boolean. This is consistent with how JobType already handles it.

For the list endpoint, the repository query includes `status: VisitTypeStatus.ACTIVE` by default, which the existing `findMany` pagination handles naturally via the filter object.

## Code Examples

Verified patterns from the existing codebase (source: files read during this research session):

### Visit Type Entity
```typescript
// Source: mirrors src/job/entities/job-type.entity.ts
import { ObjectId } from "mongodb";
import { IBaseEntity } from "@core/entities/base.entity";
import { VisitTypeStatus } from "@visit-type/enums/visit-type-status.enum";

export interface IVisitTypeEntity extends IBaseEntity {
  businessId: ObjectId;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}
```

### Visit Type DTO
```typescript
// Source: mirrors src/job/data-transfer-objects/job-type.dto.ts
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { VisitTypeStatus } from "@visit-type/enums/visit-type-status.enum";

export interface IVisitTypeDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  name: string;
  description: string | null;
  color: string;
  icon: string;
  status: VisitTypeStatus;
}
```

### Create Request (with validation)
```typescript
// Source: mirrors src/job/requests/create-job-type.request.ts
import { IsString, IsNotEmpty, IsOptional, Length, Matches } from "class-validator";

export class CreateVisitTypeRequest {
  @IsString()
  @IsNotEmpty()
  @Length(1, 100)
  name: string;

  @IsString()
  @IsOptional()
  description?: string | null;

  @IsString()
  @IsNotEmpty()
  @Matches(/^#[0-9A-Fa-f]{6}$/)
  color: string;

  @IsString()
  @IsNotEmpty()
  icon: string;
}
```

### Update Request (name excluded)
```typescript
// Source: mirrors src/job/requests/update-job-type.request.ts but WITHOUT name
import { VisitTypeStatus } from "@visit-type/enums/visit-type-status.enum";
import { IsString, IsOptional, IsEnum, Matches, IsNotEmpty } from "class-validator";

export class UpdateVisitTypeRequest {
  @IsString()
  @IsOptional()
  description?: string | null;

  @IsString()
  @IsNotEmpty()
  @IsOptional()
  @Matches(/^#[0-9A-Fa-f]{6}$/)
  color?: string;

  @IsString()
  @IsNotEmpty()
  @IsOptional()
  icon?: string;

  @IsEnum(VisitTypeStatus)
  @IsOptional()
  status?: VisitTypeStatus;
}
```

### Default Visit Types Creator
```typescript
// Source: mirrors src/business/services/default-job-types-creator.service.ts
// and src/business/services/default-tax-rates-creator.service.ts
import { IBusinessDto } from "@business/data-transfer-objects/business.dto";
import { BusinessTrade } from "@business/enums/business-trade.enum";
import { AppLogger } from "@core/services/app-logger.service";
import { IVisitTypeDto } from "@visit-type/data-transfer-objects/visit-type.dto";
import { VisitTypeStatus } from "@visit-type/enums/visit-type-status.enum";
import { VisitTypeCreatorService } from "@visit-type/services/visit-type-creator.service";
import { Injectable } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { ObjectId } from "mongodb";

interface VisitTypeTemplate {
  name: string;
  description: string;
  color: string;
  icon: string;
}

const DEFAULT_VISIT_TYPES: VisitTypeTemplate[] = [
  { name: "Quote", description: "Initial assessment and quotation visit", color: "#3B82F6", icon: "clipboard-list" },
  { name: "Job", description: "Scheduled work visit", color: "#10B981", icon: "wrench" },
  { name: "Follow-up", description: "Return visit to check or complete work", color: "#F59E0B", icon: "rotate-cw" },
  { name: "Inspection", description: "Review or inspection of completed work", color: "#8B5CF6", icon: "search" },
];

@Injectable()
export class DefaultVisitTypesCreatorService {
  private readonly logger = new AppLogger(DefaultVisitTypesCreatorService.name);

  constructor(private readonly visitTypeCreator: VisitTypeCreatorService) {}

  public async createDefaultVisitTypes(authUser: IUserDto, business: IBusinessDto): Promise<void> {
    const { primaryTrade } = business.trade;

    if (primaryTrade === BusinessTrade.OTHER) {
      this.logger.log("Skipping default visit types for OTHER trade", { businessId: business.id });
      return;
    }

    for (const template of DEFAULT_VISIT_TYPES) {
      const visitType: IVisitTypeDto = {
        id: new ObjectId().toString(),
        businessId: business.id,
        name: template.name,
        description: template.description,
        color: template.color,
        icon: template.icon,
        status: VisitTypeStatus.ACTIVE,
      };
      await this.visitTypeCreator.create(authUser, visitType);
    }

    this.logger.log("Default visit types created", { businessId: business.id, count: DEFAULT_VISIT_TYPES.length });
  }
}
```

### Migration: Unique Compound Index
```typescript
// Source: pattern from src/migration/migrations/20240101000000-example-create-users-index.migration.ts
import { Db } from "mongodb";
import { IMigration } from "@migration/interfaces/migration.interface";

export default class CreateVisitTypesIndexesMigration implements IMigration {
  migrationName = "YYYYMMDDHHMMSS-create-visit-types-indexes";

  async up(db: Db): Promise<void> {
    // Unique compound index: no two visit types with the same name per business
    await db.collection("visitTypes").createIndex(
      { businessId: 1, name: 1 },
      { unique: true }
    );
    // Performance index: list visit types by business filtered by status
    await db.collection("visitTypes").createIndex(
      { businessId: 1, status: 1 }
    );
  }

  async down(db: Db): Promise<void> {
    await db.collection("visitTypes").dropIndex("businessId_1_name_1");
    await db.collection("visitTypes").dropIndex("businessId_1_status_1");
  }

  isBootstrap(): boolean {
    return false;
  }
}
```

### Repository: List with Status Filter
```typescript
// Source: enhanced version of JobTypeRepository.findJobTypesByBusinessId
public async findVisitTypesByBusinessId(
  businessId: string,
  queryOptions: IBaseQueryOptionsDto,
): Promise<DtoCollection<IVisitTypeDto>> {
  const filter: Filter<IVisitTypeEntity> = {
    businessId: new ObjectId(businessId),
    status: VisitTypeStatus.ACTIVE,
  };
  const results = await this.fetcher.findMany<IVisitTypeEntity>(
    VisitTypeRepository.COLLECTION,
    filter,
    queryOptions,
  );
  return this.toDtoCollection(results);
}
```

### Last Active Check in Updater Service
```typescript
// Source: new logic specific to visit types
public async update(authUser: IUserDto, existing: IVisitTypeDto, updates: IVisitTypeDto): Promise<IVisitTypeDto> {
  const accessController = this.accessControllerFactory.create(this.visitTypePolicy);
  accessController.canUpdate(authUser, existing);

  // Prevent archiving the last active visit type
  if (existing.status === VisitTypeStatus.ACTIVE && updates.status === VisitTypeStatus.INACTIVE) {
    const activeCount = await this.visitTypeRepository.countActiveByBusinessId(existing.businessId);
    if (activeCount <= 1) {
      throw new InvalidRequestError(
        ErrorCodes.VISIT_TYPE_LAST_ACTIVE_CANNOT_ARCHIVE,
        "Cannot archive the last active visit type for this business",
      );
    }
  }

  return this.visitTypeRepository.update(updates);
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mongoose ODM | Native MongoDB driver 7.x | Project inception | All data access uses MongoDbFetcher/MongoDbWriter, not Mongoose |
| Trade-specific job type defaults | Same 4 defaults for all trades (visit types) | This phase (design decision) | Simpler than JobType's per-trade templates |

**Deprecated/outdated:**
- `mongoose` is in `package.json` but unused in code. Do not use it.

## Open Questions

1. **Icon identifier format -- what icon library will the frontend use?**
   - What we know: Backend stores a string identifier. Frontend maps it to an icon component.
   - What's unclear: Whether the frontend uses Lucide, Heroicons, Material Icons, or another library. The icon string format should match what the frontend expects.
   - Recommendation: Use generic descriptive names (`clipboard-list`, `wrench`, `rotate-cw`, `search`). These loosely match Lucide icon names which is common in React projects. Confirm with frontend implementation in Phase 2.

2. **Should the list endpoint support an `includeInactive` query parameter?**
   - What we know: The default filter returns only active visit types. Some admin views may need to see archived types.
   - What's unclear: Whether Phase 2 UI needs to show inactive types for reactivation.
   - Recommendation: Implement active-only filtering for now. Adding an optional `status` query parameter is trivial if needed later. Keep the default simple.

3. **Reactivation of archived visit types**
   - What we know: Names are unique per business across all statuses. If a user archives "Quote" and later wants it back, they cannot create a new one with the same name.
   - What's unclear: Whether reactivation (setting status back to ACTIVE) should be explicitly supported.
   - Recommendation: The PATCH endpoint already allows setting `status: "active"`, so reactivation works implicitly. No extra work needed, but the UI (Phase 2) should surface this.

4. **Repository `countActiveByBusinessId` method**
   - What we know: The "last active" guard requires knowing how many active visit types exist.
   - What's unclear: Whether to use `countDocuments` (native MongoDB) or fetch all and count in code.
   - Recommendation: Add a `countActiveByBusinessId` method to the repository using `MongoDbFetcher` or direct `countDocuments` on the native driver. The `MongoDbFetcher` does not currently expose a count method, so either extend it or access the collection directly through `MongoConnectionService.getDb()`.

## Sources

### Primary (HIGH confidence)
- Existing codebase files -- all patterns verified by reading actual source code:
  - `src/job/` module (entity, dto, repository, controller, services, policy, requests, responses, mappers, enums)
  - `src/business/services/business-creator.service.ts` -- onboarding default creation flow
  - `src/business/services/default-job-types-creator.service.ts` -- default generation pattern
  - `src/business/services/default-tax-rates-creator.service.ts` -- simpler default generation pattern
  - `src/core/` services (MongoDbFetcher, MongoDbWriter, MongoConnectionService)
  - `src/core/` errors, factories, policies, interfaces
  - `src/migration/` migration pattern and interface
  - `.github/copilot-instructions/` -- API standards, module standards, naming standards, testing standards, TypeScript standards
  - `tsconfig.json`, `package.json` -- path aliases and dependencies

### Secondary (MEDIUM confidence)
- Color + icon recommendations -- based on common scheduling UI conventions and icon library naming patterns

### Tertiary (LOW confidence)
- None. All findings are directly sourced from the codebase.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- verified against actual codebase dependencies and usage patterns
- Architecture: HIGH -- directly cloning existing JobType module structure with verified patterns
- Pitfalls: HIGH -- identified from actual codebase inconsistencies (Mongoose in package.json but unused, JobType allowing name updates where VisitType should not)
- Discretion areas: MEDIUM -- color/icon choices are aesthetic; API patterns are HIGH confidence from codebase

**Research date:** 2026-02-21
**Valid until:** 2026-03-21 (stable codebase, no fast-moving external dependencies)
