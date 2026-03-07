# Phase 3: Schedule Data Model and Create API - Research

**Researched:** 2026-03-01
**Domain:** NestJS backend module -- MongoDB data model and REST create endpoint for schedule entries
**Confidence:** HIGH

## Summary

This phase creates a new `schedule` NestJS module with a MongoDB data model and a single POST endpoint for creating schedule entries on jobs. The codebase has a mature, well-documented pattern for new modules (visit-type, job, item) that this phase follows exactly. There are no novel technology decisions -- every building block (entity, DTO, repository, service, controller, policy, request validation, response mapping) has a direct analog in the existing codebase.

The schedule module introduces cross-module validation (job exists, visit type exists, assignee belongs to business) which follows the same pattern established by `JobCreatorService` (validating customer and job type). The main technical nuances are: (1) a request class with date, time, and integer-minute duration fields requiring specific class-validator decorators (`@Matches` for YYYY-MM-DD, `@IsMilitaryTime` for HH:mm, `@IsInt`/`@IsDivisibleBy` for 15-minute increments), and (2) defaulting the assignee to the authenticated user and duration to 60 minutes in the request-to-DTO mapper.

**Primary recommendation:** Clone the visit-type module structure exactly, add cross-module validation following the JobCreatorService pattern, and use class-validator's built-in decorators for all field validation.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **Date/time representation:** Separate `date` (string, `YYYY-MM-DD`) and `startTime` (string, `HH:mm` 24-hour format) fields. Stored in UTC, frontend converts. Start time accepts any minute value (not snapped). No date restrictions -- past and future allowed.
- **Duration format and granularity:** Stored as integer minutes. 15-minute increment validation (rejects values not divisible by 15). Minimum 15 minutes, maximum 1440 minutes (24 hours). Default 60 minutes when not provided.
- **Assignee modeling:** Single `assigneeId` field referencing user ID. Optional on create -- defaults to authenticated user. Validated against business membership. Returned as ID only.
- **Visit type on schedule entries:** `visitTypeId` is optional. Validated against business ownership when provided. No cascade on deactivation.
- **Notes field:** Included from day one as optional string to avoid migration in Phase 6.

### Claude's Discretion
- MongoDB index strategy for the schedules collection
- Status field initial value and enum design (scaffolding for Phase 4)
- Request validation decorator choices (class-validator patterns)
- Response shape following existing IResponse<T> conventions
- Error message wording for validation failures

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SCHED-06 | Schedule assignee defaults to the logged-in user | Request-to-DTO mapper sets `assigneeId` from `request.user.id` when omitted. Auth user available via `JwtAuthGuard` on `request.user`. Pattern proven in existing mappers. |
| SCHED-07 | Duration defaults to 1 hour when not specified | Request-to-DTO mapper uses `requestBody.durationMinutes ?? 60`. `@IsOptional()` on request field. Nullish coalescing pattern already used in codebase (e.g., `isDefault ?? false`). |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Framework (modules, DI, guards, controllers) | Project framework -- all modules use it |
| class-validator | 0.14.x | Request body validation decorators | Project standard for all request classes |
| class-transformer | 0.5.x | DTO transformation (used with `@Type()` if nested) | Project standard, paired with class-validator |
| mongodb (native driver) | 7.x | Database operations via MongoDbFetcher/MongoDbWriter | Project uses native driver, not Mongoose |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| luxon | 3.5.x | Date/time handling | Already a project dependency; use if any date manipulation needed in service layer |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `@IsMilitaryTime()` for startTime | `@Matches(/^\d{2}:\d{2}$/)` regex | IsMilitaryTime is a class-validator built-in that validates HH:MM format including range (00:00-23:59). Regex would need additional range validation. Use IsMilitaryTime. |
| `@Matches(/^\d{4}-\d{2}-\d{2}$/)` for date | `@IsDateString()` | IsDateString is an alias for IsISO8601 which accepts full ISO timestamps, not just YYYY-MM-DD. The regex pattern ensures exactly the date-only format the user decided on. Use Matches. |
| `@IsMongoId()` for ObjectId strings | `@IsString() @Length(24, 24)` | Codebase uses `@IsString() @IsNotEmpty() @Length(24, 24)` for ObjectId references (see CreateJobRequest). Follow established pattern. |

