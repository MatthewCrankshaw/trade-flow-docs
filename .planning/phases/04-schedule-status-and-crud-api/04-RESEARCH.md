# Phase 4: Schedule Status and CRUD API - Research

**Researched:** 2026-03-01
**Domain:** NestJS API -- status state machine, CRUD endpoints, list/filter queries
**Confidence:** HIGH

## Summary

Phase 4 adds schedule lifecycle management to the existing schedule module: a status transition endpoint with state machine validation, list endpoints with filtering/pagination, an update endpoint with field-level constraints, and notes support. The existing codebase already has all the foundational patterns needed -- updater services (JobUpdater, VisitTypeUpdater), retriever services with list+pagination (JobRetriever), DtoCollection, AccessControllerFactory, and the schedule module scaffolding from Phase 3.

The status state machine is the primary new concept. It is a simple directed graph with 5 states and 5 transitions, with 3 terminal states. No external libraries are needed -- a plain TypeScript Map from status to allowed-next-statuses is the standard approach for this complexity level. The transition endpoint (POST .../transition) is architecturally separate from the update endpoint (PATCH), which matches the CONTEXT.md decision to separate lifecycle changes from field updates.

The existing migration creates indexes on `jobId + date + startTime` but Phase quick-2 merged those into `startDateTime`. The index needs updating to `jobId + startDateTime` for the list-by-job query. Additional indexes are needed for business-wide listing (`businessId + startDateTime`) and status filtering.

**Primary recommendation:** Follow established updater/retriever patterns exactly. The state machine is a pure function (no library needed). The main implementation risk is the MongoDbFetcher not supporting sort -- handle sort at the repository level using the MongoDB native driver directly, or extend the fetcher.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Allowed transitions per roadmap: scheduled->confirmed, confirmed->completed, scheduled->canceled, scheduled->no-show, confirmed->canceled
- Terminal states (completed, canceled, no-show) are final -- no reversals. If a mistake is made, create a new schedule
- Dedicated transition endpoint: POST /v1/business/:businessId/job/:jobId/schedule/:scheduleId/transition with { status: 'confirmed' }
- Invalid transition errors include current status and list of valid next statuses for clarity
- No extra data required for any transition -- notes can be added separately via update endpoint
- Cancel is a status transition (scheduled->canceled or confirmed->canceled), not a delete
- No hard delete endpoint -- SchedulePolicy.canDelete already returns false, keep it that way
- No cancellation reason field for MVP -- keep it simple, can be added later
- Canceled schedules are preserved in the database with all original data intact
- Cancel metadata: status change only, existing updatedAt timestamp captures when
- List by job: GET /v1/business/:businessId/job/:jobId/schedule (required by success criteria)
- Also support business-wide listing: GET /v1/business/:businessId/schedule (needed for Phase 5/6 calendar and list views)
- Filters: status filter (?status=scheduled,confirmed) and date range (?from=2026-03-01&to=2026-03-31)
- Pagination using existing DtoCollection pattern (limit/offset) for consistency
- Default sort: chronological, oldest first (upcoming schedules at top)
- Updates only allowed on scheduled and confirmed statuses -- completed, canceled, and no-show are locked
- Updatable fields: startDateTime, durationMinutes, assigneeId, visitTypeId, notes (all optional). Job association is permanent
- Time/duration changes on a confirmed schedule auto-reset status to scheduled (re-confirmation needed)
- Notes field uses replace (overwrite) pattern, matching standard PATCH update behavior

