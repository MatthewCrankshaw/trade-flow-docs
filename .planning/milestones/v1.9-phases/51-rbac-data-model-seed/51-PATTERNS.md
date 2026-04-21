# Phase 51: RBAC Data Model & Seed - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 14 new/modified files
**Analogs found:** 14 / 14

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `src/user/entities/permission.entity.ts` | entity | CRUD | `src/user/entities/support-role.entity.ts` | exact |
| `src/user/data-transfer-objects/permission.dto.ts` | model | CRUD | `src/user/data-transfer-objects/support-role.dto.ts` | exact |
| `src/user/repositories/permission.repository.ts` | repository | CRUD | `src/user/repositories/support-role.repository.ts` | exact |
| `src/user/enums/permission-name.enum.ts` | config | CRUD | `src/user/enums/support-role-name.enum.ts` | exact |
| `src/user/enums/permission-category.enum.ts` | config | CRUD | `src/user/enums/support-role-name.enum.ts` | exact |
| `src/user/responses/permission.response.ts` | model | request-response | `src/user/responses/user.response.ts` | role-match |
| `src/user/services/rbac-seeder.service.ts` | service | batch | `src/subscription/repositories/subscription.repository.ts` | partial (onModuleInit) |
| `src/user/entities/support-role.entity.ts` | entity | CRUD | self (extend) | exact |
| `src/user/entities/business-role.entity.ts` | entity | CRUD | self (extend) | exact |
| `src/user/data-transfer-objects/support-role.dto.ts` | model | CRUD | self (extend) | exact |
| `src/user/data-transfer-objects/business-role.dto.ts` | model | CRUD | self (extend) | exact |
| `src/user/repositories/support-role.repository.ts` | repository | CRUD | self (extend with permission hydration) | exact |
| `src/user/repositories/business-role.repository.ts` | repository | CRUD | self (extend with permission hydration) | exact |
| `src/user/user.module.ts` | config | CRUD | self (add providers/exports) | exact |
| `src/user/repositories/user.repository.ts` | repository | CRUD | self (add setSupportRoleIds) | exact |
| `src/user/test/services/rbac-seeder.service.spec.ts` | test | batch | `src/subscription/test/repositories/subscription.repository.spec.ts` | role-match |
| `src/user/test/repositories/permission.repository.spec.ts` | test | CRUD | `src/subscription/test/repositories/subscription.repository.spec.ts` | role-match |
| `src/user/test/mocks/permission-mock-generator.ts` | test | CRUD | `src/subscription/test/mocks/subscription-mock-generator.ts` | exact |

## Pattern Assignments

### `src/user/entities/permission.entity.ts` (entity, NEW)

**Analog:** `src/user/entities/support-role.entity.ts`

**Imports pattern** (lines 1):
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
```

**Core pattern** (lines 3-5):
```typescript
export interface SupportRoleEntity extends IBaseEntity {
  roleName: string;
}
```

**Base entity contract** (`src/core/entities/base.entity.ts` lines 1-7):
```typescript
import { ObjectId } from "mongodb";

export interface IBaseEntity extends Record<string, unknown> {
  _id: ObjectId;
  createdAt: Date;
  updatedAt: Date;
}
```

---

### `src/user/data-transfer-objects/permission.dto.ts` (model, NEW)

**Analog:** `src/user/data-transfer-objects/support-role.dto.ts`

**Core pattern** (lines 1-11):
```typescript
export interface ISupportRoleDto {
  /**
   * Unique identifier for the support role.
   */
  id: string;

