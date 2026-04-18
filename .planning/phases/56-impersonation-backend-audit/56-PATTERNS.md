# Phase 56: Impersonation Backend & Audit - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 16 new/modified files
**Analogs found:** 14 / 16

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `src/impersonation/impersonation.module.ts` | config | request-response | `src/tax-rate/tax-rate.module.ts` | exact |
| `src/impersonation/controllers/impersonation.controller.ts` | controller | request-response | `src/tax-rate/controllers/tax-rate.controller.ts` | exact |
| `src/impersonation/services/impersonation-creator.service.ts` | service | request-response | `src/document-token/services/document-token-creator.service.ts` | role-match |
| `src/impersonation/services/impersonation-terminator.service.ts` | service | request-response | `src/document-token/services/document-token-revoker.service.ts` | role-match |
| `src/impersonation/repositories/impersonation-audit.repository.ts` | repository | CRUD | `src/document-token/repositories/document-token.repository.ts` | role-match |
| `src/impersonation/entities/impersonation-audit.entity.ts` | model | CRUD | `src/document-token/entities/document-token.entity.ts` | exact |
| `src/impersonation/data-transfer-objects/impersonation-audit.dto.ts` | model | CRUD | `src/document-token/data-transfer-objects/document-token.dto.ts` | exact |
| `src/impersonation/requests/start-impersonation.request.ts` | model | request-response | `src/document-token/requests/decline-quote.request.ts` | role-match |
| `src/impersonation/requests/end-impersonation.request.ts` | model | request-response | `src/document-token/requests/decline-quote.request.ts` | exact |
| `src/impersonation/responses/impersonation.response.ts` | model | request-response | `src/tax-rate/responses/tax-rate.response.ts` | exact |
| `src/impersonation/enums/impersonation-status.enum.ts` | model | N/A | `src/quote/enums/quote-status.enum.ts` | exact |
| `src/auth/auth.guard.ts` (MODIFY) | middleware | request-response | N/A (self -- extending existing file) | self |
| `src/app.module.ts` (MODIFY) | config | N/A | N/A (self -- adding import) | self |
| `src/impersonation/test/services/impersonation-creator.service.spec.ts` | test | N/A | `src/document-token/test/services/document-token-creator.service.spec.ts` | exact |
| `src/impersonation/test/services/impersonation-terminator.service.spec.ts` | test | N/A | `src/document-token/test/services/document-token-creator.service.spec.ts` | role-match |
| `src/impersonation/test/mocks/impersonation-mock-generator.ts` | test | N/A | `src/document-token/test/mocks/document-token-mock-generator.ts` | exact |

## Pattern Assignments

### `src/impersonation/impersonation.module.ts` (config, request-response)

**Analog:** `src/tax-rate/tax-rate.module.ts`

**Module pattern** (lines 1-17):
```typescript
import { forwardRef, Module } from "@nestjs/common";
import { CoreModule } from "@core/core.module";
import { TaxRateController } from "@tax-rate/controllers/tax-rate.controller";
import { TaxRateRetrieverService } from "@tax-rate/services/tax-rate-retriever.service";
import { TaxRateCreatorService } from "@tax-rate/services/tax-rate-creator.service";
import { TaxRateUpdaterService } from "@tax-rate/services/tax-rate-updater.service";
import { TaxRateRepository } from "@tax-rate/repositories/tax-rate.repository";
import { TaxRatePolicy } from "@tax-rate/policies/tax-rate.policy";
import { UserModule } from "@user/user.module";

@Module({
  imports: [CoreModule, forwardRef(() => UserModule)],
  controllers: [TaxRateController],
  providers: [TaxRateRetrieverService, TaxRateCreatorService, TaxRateUpdaterService, TaxRateRepository, TaxRatePolicy],
  exports: [TaxRateRetrieverService, TaxRateCreatorService, TaxRateRepository],
})
export class TaxRateModule {}
```

**Adaptation notes:** ImpersonationModule imports `CoreModule` and `forwardRef(() => UserModule)` (for UserRetriever). Providers: ImpersonationCreator, ImpersonationTerminator, ImpersonationAuditRepository. No policy needed (permission-based auth via guard). No exports needed unless other modules consume impersonation services.