### Claude's Discretion
- Exact error code assignments for new error scenarios (transition errors, update-on-locked-status errors)
- Whether to create a separate ScheduleTransitionService or handle transitions in ScheduleUpdaterService
- Index strategy for the new list/filter queries
- UpdateScheduleRequest validation details

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| VSTAT-01 | Schedule entries have status: Scheduled, Confirmed, Completed, Canceled, No-show | ScheduleStatus enum already defines all 5 values (Phase 3 scaffolding). Status field exists on entity, DTO, and response. No changes needed to the status enum itself. |
| VSTAT-02 | User can transition status following valid state machine (scheduled->confirmed->completed, scheduled->canceled, scheduled->no-show, confirmed->canceled) | State machine as a TypeScript Map defining allowed transitions. Transition endpoint POST .../transition with { status } body. ScheduleTransitionService validates current status against allowed map, rejects invalid transitions with error including valid next statuses. |
| VSTAT-03 | Invalid status transitions are rejected by the API | InvalidRequestError with new SCHEDULE_INVALID_TRANSITION error code. Error details include current status and list of valid next statuses. IResponseError.meta can carry structured data (allowedTransitions, currentStatus). |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Already in use, provides DI, guards, controllers |
| class-validator | Already installed | Request body validation | Decorators for UpdateScheduleRequest and TransitionScheduleRequest |
| class-transformer | Already installed | Request transformation | @Type decorator for nested validation |
| mongodb (native) | Already installed | Database driver | Used by MongoDbFetcher/MongoDbWriter, needed for filter building |
| luxon | Already installed | DateTime handling | DateTime used in DTO for startDateTime, fromISO for parsing |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Jest | Already installed | Unit testing | All new services, utilities, and repository methods need tests |
| @nestjs/testing | Already installed | Test module setup | TestingModule for DI-based service tests |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Plain TS Map for state machine | xstate library | xstate is overkill for 5 states and 5 transitions -- adds 30KB+ dependency for no benefit |
| Repository-level sort | Extend MongoDbFetcher | Extending fetcher is cleaner long-term but higher blast radius -- repository-level is safer for this phase |

**Installation:**
No new packages needed. All dependencies are already installed.

## Architecture Patterns

### Recommended Project Structure
New and modified files for Phase 4:
```
schedule/
├── controllers/
│   ├── schedule.controller.ts          # ADD: list, getById, update, transition endpoints
│   └── mappers/
│       ├── map-schedule-to-response.utility.ts       # EXISTS (no change)
│       ├── map-create-schedule-request-to-dto.utility.ts  # EXISTS (no change)
│       └── merge-existing-schedule-with-changes.utility.ts  # NEW: merge update request onto existing DTO
├── services/
│   ├── schedule-creator.service.ts     # EXISTS (no change)
│   ├── schedule-retriever.service.ts   # NEW: findByIdOrFail, findByJobId, findByBusinessId
│   ├── schedule-updater.service.ts     # NEW: update fields with status-lock check
│   └── schedule-transition.service.ts  # NEW: status transitions with state machine
├── repositories/
│   └── schedule.repository.ts          # MODIFY: add update, findByJobId, findByBusinessId
├── requests/
│   ├── create-schedule.request.ts      # EXISTS (no change)
│   ├── update-schedule.request.ts      # NEW: optional fields with validation
│   └── transition-schedule.request.ts  # NEW: { status: ScheduleStatus }
├── enum/
│   ├── schedule-status.enum.ts         # EXISTS (no change)
│   └── schedule-transitions.ts         # NEW: state machine map
├── schedule.module.ts                  # MODIFY: register new services
└── test/
    ├── services/
    │   ├── schedule-retriever.service.spec.ts    # NEW
    │   ├── schedule-updater.service.spec.ts      # NEW
    │   └── schedule-transition.service.spec.ts   # NEW
    ├── controllers/
    │   └── mappers/
    │       └── merge-existing-schedule-with-changes.utility.spec.ts  # NEW
    └── mocks/
        └── schedule-mock-generator.ts  # MODIFY: add helpers for update/transition scenarios
```

### Pattern 1: Status State Machine as a Static Map
**What:** Define allowed transitions as a `Map<ScheduleStatus, ScheduleStatus[]>` (or `Record`).
**When to use:** When the state machine is small (< 10 states) and transitions have no side effects beyond the status change itself.
**Example:**
```typescript
// schedule/enum/schedule-transitions.ts
import { ScheduleStatus } from "@schedule/enum/schedule-status.enum";

export const ALLOWED_TRANSITIONS: ReadonlyMap<ScheduleStatus, readonly ScheduleStatus[]> = new Map([
  [ScheduleStatus.SCHEDULED, [ScheduleStatus.CONFIRMED, ScheduleStatus.CANCELED, ScheduleStatus.NO_SHOW]],
  [ScheduleStatus.CONFIRMED, [ScheduleStatus.COMPLETED, ScheduleStatus.CANCELED]],
  [ScheduleStatus.COMPLETED, []],
  [ScheduleStatus.CANCELED, []],
  [ScheduleStatus.NO_SHOW, []],
]);

export const isValidTransition = (from: ScheduleStatus, to: ScheduleStatus): boolean => {
  const allowed = ALLOWED_TRANSITIONS.get(from);
  return allowed !== undefined && allowed.includes(to);
};

export const getValidTransitions = (from: ScheduleStatus): readonly ScheduleStatus[] => {
  return ALLOWED_TRANSITIONS.get(from) ?? [];
};
```
**Confidence:** HIGH -- this is a standard TypeScript pattern, no external dependencies.