  /**
   * The name of the support role assigned to a user.
   */
  roleName: string;
}
```

**NOTE:** `IPermissionDto` must extend `IBaseResourceDto` (which requires `id: string`) because it will be used inside `DtoCollection<IPermissionDto>`. The `DtoCollection<T>` generic constraint is `T extends IBaseResourceDto`.

**Base resource contract** (`src/core/data-transfer-objects/base-resource.dto.ts` lines 1-3):
```typescript
export interface IBaseResourceDto {
  id: string;
}
```

---

### `src/user/repositories/permission.repository.ts` (repository, NEW)

**Analog:** `src/user/repositories/support-role.repository.ts`

**Imports pattern** (lines 1-9):
```typescript
import { Injectable } from "@nestjs/common";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { SupportRoleEntity } from "@user/entities/support-role.entity";
import { ISupportRoleDto } from "@user/data-transfer-objects/support-role.dto";
import { ObjectId } from "mongodb";
import { EntityCollection } from "@core/collections/entity.collection";
import { DtoCollection } from "@core/collections/dto.collection";
import { IByIdRetrieverRepository } from "@core/interfaces/by-id-retriever-repository.interface";
import { AppLogger } from "@core/services/app-logger.service";
```

**Class declaration and constructor pattern** (lines 11-16):
```typescript
@Injectable()
export class SupportRoleRepository implements IByIdRetrieverRepository {
  private static readonly COLLECTION = "supportroles";
  private readonly logger = new AppLogger(SupportRoleRepository.name);

  constructor(private readonly fetcher: MongoDbFetcher) {}
```

**findByIds pattern** (lines 23-34):
```typescript
public async findByIds(ids: string[]): Promise<DtoCollection<ISupportRoleDto>> {
  if (!ids || ids.length === 0) {
    return DtoCollection.empty();
  }

  const objectIds = ids.map((id) => new ObjectId(id));
  const entities = await this.fetcher.findMany<SupportRoleEntity>(SupportRoleRepository.COLLECTION, {
    _id: { $in: objectIds },
  });
  this.logger.debug("Fetched support roles", { ids });
  return this.toDtoCollection(entities);
}
```

**mapToDto pattern** (lines 36-41):
```typescript
private mapToDto(entity: SupportRoleEntity): ISupportRoleDto {
  return {
    id: entity._id.toString(),
    roleName: entity.roleName,
  };
}
```

**toDtoCollection pattern** (lines 43-46):
```typescript
private toDtoCollection(entityCollection: EntityCollection<SupportRoleEntity>): DtoCollection<ISupportRoleDto> {
  const dtoData = entityCollection.map((entity) => this.mapToDto(entity));
  return DtoCollection.create(dtoData, entityCollection.queryResults, entityCollection.queryOptions);
}
```

---

### `src/user/enums/permission-name.enum.ts` (config, NEW)

**Analog:** `src/user/enums/support-role-name.enum.ts`

**Core pattern** (lines 1-4):
```typescript
export enum SupportRoleName {
  SUPER_USER = "super_user",
  SUPPORT_ADMINISTRATOR = "support_administrator",
}
```

---

### `src/user/enums/permission-category.enum.ts` (config, NEW)

**Analog:** `src/user/enums/support-role-name.enum.ts` (same enum style)

Same UPPER_SNAKE_CASE keys with lowercase string values pattern.

---

### `src/user/services/rbac-seeder.service.ts` (service, NEW)

**Analog (onModuleInit):** `src/subscription/repositories/subscription.repository.ts`

**onModuleInit lifecycle pattern** (lines 1, 12-25):
```typescript
import { Injectable, OnModuleInit } from "@nestjs/common";

@Injectable()
export class SubscriptionRepository implements OnModuleInit {
  private readonly logger = new AppLogger(SubscriptionRepository.name);

  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
    private readonly connection: MongoConnectionService,
  ) {}

