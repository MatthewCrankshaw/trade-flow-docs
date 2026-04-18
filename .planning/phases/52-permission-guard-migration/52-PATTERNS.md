# Phase 52: Permission Guard & Migration - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 22 (4 new, 16 modify, 2 delete)
**Analogs found:** 22 / 22

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `src/auth/decorators/requires-permission.decorator.ts` | decorator | request-response | `src/subscription/decorators/skip-subscription-check.decorator.ts` | exact |
| `src/auth/guards/permission.guard.ts` | guard | request-response | `src/subscription/guards/subscription.guard.ts` | exact |
| `src/auth/utilities/has-permission.utility.ts` | utility | transform | `src/user/utilities/is-support-user.utility.ts` | exact |
| `src/auth/utilities/has-any-permission.utility.ts` | utility | transform | `src/user/utilities/is-support-user.utility.ts` | exact |
| `src/auth/test/guards/permission.guard.spec.ts` | test | request-response | `src/subscription/test/guards/subscription.guard.spec.ts` | exact |
| `src/auth/test/utilities/has-permission.utility.spec.ts` | test | transform | (no utility test exists) | no-analog |
| `src/subscription/guards/subscription.guard.ts` | guard (modify) | request-response | self (line 48 changes) | self |
| `src/business/policies/business.policy.ts` | policy (modify) | CRUD | self (import + call swap) | self |
| `src/customer/policies/customer.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/job/policies/job.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/job/policies/job-type.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/quote/policies/quote.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/quote/policies/quote-line-item.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/quote-settings/policies/quote-settings.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/schedule/policies/schedule.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/visit-type/policies/visit-type.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/tax-rate/policies/tax-rate.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/item/policies/item.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/estimate/policies/estimate.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/estimate/policies/estimate-line-item.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/estimate-settings/policies/estimate-settings.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/user/policies/support-role.policy.ts` | policy (modify) | CRUD | `src/business/policies/business.policy.ts` | exact |
| `src/migration/policies/migration.policy.ts` | policy (modify) | CRUD | self (isSuperUser -> hasPermission swap) | self |
| `src/auth/auth.module.ts` | module (modify) | config | self (add PermissionGuard to providers/exports) | self |
| `src/user/utilities/is-support-user.utility.ts` | utility (delete) | -- | -- | -- |
| `src/user/utilities/is-super-user.utility.ts` | utility (delete) | -- | -- | -- |

## Pattern Assignments

### `src/auth/decorators/requires-permission.decorator.ts` (NEW, decorator, request-response)

**Analog:** `src/subscription/decorators/skip-subscription-check.decorator.ts`

**Complete file pattern** (lines 1-3):
```typescript
import { SetMetadata } from "@nestjs/common";

export const SkipSubscriptionCheck = () => SetMetadata("skipSubscriptionCheck", true);
```

**Adaptation:** Change from boolean metadata to string array metadata. The decorator accepts variadic permission names via `...permissions: string[]` and stores them under a `PERMISSIONS_KEY` constant. Export the key constant so the guard can import it.

---

### `src/auth/guards/permission.guard.ts` (NEW, guard, request-response)

**Analog:** `src/subscription/guards/subscription.guard.ts`

**Imports pattern** (lines 1-7):
```typescript
import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { SubscriptionRepository } from "@subscription/repositories/subscription.repository";
import { SubscriptionStatus } from "@subscription/enums/subscription-status.enum";
import { ForbiddenError } from "@core/errors/forbidden-error.error";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
```

**Guard class structure** (lines 9-14):
```typescript
@Injectable()
export class SubscriptionGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly subscriptionRepository: SubscriptionRepository,
  ) {}
```

**Reflector metadata reading** (lines 18-21, 24-27):
```typescript
const isPublic = this.reflector.getAllAndOverride<boolean>("isPublic", [context.getHandler(), context.getClass()]);
if (isPublic) {
  return true;
}
```

**Request user extraction** (lines 32-33, 40):
```typescript
const request = context.switchToHttp().getRequest();
// ...
const user = request.user as IUserDto | undefined;
```

**Error throwing pattern** (line 59):
```typescript
throw new ForbiddenError(ErrorCodes.SUBSCRIPTION_REQUIRED, "Active subscription required");
```

**Adaptation:** PermissionGuard is simpler than SubscriptionGuard. No repository dependency (constructor only needs `Reflector`). No async DB lookup. Read `PERMISSIONS_KEY` metadata via `getAllAndOverride<string[]>()`. If no metadata, return `true`. If metadata exists, extract `request.user`, call `hasAnyPermission()`, throw `ForbiddenError` with `ErrorCodes.ACTION_FORBIDDEN` if check fails.

---

### `src/auth/utilities/has-permission.utility.ts` (NEW, utility, transform)

**Analog:** `src/user/utilities/is-support-user.utility.ts`