### Pattern 2: Separate Transition Service (Recommended)
**What:** Create a `ScheduleTransitionService` separate from `ScheduleUpdaterService`.
**When to use:** When lifecycle state changes have different authorization/validation rules than field updates.
**Why separate:** The CONTEXT.md explicitly separates the endpoints (POST .../transition vs PATCH). The transition service validates the state machine; the updater service validates field-level constraints and the "updates only on scheduled/confirmed" rule. Separation avoids a god-service.
**Example:**
```typescript
// schedule/services/schedule-transition.service.ts
@Injectable()
export class ScheduleTransitionService {
  constructor(
    private readonly scheduleRepository: ScheduleRepository,
    private readonly schedulePolicy: SchedulePolicy,
    private readonly accessControllerFactory: AccessControllerFactory,
  ) {}

  public async transition(
    authUser: IUserDto,
    existing: IScheduleDto,
    targetStatus: ScheduleStatus,
  ): Promise<IScheduleDto> {
    const accessController = this.accessControllerFactory.create(this.schedulePolicy);
    accessController.canUpdate(authUser, existing);

    if (!isValidTransition(existing.status, targetStatus)) {
      const validNext = getValidTransitions(existing.status);
      throw new InvalidRequestError(
        ErrorCodes.SCHEDULE_INVALID_TRANSITION,
        `Cannot transition from '${existing.status}' to '${targetStatus}'. Valid transitions: ${validNext.join(", ") || "none (terminal state)"}`,
      );
    }

    const updated: IScheduleDto = { ...existing, status: targetStatus };
    return this.scheduleRepository.update(updated);
  }
}
```
**Confidence:** HIGH -- follows existing service patterns (VisitTypeUpdater, JobUpdater).

### Pattern 3: Update with Status-Lock Guard
**What:** The updater service checks the current status before allowing field updates.
**When to use:** When certain statuses should freeze the record.
**Example:**
```typescript
// schedule/services/schedule-updater.service.ts
const UPDATABLE_STATUSES: readonly ScheduleStatus[] = [
  ScheduleStatus.SCHEDULED,
  ScheduleStatus.CONFIRMED,
];

public async update(authUser: IUserDto, existing: IScheduleDto, updates: IScheduleDto): Promise<IScheduleDto> {
  const accessController = this.accessControllerFactory.create(this.schedulePolicy);
  accessController.canUpdate(authUser, existing);

  if (!UPDATABLE_STATUSES.includes(existing.status)) {
    throw new InvalidRequestError(
      ErrorCodes.SCHEDULE_UPDATE_NOT_ALLOWED,
      `Cannot update schedule in '${existing.status}' status. Only 'scheduled' and 'confirmed' schedules can be updated.`,
    );
  }

  // Cross-module validation for visitTypeId if changed
  if (updates.visitTypeId && updates.visitTypeId !== existing.visitTypeId) {
    await this.validateVisitType(authUser, updates.visitTypeId);
  }

  // Auto-reset confirmed -> scheduled when time/duration changes
  let finalUpdates = updates;
  if (existing.status === ScheduleStatus.CONFIRMED) {
    const timeChanged = !updates.startDateTime.equals(existing.startDateTime);
    const durationChanged = updates.durationMinutes !== existing.durationMinutes;
    if (timeChanged || durationChanged) {
      finalUpdates = { ...updates, status: ScheduleStatus.SCHEDULED };
    }
  }

  return this.scheduleRepository.update(finalUpdates);
}
```
**Confidence:** HIGH -- follows JobUpdater/VisitTypeUpdater pattern with added business rules.