**Installation:**
```bash
# No new packages needed -- all dependencies already installed
```

## Architecture Patterns

### Recommended Project Structure
```
src/schedule/
├── schedule.module.ts                          # Module definition
├── controllers/
│   ├── schedule.controller.ts                  # POST endpoint
│   └── mappers/
│       ├── map-create-schedule-request-to-dto.utility.ts   # Request → DTO with defaults
│       └── map-schedule-to-response.utility.ts             # DTO → Response
├── services/
│   └── schedule-creator.service.ts             # Business logic + cross-module validation
├── repositories/
│   └── schedule.repository.ts                  # MongoDB CRUD
├── entities/
│   └── schedule.entity.ts                      # MongoDB document interface
├── data-transfer-objects/
│   └── schedule.dto.ts                         # DTO interface
├── requests/
│   └── create-schedule.request.ts              # class-validator decorated request
├── responses/
│   └── schedule.response.ts                    # Response interface
├── policies/
│   └── schedule.policy.ts                      # Business membership authorization
├── enum/
│   └── schedule-status.enum.ts                 # Status enum (scaffolding for Phase 4)
└── test/
    ├── services/
    │   └── schedule-creator.service.spec.ts
    ├── controllers/
    │   └── mappers/
    │       ├── map-create-schedule-request-to-dto.utility.spec.ts
    │       └── map-schedule-to-response.utility.spec.ts
    ├── repositories/
    │   └── schedule.repository.spec.ts
    ├── policies/
    │   └── schedule.policy.spec.ts
    └── mocks/
        └── schedule-mock-generator.ts
```

### Pattern 1: Entity/DTO/Response Layer Separation
**What:** Three separate interfaces for database document, inter-layer transfer, and API output
**When to use:** Every module -- this is the project's core architectural rule
**Example:**
```typescript
// Entity (repository layer only) -- uses ObjectId
interface IScheduleEntity extends IBaseEntity {
  businessId: ObjectId;
  jobId: ObjectId;
  visitTypeId: ObjectId | null;
  assigneeId: ObjectId;
  date: string;           // "YYYY-MM-DD"
  startTime: string;      // "HH:mm"
  durationMinutes: number; // integer, divisible by 15
  notes: string | null;
  status: ScheduleStatus;
}

// DTO (between layers) -- uses string IDs
interface IScheduleDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  jobId: string;
  visitTypeId: string | null;
  assigneeId: string;
  date: string;
  startTime: string;
  durationMinutes: number;
  notes: string | null;
  status: ScheduleStatus;
}

// Response (API output) -- independent, never extends DTO
interface IScheduleResponse {
  id: string;
  jobId: string;
  visitTypeId: string | null;
  assigneeId: string;
  date: string;
  startTime: string;
  durationMinutes: number;
  notes: string | null;
  status: string;
}
```
Source: Verified against visit-type entity/DTO/response pattern in codebase

