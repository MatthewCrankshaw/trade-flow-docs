# Phase 54: User Management - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 24 new/modified files
**Analogs found:** 20 / 24

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| **Backend -- Core Query Infrastructure** | | | | |
| `api:src/core/interfaces/query-filter.interface.ts` | interface | -- | `api:src/core/data-transfer-objects/base-query-options.dto.ts` | role-match |
| `api:src/core/utilities/query-filter-parser.utility.ts` | utility | transform | `api:src/core/utilities/pagination-query-to-base-query-options.utility.ts` | exact |
| `api:src/core/utilities/query-sort-parser.utility.ts` | utility | transform | `api:src/core/utilities/pagination-query-to-base-query-options.utility.ts` | exact |
| `api:src/core/data-transfer-objects/base-query-options.dto.ts` | dto | -- | (self -- modify) | exact |
| `api:src/core/services/mongo/mongo-db-fetcher.service.ts` | service | CRUD | (self -- modify, add `aggregate`) | exact |
| **Backend -- Support Module** | | | | |
| `api:src/support/support.module.ts` | module | -- | `api:src/user/user.module.ts` | exact |
| `api:src/support/controllers/support-user.controller.ts` | controller | request-response | `api:src/estimate/controllers/estimate.controller.ts` | exact |
| `api:src/support/services/support-user-retriever.service.ts` | service | CRUD | `api:src/customer/services/customer-retriever.service.ts` | role-match |
| `api:src/support/services/support-dashboard-metrics.service.ts` | service | CRUD | `api:src/customer/services/customer-retriever.service.ts` | partial |
| `api:src/support/services/firebase-auth-metadata.service.ts` | service | request-response | -- (no analog) | none |
| `api:src/support/repositories/support-user.repository.ts` | repository | CRUD | `api:src/user/repositories/user.repository.ts` | role-match |
| `api:src/support/data-transfer-objects/support-user.dto.ts` | dto | -- | `api:src/user/data-transfer-objects/user.dto.ts` | role-match |
| `api:src/support/data-transfer-objects/dashboard-metrics.dto.ts` | dto | -- | `api:src/core/data-transfer-objects/query-results.dto.ts` | partial |
| `api:src/support/responses/support-user.response.ts` | response | -- | `api:src/user/responses/user.response.ts` | exact |
| `api:src/support/responses/support-user-detail.response.ts` | response | -- | `api:src/user/responses/user.response.ts` | role-match |
| `api:src/support/responses/dashboard-metrics.response.ts` | response | -- | `api:src/core/response/response.interface.ts` | partial |
| **Backend -- Tests** | | | | |
| `api:src/support/test/services/support-user-retriever.service.spec.ts` | test | -- | `api:src/estimate/test/services/estimate-retriever.service.spec.ts` | exact |
| `api:src/support/test/services/support-dashboard-metrics.service.spec.ts` | test | -- | `api:src/estimate/test/services/estimate-retriever.service.spec.ts` | role-match |
| `api:src/core/test/utilities/query-filter-parser.utility.spec.ts` | test | -- | (pure function test -- no complex analog needed) | partial |
| **Frontend** | | | | |
| `ui:src/features/support/api/supportApi.ts` | api | request-response | `ui:src/features/customers/api/customerApi.ts` | exact |
| `ui:src/features/support/hooks/useSupportUsers.ts` | hook | request-response | `ui:src/features/customers/hooks/useCustomersList.ts` | role-match |
| `ui:src/features/support/components/MembershipMetrics.tsx` | component | request-response | `ui:src/pages/support/SupportDashboardPage.tsx` (stats cards) | exact |
| `ui:src/pages/support/SupportUserDetailPage.tsx` | page | request-response | `ui:src/pages/support/SupportUsersPage.tsx` | role-match |
| `ui:src/pages/support/SupportUsersPage.tsx` | page | request-response | (self -- modify, replace mocks) | exact |
| `ui:src/pages/support/SupportDashboardPage.tsx` | page | request-response | (self -- modify, add metrics) | exact |
| `ui:src/types/api.types.ts` | types | -- | (self -- modify) | exact |