### Pattern 4: Controller Merge Utility for PATCH
**What:** A pure function that merges the existing DTO with incoming partial request, preserving unchanged fields.
**When to use:** For all PATCH update endpoints.
**Example (following `mergeExistingJobWithChanges`):**
```typescript
// schedule/controllers/mappers/merge-existing-schedule-with-changes.utility.ts
export const mergeExistingScheduleWithChanges = (
  existing: IScheduleDto,
  changes: UpdateScheduleRequest,
): IScheduleDto => {
  return {
    ...existing,
    startDateTime: changes.startDateTime
      ? DateTime.fromISO(changes.startDateTime, { zone: "utc" })
      : existing.startDateTime,
    durationMinutes: changes.durationMinutes ?? existing.durationMinutes,
    assigneeId: changes.assigneeId ?? existing.assigneeId,
    visitTypeId: changes.visitTypeId === undefined ? existing.visitTypeId : (changes.visitTypeId ?? null),
    notes: changes.notes === undefined ? existing.notes : (changes.notes ?? null),
  };
};
```
**Confidence:** HIGH -- directly mirrors `mergeExistingJobWithChanges` pattern.

### Pattern 5: Repository findMany with Sort (for List Endpoints)
**What:** The ScheduleRepository builds MongoDB filter+sort and calls the native driver, since MongoDbFetcher.findMany does not support sort.
**When to use:** When list queries need sorting (chronological, oldest first).
**Example:**
```typescript
// In ScheduleRepository
public async findByJobId(
  jobId: string,
  queryOptions: IBaseQueryOptionsDto,
  filters?: IScheduleFilterDto,
): Promise<DtoCollection<IScheduleDto>> {
  const db = await this.connection.getDb();
  const filter = this.buildFilter({ jobId: new ObjectId(jobId) }, filters);
  const pagination = this.resolvePagination(queryOptions);

  const results = await db
    .collection<IScheduleEntity>(ScheduleRepository.COLLECTION)
    .find(filter, {
      limit: pagination.limit,
      skip: (Math.max(pagination.page - 1, 0)) * pagination.limit,
      sort: { startDateTime: 1 },
    })
    .toArray();

  const total = await db
    .collection(ScheduleRepository.COLLECTION)
    .countDocuments(filter);

  return this.toDtoCollection(results, { total, limit: pagination.limit, page: pagination.page }, queryOptions);
}
```
**Confidence:** HIGH -- the VisitTypeRepository already uses `this.connection.getDb()` directly for `countActiveByBusinessId`. This follows the same pattern for more complex queries.

### Anti-Patterns to Avoid
- **Putting state machine logic in the controller:** Business rules belong in services. The controller only maps request to DTO and calls the service.
- **Using the update endpoint for status changes:** CONTEXT.md explicitly separates transition from update. Don't combine them into a single PATCH that also accepts status.
- **Allowing status field in UpdateScheduleRequest:** The update request should NOT include status. Status changes go through the transition endpoint only.
- **Hard-coding filter building in the controller:** Build MongoDB filters in the repository layer. Controllers should pass typed filter DTOs/options.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Request validation | Custom validation functions | class-validator decorators (@IsEnum, @IsOptional, @IsISO8601, @Length) | Handles edge cases, integrates with NestJS ValidationPipe |
| DateTime parsing | Manual date string parsing | `DateTime.fromISO(value, { zone: "utc" })` from Luxon | Timezone handling, validation, ISO8601 compliance |
| Pagination | Custom skip/limit math | Existing `paginationQueryToBaseQueryOptions` utility | Already handles validation and defaults |
| Authorization | Inline permission checks | `AccessControllerFactory.create(policy).canUpdate(user, resource)` | Consistent pattern, throws ForbiddenError automatically |
| Error responses | Manual HTTP exception building | `throw new InvalidRequestError(code, details)` + `createHttpError` | Maps to correct HTTP status (422), consistent error format |

**Key insight:** Every infrastructure piece needed already exists in the codebase. The only new concept is the state machine map, which is ~15 lines of TypeScript.

## Common Pitfalls

### Pitfall 1: Stale Migration Index
**What goes wrong:** The existing migration `20260301000000-create-schedules-indexes` creates an index on `jobId + date + startTime`, but Phase quick-2 merged those into a single `startDateTime` field. The index references fields that no longer exist.
**Why it happens:** The migration was written before the schema change.
**How to avoid:** Create a new migration that drops the old index and creates updated indexes: `{ jobId: 1, startDateTime: 1 }` for list-by-job, `{ businessId: 1, startDateTime: 1 }` for business-wide listing. Optionally add a compound index including status for filtered queries.
**Warning signs:** List queries return results in wrong order or perform full collection scans.

