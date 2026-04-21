# Phase 55: Role Administration - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 11 new/modified files
**Analogs found:** 11 / 11

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-api/src/user/services/support-role-assigner.service.ts` | service | request-response | `trade-flow-api/src/user/services/business-role-assigner.service.ts` | exact |
| `trade-flow-api/src/user/controllers/support-role-admin.controller.ts` | controller | request-response | `trade-flow-api/src/user/controllers/support-role.controller.ts` | role-match |
| `trade-flow-api/src/user/repositories/user.repository.ts` | repository | CRUD | (self -- add methods) | exact |
| `trade-flow-api/src/user/repositories/support-role.repository.ts` | repository | CRUD | (self -- add method) | exact |
| `trade-flow-api/src/user/user.module.ts` | config | N/A | (self -- register new providers) | exact |
| `trade-flow-api/src/user/test/services/support-role-assigner.service.spec.ts` | test | N/A | `trade-flow-api/src/business/test/services/business-creator.service.spec.ts` | role-match |
| `trade-flow-api/src/user/test/controllers/support-role-admin.controller.spec.ts` | test | N/A | `trade-flow-api/src/business/test/controllers/business.controller.spec.ts` | role-match |
| `trade-flow-ui/src/services/userApi.ts` | service | request-response | (self -- add mutations) | exact |
| `trade-flow-ui/src/features/support/components/RoleActions.tsx` | component | event-driven | `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` | role-match |
| `trade-flow-ui/src/features/support/components/GrantRoleDialog.tsx` | component | event-driven | `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` (AlertDialog pattern) | role-match |
| `trade-flow-ui/src/features/support/components/RevokeRoleDialog.tsx` | component | event-driven | `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` (AlertDialog pattern) | role-match |

## Pattern Assignments

### `trade-flow-api/src/user/services/support-role-assigner.service.ts` (service, request-response)

**Analog:** `trade-flow-api/src/user/services/business-role-assigner.service.ts`

**Imports pattern** (lines 1-7):
```typescript
import { Injectable } from "@nestjs/common";
import { BusinessRoleRepository } from "@user/repositories/business-role.repository";
import { BusinessRoleName } from "@user/enums/business-role-name.enum";
import { UserRepository } from "@user/repositories/user.repository";
import { ObjectId } from "mongodb";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { AppLogger } from "@core/services/app-logger.service";
```

**Core pattern** (lines 9-46 -- entire file):
```typescript
@Injectable()
export class BusinessRoleAssigner {
  private readonly logger = new AppLogger(BusinessRoleAssigner.name);

  constructor(
    private readonly businessRoleRepository: BusinessRoleRepository,
    private readonly userRepository: UserRepository,
  ) {}

  public async ensureBusinessAdministratorRole(user: IUserDto, businessId: string): Promise<IUserDto> {
    const existingRole = await this.businessRoleRepository.findByBusinessIdAndRoleName(
      businessId,
      BusinessRoleName.BUSINESS_ADMINISTRATOR,
    );

    const role =
      existingRole ??
      (await this.businessRoleRepository.create({
        id: new ObjectId().toString(),
        businessId,
        roleName: BusinessRoleName.BUSINESS_ADMINISTRATOR,
      }));

    await this.userRepository.setBusinessRoleIds(user.id, [role.id]);
    this.logger.debug("Assigned business administrator role", {
      userId: user.id,
      businessId,
      roleId: role.id,
    });

    const updatedUser: IUserDto = {
      ...user,
      businessIds: [...user.businessIds, businessId],
      businessRoleIds: [...user.businessRoleIds, role.id],
    };
    return updatedUser;
  }
}
```

**Key differences for SupportRoleAssigner:** Two methods (`grant` and `revoke`) instead of one. Uses `SupportRoleRepository` and `SupportRoleName`. Must add error guards: self-revocation (`ForbiddenError`), last-admin count check (`ForbiddenError`), super-user protection, and already-has-role idempotency check.

**Error pattern** (from `trade-flow-api/src/core/errors/forbidden-error.error.ts`, lines 1-31):
```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ERRORS_MAP } from "@core/errors/errors-map.constant";

