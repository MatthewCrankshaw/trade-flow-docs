# Phase 1: Job Detail Cleanup -- Remove mocked data and wire real data - Pattern Map

**Mapped:** 2026-04-21
**Files analyzed:** 25 (new + modified)
**Analogs found:** 22 / 25

## File Classification

### Backend -- New Files (job-event module)

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-api/src/job-event/job-event.module.ts` | config | request-response | `src/schedule/schedule.module.ts` | exact |
| `trade-flow-api/src/job-event/controllers/job-event.controller.ts` | controller | request-response | `src/quote/controllers/quote.controller.ts` (GET endpoints) | role-match |
| `trade-flow-api/src/job-event/services/job-event-creator.service.ts` | service | CRUD (create-only) | `src/quote/services/quote-creator.service.ts` | role-match |
| `trade-flow-api/src/job-event/services/job-event-retriever.service.ts` | service | CRUD (read-only) | `src/quote/services/quote-retriever.service.ts` | exact |
| `trade-flow-api/src/job-event/repositories/job-event.repository.ts` | repository | CRUD | `src/quote/repositories/quote.repository.ts` | exact |
| `trade-flow-api/src/job-event/entities/job-event.entity.ts` | model | CRUD | `src/quote/entities/quote.entity.ts` | exact |
| `trade-flow-api/src/job-event/data-transfer-objects/job-event.dto.ts` | model | CRUD | `src/quote/data-transfer-objects/quote.dto.ts` | exact |
| `trade-flow-api/src/job-event/enums/job-event-type.enum.ts` | model | CRUD | `src/quote/enums/quote-status.enum.ts` | exact |
| `trade-flow-api/src/job-event/responses/job-event.response.ts` | model | request-response | `src/quote/responses/quote.responses.ts` | exact |
| `trade-flow-api/src/job-event/policies/job-event.policy.ts` | middleware | request-response | `src/quote/policies/quote.policy.ts` | exact |
| `trade-flow-api/src/job-event/test/services/job-event-creator.service.spec.ts` | test | -- | `src/quote/test/services/quote-transition.service.spec.ts` | role-match |
| `trade-flow-api/src/job-event/test/services/job-event-retriever.service.spec.ts` | test | -- | `src/quote/test/services/quote-transition.service.spec.ts` | role-match |

### Backend -- Modified Files (inline event writes)

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-api/src/quote/services/quote-creator.service.ts` | service | CRUD | self (add inline event call) | exact |
| `trade-flow-api/src/quote/services/quote-transition.service.ts` | service | CRUD | self (add inline event call) | exact |
| `trade-flow-api/src/schedule/services/schedule-creator.service.ts` | service | CRUD | self (add inline event call) | exact |
| `trade-flow-api/src/schedule/services/schedule-updater.service.ts` | service | CRUD | self (add inline event call) | exact |
| `trade-flow-api/src/job/services/job-updater.service.ts` | service | CRUD | self (add inline event call) | exact |
| `trade-flow-api/src/estimate/services/estimate-*.service.ts` | service | CRUD | self (add inline event call) | exact |

### Frontend -- New Files

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-ui/src/features/jobs/api/jobEventApi.ts` | service | request-response | `src/features/quotes/api/quoteApi.ts` | exact |

### Frontend -- Modified Files

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-ui/src/pages/JobDetailPage.tsx` | component | request-response | self (replace mocks with RTK hooks) | exact |
| `trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx` | component | request-response | self (remove mock interfaces, wire real data) | exact |
| `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` | component | request-response | self (remove Invoices/Notes tabs) | exact |
| `trade-flow-ui/src/features/jobs/components/JobTimeline.tsx` | component | request-response | self (wire to real API events) | exact |
| `trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx` | component | request-response | self (remove Create Invoice button) | exact |
| `trade-flow-ui/src/types/api.types.ts` | model | -- | self (add jobTypeId, address fields to Job) | exact |
| `trade-flow-ui/src/services/api.ts` | config | -- | self (add "JobEvent" tag type) | exact |

### Frontend -- Deleted Files

| File | Reason |
|------|--------|
| `trade-flow-ui/src/features/jobs/hooks/generate-mock-timeline.ts` | Replaced by real `jobEventApi.ts` |

---

## Pattern Assignments

### `trade-flow-api/src/job-event/job-event.module.ts` (config, request-response)