### Pitfall 2: MongoDbFetcher Lacks Sort Support
**What goes wrong:** Using `MongoDbFetcher.findMany` for list queries returns results in insertion order, not chronological order.
**Why it happens:** The `findMany` method accepts `IBaseQueryOptionsDto` which has a `sort` field, but the implementation ignores it -- no `.sort()` call is chained.
**How to avoid:** Use `this.connection.getDb()` directly in the repository for sorted queries (same approach as `countActiveByBusinessId` in VisitTypeRepository). This keeps the change scoped to the schedule module without modifying shared infrastructure.
**Warning signs:** List results appear in random order despite having a sort parameter.

### Pitfall 3: Forgetting Auto-Reset on Time Change
**What goes wrong:** A confirmed schedule has its time changed via update, but status stays "confirmed", giving a false sense that the new time is confirmed.
**Why it happens:** The auto-reset rule (confirmed -> scheduled on time/duration change) is easy to forget in implementation.
**How to avoid:** Implement the check explicitly in ScheduleUpdaterService. Use `DateTime.equals()` for startDateTime comparison and simple `!==` for durationMinutes. Add dedicated test cases for this behavior.
**Warning signs:** Tests for "update time on confirmed schedule" pass without status check assertions.

### Pitfall 4: Nullable vs Undefined in Merge Utility
**What goes wrong:** A PATCH request with `{ "visitTypeId": null }` should clear the field. A request without `visitTypeId` should preserve it. Using `??` conflates the two.
**Why it happens:** TypeScript/JSON treat `undefined` (field absent) and `null` (field explicitly set to null) differently, but `??` treats both as "use the default".
**How to avoid:** Use `=== undefined` checks for "field not provided" (preserve existing) vs explicit null checks for "clear the field". The `mergeExistingJobWithChanges` utility already demonstrates this pattern for the `address` field with its three-way check (undefined/null/value).
**Warning signs:** Cannot clear visitTypeId or notes once set.

### Pitfall 5: Missing Error Codes in ERRORS_MAP
**What goes wrong:** New ErrorCodes enum values are added but not registered in `ERRORS_MAP`. The `InvalidRequestError` constructor falls back to "Error code unknown".
**Why it happens:** Two files need to stay in sync: `error-codes.enum.ts` and `errors-map.constant.ts`.
**How to avoid:** Add entries to both files in the same task. Each new error code needs: (1) enum value in ErrorCodes, (2) entry in ERRORS_MAP with code + message.
**Warning signs:** API returns "Error code unknown" instead of the expected error message.

### Pitfall 6: Query Parameter Parsing for Comma-Separated Status Filter
**What goes wrong:** The `?status=scheduled,confirmed` query parameter arrives as a single string, not an array. Using `@IsEnum` on it directly fails.
**Why it happens:** Query parameters are strings by default in NestJS. Comma-separated values need manual parsing.
**How to avoid:** Parse the status query parameter manually: split on comma, validate each value against ScheduleStatus enum, then pass as an array to the repository filter. Similarly, parse `from` and `to` date range parameters with DateTime.fromISO validation.
**Warning signs:** Status filter returns 400 validation error or filters nothing.

## Code Examples

### TransitionScheduleRequest (class-validator)
```typescript
// schedule/requests/transition-schedule.request.ts
import { IsEnum } from "class-validator";
import { ScheduleStatus } from "@schedule/enum/schedule-status.enum";

export class TransitionScheduleRequest {
  @IsEnum(ScheduleStatus, {
    message: `status must be one of: scheduled, confirmed, completed, canceled, no_show`,
  })
  status: ScheduleStatus;
}
```
**Confidence:** HIGH -- follows CreateScheduleRequest pattern.

### UpdateScheduleRequest (all fields optional)
```typescript
// schedule/requests/update-schedule.request.ts
import {
  IsString,
  IsNotEmpty,
  IsOptional,
  IsInt,
  Min,
  Max,
  IsDivisibleBy,
  IsISO8601,
  Length,
  MaxLength,
} from "class-validator";

export class UpdateScheduleRequest {
  @IsOptional()
  @IsString()
  @IsNotEmpty()
  @IsISO8601({ strict: true }, { message: "startDateTime must be a valid ISO8601 date-time string" })
  startDateTime?: string;

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
  visitTypeId?: string | null;

  @IsOptional()
  @IsString()
  @IsNotEmpty()
  @Length(24, 24)
  assigneeId?: string;

  @IsOptional()
  @IsString()
  @MaxLength(2000)
  notes?: string | null;
}
```
**Confidence:** HIGH -- mirrors CreateScheduleRequest decorators with all fields optional.