---

### `src/impersonation/controllers/impersonation.controller.ts` (controller, request-response)

**Analog:** `src/tax-rate/controllers/tax-rate.controller.ts`

**Imports pattern** (lines 1-16):
```typescript
import { Controller, Get, Post, Patch, Body, Req, UseGuards } from "@nestjs/common";
import { JwtAuthGuard } from "@auth/auth.guard";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { IResponse } from "@core/response/response.interface";
import { createResponse } from "@core/response/create-response.utility";
import { createHttpError } from "@core/errors/handle-error.utility";
```

**Controller class + route handler pattern** (lines 18-37):
```typescript
@Controller("v1")
export class TaxRateController {
  constructor(
    private readonly taxRateRetriever: TaxRateRetrieverService,
    private readonly taxRateCreator: TaxRateCreatorService,
  ) {}

  @UseGuards(JwtAuthGuard)
  @Get("business/:businessId/tax-rates")
  public async getByBusinessId(
    @Req() request: { user: IUserDto; params: { businessId: string } },
  ): Promise<IResponse<ITaxRateResponse>> {
    try {
      const taxRates = await this.taxRateRetriever.findByBusinessId(request.user, request.params.businessId);
      return createResponse(taxRates.map((taxRate) => this.mapToResponse(taxRate)));
    } catch (error) {
      throw createHttpError(error);
    }
  }
```

**POST handler with body validation** (lines 53-73):
```typescript
  @UseGuards(JwtAuthGuard)
  @Post("business/:businessId/tax-rate")
  public async create(
    @Req() request: { user: IUserDto; params: { businessId: string } },
    @Body() body: CreateTaxRateRequest,
  ): Promise<IResponse<ITaxRateResponse>> {
    try {
      const taxRate: ITaxRateDto = {
        id: new ObjectId().toString(),
        businessId: request.params.businessId,
        name: body.name,
        rate: body.rate,
        rateType: body.rateType ?? TaxRateType.PERCENTAGE,
        isDefault: false,
        status: TaxRateStatus.ENABLED,
      };

      const created = await this.taxRateCreator.create(request.user, taxRate);
      return createResponse([this.mapToResponse(created)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }
```

**Response mapping pattern** (lines 90-99):
```typescript
  private mapToResponse(dto: ITaxRateDto): ITaxRateResponse {
    return {
      id: dto.id,
      name: dto.name,
      rate: dto.rate,
      rateType: dto.rateType,
      isDefault: dto.isDefault,
      status: dto.status,
    };
  }
```

**Adaptation notes:** Controller uses `@Controller("v1")` prefix. Routes: `POST impersonation/start` and `POST impersonation/end`. Both guarded with `@UseGuards(JwtAuthGuard)`. When Phase 52 lands, add `@RequiresPermission('impersonate_user')` decorator. Request typing uses inline `{ user: IUserDto }`. Controller delegates to ImpersonationCreator/ImpersonationTerminator services.

---

### `src/impersonation/services/impersonation-creator.service.ts` (service, request-response)

**Analog:** `src/document-token/services/document-token-creator.service.ts`

**Creator service pattern** (lines 1-35):
```typescript
import { Injectable } from "@nestjs/common";
import { randomBytes } from "crypto";
import { DateTime } from "luxon";
import { ObjectId } from "mongodb";
import { IDocumentTokenDto } from "@document-token/data-transfer-objects/document-token.dto";
import { DocumentTokenRepository } from "@document-token/repositories/document-token.repository";

@Injectable()
export class DocumentTokenCreator {
  private static readonly TOKEN_BYTES = 32;
  private static readonly EXPIRY_DAYS = 30;

  constructor(private readonly documentTokenRepository: DocumentTokenRepository) {}

  public async create(
    documentId: string,
    documentType: "quote" | "estimate",
    recipientEmail: string,
  ): Promise<IDocumentTokenDto> {
    const token = randomBytes(DocumentTokenCreator.TOKEN_BYTES).toString("base64url");
    const now = DateTime.now();

    const dto: IDocumentTokenDto = {
      id: new ObjectId().toString(),
      token,
      documentType,
      documentId,
      expiresAt: now.plus({ days: DocumentTokenCreator.EXPIRY_DAYS }),
      sentAt: now,
      recipientEmail,
    };

    return this.documentTokenRepository.create(dto);
  }
}
```