**Complete file pattern** (lines 1-8):
```typescript
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { SupportRoleName } from "@user/enums/support-role-name.enum";

export function isSupportUser(authUser: IUserDto): boolean {
  return authUser.supportRoles.some((role) =>
    [SupportRoleName.SUPER_USER, SupportRoleName.SUPPORT_ADMINISTRATOR].includes(role.roleName as SupportRoleName),
  );
}
```

**Adaptation:** Instead of checking `role.roleName`, iterate over `supportRoles` and `businessRoles`, checking `role.permissions.some((p) => p.name === permission)`. Note: Phase 51 will add `permissions: DtoCollection<IPermissionDto>` to `ISupportRoleDto` and `IBusinessRoleDto`. The `DtoCollection.some()` method exists (line 83 of `src/core/collections/dto.collection.ts`).

---

### `src/auth/utilities/has-any-permission.utility.ts` (NEW, utility, transform)

**Analog:** Same as `has-permission.utility.ts` above.

**Pattern:** Thin wrapper calling `hasPermission()` for each permission in the array, returning `true` on first match. Follows the single-export-per-utility-file convention.

---

### `src/auth/test/guards/permission.guard.spec.ts` (NEW, test, request-response)

**Analog:** `src/subscription/test/guards/subscription.guard.spec.ts`

**Mock context factory** (lines 13-20):
```typescript
const createMockContext = (method: string, user?: IUserDto): ExecutionContext =>
  ({
    getHandler: () => ({}),
    getClass: () => ({}),
    switchToHttp: () => ({
      getRequest: () => ({ method, user }),
    }),
  }) as unknown as ExecutionContext;
```

**TestingModule setup with mocked Reflector** (lines 28-44):
```typescript
const module: TestingModule = await Test.createTestingModule({
  providers: [
    SubscriptionGuard,
    {
      provide: Reflector,
      useValue: {
        getAllAndOverride: jest.fn(),
      },
    },
    {
      provide: SubscriptionRepository,
      useValue: {
        findByUserId: jest.fn(),
      },
    },
  ],
}).compile();
```

**Reflector mock for metadata** (lines 58-59):
```typescript
reflector.getAllAndOverride.mockImplementation((key: string) => {
  return key === "isPublic";
});
```

**Error assertion pattern** (line 174):
```typescript
await expect(guard.canActivate(context)).rejects.toThrow(ForbiddenError);
```

**Adaptation:** PermissionGuard has no repository dependency -- simpler provider setup. Mock `Reflector.getAllAndOverride` to return permission string arrays. Test cases: (1) no metadata returns true, (2) user with matching permission returns true, (3) user without permission throws ForbiddenError, (4) no user throws ForbiddenError. Since `canActivate()` is synchronous (not async), use `expect(() => guard.canActivate(context)).toThrow(ForbiddenError)` instead of `rejects.toThrow`.

---

### `src/subscription/guards/subscription.guard.ts` (MODIFY, guard)

**Current support bypass** (lines 47-50):
```typescript
// Support users bypass subscription check
if (user.supportRoles && user.supportRoles.length > 0) {
  return true;
}
```

**Migration:** Replace with:
```typescript
import { hasPermission } from "@auth/utilities/has-permission.utility";
// ...
if (hasPermission(user, "bypass_subscription")) {
  return true;
}
```

---

### `src/business/policies/business.policy.ts` (MODIFY, policy -- template for all 14 policy migrations)

**Current import** (line 4):
```typescript
import { isSupportUser } from "@user/utilities/is-support-user.utility";
```

**Current usage** (lines 15, 25):
```typescript
if (isSupportUser(authUser)) {
  return true;
}
```

**Migration import:**
```typescript
import { hasPermission } from "@auth/utilities/has-permission.utility";
```

**Migration usage:**
```typescript
if (hasPermission(authUser, "view_support_dashboard")) {
  return true;
}
```

This exact pattern applies to all 14 policy files using `isSupportUser()`. The change is mechanical: swap the import and replace `isSupportUser(authUser)` with `hasPermission(authUser, "view_support_dashboard")`.

---

### `src/migration/policies/migration.policy.ts` (MODIFY, policy -- isSuperUser variant)

**Current import** (line 5):
```typescript
import { isSuperUser } from "@user/utilities/is-super-user.utility";
```

**Current usage** (lines 10, 14, 18, 22):
```typescript
return isSuperUser(authUser);
```

**Migration:**
```typescript
import { hasPermission } from "@auth/utilities/has-permission.utility";
// ...
return hasPermission(authUser, "manage_migrations");
```

---

### `src/auth/auth.module.ts` (MODIFY, module)

**Current providers** (line 28):
```typescript
providers: [JwtStrategy, JwtAuthGuard, FirebaseKeyProvider],
exports: [JwtAuthGuard, UserModule],
```