**Analog:** `trade-flow-api/src/schedule/schedule.module.ts`

**Full module pattern** (lines 1-27):
```typescript
import { forwardRef, Module } from "@nestjs/common";
import { CoreModule } from "@core/core.module";
import { UserModule } from "@user/user.module";
import { JobModule } from "@job/job.module";
import { VisitTypeModule } from "@visit-type/visit-type.module";
import { ScheduleRepository } from "@schedule/repositories/schedule.repository";
import { SchedulePolicy } from "@schedule/policies/schedule.policy";
import { ScheduleController } from "@schedule/controllers/schedule.controller";
import { ScheduleCreatorService } from "@schedule/services/schedule-creator.service";
import { ScheduleRetrieverService } from "@schedule/services/schedule-retriever.service";
import { ScheduleUpdaterService } from "@schedule/services/schedule-updater.service";
import { ScheduleTransitionService } from "@schedule/services/schedule-transition.service";

@Module({
  imports: [CoreModule, forwardRef(() => UserModule), forwardRef(() => JobModule), forwardRef(() => VisitTypeModule)],
  controllers: [ScheduleController],
  providers: [
    ScheduleRepository,
    SchedulePolicy,
    ScheduleCreatorService,
    ScheduleRetrieverService,
    ScheduleUpdaterService,
    ScheduleTransitionService,
  ],
  exports: [ScheduleRetrieverService],
})
export class ScheduleModule {}
```

**Key adaptation:** JobEventModule is simpler -- no updater/transition. Export `JobEventCreator` so other modules can inject it for inline event writes. Import `CoreModule` and `JobModule` (for policy lookups). Use `forwardRef(() => JobModule)` if circular dependency arises.

---

### `trade-flow-api/src/job-event/controllers/job-event.controller.ts` (controller, request-response)

**Analog:** `trade-flow-api/src/quote/controllers/quote.controller.ts`

**Imports pattern** (lines 1-11):
```typescript
import { JwtAuthGuard } from "@auth/auth.guard";
import { createHttpError } from "@core/errors/handle-error.utility";
import { createResponse } from "@core/response/create-response.utility";
import { IResponse } from "@core/response/response.interface";
import { Controller, Get, Query, Req, UseGuards } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
```

**Controller class + GET pattern** (lines 30-31, 43-55):
```typescript
@Controller("v1")
export class QuoteController {
  constructor(
    private readonly quoteRetriever: QuoteRetriever,
  ) {}

  @UseGuards(JwtAuthGuard)
  @Get("quote/:quoteId")
  public async findOne(@Req() request: { user: IUserDto; params: { quoteId: string } }): Promise<IResponse<IQuoteResponse>> {
    const quoteId = request.params.quoteId;
    try {
      const quote = await this.quoteRetriever.findByIdOrFail(request.user, quoteId);
      const response = await this.enrichAndMapToResponse(request.user, quote);
      return createResponse([response]);
    } catch (error) {
      const httpError = createHttpError(error);
      throw httpError;
    }
  }
```

**Key adaptation:** JobEventController only has a single GET endpoint using `@Query("jobId")` instead of `:params`. Pattern: `@Get("job-event")` with `@Query("jobId") jobId: string`. Response mapping is a simple private `mapToResponse` method (no enrichment needed).

---

### `trade-flow-api/src/job-event/services/job-event-creator.service.ts` (service, create-only)

**Analog:** `trade-flow-api/src/quote/services/quote-creator.service.ts`

**Service class pattern** (lines 1-17):
```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { InvalidRequestError } from "@core/errors/invalid-request.error";
import { AuthorizedCreatorFactory } from "@core/factories/authorized-creator.factory";
import { ICreatorService } from "@core/interfaces/creator-service.interface";
import { IQuoteDto } from "@quote/data-transfer-objects/quote.dto";
import { QuotePolicy } from "@quote/policies/quote.policy";
import { QuoteRepository } from "@quote/repositories/quote.repository";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { Injectable } from "@nestjs/common";

@Injectable()
export class QuoteCreator implements ICreatorService {
  constructor(
    private readonly quoteRepository: QuoteRepository,
    private readonly quotePolicy: QuotePolicy,
    private readonly authorizedCreatorFactory: AuthorizedCreatorFactory,
  ) {}
```