### Pattern 2: Cross-Module Validation in Creator Service
**What:** Service validates foreign key references exist before creating the resource
**When to use:** When a new resource references entities from other modules (job, visit type, user/business)
**Example:**
```typescript
// Source: JobCreatorService pattern (src/job/services/job-creator.service.ts)
@Injectable()
export class ScheduleCreatorService implements ICreatorService {
  constructor(
    private readonly scheduleRepository: ScheduleRepository,
    private readonly schedulePolicy: SchedulePolicy,
    private readonly authorizedCreatorFactory: AuthorizedCreatorFactory,
    private readonly jobRetriever: JobRetrieverService,
    private readonly visitTypeRetriever: VisitTypeRetrieverService,
  ) {}

  public async create(authUser: IUserDto, schedule: IScheduleDto): Promise<IScheduleDto> {
    // 1. Validate job exists and user has access
    await this.jobRetriever.findByIdOrFail(authUser, schedule.jobId);

    // 2. Validate visit type if provided
    if (schedule.visitTypeId) {
      await this.visitTypeRetriever.findByIdOrFail(authUser, schedule.visitTypeId);
    }

    // 3. Validate assignee belongs to same business
    // (validate business membership via user's businessIds)

    // 4. Create via authorized creator (handles policy check)
    const authorizedCreator = this.authorizedCreatorFactory.createFor(
      this.scheduleRepository,
      this.schedulePolicy,
    );
    return authorizedCreator.create(authUser, schedule);
  }
}
```
Source: Verified against `src/job/services/job-creator.service.ts` lines 29-54

### Pattern 3: Request-to-DTO Mapper with Defaults
**What:** Controller mapper function applies smart defaults when converting request body to DTO
**When to use:** When create request has optional fields with default values
**Example:**
```typescript
// Source: mapCreateVisitTypeRequestToDto pattern
export const mapCreateScheduleRequestToDto = (
  requestBody: CreateScheduleRequest,
  jobId: string,
  businessId: string,
  authUserId: string,
): IScheduleDto => {
  return {
    id: new ObjectId().toString(),
    businessId,
    jobId,
    visitTypeId: requestBody.visitTypeId ?? null,     // optional
    assigneeId: requestBody.assigneeId ?? authUserId, // SCHED-06: default to auth user
    date: requestBody.date,
    startTime: requestBody.startTime,
    durationMinutes: requestBody.durationMinutes ?? 60, // SCHED-07: default 1 hour
    notes: requestBody.notes ?? null,                   // optional
    status: ScheduleStatus.SCHEDULED,                   // initial status
  };
};
```
Source: Verified against `src/visit-type/controllers/mappers/map-create-visit-type-request-to-dto.utility.ts`

### Pattern 4: Nested Resource URL with Job Scoping
**What:** Schedule is created under a job, which is itself under a business
**When to use:** Resources that belong to a parent resource
**Example:**
```typescript
// URL pattern following existing conventions:
// POST /v1/business/:businessId/job/:jobId/schedule
@UseGuards(JwtAuthGuard)
@Post("business/:businessId/job/:jobId/schedule")
public async create(
  @Req() request: { user: IUserDto; params: { businessId: string; jobId: string } },
  @Body() requestBody: CreateScheduleRequest,
): Promise<IResponse<IScheduleResponse>> { ... }
```
Source: Follows `POST /v1/business/:businessId/job` and `POST /v1/business/:businessId/visit-type` patterns

### Anti-Patterns to Avoid
- **Embedding schedules in the Job document:** Decision already made (STATE.md) to use a separate collection. Embedded arrays cause document growth issues and complicate queries.
- **Using Mongoose ODM:** Project uses native MongoDB driver exclusively. Do not introduce Mongoose schemas.
- **Validating business rules in the controller:** All cross-module validation (job exists, visit type exists, assignee membership) happens in the service layer, never the controller.
- **Returning entity interfaces from repository:** Always convert to DTO before returning from repository methods.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Time format validation (HH:mm) | Custom regex with hour/minute range check | `@IsMilitaryTime()` from class-validator | Built-in validates format AND range (00:00-23:59). Confirmed available in class-validator 0.14.x. |
| Date format validation (YYYY-MM-DD) | Custom date parser or luxon validation | `@Matches(/^\d{4}-\d{2}-\d{2}$/)` | Simple regex sufficient since no date restrictions (past/future both allowed). Avoids importing luxon in request layer. |
| Duration 15-minute increment check | Custom validation decorator | `@IsInt()` + `@IsDivisibleBy(15)` + `@Min(15)` + `@Max(1440)` | All four decorators are built into class-validator 0.14.x. Verified `IsDivisibleBy` exists in node_modules. |
| Authorization (business membership) | Custom middleware | `BasePolicy` + `AccessControllerFactory` + `AuthorizedCreatorFactory` | Project's established authorization framework. Policy checks business membership via `authUser.businessIds.includes(resource.businessId)`. |
| Audit timestamps | Manual `new Date()` calls | `createAuditFields()` / `updateAuditFields()` | Core utilities that ensure consistent timestamp creation. |
| ObjectId generation | `uuid()` or custom ID | `new ObjectId().toString()` in mapper | Established pattern -- all DTOs use MongoDB ObjectId strings. |