## Pattern Assignments

### `api:src/core/utilities/query-filter-parser.utility.ts` (utility, transform)

**Analog:** `api:src/core/utilities/pagination-query-to-base-query-options.utility.ts`

**Imports pattern** (lines 1-9):
```typescript
import { IBaseQueryOptionsDto } from "@core/data-transfer-objects/base-query-options.dto";
import {
  DEFAULT_PAGINATION_LIMIT,
  DEFAULT_PAGINATION_PAGE,
  MAX_PAGINATION_LIMIT,
  MIN_PAGINATION_VALUE,
} from "@core/constants/pagination.constant";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { InvalidRequestError } from "@core/errors/invalid-request.error";
```

**Core pattern -- single exported function with query parsing** (lines 42-52):
```typescript
export const paginationQueryToBaseQueryOptions = (query: Record<string, QueryValue>): IBaseQueryOptionsDto => {
  const page = parsePositiveInteger(query.page, "page", DEFAULT_PAGINATION_PAGE);
  const limit = parsePositiveInteger(query.limit, "limit", DEFAULT_PAGINATION_LIMIT, MAX_PAGINATION_LIMIT);

  return {
    pagination: {
      page,
      limit,
    },
  };
};
```

**Error handling -- throw InvalidRequestError for invalid values** (lines 23-30):
```typescript
  if (!Number.isInteger(parsedValue) || parsedValue < MIN_PAGINATION_VALUE) {
    throw new InvalidRequestError(
      ErrorCodes.INVALID_QUERY_PARAMETER,
      `Query parameter "${fieldName}" must be an integer greater than or equal to ${MIN_PAGINATION_VALUE}`,
    );
  }
```

---

### `api:src/support/controllers/support-user.controller.ts` (controller, request-response)

**Analog:** `api:src/estimate/controllers/estimate.controller.ts`

**Imports pattern** (lines 1-8):
```typescript
import { JwtAuthGuard } from "@auth/auth.guard";
import { createHttpError } from "@core/errors/handle-error.utility";
import { createResponse } from "@core/response/create-response.utility";
import { IResponse } from "@core/response/response.interface";
import { Body, Controller, Get, Query, Req, UseGuards } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
```

**Controller class structure** (lines 38-49):
```typescript
@Controller("v1")
export class EstimateController {
  constructor(
    private readonly estimateRetriever: EstimateRetriever,
    // ... services injected
  ) {}
```

**Paginated list endpoint with @Query()** (lines 66-81):
```typescript
  @UseGuards(JwtAuthGuard)
  @Get("estimates")
  public async findAll(
    @Req() request: { user: IUserDto },
    @Query() query: ListEstimatesRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const { items, pagination } = await this.estimateRetriever.findPaginated(request.user, query);
      return {
        data: items.map((e) => this.mapToResponse(e)),
        pagination,
      };
    } catch (error) {
      throw createHttpError(error);
    }
  }
```

**Single resource GET handler** (lines 83-94):
```typescript
  @UseGuards(JwtAuthGuard)
  @Get("estimates/:estimateId")
  public async findOne(
    @Req() request: { user: IUserDto; params: { estimateId: string } },
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const estimate = await this.estimateRetriever.findByIdOrFail(request.user, request.params.estimateId);
      return createResponse([this.mapToResponse(estimate)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }
```

**Error handling pattern** (consistent across all handlers):
```typescript
    try {
      // ... business logic
    } catch (error) {
      throw createHttpError(error);
    }
```

**Response mapping via private method** (lines 286-327):
```typescript
  private mapToResponse(estimate: IEstimateDto): IEstimateResponse {
    return {
      id: estimate.id,
      businessId: estimate.businessId,
      // ... map DTO fields to response fields
    };
  }
```

**Request validation via class-validator** (from `ListEstimatesRequest`):
```typescript
export class ListEstimatesRequest {
  @IsOptional()
  @IsEnum(EstimateStatus)
  public status?: EstimateStatus;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  public limit?: number;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  public offset?: number;
}
```