**Create method pattern** (lines 28-38):
```typescript
  public async create(authUser: IUserDto, quote: IQuoteDto): Promise<IQuoteDto> {
    await this.validateQuote(authUser, quote);
    const authorizedCreator = this.authorizedCreatorFactory.createFor(this.quoteRepository, this.quotePolicy);
    const createdQuote = await authorizedCreator.create(authUser, quoteWithNumber);
    return this.quoteTotalsCalculator.calculateTotals(createdQuote);
  }
```

**Key adaptation:** JobEventCreator is simpler -- no validation of external resources needed. It receives a pre-built DTO (jobId, businessId, type, metadata) and writes directly. It may skip `AuthorizedCreatorFactory` since events are always written by the system on behalf of the authenticated user (the calling service already validated authorization). If skipping factory, inject repository directly and call `repository.create(dto)`.

---

### `trade-flow-api/src/job-event/services/job-event-retriever.service.ts` (service, read-only)

**Analog:** `trade-flow-api/src/quote/services/quote-retriever.service.ts`

**Retriever pattern** (lines 1-14, 23-33):
```typescript
import { AccessControllerFactory } from "@core/factories/access-controller.factory";
import { IByIdRetrieverService } from "@core/interfaces/by-id-retriever-service.interface";
import { Injectable } from "@nestjs/common";
import { IQuoteDto } from "@quote/data-transfer-objects/quote.dto";
import { QuotePolicy } from "@quote/policies/quote.policy";
import { QuoteRepository } from "@quote/repositories/quote.repository";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Injectable()
export class QuoteRetriever implements IByIdRetrieverService {
  constructor(
    private readonly quoteRepository: QuoteRepository,
    private readonly quotePolicy: QuotePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
  ) {}

  public async findByIdOrFail(authUser: IUserDto, quoteId: string): Promise<IQuoteDto> {
    const quote = await this.quoteRepository.findByIdOrFail(quoteId);
    const accessController = this.accessControllerFactory.create<IQuoteDto>(this.quotePolicy);
    accessController.canRead(authUser, quote);
    return quote;
  }
```

**Key adaptation:** JobEventRetriever needs a `getByJobId(authUser, jobId)` method instead of `findByIdOrFail`. Authorization pattern: look up the job first (via JobRetrieverService), then verify user can read the job. This gives businessId-based access control without needing per-event policy checks. Return `IJobEventDto[]` sorted by `occurredAt` descending.

---

### `trade-flow-api/src/job-event/repositories/job-event.repository.ts` (repository, CRUD)

**Analog:** `trade-flow-api/src/quote/repositories/quote.repository.ts`

**Repository class pattern** (lines 1-25):
```typescript
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ResourceNotFoundError } from "@core/errors/resource-not-found.error";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { MongoDbWriter } from "@core/services/mongo/mongo-db-writer.service";
import { ICreatorRepository } from "@core/interfaces/creator-repository.interface";
import { AppLogger } from "@core/services/app-logger.service";
import { createAuditFields } from "@core/utilities/create-audit-fields.utility";
import { Filter, ObjectId } from "mongodb";

@Injectable()
export class QuoteRepository implements ICreatorRepository, IByIdRetrieverRepository {
  private static readonly COLLECTION = "quotes";
  private readonly logger = new AppLogger(QuoteRepository.name);

  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
  ) {}
```

**Create method** (lines 66-71):
```typescript
  public async create(dto: IQuoteDto): Promise<IQuoteDto> {
    const entity = this.toEntity(dto);
    await this.writer.insertOne<IQuoteEntity>(QuoteRepository.COLLECTION, entity);
    this.logger.debug("Created quote", { quoteId: entity._id.toString() });
    return this.toDto(entity);
  }
```

**findMany pattern** (lines 48-64):
```typescript
  public async findQuotesByBusinessId(businessId: string): Promise<IQuoteDto[]> {
    const businessObjectId = new ObjectId(businessId);
    const filter: Filter<IQuoteEntity> = { businessId: businessObjectId, status: { $ne: QuoteStatus.DELETED } };
    const entities = await this.fetcher.findMany<IQuoteEntity>(QuoteRepository.COLLECTION, filter);
    this.logger.debug("Found quotes by businessId", { businessId, total: entities.length });
    return Promise.all(entities.map(async (entity) => { ... return this.toDto(entity); }));
  }
```