**Migration:** Add `PermissionGuard` to providers and exports so other modules can use it with `@UseGuards(PermissionGuard)`:
```typescript
providers: [JwtStrategy, JwtAuthGuard, PermissionGuard, FirebaseKeyProvider],
exports: [JwtAuthGuard, PermissionGuard, UserModule],
```

---

## Shared Patterns

### Error Handling for Permission Denials
**Source:** `src/core/errors/forbidden-error.error.ts` (lines 1-31)
**Apply to:** `permission.guard.ts`
```typescript
import { ForbiddenError } from "@core/errors/forbidden-error.error";
import { ErrorCodes } from "@core/errors/error-codes.enum";

// ErrorCodes.ACTION_FORBIDDEN = "CORE_7" (line 8 of error-codes.enum.ts)
throw new ForbiddenError(
  ErrorCodes.ACTION_FORBIDDEN,
  `Missing required permission: ${requiredPermissions.join(", ")}`,
);
```
Note: `ForbiddenError` constructor takes `(code: ErrorCodes, details: string)`. The `details` field is a string, not a structured object. D-07 mentions structured details but the existing pattern uses strings -- keep string format to match.

### DtoCollection Iteration for Permission Checking
**Source:** `src/core/collections/dto.collection.ts` (lines 83-85)
**Apply to:** `has-permission.utility.ts`
```typescript
public some(callback: (item: T, index: number, array: T[]) => boolean): boolean {
  return this._data.some(callback);
}
```
`DtoCollection` supports `.some()` and is iterable via `Symbol.iterator`. Both `supportRoles` and `businessRoles` on `IUserDto` are `DtoCollection` instances. Safe to call `.some()` on `DtoCollection.empty()` -- returns `false`.

### User Mock Generator Pattern
**Source:** `src/business/test/mocks/business-mock-generator.ts` (lines 65-79)
**Apply to:** All new and updated test files
```typescript
public static createUserDto(overrides: Partial<IUserDto> = {}): IUserDto {
  return {
    id: new ObjectId().toString(),
    externalAuthUserId: "firebase|123456789",
    nickname: "testuser",
    name: "Test User",
    email: "test@example.com",
    businessIds: [],
    supportRoleIds: [],
    businessRoleIds: [],
    supportRoles: DtoCollection.empty(),
    businessRoles: DtoCollection.empty(),
    ...overrides,
  };
}
```
After Phase 51, `ISupportRoleDto` will have `permissions: DtoCollection<IPermissionDto>`. Tests must build support users with permissions populated on their role DTOs.

### Guard Test Module Setup
**Source:** `src/subscription/test/guards/subscription.guard.spec.ts` (lines 1-49)
**Apply to:** `permission.guard.spec.ts`
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { ExecutionContext } from "@nestjs/common";
import { Reflector } from "@nestjs/core";

// Mock context factory
const createMockContext = (method: string, user?: IUserDto): ExecutionContext =>
  ({
    getHandler: () => ({}),
    getClass: () => ({}),
    switchToHttp: () => ({
      getRequest: () => ({ method, user }),
    }),
  }) as unknown as ExecutionContext;

// Test module setup
const module: TestingModule = await Test.createTestingModule({
  providers: [
    PermissionGuard,
    {
      provide: Reflector,
      useValue: {
        getAllAndOverride: jest.fn(),
      },
    },
  ],
}).compile();
```

### Policy Test Structure
**Source:** `src/business/test/policies/business.policy.spec.ts` (lines 1-14, 53-67)
**Apply to:** All policy test files that need updating
```typescript
import { BusinessPolicy } from "@business/policies/business.policy";
import { BusinessMockGenerator } from "@business-test/mocks/business-mock-generator";
import { DtoCollection } from "@core/collections/dto.collection";

describe("BusinessPolicy", () => {
  let businessPolicy: BusinessPolicy;

  beforeEach(() => {
    businessPolicy = new BusinessPolicy();
  });

  // Support user test (lines 54-67) -- THIS NEEDS UPDATING
  it("should return true when user is support user", () => {
    const supportUser = BusinessMockGenerator.createUserDto({
      supportRoleIds: ["support-role-id"],
      supportRoles: DtoCollection.create([{ id: "support-role-id", roleName: "super_user" }]),
      businessIds: [],
    });
    const result = businessPolicy.canRead(supportUser, business);
    expect(result).toBe(true);
  });
});
```
After migration, the support role mock must include `permissions` with the `view_support_dashboard` permission DTO, because `hasPermission()` reads permissions from role DTOs, not role names.

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `src/auth/test/utilities/has-permission.utility.spec.ts` | test | transform | No existing utility test files in `src/auth/test/` or `src/user/test/`. Follow standard Jest `describe/it` structure from policy tests. No NestJS TestingModule needed -- pure function tests. |

## Metadata

**Analog search scope:** `trade-flow-api/src/`
**Files scanned:** 22 target files, 8 analog files read in full
**Pattern extraction date:** 2026-04-18