export class ForbiddenError extends Error {
  private code: ErrorCodes;
  private details: string;

  constructor(code: ErrorCodes, details: string) {
    const { message } = ERRORS_MAP.get(code) ?? {
      message: "Error code unknown",
      code: ErrorCodes.ERROR_CODE_UNKNOWN,
    };
    super(message);
    this.name = "ForbiddenError";
    this.details = details;
    this.code = code;
  }

  public getCode() { return this.code; }
  public getMessage() { return this.message; }
  public getDetails() { return this.details; }
}
```

**Note:** ForbiddenError `details` is a `string`, not an object. RESEARCH.md assumed `{ type: "last_admin_protection" }` but the actual constructor takes `(code: ErrorCodes, details: string)`. Either pass the type as a string directly, or extend `ForbiddenError` to accept structured details. The planner should decide the approach.

---

### `trade-flow-api/src/user/controllers/support-role-admin.controller.ts` (controller, request-response)

**Analog:** `trade-flow-api/src/user/controllers/support-role.controller.ts`

**Imports pattern** (lines 1-8):
```typescript
import { JwtAuthGuard } from "@auth/auth.guard";
import { createResponse } from "@core/response/create-response.utility";
import { IResponse } from "@core/response/response.interface";
import { Controller, Get, Request, UseGuards } from "@nestjs/common";
import { ISupportRoleResponse } from "@user/responses/support-role.response";
import { SupportRoleRetriever } from "@user/services/support-role-retriever.service";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { createHttpError } from "@core/errors/handle-error.utility";
```

**Controller + Guard pattern** (lines 10-28):
```typescript
@Controller("v1/roles/support")
export class SupportRoleController {
  constructor(private readonly supportRoleRetriever: SupportRoleRetriever) {}