**Adaptation notes:** ImpersonationCreator constructor takes `ImpersonationAuditRepository`, `UserRetriever`, and `ConfigService`. Method: `create(authUser: IUserDto, request: StartImpersonationRequest)`. Flow: (1) fetch target user via UserRetriever.getById(), (2) validate target has no support roles, (3) create audit DTO with `new ObjectId().toString()`, (4) sign JWT with `jsonwebtoken`, (5) return token + sessionId. Use `DateTime.utc().toISO()` for timestamps per project date convention.

---

### `src/impersonation/repositories/impersonation-audit.repository.ts` (repository, CRUD)

**Analog:** `src/document-token/repositories/document-token.repository.ts`

**Repository class structure** (lines 1-21, 114-127):
```typescript
import { Injectable } from "@nestjs/common";
import { ObjectId } from "mongodb";
import { DateTime } from "luxon";
import { IDocumentTokenDto } from "@document-token/data-transfer-objects/document-token.dto";
import { IDocumentTokenEntity } from "@document-token/entities/document-token.entity";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { MongoDbWriter } from "@core/services/mongo/mongo-db-writer.service";
import { AppLogger } from "@core/services/app-logger.service";
import { createAuditFields } from "@core/utilities/create-audit-fields.utility";

@Injectable()
export class DocumentTokenRepository {
  private static readonly COLLECTION = "document_tokens";
  private readonly logger = new AppLogger(DocumentTokenRepository.name);

  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
  ) {}
```

**Create method pattern** (lines 23-39):
```typescript
  public async create(dto: IDocumentTokenDto): Promise<IDocumentTokenDto> {
    const entity: IDocumentTokenEntity = {
      _id: new ObjectId(dto.id),
      ...createAuditFields(),
      token: dto.token,
      documentType: dto.documentType,
      documentId: new ObjectId(dto.documentId),
      expiresAt: dto.expiresAt.toJSDate(),
      sentAt: dto.sentAt.toJSDate(),
      recipientEmail: dto.recipientEmail,
    };

    this.logger.log("Creating document token", { documentId: dto.documentId, documentType: dto.documentType });
    await this.writer.insertOne<IDocumentTokenEntity>(DocumentTokenRepository.COLLECTION, entity);

    return this.toDto(entity);
  }
```

**findOne query pattern** (lines 41-50):
```typescript
  public async findByToken(token: string): Promise<IDocumentTokenDto | null> {
    const entity = await this.fetcher.findOne<IDocumentTokenEntity>(DocumentTokenRepository.COLLECTION, { token });

    if (!entity) {
      this.logger.log("Token not found", { token: token.substring(0, 8) + "..." });
      return null;
    }

    return this.toDto(entity);
  }
```

**Controlled updateOne pattern** (lines 52-58):
```typescript
  public async updateFirstViewedAt(tokenId: string): Promise<void> {
    await this.writer.updateOne<IDocumentTokenEntity>(
      DocumentTokenRepository.COLLECTION,
      { _id: new ObjectId(tokenId), firstViewedAt: { $exists: false } } as never,
      { $set: { firstViewedAt: new Date() } } as never,
    );
  }
```

**toDto conversion pattern** (lines 114-127):
```typescript
  private toDto(entity: IDocumentTokenEntity): IDocumentTokenDto {
    return {
      id: entity._id.toString(),
      token: entity.token,
      documentType: entity.documentType,
      documentId: entity.documentId.toString(),
      expiresAt: DateTime.fromJSDate(entity.expiresAt),
      revokedAt: entity.revokedAt ? DateTime.fromJSDate(entity.revokedAt) : undefined,
      sentAt: DateTime.fromJSDate(entity.sentAt),
      recipientEmail: entity.recipientEmail,
      firstViewedAt: entity.firstViewedAt ? DateTime.fromJSDate(entity.firstViewedAt) : undefined,
    };
  }
```

