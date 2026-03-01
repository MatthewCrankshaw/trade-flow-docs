---
phase: quick-1
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts
  - trade-flow-api/src/schedule/repositories/schedule.repository.ts
  - trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts
  - trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts
  - trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts
  - trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts
  - trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts
  - trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts
  - trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts
  - trade-flow-api/src/quote/repositories/quote.repository.ts
  - trade-flow-api/src/migration/data-transfer-objects/migration.dto.ts
  - trade-flow-api/src/migration/repositories/migration.repository.ts
autonomous: true
requirements: [LUXON-DTO-01]

must_haves:
  truths:
    - "All date/time fields in DTOs use Luxon DateTime instead of JS Date or raw strings"
    - "Entity layer remains unchanged (strings/Date objects for MongoDB storage)"
    - "Repository toDto methods convert entity dates to Luxon DateTime"
    - "Repository toEntity methods convert Luxon DateTime back to storage format"
    - "Response mappers serialize Luxon DateTime to string format for API consumers"
    - "All existing tests pass with updated types"
  artifacts:
    - path: "trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts"
      provides: "Schedule DTO with Luxon DateTime for date field"
      contains: "DateTime"
    - path: "trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts"
      provides: "Quote DTO with Luxon DateTime for date fields"
      contains: "DateTime"
    - path: "trade-flow-api/src/migration/data-transfer-objects/migration.dto.ts"
      provides: "Migration DTO with Luxon DateTime for executedAt"
      contains: "DateTime"
  key_links:
    - from: "trade-flow-api/src/schedule/repositories/schedule.repository.ts"
      to: "trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts"
      via: "toDto converts entity string date to DateTime, toEntity converts DateTime back to string"
      pattern: "DateTime\\.fromISO|DateTime\\.fromFormat|\\.toISODate\\(\\)|\\.toFormat\\("
    - from: "trade-flow-api/src/quote/repositories/quote.repository.ts"
      to: "trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts"
      via: "toDto converts entity JS Date to Luxon DateTime"
      pattern: "DateTime\\.fromJSDate"
---

<objective>
Enforce Luxon DateTime usage across all DTOs that contain date, time, or timestamp fields.

Purpose: Establish a consistent pattern where entities use raw storage types (strings, JS Date) but DTOs -- the contract between all layers -- use Luxon DateTime. This ensures timezone-safe date handling throughout the business logic and service layers, following the pattern already established in IOnboardingProgressDto.

Output: Updated DTO interfaces, repository mappers, request-to-DTO mappers, response mappers, mock generators, and all associated tests.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-api/CLAUDE.md
@trade-flow-api/.github/copilot-instructions.md

<interfaces>
<!-- Established Luxon pattern to follow (from onboarding-progress module): -->

From trade-flow-api/src/user/data-transfer-objects/onboarding-progress.dto.ts:
```typescript
import { DateTime } from "luxon";
export interface IOnboardingProgressDto extends IBaseResourceDto {
  completedAt: DateTime | null;
}
```

From trade-flow-api/src/user/repositories/onboarding-progress.repository.ts (toDto):
```typescript
completedAt: entity.completedAt ? DateTime.fromJSDate(entity.completedAt) : null,
```

<!-- Files to modify - current state: -->

From trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts:
```typescript
date: string;           // YYYY-MM-DD format - convert to DateTime
startTime: string;      // HH:mm format - convert to DateTime
durationMinutes: number; // Keep as number (not a date/time type)
```

From trade-flow-api/src/schedule/entities/schedule.entity.ts:
```typescript
date: string;           // Storage format - DO NOT CHANGE
startTime: string;      // Storage format - DO NOT CHANGE
durationMinutes: number; // Storage format - DO NOT CHANGE
```

From trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts:
```typescript
validUntil?: Date;     // JS Date - convert to DateTime
sentAt?: Date;         // JS Date - convert to DateTime
acceptedAt?: Date;     // JS Date - convert to DateTime
rejectedAt?: Date;     // JS Date - convert to DateTime
```

From trade-flow-api/src/migration/data-transfer-objects/migration.dto.ts:
```typescript
executedAt: Date;      // JS Date - convert to DateTime
```