  @UseGuards(JwtAuthGuard)
  @Get()
  public async getSupportRoles(@Request() request: { user: IUserDto }): Promise<IResponse<ISupportRoleResponse>> {
    try {
      const roles = await this.supportRoleRetriever.getAll(request.user);
      const response = roles.map((role) => ({
        id: role.id,
        roleName: role.roleName,
      }));
      return createResponse(response);
    } catch (error) {
      throw createHttpError(error);
    }
  }
}
```

**Response mapping pattern** (from `trade-flow-api/src/user/controllers/user.controller.ts`, lines 82-102):
```typescript
private mapToResponse(user: IUserDto): IUserResponse {
  const supportRoles: IUserSupportRoleResponse[] = user.supportRoles.map((role) => ({
    id: role.id,
    roleName: role.roleName,
  }));

  const businessRoles: IUserBusinessRoleResponse[] = user.businessRoles.map((role) => ({
    id: role.id,
    roleName: role.roleName,
    businessId: role.businessId,
  }));

  return {
    id: user.id,
    email: user.email,
    nickname: user.nickname,
    name: user.name,
    supportRoles,
    businessRoles,
  };
}
```

**Key differences:** New controller at `v1/users/:id/support-role` with `POST` and `DELETE` methods. Uses `@UseGuards(JwtAuthGuard, PermissionGuard)` and `@RequiresPermission("manage_roles")` (Phase 52 infrastructure -- not yet in codebase). Uses `@Param("id")` for target user ID. Returns `IUserResponse` from `mapToResponse()`.

---

### `trade-flow-api/src/user/repositories/user.repository.ts` (repository, CRUD -- MODIFY)

**Analog:** Self -- add new methods following `setBusinessRoleIds` pattern.

**Existing pattern to extend** (lines 158-169):
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

**MongoDbWriter supports `$addToSet` and `$pull`:** Confirmed -- `updateOne` and `findOneAndUpdate` accept `UpdateFilter<TSchema>` from mongodb driver, which supports all MongoDB update operators (lines 27-34 of `mongo-db-writer.service.ts`):
```typescript
async updateOne<TSchema extends Record<string, unknown>>(
  collectionName: string,
  filter: Filter<TSchema>,
  update: UpdateFilter<TSchema>,
  options?: UpdateOptions,
) {
  const db = await this.connection.getDb();
  return db.collection<TSchema>(collectionName).updateOne(filter, update, options);
}
```

**New methods to add:** `addSupportRoleId(userId, roleId)` using `$addToSet`, `removeSupportRoleId(userId, roleId)` using `$pull`, and `countBySupportRoleId(roleId)` for last-admin protection check.

---

### `trade-flow-api/src/user/repositories/support-role.repository.ts` (repository, CRUD -- MODIFY)

**Analog:** Self -- add `findByRoleName` method following existing `findByIds` pattern.

**Existing pattern** (lines 23-34):
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

**`mapToDto` pattern** (lines 36-41):
```typescript
private mapToDto(entity: SupportRoleEntity): ISupportRoleDto {
  return {
    id: entity._id.toString(),
    roleName: entity.roleName,
  };
}
```

**New method:** `findByRoleName(name: SupportRoleName): Promise<ISupportRoleDto | null>` using `this.fetcher.findOne` with `{ roleName: name }` filter, returning `this.mapToDto(entity)` or `null`.

---

### `trade-flow-api/src/user/user.module.ts` (config -- MODIFY)

**Existing pattern** (lines 1-53):
```typescript
@Module({
  imports: [CoreModule, forwardRef(() => BusinessModule)],
  controllers: [UserController, SupportRoleController, BusinessRoleController],
  providers: [
    UserCreator, UserRetriever, UserUpdater, UserRepository,
    SupportRoleRepository, BusinessRoleRepository,
    SupportRoleRetriever, BusinessRoleRetriever, BusinessRoleAssigner,
    SupportRolePolicy, BusinessRolePolicy,
    OnboardingProgressRepository, OnboardingProgressRetriever,
    OnboardingProgressUpdater, OnboardingProgressPolicy,
  ],
  exports: [
    UserCreator, UserRetriever, UserRepository,
    SupportRoleRepository, BusinessRoleRepository,
    BusinessRoleAssigner, OnboardingProgressUpdater,
  ],
})
export class UserModule {}
```

**Changes needed:** Add `SupportRoleAdminController` to `controllers`, add `SupportRoleAssigner` to `providers` and `exports`.

---

### `trade-flow-api/src/user/test/services/support-role-assigner.service.spec.ts` (test)

**Analog:** `trade-flow-api/src/business/test/services/business-creator.service.spec.ts`

**Test setup pattern** (lines 1-23, 30-119):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { BusinessCreator } from "@business/services/business-creator.service";
import { BusinessRepository } from "@business/repositories/business.repository";
// ... more imports

describe("BusinessCreator", () => {
  let businessCreator: BusinessCreator;
  let businessRepository: jest.Mocked<BusinessRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BusinessCreator,
        {
          provide: BusinessRepository,
          useValue: {
            create: jest.fn(),
          },
        },
        // ... more mock providers
      ],
    }).compile();

    businessCreator = module.get<BusinessCreator>(BusinessCreator);
    businessRepository = module.get(BusinessRepository);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
```

**Test case pattern** (lines 126-159):
```typescript
it("should create business successfully", async () => {
  // Arrange
  const businessData = BusinessMockGenerator.createBusinessDto();
  const user = BusinessMockGenerator.createUserDto();
  // ... setup mocks

  // Act
  const result = await businessCreator.create(user, businessData);

  // Assert
  expect(authorizedCreatorFactory.createFor).toHaveBeenCalledWith(businessRepository, businessPolicy);
  expect(result).toEqual(createdBusiness);
});
```

**Error propagation test pattern** (lines 162-175):
```typescript
it("should propagate error when access control fails", async () => {
  // Arrange
  const accessError = new Error("Access denied");
  mockAuthorizedCreator.create.mockRejectedValue(accessError);

  // Act & Assert
  await expect(businessCreator.create(user, businessData)).rejects.toThrow(accessError);
});
```