**Adaptation notes:** ImpersonationAuditRepository uses `COLLECTION = "impersonation_audit"`. Exposes only: `create()`, `findBySessionId()`, `findBySupportUserId()`, and `markEnded()`. The `markEnded()` follows the `updateFirstViewedAt` pattern -- a controlled single-field update using `writer.updateOne()` with `as never` casts. No general `update()` or `delete()` methods. The `toDto` converts Date fields to DateTime using `DateTime.fromJSDate()`, and nullable fields use ternary pattern.

---

### `src/impersonation/entities/impersonation-audit.entity.ts` (model)

**Analog:** `src/document-token/entities/document-token.entity.ts`

**Entity pattern** (lines 1-13):
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface IDocumentTokenEntity extends IBaseEntity {
  token: string;
  documentType: "quote" | "estimate";
  documentId: ObjectId;
  expiresAt: Date;
  revokedAt?: Date;
  sentAt: Date;
  recipientEmail: string;
  firstViewedAt?: Date;
}
```

**Base entity provides** (`src/core/entities/base.entity.ts` lines 1-7):
```typescript
import { ObjectId } from "mongodb";

export interface IBaseEntity extends Record<string, unknown> {
  _id: ObjectId;
  createdAt: Date;
  updatedAt: Date;
}
```

**Adaptation notes:** `IImpersonationAuditEntity extends IBaseEntity` with fields: `supportUserId: string`, `targetUserId: string`, `reason: string`, `startedAt: Date`, `endedAt: Date | null`, `status: string`. Dates stored as native JS `Date` in entities. Use `string` for userId fields (not ObjectId) since user IDs are stored as strings elsewhere in the codebase.

---

### `src/impersonation/data-transfer-objects/impersonation-audit.dto.ts` (model)

**Analog:** `src/document-token/data-transfer-objects/document-token.dto.ts`

**DTO pattern** (lines 1-14):
```typescript
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { DateTime } from "luxon";

export interface IDocumentTokenDto extends IBaseResourceDto {
  id: string;
  token: string;
  documentType: "quote" | "estimate";
  documentId: string;
  expiresAt: DateTime;
  revokedAt?: DateTime;
  sentAt: DateTime;
  recipientEmail: string;
  firstViewedAt?: DateTime;
}
```

**Base DTO** (`src/core/data-transfer-objects/base-resource.dto.ts` lines 1-3):
```typescript
export interface IBaseResourceDto {
  id: string;
}
```

**Adaptation notes:** `IImpersonationAuditDto extends IBaseResourceDto` with: `supportUserId: string`, `targetUserId: string`, `reason: string`, `startedAt: DateTime`, `endedAt: DateTime | null`, `status: ImpersonationStatus`. Date fields use Luxon `DateTime` per project convention.

---

### `src/impersonation/requests/start-impersonation.request.ts` (model, request-response)

**Analog:** `src/document-token/requests/decline-quote.request.ts`

**Request class with class-validator** (lines 1-8):
```typescript
import { IsOptional, IsString, MaxLength } from "class-validator";

export class DeclineQuoteRequest {
  @IsOptional()
  @IsString()
  @MaxLength(500)
  reason?: string;
}
```

**Adaptation notes:** `StartImpersonationRequest` has two required fields: `@IsString() @IsNotEmpty() targetUserId: string` and `@IsString() @IsNotEmpty() @MinLength(10) reason: string`. Import from `class-validator`.

---

### `src/impersonation/requests/end-impersonation.request.ts` (model, request-response)

**Analog:** Same as above.

**Adaptation notes:** `EndImpersonationRequest` has one required field: `@IsString() @IsNotEmpty() sessionId: string`.

---

### `src/impersonation/responses/impersonation.response.ts` (model, request-response)

**Analog:** `src/tax-rate/responses/tax-rate.response.ts`

**Response interface pattern** (lines 1-11):
```typescript
import { TaxRateType } from "@tax-rate/enums/tax-rate-type.enum";
import { TaxRateStatus } from "@tax-rate/enums/tax-rate-status.enum";