  async onModuleInit(): Promise<void> {
    await this.ensureIndexes();
  }
```

**Analog (upsert pattern):** `src/subscription/repositories/subscription.repository.ts`

**Upsert with $setOnInsert/$set pattern** (lines 44-56):
```typescript
public async upsertByUserId(userId: string, dto: Partial<ISubscriptionDto>): Promise<ISubscriptionDto> {
  const now = new Date();
  const setFields: Record<string, unknown> = { ...this.toEntityFields(dto), updatedAt: now };

  const result = await this.writer.findOneAndUpdate<ISubscriptionEntity>(
    SubscriptionRepository.COLLECTION,
    { userId },
    {
      $set: setFields as Partial<ISubscriptionEntity>,
      $setOnInsert: { _id: new ObjectId(), createdAt: now } as Partial<ISubscriptionEntity>,
    },
    { upsert: true },
  );
```

**Analog (raw collection access):** `src/core/services/mongo/mongo-connection.service.ts`

**Raw DB access pattern** (lines 13-14 of MongoConnectionService):
```typescript
// Get raw Db instance for operations MongoDbWriter/MongoDbFetcher don't cover
const db = await this.connection.getDb();
const collection = db.collection("permissions");
```

**Analog (logging):** `src/user/services/business-role-assigner.service.ts`

**Logger with contextual data pattern** (lines 7, 11, 33-37):
```typescript
import { AppLogger } from "@core/services/app-logger.service";

private readonly logger = new AppLogger(BusinessRoleAssigner.name);

this.logger.debug("Assigned business administrator role", {
  userId: user.id,
  businessId,
  roleId: role.id,
});
```

**Analog (ConfigService injection):** Import `ConfigService` from `@nestjs/config` for `SUPER_USER_EMAIL` env var access.

---

### `src/user/entities/support-role.entity.ts` (entity, MODIFY)

**Current file** (lines 1-5):
```typescript
import { IBaseEntity } from "@core/entities/base.entity";

export interface SupportRoleEntity extends IBaseEntity {
  roleName: string;
}
```

**Add fields:** `permissionIds: ObjectId[]` and optionally `isUnrestrictable?: boolean`. Import `ObjectId` from `mongodb` (same as `src/user/entities/business-role.entity.ts` line 2).

---

### `src/user/entities/business-role.entity.ts` (entity, MODIFY)

**Current file** (lines 1-7):
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface BusinessRoleEntity extends IBaseEntity {
  roleName: string;
  businessId: ObjectId;
}
```

**Add field:** `permissionIds: ObjectId[]`.

---

### `src/user/data-transfer-objects/support-role.dto.ts` (model, MODIFY)

**Current file** (lines 1-11):
```typescript
export interface ISupportRoleDto {
  id: string;
  roleName: string;
}
```

**Add fields:** `permissionIds: string[]`, `permissions: DtoCollection<IPermissionDto>`, `isUnrestrictable?: boolean`. Import `DtoCollection` from `@core/collections/dto.collection` and `IPermissionDto`.

---

### `src/user/data-transfer-objects/business-role.dto.ts` (model, MODIFY)

**Current file** (lines 1-16):
```typescript
export interface IBusinessRoleDto {
  id: string;
  roleName: string;
  businessId: string;
}
```

**Add fields:** `permissionIds: string[]`, `permissions: DtoCollection<IPermissionDto>`. Same import pattern as support-role.dto.ts modification.

---

### `src/user/repositories/support-role.repository.ts` (repository, MODIFY)

**Current mapToDto** (lines 36-41):
```typescript
private mapToDto(entity: SupportRoleEntity): ISupportRoleDto {
  return {
    id: entity._id.toString(),
    roleName: entity.roleName,
  };
}
```

**Must add:** Inject `PermissionRepository`, call `findByIds(entity.permissionIds)` to hydrate permissions on the DTO. Add `permissionIds` and `permissions` to mapToDto output.

---

### `src/user/repositories/business-role.repository.ts` (repository, MODIFY)

**Current mapToDto** (lines 72-78):
```typescript
private mapToDto(entity: BusinessRoleEntity): IBusinessRoleDto {
  return {
    id: entity._id.toString(),
    roleName: entity.roleName,
    businessId: entity.businessId.toString(),
  };
}
```

**Must add:** Same permission hydration pattern as support-role.repository.ts modification.

---

### `src/user/repositories/user.repository.ts` (repository, MODIFY)

**Existing setBusinessRoleIds pattern** (lines 158-169):
```typescript
public async setBusinessRoleIds(userId: string, roleIds: string[]): Promise<void> {
  await this.writer.findOneAndUpdate<UserEntity>(
    UserRepository.COLLECTION,
    { _id: new ObjectId(userId) },
    {
      $set: {
        businessRoleIds: roleIds.map((id) => new ObjectId(id)),
        ...updateAuditFields(),
      },
    },
  );
}
```

**Add:** `setSupportRoleIds` method following the identical pattern, replacing `businessRoleIds` with `supportRoleIds`.

---

### `src/user/user.module.ts` (config, MODIFY)

**Current providers array** (lines 26-41):
```typescript
providers: [
  UserCreator,
  UserRetriever,
  UserUpdater,
  UserRepository,
  SupportRoleRepository,
  BusinessRoleRepository,
  SupportRoleRetriever,
  BusinessRoleRetriever,
  BusinessRoleAssigner,
  SupportRolePolicy,
  BusinessRolePolicy,
  OnboardingProgressRepository,
  OnboardingProgressRetriever,
  OnboardingProgressUpdater,
  OnboardingProgressPolicy,
],
```

**Add to providers:** `PermissionRepository`, `RbacSeeder`.
**Add to exports:** `PermissionRepository` (for use by other modules if needed).

---

### `src/user/test/mocks/permission-mock-generator.ts` (test mock, NEW)

**Analog:** `src/subscription/test/mocks/subscription-mock-generator.ts`

**Mock generator pattern** (lines 1-10):
```typescript
import { ObjectId } from "mongodb";
import { ISubscriptionDto } from "@subscription/data-transfer-objects/subscription.dto";
import { SubscriptionStatus } from "@subscription/enums/subscription-status.enum";

export class SubscriptionMockGenerator {
  public static createSubscriptionDto(overrides: Partial<ISubscriptionDto> = {}): ISubscriptionDto {
    return {
      id: new ObjectId().toString(),
      // ... default values ...
      ...overrides,
    };
  }
}
```

---

### `src/user/test/repositories/permission.repository.spec.ts` (test, NEW)

**Analog:** `src/subscription/test/repositories/subscription.repository.spec.ts`

**Test setup pattern** (lines 1-56):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { ObjectId } from "mongodb";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { MongoDbWriter } from "@core/services/mongo/mongo-db-writer.service";
import { MongoConnectionService } from "@core/services/mongo/mongo-connection.service";

describe("PermissionRepository", () => {
  let repository: PermissionRepository;
  let mongoDbFetcher: jest.Mocked<MongoDbFetcher>;

  const mockCollection = {
    createIndex: jest.fn(),
  };

  const mockDb = {
    collection: jest.fn().mockReturnValue(mockCollection),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PermissionRepository,
        {
          provide: MongoDbFetcher,
          useValue: {
            findOne: jest.fn(),
            findMany: jest.fn(),
          },
        },
        {
          provide: MongoConnectionService,
          useValue: {
            getDb: jest.fn().mockResolvedValue(mockDb),
          },
        },
      ],
    }).compile();

    repository = module.get<PermissionRepository>(PermissionRepository);
    mongoDbFetcher = module.get(MongoDbFetcher);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
```

---

### `src/user/test/services/rbac-seeder.service.spec.ts` (test, NEW)

**Analog:** `src/subscription/test/repositories/subscription.repository.spec.ts`

Same NestJS `Test.createTestingModule` setup pattern. Mock `MongoConnectionService`, `ConfigService`, `UserRepository`, and `PermissionRepository`. Test `onModuleInit()` calls seed methods. Verify `updateOne` called with upsert for each permission and role.

---

## Shared Patterns

### Entity Definition
**Source:** `src/core/entities/base.entity.ts` (lines 1-7)
**Apply to:** `permission.entity.ts`, modified `support-role.entity.ts`, modified `business-role.entity.ts`
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface XyzEntity extends IBaseEntity {
  // domain fields here
}
```

### DTO with DtoCollection
**Source:** `src/user/data-transfer-objects/user.dto.ts` (lines 1-6, 49-55)
**Apply to:** Modified role DTOs (support-role.dto.ts, business-role.dto.ts)
```typescript
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { DtoCollection } from "@core/collections/dto.collection";

// Pattern: store ID array + hydrated collection
permissionIds: string[];
permissions: DtoCollection<IPermissionDto>;
```

### Repository findByIds
**Source:** `src/user/repositories/support-role.repository.ts` (lines 23-34)
**Apply to:** `permission.repository.ts`
```typescript
public async findByIds(ids: string[]): Promise<DtoCollection<IPermissionDto>> {
  if (!ids || ids.length === 0) {
    return DtoCollection.empty();
  }
  const objectIds = ids.map((id) => new ObjectId(id));
  const entities = await this.fetcher.findMany<PermissionEntity>(PermissionRepository.COLLECTION, {
    _id: { $in: objectIds },
  });
  return this.toDtoCollection(entities);
}
```

### Idempotent Upsert
**Source:** `src/subscription/repositories/subscription.repository.ts` (lines 44-56)
**Apply to:** `rbac-seeder.service.ts`
```typescript
const now = new Date();
await collection.updateOne(
  { name: permissionName },
  {
    $setOnInsert: { _id: new ObjectId(), createdAt: now },
    $set: { description, category, updatedAt: now },
  },
  { upsert: true },
);
```

### onModuleInit Lifecycle
**Source:** `src/subscription/repositories/subscription.repository.ts` (lines 1, 13, 23-25)
**Apply to:** `rbac-seeder.service.ts`
```typescript
import { Injectable, OnModuleInit } from "@nestjs/common";

@Injectable()
export class RbacSeeder implements OnModuleInit {
  async onModuleInit(): Promise<void> {
    // seed operations
  }
}
```

### Logging
**Source:** `src/user/services/business-role-assigner.service.ts` (lines 7, 11)
**Apply to:** All new service and repository files
```typescript
import { AppLogger } from "@core/services/app-logger.service";

private readonly logger = new AppLogger(ClassName.name);
```

### Audit Fields
**Source:** `src/core/utilities/create-audit-fields.utility.ts` (lines 12-15)
**Apply to:** `business-role.repository.ts` create method (already used), `rbac-seeder.service.ts` (for entity creation)
```typescript
import { createAuditFields } from "@core/utilities/create-audit-fields.utility";

const entity = {
  _id: new ObjectId(),
  ...createAuditFields(),
  // domain fields
};
```

### Set Role IDs on User
**Source:** `src/user/repositories/user.repository.ts` (lines 158-169)
**Apply to:** New `setSupportRoleIds` method on UserRepository
```typescript
public async setBusinessRoleIds(userId: string, roleIds: string[]): Promise<void> {
  await this.writer.findOneAndUpdate<UserEntity>(
    UserRepository.COLLECTION,
    { _id: new ObjectId(userId) },
    {
      $set: {
        businessRoleIds: roleIds.map((id) => new ObjectId(id)),
        ...updateAuditFields(),
      },
    },
  );
}
```

### Mock Generator
**Source:** `src/subscription/test/mocks/subscription-mock-generator.ts` (lines 8-20)
**Apply to:** `permission-mock-generator.ts`
```typescript
export class PermissionMockGenerator {
  public static createPermissionDto(overrides: Partial<IPermissionDto> = {}): IPermissionDto {
    return {
      id: new ObjectId().toString(),
      // default values
      ...overrides,
    };
  }
}
```

### Test Module Setup
**Source:** `src/subscription/test/repositories/subscription.repository.spec.ts` (lines 1-56)
**Apply to:** All new test files
```typescript
import { Test, TestingModule } from "@nestjs/testing";

describe("ClassName", () => {
  let subject: ClassName;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ClassName,
        { provide: DependencyClass, useValue: { methodName: jest.fn() } },
      ],
    }).compile();

    subject = module.get<ClassName>(ClassName);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
});
```

## No Analog Found

No files in this phase lack an analog. All new files closely follow existing patterns within the `user` module.

## Metadata

**Analog search scope:** `trade-flow-api/src/user/`, `trade-flow-api/src/core/`, `trade-flow-api/src/subscription/`, `trade-flow-api/src/business/`
**Files scanned:** ~30 source files read for pattern extraction
**Pattern extraction date:** 2026-04-18