**toDto/toEntity pattern** (lines 104-153):
```typescript
  private toDto(entity: IQuoteEntity): IQuoteDto {
    return {
      id: entity._id.toString(),
      businessId: entity.businessId.toString(),
      // ... map ObjectId to string, Date to DateTime
    };
  }

  private toEntity(dto: IQuoteDto): IQuoteEntity {
    return {
      _id: new ObjectId(),
      ...createAuditFields(),
      businessId: new ObjectId(dto.businessId),
      // ... map string to ObjectId, DateTime to Date
    };
  }
```

**Key adaptation:** Collection name = `"job_events"`. Repository needs `create()` and `findByJobId(jobId)` (sorted by `occurredAt` descending). No update/delete methods. The `findByJobId` method filters by `{ jobId: new ObjectId(jobId) }`.

---

### `trade-flow-api/src/job-event/entities/job-event.entity.ts` (model)

**Analog:** `trade-flow-api/src/quote/entities/quote.entity.ts`

**Entity pattern** (lines 1-22):
```typescript
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface IQuoteEntity extends IBaseEntity {
  businessId: ObjectId;
  customerId: ObjectId;
  jobId: ObjectId;
  number: string;
  // ... domain-specific fields using native types (Date, ObjectId)
}
```

**Base entity** (`src/core/entities/base.entity.ts`):
```typescript
export interface IBaseEntity extends Record<string, unknown> {
  _id: ObjectId;
  createdAt: Date;
  updatedAt: Date;
}
```

**Key adaptation:** JobEvent entity: `jobId: ObjectId`, `businessId: ObjectId`, `type: JobEventType` (string enum), `metadata: Record<string, unknown>`, `occurredAt: Date`.

---

### `trade-flow-api/src/job-event/data-transfer-objects/job-event.dto.ts` (model)

**Analog:** `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts`

**DTO pattern** (lines 1-8):
```typescript
import { DateTime } from "luxon";
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";

export interface IQuoteDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  // ... domain fields using string IDs and Luxon DateTime
}
```

**Key adaptation:** `IJobEventDto extends IBaseResourceDto` with: `id: string`, `jobId: string`, `businessId: string`, `type: JobEventType`, `metadata: Record<string, unknown>`, `occurredAt: DateTime`, `createdAt?: DateTime`, `updatedAt?: DateTime`.

---

### `trade-flow-api/src/job-event/enums/job-event-type.enum.ts` (model)

**Analog:** `trade-flow-api/src/quote/enums/quote-status.enum.ts`

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

**Key adaptation:** `JobEventType` enum with UPPER_SNAKE_CASE keys and lowercase string values per convention.

---

### `trade-flow-api/src/job-event/responses/job-event.response.ts` (model)

**Analog:** `trade-flow-api/src/quote/responses/quote.responses.ts`

**Response pattern** (lines 20-45):
```typescript
export interface IQuoteResponse {
  id: string;
  businessId: string;
  customerId: string;
  jobId: string;
  number: string;
  quoteDate: string;
  notes?: string;
  status: string;
  // ... all fields as primitives (string, number, Date) -- no Luxon, no ObjectId
}
```

**Key adaptation:** `IJobEventResponse` with: `id: string`, `jobId: string`, `businessId: string`, `type: string`, `metadata: Record<string, unknown>`, `occurredAt: string` (ISO 8601), `createdAt: string`, `updatedAt: string`. Response interfaces are independent of DTOs per CLAUDE.md.

---

### `trade-flow-api/src/job-event/policies/job-event.policy.ts` (middleware)

**Analog:** `trade-flow-api/src/quote/policies/quote.policy.ts`

**Policy pattern** (lines 1-15):
```typescript
import { BasePolicy } from "@core/policies/base.policy";
import { Injectable } from "@nestjs/common";
import { IQuoteDto } from "@quote/data-transfer-objects/quote.dto";
import { IUserDto } from "@user/data-transfer-objects/user.dto";
import { hasPermission } from "@auth/utilities/has-permission.utility";

@Injectable()
export class QuotePolicy extends BasePolicy<IQuoteDto> {
  public canCreate(authUser: IUserDto, resource: Omit<IQuoteDto, "id">): boolean {
    if (hasPermission(authUser, "view_support_dashboard")) return true;
    if (authUser.businessIds.includes(resource.businessId)) return true;
    return false;
  }

  public canRead(authUser: IUserDto, resource: IQuoteDto): boolean {
    if (hasPermission(authUser, "view_support_dashboard")) return true;
    if (authUser.businessIds.includes(resource.businessId)) return true;
    return false;
  }
```