export interface ITaxRateResponse {
  id: string;
  name: string;
  rate: number;
  rateType: TaxRateType;
  isDefault: boolean;
  status: TaxRateStatus;
}
```

**Adaptation notes:** `IImpersonationResponse` for the start endpoint: `token: string`, `sessionId: string`, `expiresInMinutes: number`. Response interfaces are independent of DTOs per project convention.

---

### `src/impersonation/enums/impersonation-status.enum.ts` (model)

**Analog:** `src/quote/enums/quote-status.enum.ts`

**Enum pattern** (lines 1-8):
```typescript
export enum QuoteStatus {
  DRAFT = "draft",
  SENT = "sent",
  ACCEPTED = "accepted",
  REJECTED = "rejected",
  EXPIRED = "expired",
  DELETED = "deleted",
}
```

**Adaptation notes:** `ImpersonationStatus` with values: `ACTIVE = "active"`, `ENDED = "ended"`, `EXPIRED = "expired"`.

---

### `src/auth/auth.guard.ts` (MODIFY -- middleware, request-response)

**Current file** (lines 1-111):

**Token extraction** (lines 89-91):
```typescript
  private getTokenFromRequest(request: Request): string | null {
    return request.headers?.authorization?.split(" ")[1] ?? null;
  }
```

**JWT decode** (lines 93-110):
```typescript
  private extractUserInfoFromToken(token: string): IFirebaseUserInfoResponse {
    const decoded = jwt.decode(token);
    if (!decoded || typeof decoded === "string") {
      this.logger.error("Failed to decode JWT token");
      throw new UnauthorizedException("Invalid token");
    }
    // ...
  }
```

**canActivate entry point** (lines 24-86):
```typescript
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const canActivate = (await super.canActivate(context)) as boolean;
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      throw new UnauthorizedException();
    }

    if (canActivate) {
      try {
        // ... existing Firebase flow: lookup/create user, hydrate ...
        request.user = hydratedUser ?? createdUser;
      } catch (error) {
        this.logger.error("Error retrieving or creating user", { error });
        throw new UnauthorizedException();
      }
      return canActivate;
    }
    return false;
  }