---

### `trade-flow-api/src/user/test/controllers/support-role-admin.controller.spec.ts` (test)

**Analog:** `trade-flow-api/src/business/test/controllers/business.controller.spec.ts`

**Controller test setup with guard override** (lines 1-65):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { BusinessController } from "@business/controllers/business.controller";
import { createResponse } from "@core/response/create-response.utility";
import { createHttpError } from "@core/errors/handle-error.utility";
import { JwtAuthGuard } from "@auth/auth.guard";

jest.mock("@core/response/create-response.utility");
jest.mock("@core/errors/handle-error.utility");

describe("BusinessController", () => {
  let controller: BusinessController;
  let businessCreator: jest.Mocked<BusinessCreator>;

  const mockCreateResponse = createResponse as jest.MockedFunction<typeof createResponse>;
  const mockCreateHttpError = createHttpError as jest.MockedFunction<typeof createHttpError>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [BusinessController],
      providers: [
        {
          provide: BusinessCreator,
          useValue: { create: jest.fn() },
        },
        // ...
      ],
    })
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: jest.fn(() => true) })
      .compile();

    controller = module.get<BusinessController>(BusinessController);
    // ...

    mockCreateResponse.mockImplementation((data) => ({
      data,
      success: true,
      message: "Success",
    }));
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
```

---

### `trade-flow-ui/src/services/userApi.ts` (service -- MODIFY)

**Analog:** Self -- add mutation endpoints following existing `updateUser` pattern.

**Mutation pattern** (lines 22-35):
```typescript
updateUser: builder.mutation<User, UpdateUserPayload>({
  query: (body) => ({
    url: "/v1/user/me",
    method: "PATCH",
    body,
  }),
  transformResponse: (response: StandardResponse<User>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No user data returned");
  },
  invalidatesTags: ["User"],
}),
```

**New mutations to add:** `grantSupportRole` (POST to `/v1/users/${userId}/support-role`) and `revokeSupportRole` (DELETE to same). Both invalidate `["User", "UserList"]` tags. No request body needed -- userId is in the URL path.

---

### `trade-flow-ui/src/features/support/components/RoleActions.tsx` (component, event-driven)

**Analog:** `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx`

**Button rendering with conditional visibility** (lines 106-124):
```tsx
<div className="flex flex-wrap items-center gap-2">
  {quote.status === "draft" && (
    <>
      <Button size="sm" className="gap-1.5" onClick={handleOpenSendDialog} disabled={isLoading || isDeleting}>
        <Send className="h-4 w-4" />
        Send Quote
      </Button>
      <Button
        variant="destructive"
        size="sm"
        className="gap-1.5"
        onClick={() => setShowDeleteConfirm(true)}
        disabled={isLoading || isDeleting}
      >
        <Trash2 className="h-4 w-4" />
        Delete
      </Button>
    </>
  )}
```

**Key adaptation:** Conditional rendering based on viewer's role (super_user), target user's role state, and self-view check instead of quote status.

---

### `trade-flow-ui/src/features/support/components/GrantRoleDialog.tsx` and `RevokeRoleDialog.tsx` (component, event-driven)

**Analog:** `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` (AlertDialog sections)

**AlertDialog imports** (lines 7-15):
```tsx
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { buttonVariants } from "@/components/ui/button.variants";
import { toast } from "@/lib/toast";
```

**AlertDialog with state control** (lines 189-213):
```tsx
<AlertDialog
  open={confirmAction !== null}
  onOpenChange={(open) => {
    if (!open) setConfirmAction(null);
  }}
>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>
        {confirmAction === "accepted" ? "Mark quote as accepted?" : "Mark quote as rejected?"}
      </AlertDialogTitle>
      <AlertDialogDescription>
        {confirmAction === "accepted"
          ? "Are you sure you want to mark this quote as accepted? This cannot be undone."
          : "Are you sure you want to mark this quote as rejected? This cannot be undone."}
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={handleConfirm}>
        {confirmAction === "accepted" ? "Accept" : "Reject"}
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