From trade-flow-api/src/schedule/responses/schedule.response.ts:
```typescript
date: string;       // API response stays as string
startTime: string;  // API response stays as string
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Update schedule DTO and all cascade files (repository, mappers, mocks, tests)</name>
  <files>
    trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts
    trade-flow-api/src/schedule/repositories/schedule.repository.ts
    trade-flow-api/src/schedule/controllers/mappers/map-create-schedule-request-to-dto.utility.ts
    trade-flow-api/src/schedule/controllers/mappers/map-schedule-to-response.utility.ts
    trade-flow-api/src/schedule/test/mocks/schedule-mock-generator.ts
    trade-flow-api/src/schedule/test/controllers/mappers/map-create-schedule-request-to-dto.utility.spec.ts
    trade-flow-api/src/schedule/test/controllers/mappers/map-schedule-to-response.utility.spec.ts
    trade-flow-api/src/schedule/test/repositories/schedule.repository.spec.ts
  </files>
  <action>
    **1. Update schedule.dto.ts:**
    - Import `{ DateTime } from "luxon"`
    - Change `date: string` to `date: DateTime` (represents the schedule date; stores full DateTime but only the date portion is meaningful)
    - Change `startTime: string` to `startTime: DateTime` (represents the start time; stores full DateTime but only the time portion is meaningful)
    - Keep `durationMinutes: number` unchanged (it is a numeric count, not a date/time type)
    - Update JSDoc comments: `date` = "The date of the schedule entry as a Luxon DateTime", `startTime` = "The start time of the schedule entry as a Luxon DateTime"

    **2. Update schedule.repository.ts toDto method:**
    - Import `{ DateTime } from "luxon"`
    - Convert `date`: `DateTime.fromISO(entity.date)` (entity.date is "YYYY-MM-DD" string, DateTime.fromISO handles this)
    - Convert `startTime`: `DateTime.fromFormat(entity.startTime, "HH:mm")` (entity.startTime is "HH:mm" string)
    - All other fields remain the same

    **3. Update schedule.repository.ts toEntity method:**
    - Convert `date` back: `dto.date.toISODate()!` (DateTime to "YYYY-MM-DD" string; non-null assertion is safe because we construct valid DateTimes)
    - Convert `startTime` back: `dto.startTime.toFormat("HH:mm")` (DateTime to "HH:mm" string)

    **4. Update map-create-schedule-request-to-dto.utility.ts:**
    - Import `{ DateTime } from "luxon"`
    - Convert `date`: `DateTime.fromISO(requestBody.date)` (request body has validated "YYYY-MM-DD" string)
    - Convert `startTime`: `DateTime.fromFormat(requestBody.startTime, "HH:mm")` (request body has validated "HH:mm" string)

    **5. Update map-schedule-to-response.utility.ts:**
    - Serialize `date`: `schedule.date.toISODate()!` (DateTime back to "YYYY-MM-DD" string for API response)
    - Serialize `startTime`: `schedule.startTime.toFormat("HH:mm")` (DateTime back to "HH:mm" string for API response)

    **6. Update schedule-mock-generator.ts:**
    - Import `{ DateTime } from "luxon"`
    - Change `date` default: `DateTime.fromISO("2026-03-15")` instead of `"2026-03-15"`
    - Change `startTime` default: `DateTime.fromFormat("09:00", "HH:mm")` instead of `"09:00"`
    - Schedule response mock stays with strings (responses are serialized)

    **7. Update all schedule test files:**
    - In map-create-schedule-request-to-dto.utility.spec.ts: Change assertions from `expect(result.date).toBe("2026-04-10")` to `expect(result.date.toISODate()).toBe("2026-04-10")` and from `expect(result.startTime).toBe("14:30")` to `expect(result.startTime.toFormat("HH:mm")).toBe("14:30")`
    - In map-schedule-to-response.utility.spec.ts: The response fields are already strings from the mapper, so test assertions comparing to `dto.date` need updating to compare to `dto.date.toISODate()` etc. Or better: since mapScheduleToResponse now serializes, tests should verify the output strings match expected values.
    - In schedule.repository.spec.ts: Update the expected DTO output assertions. Where tests check `date: dto.date` on the toDto result, the result will now be a DateTime object. Update to check `date: expect.any(Object)` or use `result.date.toISODate()` assertions. For the toEntity direction (create test checking insertOne args), entity.date should still be a string -- verify this is the case. For findByIdOrFail test, the expected result object should use DateTime comparisons.
  </action>
  <verify>
    <automated>cd trade-flow-api && npx jest --testPathPattern="schedule" --no-coverage 2>&1 | tail -20</automated>
  </verify>
  <done>
    - IScheduleDto.date is DateTime type, IScheduleDto.startTime is DateTime type
    - Schedule entity unchanged (still uses strings)
    - Repository converts between DateTime and string in both directions
    - Request mapper creates DateTime from validated string input
    - Response mapper serializes DateTime back to string for API output
    - All 34 schedule tests pass
  </done>
</task>

<task type="auto">
  <name>Task 2: Update quote and migration DTOs and their repository mappers</name>
  <files>
    trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts
    trade-flow-api/src/quote/repositories/quote.repository.ts
    trade-flow-api/src/migration/data-transfer-objects/migration.dto.ts
    trade-flow-api/src/migration/repositories/migration.repository.ts
  </files>
  <action>
    **1. Update quote.dto.ts:**
    - Import `{ DateTime } from "luxon"`
    - Change `validUntil?: Date` to `validUntil?: DateTime`
    - Change `sentAt?: Date` to `sentAt?: DateTime`
    - Change `acceptedAt?: Date` to `acceptedAt?: DateTime`
    - Change `rejectedAt?: Date` to `rejectedAt?: DateTime`
    - Update JSDoc comments to mention "Luxon DateTime"

    **2. Update quote.repository.ts toDto method:**
    - Import `{ DateTime } from "luxon"`
    - Convert optional Date fields using the onboarding pattern: `validUntil: entity.validUntil ? DateTime.fromJSDate(entity.validUntil) : undefined`
    - Same pattern for sentAt, acceptedAt, rejectedAt (use `undefined` not `null` since these are optional fields with `?:`)

    **3. Update quote.repository.ts toEntity method:**
    - Convert DateTime back to JS Date: `validUntil: dto.validUntil?.toJSDate()` (Luxon DateTime.toJSDate() returns a native Date)
    - Same pattern for sentAt, acceptedAt, rejectedAt

    **4. Update migration.dto.ts:**
    - Import `{ DateTime } from "luxon"`
    - Change `executedAt: Date` to `executedAt: DateTime`
    - Update JSDoc comment

    **5. Update migration.repository.ts mapToDto method:**
    - Import `{ DateTime } from "luxon"`
    - Convert: `executedAt: DateTime.fromJSDate(entity.executedAt)`

    Note: The migration module is internal infrastructure. The migration entity stores `executedAt` as JS Date in MongoDB. The repository mapToDto should convert it. There is no toEntity equivalent that needs changing (migrations are created with raw Date objects in the migration runner, not through the DTO path).

    **Important:** Do NOT modify entity interfaces. Entities remain with JS Date/string types as they represent the MongoDB storage layer.
  </action>
  <verify>
    <automated>cd trade-flow-api && npx jest --testPathPattern="(quote|migration)" --no-coverage 2>&1 | tail -20</automated>
  </verify>
  <done>
    - IQuoteDto uses DateTime for validUntil, sentAt, acceptedAt, rejectedAt
    - IMigrationDto uses DateTime for executedAt
    - Quote repository converts between DateTime and JS Date in both directions
    - Migration repository converts JS Date to DateTime in mapToDto
    - Quote and migration entities unchanged
    - All quote and migration tests pass
  </done>
</task>

<task type="auto">
  <name>Task 3: Run full validation suite and verify no regressions</name>
  <files>
    (no new files -- validation only)
  </files>
  <action>
    Run the full project validation to ensure no TypeScript errors or lint issues were introduced:

    1. Run `npm run typecheck:strict` -- must pass with zero errors. This will catch any file that still passes a string/Date where DateTime is now expected, or vice versa.
    2. Run `npm run lint:check` -- must pass with zero lint errors.
    3. Run `npm run test` -- full test suite must pass (all modules, not just schedule/quote/migration).
    4. If any failures, fix them. Common issues to watch for:
       - Quote response mappers may need DateTime-to-string serialization (check if quote has response mappers similar to schedule)
       - Any service that creates a DTO literal with raw string dates needs to use DateTime.fromISO()
       - Test mocks that create DTOs inline need DateTime values instead of strings/Dates
  </action>
  <verify>
    <automated>cd trade-flow-api && npm run validate && npm run test 2>&1 | tail -30</automated>
  </verify>
  <done>
    - `npm run typecheck:strict` passes with zero errors
    - `npm run lint:check` passes with zero errors
    - `npm run test` passes with all tests green
    - No DTO in the codebase uses JS Date for date/time fields (only Luxon DateTime)
    - Entities remain unchanged with raw storage types
  </done>
</task>

</tasks>

<verification>
1. All DTOs with date/time fields use Luxon DateTime (grep for `: Date` in dto files should return zero results except in entity files)
2. All entities remain unchanged (string/Date types for MongoDB)
3. Repository mappers handle bidirectional conversion (DateTime <-> storage type)
4. Response mappers serialize DateTime to API-friendly strings
5. Full test suite passes: `cd trade-flow-api && npm run test`
6. TypeScript strict check passes: `cd trade-flow-api && npm run typecheck:strict`
</verification>

<success_criteria>
- Zero instances of `Date` type (JS native) in any DTO interface (only `DateTime` from Luxon)
- Zero instances of raw `string` for date/time fields in any DTO interface (schedule date/startTime use DateTime)
- `durationMinutes` remains `number` (it is a count, not a date/time value)
- All 34+ existing tests pass without modification to test behavior (only type/assertion updates)
- `npm run validate` passes clean
- Consistent pattern across all modules matching the IOnboardingProgressDto precedent
</success_criteria>

<output>
After completion, create `.planning/quick/1-enforce-luxon-datetime-usage-in-dtos-for/1-SUMMARY.md`
</output>