**Key adaptation:** `JobEventPolicy extends BasePolicy<IJobEventDto>` -- same businessId ownership pattern. `canCreate` and `canRead` check `authUser.businessIds.includes(resource.businessId)`. No update/delete needed.

---

### `trade-flow-api/src/job-event/test/` (tests)

**Analog:** `trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts`

**Test setup pattern** (lines 1-66):
```typescript
import { Test, TestingModule } from "@nestjs/testing";
import { DateTime } from "luxon";
import { ObjectId } from "mongodb";

describe("QuoteTransitionService", () => {
  let service: QuoteTransitionService;
  let quoteRepository: { findByIdOrFail: jest.Mock; update: jest.Mock };
  let quotePolicy: { canUpdate: jest.Mock };
  let accessControllerFactory: { create: jest.Mock };

  const quoteId = new ObjectId().toString();
  const businessId = new ObjectId().toString();

  function createMockQuoteDto(overrides?: Partial<IQuoteDto>): IQuoteDto {
    return {
      id: quoteId,
      businessId,
      // ... default values
      ...overrides,
    } as IQuoteDto;
  }

  beforeEach(async () => {
    quoteRepository = { findByIdOrFail: jest.fn(), update: jest.fn() };
    // ... mock all dependencies as { methodName: jest.fn() }

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        QuoteTransitionService,
        { provide: QuoteRepository, useValue: quoteRepository },
        { provide: QuotePolicy, useValue: quotePolicy },
        { provide: AccessControllerFactory, useValue: accessControllerFactory },
      ],
    }).compile();

    service = module.get<QuoteTransitionService>(QuoteTransitionService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });
```

**Key patterns:** Mock generators as local functions with `overrides?: Partial<T>`, NestJS `TestingModule` for DI, `jest.clearAllMocks()` in `afterEach`, descriptive `describe`/`it` blocks.

---

### `trade-flow-ui/src/features/jobs/api/jobEventApi.ts` (service, request-response)

**Analog:** `trade-flow-ui/src/features/quotes/api/quoteApi.ts`

**RTK Query endpoint pattern** (lines 1-14):
```typescript
import { apiSlice } from "@/services/api";
import type { Quote, StandardResponse } from "@/types";

export const quoteApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getQuotes: builder.query<Quote[], string>({
      query: (businessId) => `/v1/business/${businessId}/quotes`,
      transformResponse: (response: StandardResponse<Quote>) => response.data,
      providesTags: (result) =>
        result
          ? [...result.map(({ id }) => ({ type: "Quote" as const, id })), { type: "Quote", id: "LIST" }]
          : [{ type: "Quote", id: "LIST" }],
    }),
```

**Export pattern** (lines 194-204):
```typescript
export const {
  useGetQuotesQuery,
} = quoteApi;
```

**Key adaptation:** Single `getJobEvents` query endpoint. Query: `(jobId: string) => /v1/job-event?jobId=${jobId}`. ProvidesTags with `"JobEvent"` type. Export `useGetJobEventsQuery`.

---

### `trade-flow-ui/src/services/api.ts` -- Tag registration (config)

**Current tagTypes** (lines 46-62):
```typescript
export const apiSlice = createApi({
  reducerPath: "api",
  baseQuery: baseQueryWithImpersonationExpiry,
  tagTypes: [
    "User",
    "Business",
    "Customer",
    "Item",
    "Quote",
    "Estimate",
    "Migration",
    "TaxRate",
    "JobType",
    "Job",
    "VisitType",
    "Schedule",
    "Subscription",
    "SupportUser",
    "DashboardMetrics",
  ],
  endpoints: () => ({}),
});
```

**Key adaptation:** Add `"JobEvent"` to the `tagTypes` array.

---

### `trade-flow-ui/src/types/api.types.ts` -- Job type update (model)

**Current Job interface** (lines 348-355):
```typescript
export interface Job {
  id: string;
  customerId: string;
  customerName: string;
  title: string;
  description: string | null;
  status: JobStatus;
}
```