**Destructive AlertDialog variant** (lines 216-231):
```tsx
<AlertDialog open={showDeleteConfirm} onOpenChange={setShowDeleteConfirm}>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Delete {quote.number}?</AlertDialogTitle>
      <AlertDialogDescription>
        Delete {quote.number} for {quote.customerName}? This cannot be undone.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction className={buttonVariants({ variant: "destructive" })} onClick={handleDelete}>
        Delete
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

**Mutation + toast pattern** (lines 56-73):
```tsx
const handleTransition = async (status: "sent" | "accepted" | "rejected") => {
  try {
    await transitionQuote({
      businessId,
      quoteId: quote.id,
      status,
    }).unwrap();

    const labels: Record<string, string> = {
      sent: "Quote sent",
      accepted: "Quote marked as accepted",
      rejected: "Quote marked as rejected",
    };
    toast.success(labels[status]);
  } catch {
    toast.error("Failed to update quote status");
  }
};
```

---

## Shared Patterns

### Authentication Guard
**Source:** `trade-flow-api/src/auth/auth.guard.ts` (JwtAuthGuard)
**Apply to:** `SupportRoleAdminController`
```typescript
@UseGuards(JwtAuthGuard)
```
Note: Phase 52 adds `PermissionGuard` and `@RequiresPermission()`. The new controller should use both: `@UseGuards(JwtAuthGuard, PermissionGuard)` with `@RequiresPermission("manage_roles")`.

### Error Handling (API)
**Source:** `trade-flow-api/src/core/errors/handle-error.utility.ts`
**Apply to:** `SupportRoleAdminController`
```typescript
try {
  // ... service call
  return createResponse([result]);
} catch (error) {
  throw createHttpError(error);
}
```

### Response Formatting (API)
**Source:** `trade-flow-api/src/core/response/create-response.utility.ts`
**Apply to:** `SupportRoleAdminController`
```typescript
import { createResponse } from "@core/response/create-response.utility";
// Always wrap in array:
return createResponse([mappedResponse]);
```

### Logger Pattern (API)
**Source:** `trade-flow-api/src/user/services/business-role-assigner.service.ts` line 11
**Apply to:** `SupportRoleAssigner`
```typescript
private readonly logger = new AppLogger(SupportRoleAssigner.name);
```

### RTK Query Cache Invalidation (UI)
**Source:** `trade-flow-ui/src/services/userApi.ts` line 34
**Apply to:** Grant/revoke mutations
```typescript
invalidatesTags: ["User", "UserList"],
```

### Toast Notifications (UI)
**Source:** `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` lines 69-72
**Apply to:** `GrantRoleDialog`, `RevokeRoleDialog`
```tsx
import { toast } from "@/lib/toast";
toast.success("Admin role granted");
toast.error("Failed to update role");
```

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| -- | -- | -- | All files have analogs in the codebase |

**Note on Phase 52 dependencies:** `@RequiresPermission()` decorator and `PermissionGuard` are from Phase 52 (not yet implemented). The planner should ensure Phase 52 is complete before this phase, or include guard scaffolding as a prerequisite check. These are referenced in CONTEXT.md D-01 and RESEARCH.md but have no existing codebase analog to extract from.

## Critical Finding: ForbiddenError Details Type

The actual `ForbiddenError` constructor signature is `(code: ErrorCodes, details: string)` where `details` is a plain string. RESEARCH.md assumed `details` could be an object like `{ type: "last_admin_protection" }`. The planner must decide whether to:
1. Pass `type` as the details string directly (e.g., `"last_admin_protection"`)
2. Extend or modify `ForbiddenError` to accept structured details

## Metadata

**Analog search scope:** `trade-flow-api/src/user/`, `trade-flow-api/src/business/`, `trade-flow-api/src/core/`, `trade-flow-ui/src/services/`, `trade-flow-ui/src/features/quotes/`
**Files scanned:** 15 analog files read
**Pattern extraction date:** 2026-04-18