### Controller Transition Endpoint
```typescript
// In schedule.controller.ts
@UseGuards(JwtAuthGuard)
@Post("business/:businessId/job/:jobId/schedule/:scheduleId/transition")
public async transition(
  @Req() request: {
    user: IUserDto;
    params: { businessId: string; jobId: string; scheduleId: string };
  },
  @Body() requestBody: TransitionScheduleRequest,
): Promise<IResponse<IScheduleResponse>> {
  try {
    const existing = await this.scheduleRetriever.findByIdOrFail(request.user, request.params.scheduleId);
    const transitioned = await this.scheduleTransition.transition(
      request.user,
      existing,
      requestBody.status,
    );
    const response = mapScheduleToResponse(transitioned);
    return createResponse([response]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```
**Confidence:** HIGH -- follows existing controller patterns exactly.

### Controller List Endpoint
```typescript
// In schedule.controller.ts
@UseGuards(JwtAuthGuard)
@Get("business/:businessId/job/:jobId/schedule")
public async findByJob(
  @Req() request: {
    user: IUserDto;
    params: { businessId: string; jobId: string };
    query: Record<string, string | string[] | undefined>;
  },
): Promise<IResponse<IScheduleResponse>> {
  try {
    const queryOptions = paginationQueryToBaseQueryOptions(request.query);
    const filters = parseScheduleFilters(request.query);
    const collection = await this.scheduleRetriever.findByJobId(
      request.user,
      request.params.jobId,
      queryOptions,
      filters,
    );
    const responses = collection.map((dto) => mapScheduleToResponse(dto));
    return createResponse(responses, collection.queryResults);
  } catch (error) {
    throw createHttpError(error);
  }
}
```
**Confidence:** HIGH -- follows JobController.findAll pattern.

### Error Code Recommendations
```typescript
// New entries in ErrorCodes enum:
SCHEDULE_INVALID_TRANSITION = "SCHEDULE_3",
SCHEDULE_UPDATE_NOT_ALLOWED = "SCHEDULE_4",
SCHEDULE_VISIT_TYPE_NOT_FOUND_FOR_UPDATE = "SCHEDULE_5",

// New entries in ERRORS_MAP:
[ErrorCodes.SCHEDULE_INVALID_TRANSITION, {
  code: ErrorCodes.SCHEDULE_INVALID_TRANSITION,
  message: "Invalid schedule status transition",
}],
[ErrorCodes.SCHEDULE_UPDATE_NOT_ALLOWED, {
  code: ErrorCodes.SCHEDULE_UPDATE_NOT_ALLOWED,
  message: "Schedule cannot be updated in its current status",
}],
[ErrorCodes.SCHEDULE_VISIT_TYPE_NOT_FOUND_FOR_UPDATE, {
  code: ErrorCodes.SCHEDULE_VISIT_TYPE_NOT_FOUND_FOR_UPDATE,
  message: "Visit type not found for schedule update",
}],
```
**Confidence:** HIGH -- follows existing naming pattern (SCHEDULE_0, SCHEDULE_1, SCHEDULE_2 already exist).

### Index Migration
```typescript
// migration/migrations/20260301100000-update-schedules-indexes.migration.ts
export default class UpdateSchedulesIndexesMigration implements IMigration {
  migrationName = "20260301100000-update-schedules-indexes.migration";

  async up(db: Db): Promise<void> {
    const collection = db.collection("schedules");

    // Drop stale index from before startDateTime merge
    try {
      await collection.dropIndex("jobId_1_date_1_startTime_1");
    } catch (_e) {
      // Index may not exist if DB was created after the merge
    }

    // Job-scoped chronological listing
    await collection.createIndex({ jobId: 1, startDateTime: 1 });

    // Business-wide chronological listing (Phase 5/6 calendar)
    await collection.createIndex({ businessId: 1, startDateTime: 1 });
  }

  async down(db: Db): Promise<void> {
    const collection = db.collection("schedules");
    await collection.dropIndex("jobId_1_startDateTime_1");
    await collection.dropIndex("businessId_1_startDateTime_1");
  }

  isBootstrap(): boolean {
    return false;
  }
}
```
**Confidence:** HIGH -- follows existing migration pattern exactly.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate date + startTime fields | Single startDateTime ISO8601 field | Phase quick-2 (2026-03-01) | Index migration needs updating; filter queries use startDateTime range |
| No schedule status transitions | Status field exists but only SCHEDULED used | Phase 3 (2026-03-01) | All 5 enum values already defined, just need transition logic |