**API response already includes** (from `trade-flow-api/src/job/responses/job.response.ts` lines 1-17):
```typescript
export interface IJobResponse {
  id: string;
  customerId: string;
  customerName: string;
  title: string;
  description: string | null;
  status: JobStatus;
  address: {
    addressLines: string[];
    countryCode: string;
    locality: string;
    postalCode: string;
    administrativeArea: string;
  } | null;
}
```

**Key adaptation:** Add `jobTypeId: string` and `address: { addressLines: string[]; countryCode: string; locality: string; postalCode: string; administrativeArea: string } | null` to the frontend `Job` interface. Also add a `JobEvent` type and `JobEventType` union type.

---

### `trade-flow-ui/src/pages/JobDetailPage.tsx` -- Mock removal + data wiring (component)

**Current mock data** (lines 34-54):
```typescript
const MOCK_CUSTOMER = { name: "John Smith", address: "14 Oakfield Road, Bristol BS8 2BJ" };
const MOCK_JOB_TYPE = "Window Installation";
const MOCK_ACCESS_NOTES = { parking: "...", pets: "...", keys: null, gateCode: "4821" };
const MOCK_COMMERCIAL = { quoted: "...", quoteStatus: "...", invoiced: "...", paid: "...", outstanding: "..." };
```

**Existing RTK hook usage to follow** (lines 93-103):
```typescript
const { data: job, isLoading, isError } = useGetJobQuery(jobId ?? "", { skip: !jobId });
const { data: schedules, isLoading: schedulesLoading } = useGetSchedulesByJobQuery(
  { businessId: businessId!, jobId: jobId! },
  { skip: !businessId || !jobId },
);
const { data: allQuotes = [], isLoading: quotesLoading } = useGetQuotesQuery(businessId!, { skip: !businessId });
```

**Key adaptation:** Remove all 4 MOCK constants. Add:
- `useGetCustomerQuery(job?.customerId, { skip: !job?.customerId })` -- uses existing hook
- `useGetJobTypeQuery(job?.jobTypeId, { skip: !job?.jobTypeId })` -- uses existing hook
- `useGetJobEventsQuery(jobId, { skip: !jobId })` -- new hook
- Remove `generateMockTimeline` import and usage (line 154)
- Pass real customer name, address, job type to child components
- Remove `handleCreateInvoice` handler (line 126)

---

### `trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx` -- Remove invoice button (component)

**Current invoice button** (lines 38-44):
```typescript
{(status === "in_progress" || status === "completed") && (
  <Button variant="outline" size="sm" className="gap-1.5" onClick={onCreateInvoice}>
    <Receipt className="h-4 w-4" />
    Create Invoice
  </Button>
)}
```

**Key adaptation:** Remove the `onCreateInvoice` prop from interface and JSX. Remove the `Receipt` icon import if unused. Remove the entire button block.

---

### `trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx` -- Remove mock interfaces (component)

**Mock interfaces to remove** (lines 13-26):
```typescript
interface MockCommercialData {
  quoted: string;
  quoteStatus: string;
  invoiced: string;
  paid: string;
  outstanding: string;
}

interface MockAccessNotes {
  parking: string | null;
  pets: string | null;
  keys: string | null;
  gateCode: string | null;
}
```

**Key adaptation:**
- Remove `MockCommercialData`, `MockAccessNotes` interfaces
- Remove `AccessNotesCard` component entirely (lines 96-145)
- Remove `accessNotes` prop from `JobOverviewSectionProps`
- Replace `CommercialSnapshotCard` with a simplified "Total Quoted" display using real quote data
- Add `quotes` prop (or `totalQuotedAmount: number`) to `JobOverviewSectionProps`
- Remove invoiced/paid/outstanding from commercial section

---

### `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` -- Remove tabs (component)

**Invoices tab to remove** (lines 112-115, 166-169):
```typescript
<TabsTrigger value="invoices" className="gap-1.5">
  <Receipt className="h-3.5 w-3.5" />
  Invoices
</TabsTrigger>
// ...
<TabsContent value="invoices" className="mt-4">
  <TabHeader title="Invoices" count={MOCK_INVOICES.length} onAdd={() => {}} />
  <InvoicesTable />
</TabsContent>
```