---

### `api:src/support/services/support-user-retriever.service.ts` (service, CRUD)

**Analog:** `api:src/customer/services/customer-retriever.service.ts`

**Imports and class structure** (lines 1-16):
```typescript
import { DtoCollection } from "@core/collections/dto.collection";
import { AccessControllerFactory } from "@core/factories/access-controller.factory";
import { IByIdRetrieverService } from "@core/interfaces/by-id-retriever-service.interface";
import { ICustomerDto } from "@customer/data-transfer-objects/customer.dto";
import { CustomerPolicy } from "@customer/policies/customer.policy";
import { CustomerRepository } from "@customer/repositories/customer.repository";
import { Injectable } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Injectable()
export class CustomerRetriever implements IByIdRetrieverService {
  constructor(
    private readonly customerRepository: CustomerRepository,
    private readonly customerPolicy: CustomerPolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
  ) {}
```

**findByIdOrFail with policy enforcement** (lines 18-23):
```typescript
  public async findByIdOrFail(authUser: IUserDto, id: string): Promise<ICustomerDto> {
    const customer = await this.customerRepository.findByIdOrFail(id);
    const accessController = this.accessControllerFactory.create<ICustomerDto>(this.customerPolicy);
    accessController.canRead(authUser, customer);
    return customer;
  }
```

---

### `api:src/support/repositories/support-user.repository.ts` (repository, CRUD)

**Analog:** `api:src/user/repositories/user.repository.ts`

**Imports and class structure** (lines 1-24):
```typescript
import { Injectable } from "@nestjs/common";
import { ObjectId } from "mongodb";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { ResourceNotFoundError } from "@core/errors/resource-not-found.error";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { MongoDbWriter } from "@core/services/mongo/mongo-db-writer.service";
import { UserEntity } from "@user/entities/user.entity";
import { DtoCollection } from "@core/collections/dto.collection";
import { AppLogger } from "@core/services/app-logger.service";

@Injectable()
export class UserRepository implements ICreatorRepository, IByIdRetrieverRepository {
  private readonly logger = new AppLogger(UserRepository.name);
  private static readonly COLLECTION = "users";

  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
  ) {}
```

**findById pattern** (lines 38-48):
```typescript
  public async findById(userId: string): Promise<IUserDto | null> {
    const userEntity = await this.fetcher.findOne<UserEntity>(UserRepository.COLLECTION, {
      _id: new ObjectId(userId),
    });
    if (!userEntity) {
      this.logger.debug("User not found by id", { userId });
      return null;
    }
    this.logger.debug("Found user by id", { userId });
    return this.mapToDto(userEntity);
  }
```

**mapToDto pattern** (lines 143-155):
```typescript
  private mapToDto(entity: UserEntity): IUserDto {
    return {
      id: entity._id.toString(),
      externalAuthUserId: entity.externalAuthUserId,
      email: entity.email,
      nickname: entity.nickname ?? null,
      name: entity.name ?? null,
      businessIds: [],
      supportRoleIds: (entity.supportRoleIds ?? []).map((id) => id.toString()),
      businessRoleIds: (entity.businessRoleIds ?? []).map((id) => id.toString()),
      supportRoles: DtoCollection.empty(),
      businessRoles: DtoCollection.empty(),
    };
  }
```

**MongoDbFetcher aggregate method to add** (based on existing `findMany` at lines 32-62):
```typescript
  public async findMany<TSchema extends IBaseEntity>(
    collectionName: string,
    filter: Filter<TSchema>,
    options?: IBaseQueryOptionsDto,
  ): Promise<EntityCollection<TSchema>> {
    const db = await this.connection.getDb();
    // ... pagination resolution, query execution
  }

  // NEW: aggregate method follows same pattern
  public async aggregate<TResult>(
    collectionName: string,
    pipeline: Document[],
  ): Promise<TResult[]> {
    const db = await this.connection.getDb();
    this.logger.log(`aggregate on collection ${collectionName}`);
    return db.collection(collectionName).aggregate<TResult>(pipeline).toArray();
  }
```