```

**Adaptation notes:** The guard must be modified to intercept BEFORE `super.canActivate()` for impersonation tokens (since the Passport JWT strategy validates against Firebase public keys). New flow: (1) extract token from header, (2) `jwt.decode(token)` to peek at payload, (3) if `payload.type === "impersonation"`, verify with `jwt.verify(token, IMPERSONATION_JWT_SECRET, { algorithms: ["HS256"], issuer: "trade-flow-impersonation" })`, hydrate both target user and support user via `UserRetriever.getById()`, set `request.user` and `request.impersonator`, return true. (4) Otherwise, fall through to existing Firebase flow. Handle `jwt.TokenExpiredError` specifically for descriptive 401 messages. Inject `ConfigService` for secret access.

---

### `src/app.module.ts` (MODIFY -- config)

**Current imports array** (lines 30-63):
```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ ... }),
    LoggerModule.forRoot(loggerConfig),
    AuthModule,
    PingModule,
    UserModule,
    // ... other modules ...
    SubscriptionModule,
  ],
  controllers: [],
  providers: [{ provide: APP_GUARD, useClass: SubscriptionGuard }],
})
export class AppModule {}
```

**Adaptation notes:** Add `ImpersonationModule` to the imports array. Add import statement: `import { ImpersonationModule } from "@impersonation/impersonation.module";`. Requires adding `@impersonation/*` path alias to `tsconfig.json`.

---

### `src/impersonation/test/services/impersonation-creator.service.spec.ts` (test)

**Analog:** `src/document-token/test/services/document-token-creator.service.spec.ts`

**Test setup pattern** (lines 1-29):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { DateTime } from "luxon";
import { DocumentTokenCreator } from "@document-token/services/document-token-creator.service";
import { DocumentTokenRepository } from "@document-token/repositories/document-token.repository";

describe("DocumentTokenCreator", () => {
  let creator: DocumentTokenCreator;
  let repository: jest.Mocked<DocumentTokenRepository>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        DocumentTokenCreator,
        {
          provide: DocumentTokenRepository,
          useValue: {
            create: jest.fn().mockImplementation((dto) => Promise.resolve(dto)),
          },
        },
      ],
    }).compile();

    creator = module.get<DocumentTokenCreator>(DocumentTokenCreator);
    repository = module.get(DocumentTokenRepository);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
```

**Assertion patterns** (lines 31-65):
```typescript
  it("should generate a base64url token of 43 characters", async () => {
    const documentId = "507f1f77bcf86cd799439011";
    const result = await creator.create(documentId, "quote", "test@example.com");
    expect(result.token).toHaveLength(43);
  });

  it("should pass recipientEmail through to repository", async () => {
    await creator.create(documentId, "quote", "test@example.com");
    expect(repository.create).toHaveBeenCalledWith(
      expect.objectContaining({
        recipientEmail: "test@example.com",
      }),
    );
  });
```

**Adaptation notes:** Mock `ImpersonationAuditRepository`, `UserRetriever`, and `ConfigService`. Test cases: (1) creates audit entry and returns token for customer target, (2) throws ForbiddenError when target has support roles, (3) audit entry has correct fields, (4) JWT is signed with expected payload. Mock `UserRetriever.getById()` to return user DTOs with/without supportRoles.

---

### `src/impersonation/test/mocks/impersonation-mock-generator.ts` (test)

**Analog:** `src/document-token/test/mocks/document-token-mock-generator.ts`

**Mock generator pattern** (lines 1-32):
```typescript
import { ObjectId } from "mongodb";
import { DateTime } from "luxon";
import { IDocumentTokenDto } from "@document-token/data-transfer-objects/document-token.dto";

export class DocumentTokenMockGenerator {
  static create(overrides?: Partial<IDocumentTokenDto>): IDocumentTokenDto {
    return {
      id: new ObjectId().toString(),
      token: "mock-token-base64url-string-43chars-abcde",
      documentType: "quote",
      documentId: new ObjectId().toString(),
      expiresAt: DateTime.now().plus({ days: 30 }),
      sentAt: DateTime.now(),
      recipientEmail: "customer@example.com",
      ...overrides,
    };
  }

  static createExpired(overrides?: Partial<IDocumentTokenDto>): IDocumentTokenDto {
    return DocumentTokenMockGenerator.create({
      expiresAt: DateTime.now().minus({ days: 1 }),
      ...overrides,
    });
  }
}
```

**Adaptation notes:** `ImpersonationMockGenerator` with `static create(overrides?: Partial<IImpersonationAuditDto>)` returning default audit DTO. Add variant methods like `static createEnded()` and `static createExpired()`.

---

## Shared Patterns

### Error Handling
**Source:** `src/core/errors/handle-error.utility.ts` (lines 10-102)
**Apply to:** Controller (`impersonation.controller.ts`)
```typescript
// Every controller handler wraps in try/catch with createHttpError
try {
  const result = await this.service.create(request.user, body);
  return createResponse([this.mapToResponse(result)]);
} catch (error) {
  throw createHttpError(error);
}
```

### Error Classes
**Source:** `src/core/errors/forbidden-error.error.ts` (lines 1-31)
**Apply to:** ImpersonationCreator (lateral impersonation check), ImpersonationTerminator (session not found/already ended)
```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ForbiddenError } from "@core/errors/forbidden-error.error";

// Usage in service:
throw new ForbiddenError(ErrorCodes.ACTION_NOT_ALLOWED, "Cannot impersonate support users");
```

### Response Formatting
**Source:** `src/core/response/create-response.utility.ts` (lines 1-15)
**Apply to:** Controller
```typescript
import { createResponse } from "@core/response/create-response.utility";

// Always wrap single items in array
return createResponse([this.mapToResponse(result)]);
```

### Audit Fields
**Source:** `src/core/utilities/create-audit-fields.utility.ts` (lines 1-15)
**Apply to:** ImpersonationAuditRepository `create()` method
```typescript
import { createAuditFields } from "@core/utilities/create-audit-fields.utility";

const entity = {
  _id: new ObjectId(dto.id),
  ...createAuditFields(),
  // ... domain fields
};
```

### User Retrieval
**Source:** `src/user/services/user-retriever.service.ts` (lines 17-34)
**Apply to:** ImpersonationCreator (target user lookup), auth guard (hydrating users from token payload)
```typescript
// UserRetriever.getById() returns IUserDto | null
// Returns fully hydrated user with businessIds, supportRoles, businessRoles
const targetUser = await this.userRetriever.getById(request.targetUserId);
if (!targetUser) {
  throw new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "Target user not found");
}
```

### User DTO Structure (for support role check)
**Source:** `src/user/data-transfer-objects/user.dto.ts` (lines 1-56)
**Apply to:** ImpersonationCreator lateral impersonation check
```typescript
// IUserDto has supportRoles: DtoCollection<ISupportRoleDto>
// Check: targetUser.supportRoles.length > 0 means user has support roles
if (targetUser.supportRoles.length > 0) {
  throw new ForbiddenError(ErrorCodes.ACTION_NOT_ALLOWED, "Cannot impersonate support users");
}
```

### MongoDbWriter Method Signatures
**Source:** `src/core/services/mongo/mongo-db-writer.service.ts` (lines 19-35)
**Apply to:** ImpersonationAuditRepository
```typescript
// insertOne<TSchema>(collectionName: string, document: OptionalUnlessRequiredId<TSchema>): Promise<InsertOneResult<TSchema>>
await this.writer.insertOne<IImpersonationAuditEntity>(COLLECTION, entity);

// updateOne<TSchema>(collectionName: string, filter: Filter<TSchema>, update: UpdateFilter<TSchema>): void
await this.writer.updateOne<IImpersonationAuditEntity>(
  COLLECTION,
  { _id: new ObjectId(sessionId) } as never,
  { $set: { endedAt: new Date(), status: ImpersonationStatus.ENDED } } as never,
);
```

### MongoDbFetcher Method Signatures
**Source:** `src/core/services/mongo/mongo-db-fetcher.service.ts` (lines 16-29)
**Apply to:** ImpersonationAuditRepository
```typescript
// findOne<TSchema>(collectionName: string, filter: Filter<TSchema>, options?: FindOptions): Promise<TSchema | null>
const entity = await this.fetcher.findOne<IImpersonationAuditEntity>(COLLECTION, { _id: new ObjectId(sessionId) });
```

### Test Module Setup
**Source:** `src/document-token/test/services/document-token-creator.service.spec.ts` (lines 1-29)
**Apply to:** All test files
```typescript
import { Test, TestingModule } from "@nestjs/testing";

describe("ServiceName", () => {
  let service: ServiceClass;
  let dependency: jest.Mocked<DependencyClass>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ServiceClass,
        {
          provide: DependencyClass,
          useValue: {
            methodName: jest.fn().mockImplementation((dto) => Promise.resolve(dto)),
          },
        },
      ],
    }).compile();

    service = module.get<ServiceClass>(ServiceClass);
    dependency = module.get(DependencyClass);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
});
```

---

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| Express Request type extension (`request.impersonator`) | type-declaration | N/A | No existing Express Request augmentation in the codebase; use standard `declare global { namespace Express { interface Request { impersonator?: IUserDto } } }` pattern per RESEARCH.md |
| `@RequiresPermission` decorator usage | N/A | N/A | Phase 52 dependency not yet implemented; controller should be structured to easily add `@RequiresPermission('impersonate_user')` when available |

## Metadata

**Analog search scope:** `trade-flow-api/src/` (auth, core, document-token, tax-rate, quote, user modules)
**Files scanned:** 24 files read across 8 modules
**Pattern extraction date:** 2026-04-18
