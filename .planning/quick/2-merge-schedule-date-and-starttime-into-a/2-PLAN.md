---
phase: quick-2
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-api/src/schedule/entities/schedule.entity.ts
  - trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts
  - trade-flow-api/src/schedule/requests/create-schedule.request.ts
  - trade-flow-api/src/schedule/responses/schedule.response.ts
  - trade-flow-api/src/schedule/repositories/schedule.repository.ts
  - trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts
  - trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts
  - trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts
  - trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts
  - trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts
  - trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts
  - trade-flow-api/openapi.yaml
autonomous: true
requirements: [QUICK-2]

must_haves:
  truths:
    - "Entity stores a single startDateTime as ISO8601 UTC string instead of separate date and startTime"
    - "DTO exposes a single startDateTime as Luxon DateTime instead of separate date and startTime"
    - "API request accepts a single startDateTime ISO8601 string instead of separate date and startTime"
    - "API response returns a single startDateTime ISO8601 UTC string instead of separate date and startTime"
    - "All 34 existing tests pass after the refactor (updated to match new field)"
  artifacts:
    - path: "trade-flow-api/src/schedule/entities/schedule.entity.ts"
      provides: "IScheduleEntity with startDateTime: string"
      contains: "startDateTime: string"
    - path: "trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts"
      provides: "IScheduleDto with startDateTime: DateTime"
      contains: "startDateTime: DateTime"
    - path: "trade-flow-api/src/schedule/requests/create-schedule.request.ts"
      provides: "CreateScheduleRequest with startDateTime: string (ISO8601 validated)"
      contains: "startDateTime"
    - path: "trade-flow-api/src/schedule/responses/schedule.response.ts"
      provides: "IScheduleResponse with startDateTime: string"
      contains: "startDateTime: string"
  key_links:
    - from: "trade-flow-api/src/schedule/repositories/schedule.repository.ts"
      to: "schedule.entity.ts / schedule.dto.ts"
      via: "toDto converts entity.startDateTime (ISO string) to DateTime.fromISO; toEntity converts dto.startDateTime to .toUTC().toISO()"
      pattern: "startDateTime.*fromISO|toISO"
    - from: "trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts"
      to: "schedule.dto.ts"
      via: "Converts request.startDateTime string to DateTime.fromISO for the DTO"
      pattern: "DateTime\\.fromISO.*startDateTime"
---

<objective>
Merge the separate `date` (YYYY-MM-DD string) and `startTime` (HH:mm string) fields into a single `startDateTime` field across the entire schedule module. The entity stores it as an ISO8601 UTC string (for MongoDB), the DTO uses a Luxon DateTime, the request accepts an ISO8601 string, and the response returns an ISO8601 UTC string. This aligns with best practices for datetime handling and simplifies timezone-aware scheduling.

Purpose: Eliminate the date/time split that makes UTC conversion and timezone handling awkward. A single ISO8601 datetime field is the standard approach for scheduling systems.
Output: All schedule module files updated with the merged field, all tests passing.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-api/CLAUDE.md
@trade-flow-api/.github/copilot-instructions.md

<interfaces>
<!-- Current interfaces that will be modified. All references to `date` and `startTime` become `startDateTime`. -->

From trade-flow-api/src/schedule/entities/schedule.entity.ts:
```typescript
export interface IScheduleEntity extends IBaseEntity {
  businessId: ObjectId;
  jobId: ObjectId;
  visitTypeId: ObjectId | null;
  assigneeId: ObjectId;
  date: string;          // REMOVE - merge into startDateTime
  startTime: string;     // REMOVE - merge into startDateTime
  durationMinutes: number;
  notes: string | null;
  status: ScheduleStatus;
}
```

From trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts:
```typescript
export interface IScheduleDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  jobId: string;
  visitTypeId: string | null;
  assigneeId: string;
  date: DateTime;        // REMOVE - merge into startDateTime
  startTime: DateTime;   // REMOVE - merge into startDateTime
  durationMinutes: number;
  notes: string | null;
  status: ScheduleStatus;
}
```

From trade-flow-api/src/schedule/responses/schedule.response.ts:
```typescript
export interface IScheduleResponse {
  id: string;
  jobId: string;
  visitTypeId: string | null;
  assigneeId: string;
  date: string;          // REMOVE - merge into startDateTime
  startTime: string;     // REMOVE - merge into startDateTime
  durationMinutes: number;
  notes: string | null;
  status: string;
}
```

From trade-flow-api/src/schedule/repositories/schedule.repository.ts:
```typescript
// toDto: date: DateTime.fromISO(entity.date), startTime: DateTime.fromFormat(entity.startTime, "HH:mm")
// toEntity: date: dto.date.toISODate()!, startTime: dto.startTime.toFormat("HH:mm")
```

From trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts:
```typescript
// date: DateTime.fromISO(requestBody.date),
// startTime: DateTime.fromFormat(requestBody.startTime, "HH:mm"),
```

From trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts:
```typescript
// date: schedule.date.toISODate()!,
// startTime: schedule.startTime.toFormat("HH:mm"),
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Replace date+startTime with startDateTime in all production code and OpenAPI spec</name>
  <files>
    trade-flow-api/src/schedule/entities/schedule.entity.ts,
    trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts,
    trade-flow-api/src/schedule/requests/create-schedule.request.ts,
    trade-flow-api/src/schedule/responses/schedule.response.ts,
    trade-flow-api/src/schedule/repositories/schedule.repository.ts,
    trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts,
    trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts,
    trade-flow-api/openapi.yaml
  </files>
  <action>
Replace the separate `date` and `startTime` fields with a single `startDateTime` field across all schedule module files. Work through each file in this order:

1. **Entity** (`schedule.entity.ts`): Remove `date: string` and `startTime: string`. Add `startDateTime: string` (ISO8601 UTC string, e.g. `"2026-03-15T09:00:00.000Z"`).

2. **DTO** (`schedule.dto.ts`): Remove `date: DateTime` and `startTime: DateTime`. Add `startDateTime: DateTime` with JSDoc: `/** The start date and time of the schedule entry as a Luxon DateTime (UTC). */`.

3. **Request** (`create-schedule.request.ts`): Remove the `date` field (with its `@Matches` YYYY-MM-DD validator) and the `startTime` field (with its `@IsMilitaryTime` validator). Add a single `startDateTime: string` field with `@IsString()`, `@IsNotEmpty()`, and `@IsISO8601({ strict: true }, { message: "startDateTime must be a valid ISO8601 date-time string" })`. Import `IsISO8601` from `class-validator`. Remove unused imports (`IsMilitaryTime`, `Matches`).

4. **Response** (`schedule.response.ts`): Remove `date: string` and `startTime: string`. Add `startDateTime: string`.

5. **Repository** (`schedule.repository.ts`):
   - In `toDto`: Replace `date: DateTime.fromISO(entity.date)` and `startTime: DateTime.fromFormat(entity.startTime, "HH:mm")` with `startDateTime: DateTime.fromISO(entity.startDateTime, { zone: "utc" })`.
   - In `toEntity`: Replace `date: dto.date.toISODate()!` and `startTime: dto.startTime.toFormat("HH:mm")` with `startDateTime: dto.startDateTime.toUTC().toISO()!`.

6. **Request-to-DTO mapper** (`map-create-schedule-request-to-dto.utility.ts`): Replace `date: DateTime.fromISO(requestBody.date)` and `startTime: DateTime.fromFormat(requestBody.startTime, "HH:mm")` with `startDateTime: DateTime.fromISO(requestBody.startDateTime, { zone: "utc" })`. The `fromISO` call with `{ zone: "utc" }` ensures the resulting DateTime is in UTC regardless of whether the input string contained a timezone offset.

7. **DTO-to-Response mapper** (`map-schedule-to-response.utility.ts`): Replace `date: schedule.date.toISODate()!` and `startTime: schedule.startTime.toFormat("HH:mm")` with `startDateTime: schedule.startDateTime.toUTC().toISO()!`.

8. **OpenAPI spec** (`openapi.yaml`): In the `CreateScheduleRequest` schema, remove `date` and `startTime` properties and their required entries. Add `startDateTime` with `type: string`, `format: date-time`, `description: "Start date and time in ISO8601 format (e.g. 2026-03-15T09:00:00Z)"`. Update `required` to list `startDateTime` instead of `date` and `startTime`. In the `ScheduleResponse` schema, remove `date` and `startTime` properties and their required entries. Add `startDateTime` with `type: string`, `format: date-time`, `description: "Start date and time in ISO8601 UTC format"`. Update `required` to list `startDateTime` instead of `date` and `startTime`.

After all production code changes, run `npm run typecheck:strict` from `trade-flow-api/` to confirm zero type errors before moving to tests.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-api && npm run typecheck:strict 2>&1 | tail -5</automated>
  </verify>
  <done>All 8 production files and OpenAPI spec updated. No `date` or `startTime` fields remain in the schedule module (only `startDateTime`). TypeScript strict type check passes with zero errors.</done>
</task>

<task type="auto">
  <name>Task 2: Update all tests and mock generator for startDateTime</name>
  <files>
    trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts,
    trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts,
    trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts,
    trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts
  </files>
  <action>
Update all test infrastructure and test files to use `startDateTime` instead of separate `date`/`startTime`:

1. **Mock generator** (`schedule-mock-generator.ts`):
   - In `createScheduleDto`: Replace `date: DateTime.fromISO("2026-03-15")` and `startTime: DateTime.fromFormat("09:00", "HH:mm")` with `startDateTime: DateTime.fromISO("2026-03-15T09:00:00.000Z", { zone: "utc" })`.
   - In `createScheduleResponse`: Replace `date: "2026-03-15"` and `startTime: "09:00"` with `startDateTime: "2026-03-15T09:00:00.000Z"`.
   - In `createCreateScheduleRequest`: Replace `request.date = "2026-03-15"` and `request.startTime = "09:00"` with `request.startDateTime = "2026-03-15T09:00:00.000Z"`.

2. **Repository spec** (`schedule.repository.spec.ts`):
   - In the "should insert entity and return DTO" test: Update the entity assertion from `date: dto.date.toISODate()` and `startTime: dto.startTime.toFormat("HH:mm")` to `startDateTime: dto.startDateTime.toUTC().toISO()`. Update the result assertion from `result.date.toISODate()` / `result.startTime.toFormat("HH:mm")` to `expect(result.startDateTime.toISO()).toBe(dto.startDateTime.toISO())`.
   - In the "should return DTO when document exists" test: Update the inline entity literal from `date: "2026-03-15"` and `startTime: "09:00"` to `startDateTime: "2026-03-15T09:00:00.000Z"`. Update assertions from `result.date.toISODate()` / `result.startTime.toFormat("HH:mm")` to `expect(result.startDateTime).toBeInstanceOf(DateTime)` and `expect(result.startDateTime.toISO()).toBe("2026-03-15T09:00:00.000Z")`. Remove the separate `result.date` instanceof check and the `result.startTime` instanceof check; replace with a single `result.startDateTime` instanceof check.

3. **Request-to-DTO mapper spec** (`map-create-schedule-request-to-dto.utility.spec.ts`):
   - In "should map all provided fields correctly": Change the request override from `date: "2026-04-10"` and `startTime: "14:30"` to `startDateTime: "2026-04-10T14:30:00.000Z"`. Update assertions from `result.date.toISODate()` / `result.startTime.toFormat("HH:mm")` to `expect(result.startDateTime).toBeInstanceOf(DateTime)` and `expect(result.startDateTime.toISO()).toBe("2026-04-10T14:30:00.000Z")`.
   - Remove or merge the "should default assigneeId" test's date/startTime references if any are present (none expected there -- just verify it still passes).
   - All other tests in this file reference `result.durationMinutes`, `result.visitTypeId`, etc. that don't change -- leave them as-is.

4. **Response mapper spec** (`map-schedule-to-response.utility.spec.ts`):
   - In "should map all DTO fields to response": Update the expected object from `date: dto.date.toISODate()` and `startTime: dto.startTime.toFormat("HH:mm")` to `startDateTime: dto.startDateTime.toUTC().toISO()`.

After all test updates, run the full test suite from `trade-flow-api/`.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-api && npm run test -- --testPathPattern="schedule" 2>&1 | tail -20</automated>
  </verify>
  <done>All schedule module tests pass. No references to separate `date` or `startTime` fields remain in test files. Mock generator produces `startDateTime` for DTO, response, and request mocks.</done>
</task>

</tasks>

<verification>
1. `cd trade-flow-api && npm run typecheck:strict` -- zero errors
2. `cd trade-flow-api && npm run test -- --testPathPattern="schedule"` -- all tests pass
3. `cd trade-flow-api && npm run lint:check` -- no lint errors
4. Grep confirms no lingering separate date/startTime: `grep -rn "startTime\b" src/schedule/ --include="*.ts"` returns zero results (only `startDateTime` matches)
5. Grep confirms no lingering separate date field: `grep -rn "\bdate:" src/schedule/ --include="*.ts"` returns zero results
</verification>

<success_criteria>
- IScheduleEntity has `startDateTime: string` (no `date` or `startTime`)
- IScheduleDto has `startDateTime: DateTime` (no `date` or `startTime`)
- CreateScheduleRequest has `startDateTime: string` with `@IsISO8601` validation (no `date` or `startTime`)
- IScheduleResponse has `startDateTime: string` (no `date` or `startTime`)
- Repository converts between ISO string and Luxon DateTime correctly via toDto/toEntity
- OpenAPI spec reflects the new single-field contract
- All schedule tests pass
- TypeScript strict check passes
- Lint check passes
</success_criteria>

<output>
After completion, create `.planning/quick/2-merge-schedule-date-and-starttime-into-a/2-SUMMARY.md`
</output>