---

### `api:src/support/support.module.ts` (module)

**Analog:** `api:src/user/user.module.ts`

**Module definition pattern** (lines 1-53):
```typescript
import { forwardRef, Module } from "@nestjs/common";
import { CoreModule } from "@core/core.module";
// ... service/controller/repository imports

@Module({
  imports: [CoreModule, forwardRef(() => BusinessModule)],
  controllers: [UserController, SupportRoleController, BusinessRoleController],
  providers: [
    UserCreator,
    UserRetriever,
    UserRepository,
    // ... all providers
  ],
  exports: [
    UserRetriever,
    UserRepository,
    // ... exported services
  ],
})
export class UserModule {}
```

---

### `api:src/support/test/services/support-user-retriever.service.spec.ts` (test)

**Analog:** `api:src/estimate/test/services/estimate-retriever.service.spec.ts`

**Test setup pattern** (lines 1-60):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { ObjectId } from "mongodb";
import { DtoCollection } from "@core/collections/dto.collection";
import { AccessControllerFactory } from "@core/factories/access-controller.factory";

describe("EstimateRetriever", () => {
  let retriever: EstimateRetriever;
  let mockRepo: {
    findByIdOrFail: jest.Mock;
    findPaginatedByBusinessId: jest.Mock;
  };
  let mockPolicy: jest.Mocked<EstimatePolicy>;
  let mockAccessControllerFactory: { create: jest.Mock };
  let mockAccessController: { canRead: jest.Mock };

  const businessId = new ObjectId().toString();

  const authUser = {
    id: new ObjectId().toString(),
    externalAuthUserId: "firebase-uid",
    nickname: null,
    name: "Test User",
    email: "test@example.com",
    businessIds: [businessId],
    supportRoleIds: [],
    businessRoleIds: [],
    supportRoles: DtoCollection.empty(),
    businessRoles: DtoCollection.empty(),
  } as IUserDto;

  beforeEach(async () => {
    mockRepo = {
      findByIdOrFail: jest.fn(),
      findPaginatedByBusinessId: jest.fn(),
    };
    mockPolicy = { canCreate: jest.fn(), canRead: jest.fn(), canUpdate: jest.fn(), canDelete: jest.fn() } as never;
    // ... TestingModule.createTestingModule setup
  });
```

---

### `ui:src/features/support/api/supportApi.ts` (api, request-response)

**Analog:** `ui:src/features/customers/api/customerApi.ts`

**RTK Query endpoint injection** (lines 1-60):
```typescript
import { apiSlice } from "@/services/api";

import type { Customer, CreateCustomerRequest, UpdateCustomerRequest, StandardResponse } from "@/types";

export const customerApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getCustomers: builder.query<Customer[], string>({
      query: (businessId) => `/v1/business/${businessId}/customers`,
      transformResponse: (response: StandardResponse<Customer>) => response.data,
      providesTags: (result) =>
        result
          ? [...result.map(({ id }) => ({ type: "Customer" as const, id })), { type: "Customer", id: "LIST" }]
          : [{ type: "Customer", id: "LIST" }],
    }),
  }),
});

export const { useGetCustomersQuery } = customerApi;
```

**Tag type registration** (from `ui:src/services/api.ts` lines 20-35) -- note "SupportUser" tag must be added:
```typescript
export const apiSlice = createApi({
  reducerPath: "api",
  baseQuery,
  tagTypes: [
    "User",
    "Business",
    "Customer",
    // ... add "SupportUser" and "DashboardMetrics"
  ],
  endpoints: () => ({}),
});
```

---

### `ui:src/features/support/hooks/useSupportUsers.ts` (hook, request-response)

**Analog:** `ui:src/features/customers/hooks/useCustomersList.ts`

**Hook structure with filter/search state** (lines 1-125):
```typescript
import { useState, useMemo } from "react";
import { useGetCustomersQuery } from "@/services";
import type { Customer, CustomerType } from "@/types";

