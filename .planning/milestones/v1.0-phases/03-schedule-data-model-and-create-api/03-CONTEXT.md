# Phase 3: Schedule Data Model and Create API - Context

**Gathered:** 2026-03-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Backend data model and create endpoint for schedule entries on jobs. Schedules can be created via the API with smart defaults (auto-assignee, 1-hour duration). This phase covers SCHED-06 (assignee defaults to logged-in user) and SCHED-07 (duration defaults to 1 hour). No UI, no status transitions, no list/update/delete endpoints — those are later phases.

</domain>

<decisions>
## Implementation Decisions

### Date/time representation
- Separate `date` (string, `YYYY-MM-DD`) and `startTime` (string, `HH:mm` 24-hour format) fields — matches how tradespeople think ("Tuesday at 9am")
- Stored in UTC, frontend converts to local timezone for display
- Start time accepts any minute value (not snapped to increments)
- No date restrictions — past and future dates both allowed (tradespeople log visits after the fact)

### Duration format and granularity
- Stored as integer minutes (e.g., 60, 90, 120)
- 15-minute increment validation — rejects values not divisible by 15
- Minimum: 15 minutes, maximum: 1440 minutes (24 hours)
- Default: 60 minutes (1 hour) when not provided

### Assignee modeling
- Single `assigneeId` field referencing user ID
- Optional on create request — defaults to the authenticated user when omitted
- Validated against business membership (assignee must belong to the same business)
- Returned as ID only in the response (no expanded user data)

### Visit type on schedule entries
- `visitTypeId` is optional on create request — a schedule can exist without a visit type
- Validated against business ownership when provided (visit type must belong to the same business)
- No cascade on visit type deactivation — existing schedule entries keep their reference; deactivated types only blocked from new selection at the UI level (Phase 5)

### Notes field
- Included in the data model and create request from day one (optional string)
- Avoids schema migration when Phase 6 adds the notes UI (SCHED-05)

### Claude's Discretion
- MongoDB index strategy for the schedules collection
- Status field initial value and enum design (scaffolding for Phase 4)
- Request validation decorator choices (class-validator patterns)
- Response shape following existing IResponse<T> conventions
- Error message wording for validation failures

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `IBaseEntity` (`src/core/entities/base.entity`): Base entity with `_id`, `createdAt`, `updatedAt` — schedule entity extends this
- `AccessControllerFactory` / `AuthorizedCreatorFactory`: Existing authorization wrappers for repository operations
- `BusinessPolicy` pattern: Per-feature policy class checking business ownership — reuse for schedule authorization
- `createResponse()` / `createHttpError()`: Standard response and error utilities
- `AppLogger`: Pino-based logger with class context

### Established Patterns
- Controller → Service → Repository strict layering (never bypassed)
- Single-responsibility services: `ScheduleCreator` for create, future `ScheduleRetriever` etc.
- Entities stay in repository layer; DTOs are the layer contract
- Request classes with `class-validator` decorators for input validation
- Repositories handle ObjectId ↔ string conversion
- Feature modules follow: controllers/, services/, repositories/, entities/, data-transfer-objects/, responses/, requests/, policies/, enum/

### Integration Points
- Job module: Schedule references `jobId` (ObjectId) — must validate job exists and belongs to same business
- Visit-type module: Schedule optionally references `visitTypeId` (ObjectId) — must validate business ownership
- User/Auth module: `assigneeId` references user, defaults to `request.user` from JWT
- BusinessUser association: Already exists for validating business membership of assignee
- AppModule: New ScheduleModule must be registered in `src/app.module.ts`
- Path alias: Will need `@schedule/*` added to tsconfig paths

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase conventions.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-schedule-data-model-and-create-api*
*Context gathered: 2026-03-01*