**Notes tab to remove** (lines 116-119, 171-175):
```typescript
<TabsTrigger value="notes" className="gap-1.5">
  <MessageSquare className="h-3.5 w-3.5" />
  Notes
</TabsTrigger>
// ...
<TabsContent value="notes" className="mt-4">
  <TabHeader title="Notes" count={MOCK_NOTES.length} onAdd={() => {}} />
  <NotesList />
</TabsContent>
```

**Key adaptation:** Remove `MOCK_INVOICES`, `MOCK_NOTES` constants. Remove Invoices tab trigger + content. Remove Notes tab trigger + content. Remove `InvoicesTable`, `NotesList` sub-components. Remove unused icon imports (`Receipt`, `MessageSquare`). Keep Schedule, Quotes, Photos, Files tabs.

---

## Shared Patterns

### Authentication Guard
**Source:** `trade-flow-api/src/auth/auth.guard.ts`
**Apply to:** `job-event.controller.ts`
```typescript
@UseGuards(JwtAuthGuard)
@Get("job-event")
public async getByJobId(@Req() request: { user: IUserDto; ... }) {
```

### Error Handling (API controllers)
**Source:** `trade-flow-api/src/core/errors/handle-error.utility.ts`
**Apply to:** `job-event.controller.ts`
```typescript
try {
  // operation
} catch (error) {
  const httpError = createHttpError(error);
  throw httpError;
}
```

### Response Formatting
**Source:** `trade-flow-api/src/core/response/create-response.utility.ts`
**Apply to:** `job-event.controller.ts`
```typescript
return createResponse(responses); // Always wraps in array
```

### Policy-based Authorization
**Source:** `trade-flow-api/src/core/factories/access-controller.factory.ts`
**Apply to:** `job-event-retriever.service.ts`
```typescript
const accessController = this.accessControllerFactory.create<IJobEventDto>(this.jobEventPolicy);
accessController.canRead(authUser, event);
```

### Audit Fields (Entity creation)
**Source:** `trade-flow-api/src/core/utilities/create-audit-fields.utility.ts`
**Apply to:** `job-event.repository.ts` in `toEntity`
```typescript
return {
  _id: new ObjectId(),
  ...createAuditFields(),
  // ... domain fields
};
```

### Date/Time Conversion (Repository layer)
**Source:** `trade-flow-api/src/core/utilities/to-date-time.utility.ts`
**Apply to:** `job-event.repository.ts` in `toDto`
```typescript
import { toDateTime, toOptionalDateTime } from "@core/utilities/to-date-time.utility";
// In toDto:
occurredAt: toDateTime(entity.occurredAt),
createdAt: toDateTime(entity.createdAt),
```

### RTK Query Data Wiring with Skip
**Source:** `trade-flow-ui/src/pages/JobDetailPage.tsx` (lines 93-100)
**Apply to:** All new RTK Query hook calls in `JobDetailPage.tsx`
```typescript
const { data: customer } = useGetCustomerQuery(job?.customerId, { skip: !job?.customerId });
const { data: jobType } = useGetJobTypeQuery(job?.jobTypeId, { skip: !job?.jobTypeId });
const { data: events } = useGetJobEventsQuery(jobId!, { skip: !jobId });
```

### RTK Query Cache Invalidation
**Source:** `trade-flow-ui/src/features/quotes/api/quoteApi.ts` (lines 37-39)
**Apply to:** Existing quote/schedule/estimate mutations
```typescript
invalidatesTags: [
  { type: "Quote", id: "LIST" },
  { type: "Job", id: "LIST" },
  // Add "JobEvent" here for mutations that create events
],
```

---

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `trade-flow-ui/src/features/jobs/components/JobCommercialSummary.tsx` (if extracted) | component | transform | May be a new extracted component for quotes-only commercial display. Could remain inline in `JobOverviewSection.tsx` -- planner's discretion. Pattern: standard React functional component with `useCurrency` hook for formatting. |

---

## Metadata

**Analog search scope:** `trade-flow-api/src/quote/`, `trade-flow-api/src/schedule/`, `trade-flow-api/src/core/`, `trade-flow-ui/src/features/jobs/`, `trade-flow-ui/src/features/quotes/`, `trade-flow-ui/src/features/schedules/`, `trade-flow-ui/src/services/`, `trade-flow-ui/src/types/`, `trade-flow-ui/src/pages/`
**Files scanned:** 30+ across both repositories
**Pattern extraction date:** 2026-04-21