**Key insight:** This phase has zero novel technical problems. Every validation, authorization, and data access pattern has a direct codebase precedent. The risk is deviation from patterns, not missing technology.

## Common Pitfalls

### Pitfall 1: Forgetting Path Alias Registration
**What goes wrong:** TypeScript compiles but Jest tests fail with "Cannot find module @schedule/*"
**Why it happens:** `tsconfig.json` paths and `package.json` jest moduleNameMapper must both be updated
**How to avoid:** Add `@schedule/*` to BOTH `tsconfig.json` paths AND `package.json` jest moduleNameMapper in the same task. Also add `@schedule-test/*` for test imports.
**Warning signs:** TypeScript compiles but `npm test` fails with module resolution errors

### Pitfall 2: Missing Module Registration in AppModule
**What goes wrong:** Endpoint returns 404 at runtime
**Why it happens:** NestJS only discovers controllers/providers from registered modules
**How to avoid:** Add `ScheduleModule` to `AppModule.imports` array in `src/app.module.ts`
**Warning signs:** No compilation errors but HTTP requests return 404

### Pitfall 3: Not Exporting Required Services for Cross-Module Use
**What goes wrong:** Dependency injection fails in ScheduleModule when injecting JobRetrieverService or VisitTypeRetrieverService
**Why it happens:** JobModule currently only exports `JobTypeCreatorService`, not `JobRetrieverService`. VisitTypeModule only exports `VisitTypeCreatorService`.
**How to avoid:** Add `JobRetrieverService` to JobModule exports and `VisitTypeRetrieverService` to VisitTypeModule exports (or use `forwardRef` and import the modules directly).
**Warning signs:** NestJS startup error: "Nest cannot resolve dependencies of ScheduleCreatorService"

### Pitfall 4: Assignee Business Membership Validation
**What goes wrong:** A user could assign a schedule to someone outside the business
**Why it happens:** The `assigneeId` references a user, but the auth policy only checks if the *requesting* user has business access, not the *assignee*
**How to avoid:** In ScheduleCreatorService, verify the assignee's business membership. For the solo-operator MVP, the assignee will almost always be the auth user (SCHED-06 default), but the validation should still exist for correctness. Check that the assignee userId is in the business user list, or for the current phase scope, simply verify `authUser.businessIds` covers the schedule's businessId (since assignee defaults to auth user, this is implicitly validated by the policy).
**Warning signs:** No immediate failure, but becomes a security issue when team support is added

### Pitfall 5: Integer Duration from JSON
**What goes wrong:** JSON `durationMinutes: 60` arrives as a number but `@IsInt()` passes for floats in some edge cases
**Why it happens:** JSON numbers have no int/float distinction; `class-validator` `@IsInt()` checks `Number.isInteger()`
**How to avoid:** Stack `@IsInt()` (ensures integer) + `@IsDivisibleBy(15)` (ensures 15-min increments) + `@Min(15)` + `@Max(1440)`. The `@IsInt()` check handles the float edge case.
**Warning signs:** `60.5` passes validation and gets stored

### Pitfall 6: OpenAPI Specification Not Updated
**What goes wrong:** API docs are stale, causing frontend integration confusion in later phases
**Why it happens:** `openapi.yaml` requires manual updates per project convention
**How to avoid:** Update `openapi.yaml` with the new schedule schemas and endpoint in the same task as the controller
**Warning signs:** Running the API and checking docs shows missing schedule endpoints