**Deprecated/outdated:**
- The `date` and `startTime` separate fields were merged in quick-2. The existing migration `20260301000000-create-schedules-indexes` references the old field names and needs replacement.

## Open Questions

1. **Sort in MongoDbFetcher**
   - What we know: `IBaseQueryOptionsDto` has a `sort` field but `MongoDbFetcher.findMany` ignores it.
   - What's unclear: Whether this is intentional or an oversight.
   - Recommendation: Use `connection.getDb()` directly in the repository for sorted queries (same as `countActiveByBusinessId`). Do not modify the shared MongoDbFetcher -- keep the change scoped. This avoids touching shared infrastructure.

2. **Total count for paginated queries**
   - What we know: MongoDbFetcher.findMany sets `total: results.length` which is the page count, not the full collection count. JobRetriever uses this as-is.
   - What's unclear: Whether the existing pattern is intentional (total = page count) or a known limitation.
   - Recommendation: For the schedule list endpoints, use `countDocuments` alongside `.find()` when querying directly via the native driver. This gives accurate total for frontend pagination. The repository already has access to `this.connection.getDb()`.

3. **Filter DTO typing**
   - What we know: Need to pass status array and date range from controller to repository.
   - What's unclear: Whether to use a formal interface or inline parameters.
   - Recommendation: Create a lightweight `IScheduleFilterDto` interface (`status?: ScheduleStatus[]`, `from?: DateTime`, `to?: DateTime`) for type safety. Keep it in the data-transfer-objects directory.

## Sources

### Primary (HIGH confidence)
- Existing codebase files examined directly:
  - `src/schedule/` -- full module structure, entity, DTO, repository, service, controller, tests
  - `src/job/services/job-updater.service.ts` -- updater pattern reference
  - `src/job/services/job-retriever.service.ts` -- retriever pattern with list+pagination
  - `src/job/controllers/job.controller.ts` -- controller CRUD pattern reference
  - `src/job/controllers/mappers/merge-existing-job-with-chanages.utility.ts` -- merge pattern for PATCH
  - `src/visit-type/services/visit-type-updater.service.ts` -- updater with business rule validation
  - `src/visit-type/repositories/visit-type.repository.ts` -- repository update + countDocuments via connection
  - `src/core/errors/error-codes.enum.ts` -- existing error code numbering
  - `src/core/errors/errors-map.constant.ts` -- error code to message mapping
  - `src/core/services/mongo/mongo-db-fetcher.service.ts` -- findMany lacks sort support
  - `src/core/services/mongo/mongo-db-writer.service.ts` -- findOneAndUpdate with returnDocument: "after"
  - `src/core/data-transfer-objects/base-query-options.dto.ts` -- has sort field (unused by fetcher)
  - `src/core/collections/dto.collection.ts` -- pagination wrapper
  - `src/core/utilities/update-audit-fields.utility.ts` -- sets updatedAt
  - `src/core/response/response-error.interface.ts` -- meta field available for structured error data
  - `src/migration/migrations/20260301000000-create-schedules-indexes.migration.ts` -- stale index

### Secondary (MEDIUM confidence)
- `.github/copilot-instructions.md` and sub-files -- coding standards, module structure, API patterns, testing standards

### Tertiary (LOW confidence)
- None -- all findings verified against existing codebase.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and in use
- Architecture: HIGH -- every pattern has a direct precedent in the existing codebase (JobUpdater, JobRetriever, VisitTypeUpdater)
- Pitfalls: HIGH -- identified from direct code inspection (stale migration, missing sort, nullable merge)

**Research date:** 2026-03-01
**Valid until:** 2026-03-31 (stable -- no external dependencies, patterns are internal)