type TabValue = "all" | CustomerType;

interface UseCustomersListOptions {
  businessId: string;
}

interface UseCustomersListResult {
  customers: Customer[];
  isLoading: boolean;
  isEmpty: boolean;
  searchQuery: string;
  setSearchQuery: (query: string) => void;
  activeTab: TabValue;
  setActiveTab: (tab: TabValue) => void;
  tabCounts: Record<TabValue, number>;
}

export function useCustomersList({ businessId }: UseCustomersListOptions): UseCustomersListResult {
  const { data: customers = [], isLoading } = useGetCustomersQuery(businessId, {
    skip: !businessId,
  });

  const [searchQuery, setSearchQuery] = useState("");
  const [activeTab, setActiveTab] = useState<TabValue>("all");
  // ... filter logic with useMemo
}
```

Note: The support users hook will differ -- it uses server-side pagination/filtering instead of client-side. The hook will manage URL search params and pass them to the RTK Query endpoint.

---

### `ui:src/features/support/components/MembershipMetrics.tsx` (component, request-response)

**Analog:** `ui:src/pages/support/SupportDashboardPage.tsx` (stats cards at lines 138-169)

**Metric card pattern** (lines 138-169):
```typescript
<div className="grid gap-4 md:grid-cols-3">
  <Card>
    <CardHeader className="flex flex-row items-center justify-between pb-2">
      <CardTitle className="text-sm font-medium">Pending</CardTitle>
      <Clock className="h-4 w-4 text-muted-foreground" />
    </CardHeader>
    <CardContent>
      <div className="text-2xl font-bold">{pendingCount}</div>
      <p className="text-xs text-muted-foreground">migrations to run</p>
    </CardContent>
  </Card>
  // ... more cards
</div>
```

---

### `ui:src/pages/support/SupportUserDetailPage.tsx` (page, request-response)

**Analog:** `ui:src/pages/support/SupportUsersPage.tsx`

**Page wrapper pattern** (lines 182-191):
```typescript
export default function SupportUsersPage() {
  const { data: user, isLoading } = useGetCurrentUserQuery();

  return (
    <DashboardLayout user={user} isLoading={isLoading}>
      <SupportUsersContent />
    </DashboardLayout>
  );
}
```

**Header with back navigation** (lines 72-82):
```typescript
<div className="flex items-center gap-4">
  <Button variant="ghost" size="icon" asChild>
    <Link to="/support">
      <ArrowLeft className="h-4 w-4" />
    </Link>
  </Button>
  <div>
    <h1 className="text-3xl font-bold tracking-tight">All Users</h1>
    <p className="text-muted-foreground">View and manage all users in the system</p>
  </div>