## Code Examples

Verified patterns from the existing codebase:

### Schedule Entity Interface
```typescript
// Source: visit-type entity pattern (src/visit-type/entities/visit-type.entity.ts)
import { ObjectId } from "mongodb";
import { IBaseEntity } from "@core/entities/base.entity";
import { ScheduleStatus } from "@schedule/enum/schedule-status.enum";

export interface IScheduleEntity extends IBaseEntity {
  businessId: ObjectId;
  jobId: ObjectId;
  visitTypeId: ObjectId | null;
  assigneeId: ObjectId;
  date: string;
  startTime: string;
  durationMinutes: number;
  notes: string | null;
  status: ScheduleStatus;
}
```

### Create Schedule Request with Validation
```typescript
// Source: class-validator decorators verified in node_modules
import {
  IsString,
  IsNotEmpty,
  IsOptional,
  IsInt,
  Min,
  Max,
  IsDivisibleBy,
  IsMilitaryTime,
  Matches,
  Length,
  MaxLength,
} from "class-validator";

export class CreateScheduleRequest {
  @IsString()
  @IsNotEmpty()
  @Matches(/^\d{4}-\d{2}-\d{2}$/, { message: "date must be in YYYY-MM-DD format" })
  date: string;

  @IsString()
  @IsNotEmpty()
  @IsMilitaryTime({ message: "startTime must be in HH:mm format" })
  startTime: string;

  @IsOptional()
  @IsInt({ message: "durationMinutes must be a whole number" })
  @Min(15, { message: "durationMinutes must be at least 15" })
  @Max(1440, { message: "durationMinutes must be at most 1440" })
  @IsDivisibleBy(15, { message: "durationMinutes must be in 15-minute increments" })
  durationMinutes?: number;

  @IsOptional()
  @IsString()
  @IsNotEmpty()
  @Length(24, 24)
  visitTypeId?: string;

  @IsOptional()
  @IsString()
  @IsNotEmpty()
  @Length(24, 24)
  assigneeId?: string;

  @IsOptional()
  @IsString()
  @MaxLength(2000)
  notes?: string;
}
```

### Schedule Status Enum (Phase 4 Scaffolding)
```typescript
// Source: VisitTypeStatus pattern (src/visit-type/enum/visit-type-status.enum.ts)
export enum ScheduleStatus {
  SCHEDULED = "scheduled",
  CONFIRMED = "confirmed",
  COMPLETED = "completed",
  CANCELED = "canceled",
  NO_SHOW = "no_show",
}
// All statuses defined now (from VSTAT-01) but only SCHEDULED used as initial value in Phase 3.
// State transitions are Phase 4 scope.
```

### Repository with toDto/toEntity Conversion
```typescript
// Source: VisitTypeRepository pattern (src/visit-type/repositories/visit-type.repository.ts)
private toDto(entity: IScheduleEntity): IScheduleDto {
  return {
    id: entity._id.toString(),
    businessId: entity.businessId.toString(),
    jobId: entity.jobId.toString(),
    visitTypeId: entity.visitTypeId?.toString() ?? null,
    assigneeId: entity.assigneeId.toString(),
    date: entity.date,
    startTime: entity.startTime,
    durationMinutes: entity.durationMinutes,
    notes: entity.notes,
    status: entity.status,
  };
}

private toEntity(dto: IScheduleDto): IScheduleEntity {
  return {
    _id: dto.id ? new ObjectId(dto.id) : new ObjectId(),
    businessId: new ObjectId(dto.businessId),
    jobId: new ObjectId(dto.jobId),
    visitTypeId: dto.visitTypeId ? new ObjectId(dto.visitTypeId) : null,
    assigneeId: new ObjectId(dto.assigneeId),
    date: dto.date,
    startTime: dto.startTime,
    durationMinutes: dto.durationMinutes,
    notes: dto.notes,
    status: dto.status,
    ...createAuditFields(),
  };
}
```