</div>
```

**Support role guard pattern** (lines 58-63):
```typescript
const { data: user } = useGetCurrentUserQuery();
const hasSupportRole = user && user.supportRoles.length > 0;
if (!hasSupportRole) {
  return <Navigate to="/dashboard" replace />;
}
```

---

### `ui:src/pages/support/SupportUsersPage.tsx` (page -- modify)

The existing file at lines 1-191 is a mock skeleton. Replace mock data and static stats with RTK Query hooks. Keep the existing page wrapper pattern (lines 182-191) and header pattern (lines 72-82).

---

### `ui:src/pages/support/SupportDashboardPage.tsx` (page -- modify)

Add `MembershipMetrics` component above existing Quick Links section. Insert between the header (lines 378-383) and the Quick Links grid (lines 385-409).

---

### `ui:src/types/api.types.ts` (types -- modify)

**Current StandardResponse** (lines 9-12):
```typescript
export interface StandardResponse<T> {
  data: T[];
  errors?: ApiError[];
}
```

Add `PaginatedResponse<T>` extending with pagination field:
```typescript
export interface PaginatedResponse<T> extends StandardResponse<T> {
  pagination?: {
    total: number;
    page: number;
    limit: number;
  };
}
```

---

### `ui:src/App.tsx` (route -- modify)

**Existing route pattern** (lines 115-117):
```typescript
<Route path="/support" element={<SupportDashboardPage />} />
<Route path="/support/businesses" element={<SupportBusinessesPage />} />
<Route path="/support/users" element={<SupportUsersPage />} />
```

Add: `<Route path="/support/users/:id" element={<SupportUserDetailPage />} />`

## Shared Patterns

### Authentication Guard
**Source:** `api:src/user/controllers/user.controller.ts` lines 22-23
**Apply to:** All support controller endpoints
```typescript
@UseGuards(JwtAuthGuard)
@Get()
public async get(@Request() request: { user: IUserDto }): Promise<IResponse<IUserResponse>> {
```
Note: Phase 52 will introduce `@RequiresPermission('manage_users')` -- if already available, apply that instead.

### Error Handling (API)
**Source:** `api:src/core/errors/handle-error.utility.ts` lines 10-102
**Apply to:** All support controller handler methods
```typescript
try {
  // ... business logic
} catch (error) {
  throw createHttpError(error);
}
```

### Response Creation (API)
**Source:** `api:src/core/response/create-response.utility.ts` lines 5-15
**Apply to:** All controller return values
```typescript
// Single item
return createResponse([this.mapToResponse(dto)]);

// Paginated list
return {
  data: items.map((e) => this.mapToResponse(e)),
  pagination,
};
```

### Logger Pattern (API)
**Source:** `api:src/user/repositories/user.repository.ts` lines 12, 34
**Apply to:** All new services and repositories
```typescript
private readonly logger = new AppLogger(ClassName.name);
// ...
this.logger.debug("Found user by id", { userId });
```

### RTK Query Tag Invalidation (UI)
**Source:** `ui:src/features/customers/api/customerApi.ts` lines 10-13
**Apply to:** `supportApi.ts`
```typescript
providesTags: (result) =>
  result
    ? [...result.map(({ id }) => ({ type: "Customer" as const, id })), { type: "Customer", id: "LIST" }]
    : [{ type: "Customer", id: "LIST" }],
```

### Page Layout Wrapper (UI)
**Source:** `ui:src/pages/support/SupportUsersPage.tsx` lines 182-191
**Apply to:** `SupportUserDetailPage.tsx`
```typescript
export default function SupportUserDetailPage() {
  const { data: user, isLoading } = useGetCurrentUserQuery();

  return (
    <DashboardLayout user={user} isLoading={isLoading}>
      <SupportUserDetailContent />
    </DashboardLayout>
  );
}
```

### Card-based Detail Layout (UI)
**Source:** `ui:src/pages/support/SupportDashboardPage.tsx` lines 386-409
**Apply to:** `SupportUserDetailPage.tsx` section cards (Profile, Business, Subscription, Roles)
```typescript
<div className="grid gap-4 md:grid-cols-2">
  <Card>
    <CardHeader>
      <CardTitle>Section Title</CardTitle>
      <CardDescription>Section description</CardDescription>
    </CardHeader>
    <CardContent>
      {/* Section content */}
    </CardContent>
  </Card>
</div>
```

### Badge Pattern (UI)
**Source:** `ui:src/pages/support/SupportUsersPage.tsx` lines 155-159
**Apply to:** `SubscriptionBadge.tsx`, `RoleBadge.tsx`
```typescript
<Badge key={index} variant={role.roleName === "super_user" ? "default" : "secondary"}>
  {role.roleName}
</Badge>
```

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `api:src/support/services/firebase-auth-metadata.service.ts` | service | request-response | No Firebase Admin SDK usage exists in the codebase. This is a new external dependency. Follow standard `@Injectable()` NestJS service pattern with `AppLogger`. Initialize Firebase Admin once via provider, inject app instance. |

## Metadata

**Analog search scope:** `trade-flow-api/src/`, `trade-flow-ui/src/`
**Files scanned:** ~120 source files across both codebases
**Pattern extraction date:** 2026-04-18