### Module Definition with Cross-Module Imports
```typescript
// Source: JobModule pattern (src/job/job.module.ts)
import { forwardRef, Module } from "@nestjs/common";
import { CoreModule } from "@core/core.module";
import { UserModule } from "@user/user.module";
import { JobModule } from "@job/job.module";
import { VisitTypeModule } from "@visit-type/visit-type.module";

@Module({
  imports: [
    CoreModule,
    forwardRef(() => UserModule),
    forwardRef(() => JobModule),
    forwardRef(() => VisitTypeModule),
  ],
  controllers: [ScheduleController],
  providers: [
    ScheduleCreatorService,
    ScheduleRepository,
    SchedulePolicy,
  ],
  exports: [],
})
export class ScheduleModule {}
```

### Mock Generator for Tests
```typescript
// Source: VisitTypeMockGenerator pattern (src/visit-type/test/mocks/visit-type-mock-generator.ts)
export class ScheduleMockGenerator {
  public static createScheduleDto(overrides: Partial<IScheduleDto> = {}): IScheduleDto {
    return {
      id: new ObjectId().toString(),
      businessId: new ObjectId().toString(),
      jobId: new ObjectId().toString(),
      visitTypeId: null,
      assigneeId: new ObjectId().toString(),
      date: "2026-03-15",
      startTime: "09:00",
      durationMinutes: 60,
      notes: null,
      status: ScheduleStatus.SCHEDULED,
      ...overrides,
    };
  }

  public static createCreateScheduleRequest(overrides: Partial<CreateScheduleRequest> = {}): CreateScheduleRequest {
    const request = new CreateScheduleRequest();
    request.date = "2026-03-15";
    request.startTime = "09:00";
    Object.assign(request, overrides);
    return request;
  }

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
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mongoose ODM schemas | Native MongoDB driver with typed interfaces | Project inception | Entities are plain TypeScript interfaces, not Mongoose models |
| Single service per module | Split Creator/Retriever/Updater services | Project inception | Each service has one responsibility -- schedule only needs Creator for Phase 3 |
| Direct repository calls in service | AuthorizedCreatorFactory wrapping | Project inception | All creates go through policy-checked authorized creator |

**Deprecated/outdated:**
- None relevant -- the project's patterns are current and internally consistent

## MongoDB Index Strategy (Claude's Discretion)

**Recommendation:** Create a compound index on `{ jobId: 1, date: 1, startTime: 1 }` for the `schedules` collection.

**Rationale:**
- Phase 6 (SCHED-02) will query "all schedules for a job in chronological order" -- this index supports that query directly
- The compound index covers the most common query pattern (schedules by job, sorted by date/time)
- Additional indexes can be added in later phases if needed (e.g., by assigneeId for the "Today" screen in FUT-01)

**Implementation:** Add index creation in the repository constructor or as a migration step. The existing codebase uses `MongoConnectionService.getDb()` for direct collection access which supports `createIndex()`.

## Status Enum Design (Claude's Discretion)

**Recommendation:** Define the full enum from VSTAT-01 now (`SCHEDULED`, `CONFIRMED`, `COMPLETED`, `CANCELED`, `NO_SHOW`) but only use `SCHEDULED` as the initial value in Phase 3. This avoids a code change when Phase 4 adds state transitions.

**Rationale:**
- Defining the enum values costs nothing and prevents a file change in Phase 4
- The initial status for all new schedules is `SCHEDULED`
- State machine transitions are Phase 4 scope -- the enum values just exist as scaffolding

## Error Codes (Claude's Discretion)

**Recommendation:** Add these error codes to `ErrorCodes` enum and `ERRORS_MAP`:

| Code | Enum Key | Message | When Used |
|------|----------|---------|-----------|
| `SCHEDULE_0` | `SCHEDULE_JOB_NOT_FOUND` | "Job not found for schedule creation" | jobId doesn't exist or user lacks access |
| `SCHEDULE_1` | `SCHEDULE_VISIT_TYPE_NOT_FOUND` | "Visit type not found for schedule creation" | visitTypeId provided but doesn't exist or wrong business |
| `SCHEDULE_2` | `SCHEDULE_ASSIGNEE_NOT_IN_BUSINESS` | "Assignee does not belong to this business" | assigneeId fails business membership check |

**Rationale:** Follows the domain-prefixed error code pattern (JOB_0, JOB_1, VISIT_TYPE_0, etc.) and gives specific error messages for each validation failure.

## Cross-Module Dependency Analysis

### Services ScheduleModule Needs to Import

| Module | Service Needed | Currently Exported? | Action Required |
|--------|---------------|---------------------|-----------------|
| JobModule | `JobRetrieverService` | No (only `JobTypeCreatorService` exported) | Add `JobRetrieverService` to JobModule exports |
| VisitTypeModule | `VisitTypeRetrieverService` | No (only `VisitTypeCreatorService` exported) | Add `VisitTypeRetrieverService` to VisitTypeModule exports |
| UserModule | (via `forwardRef`) | Yes (standard pattern) | No change needed |
| CoreModule | `MongoDbFetcher`, `MongoDbWriter`, `AuthorizedCreatorFactory` etc. | Yes (global module) | No change needed |

**Critical:** The JobModule and VisitTypeModule export changes are prerequisites before ScheduleModule can function. These are small, safe changes (adding to the `exports` array).

## Configuration Changes Required

### tsconfig.json
```json
"@schedule/*": ["./src/schedule/*"],
"@schedule-test/*": ["./src/schedule/test/*"]
```

### package.json (Jest moduleNameMapper)
```json
"^@schedule/(.*)$": "<rootDir>/schedule/$1",
"^@schedule-test/(.*)$": "<rootDir>/schedule/test/$1"
```

### app.module.ts
```typescript
import { ScheduleModule } from "@schedule/schedule.module";
// Add ScheduleModule to imports array
```

## Open Questions

1. **Assignee business membership validation depth**
   - What we know: CONTEXT.md says "validated against business membership (assignee must belong to the same business)". For the solo-operator MVP, assignee defaults to auth user.
   - What's unclear: How to validate business membership for a different assignee. The `IUserDto` has `businessIds` but we'd need to fetch the assignee's user record to check their `businessIds`.
   - Recommendation: For Phase 3, validate that the auth user (request.user) has access to the business (via SchedulePolicy). When assignee == auth user (the default case and the only realistic solo-operator scenario), this is implicitly validated. Full assignee membership validation can be deferred to the team support feature (FUT-04). Add a TODO comment in the creator service noting this simplification.

## Sources

### Primary (HIGH confidence)
- Codebase inspection: `src/visit-type/` module -- all files read and patterns verified
- Codebase inspection: `src/job/services/job-creator.service.ts` -- cross-module validation pattern verified
- Codebase inspection: `src/job/controllers/job.controller.ts` -- nested resource URL pattern verified
- Codebase inspection: `src/core/` -- BasePolicy, AuthorizedCreatorFactory, IBaseEntity, IBaseResourceDto verified
- class-validator node_modules: `IsDivisibleBy`, `IsMilitaryTime`, `IsInt`, `Min`, `Max` decorators confirmed present in 0.14.x

### Secondary (MEDIUM confidence)
- class-validator `@IsMilitaryTime` validates HH:MM format including hour range 00-23 and minute range 00-59 (verified from type definition)
- class-validator `@IsDateString` is an ISO8601 alias that accepts full timestamps -- regex `@Matches` is more appropriate for date-only format

### Tertiary (LOW confidence)
- None -- all findings verified against codebase or node_modules

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies, all patterns exist in codebase
- Architecture: HIGH -- direct clone of visit-type module with JobCreator-style validation
- Pitfalls: HIGH -- all pitfalls derived from actual codebase patterns (path aliases, module registration, exports)

**Research date:** 2026-03-01
**Valid until:** 2026-03-31 (stable -- no external dependencies or fast-moving libraries)
