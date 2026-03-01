# Phase 4: Schedule Status and CRUD API - Context

**Gathered:** 2026-03-01
**Status:** Ready for planning

<domain>
## Phase Boundary

The API supports full schedule lifecycle including status transitions with validation, plus list/update/cancel/notes endpoints. This phase is backend-only (NestJS API). No frontend changes.

</domain>

<decisions>
## Implementation Decisions

### Status transitions
- Allowed transitions per roadmap: scheduled->confirmed, confirmed->completed, scheduled->canceled, scheduled->no-show, confirmed->canceled
- Terminal states (completed, canceled, no-show) are final -- no reversals. If a mistake is made, create a new schedule
- Dedicated transition endpoint: POST /v1/business/:businessId/job/:jobId/schedule/:scheduleId/transition with { status: 'confirmed' }
- Invalid transition errors include current status and list of valid next statuses for clarity
- No extra data required for any transition -- notes can be added separately via update endpoint

### Cancel behavior
- Cancel is a status transition (scheduled->canceled or confirmed->canceled), not a delete
- No hard delete endpoint -- SchedulePolicy.canDelete already returns false, keep it that way
- No cancellation reason field for MVP -- keep it simple, can be added later
- Canceled schedules are preserved in the database with all original data intact
- Cancel metadata: status change only, existing updatedAt timestamp captures when

### List & filter design
- List by job: GET /v1/business/:businessId/job/:jobId/schedule (required by success criteria)
- Also support business-wide listing: GET /v1/business/:businessId/schedule (needed for Phase 5/6 calendar and list views)
- Filters: status filter (?status=scheduled,confirmed) and date range (?from=2026-03-01&to=2026-03-31)
- Pagination using existing DtoCollection pattern (limit/offset) for consistency
- Default sort: chronological, oldest first (upcoming schedules at top)

### Update constraints
- Updates only allowed on scheduled and confirmed statuses -- completed, canceled, and no-show are locked
- Updatable fields: startDateTime, durationMinutes, assigneeId, visitTypeId, notes (all optional). Job association is permanent
- Time/duration changes on a confirmed schedule auto-reset status to scheduled (re-confirmation needed)
- Notes field uses replace (overwrite) pattern, matching standard PATCH update behavior

### Claude's Discretion
- Exact error code assignments for new error scenarios (transition errors, update-on-locked-status errors)
- Whether to create a separate ScheduleTransitionService or handle transitions in ScheduleUpdaterService
- Index strategy for the new list/filter queries
- UpdateScheduleRequest validation details

</decisions>

<specifics>
## Specific Ideas

- Status state machine should be easy to extend when new statuses are added in future phases
- The transition endpoint pattern separates "change what the schedule looks like" (update) from "change where the schedule is in its lifecycle" (transition)
- Cross-module validation pattern from ScheduleCreatorService should be reused for updates (validate visitTypeId if changed)

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- SchedulePolicy: already implements canCreate, canRead, canUpdate (business membership check), canDelete (returns false)
- ScheduleRepository: has create and findByIdOrFail -- needs update and list methods added
- ScheduleCreatorService: cross-module validation pattern (job + visit type) to reuse in updater
- AccessControllerFactory / AuthorizedCreatorFactory: existing authorization wrappers for services
- DtoCollection: pagination wrapper used by customer/job list endpoints
- ErrorCodes enum: add new schedule-specific error codes for transitions
- mapScheduleToResponse utility: already maps DTO to response format

### Established Patterns
- Updater services: CustomerUpdater, JobUpdater pattern -- check authorization, validate permitted updates, delegate to repository
- Repository update: findOneAndUpdate with explicit $set fields + updateAuditFields()
- Controller update: retrieve existing, merge with request body, call updater
- UpdateRequest classes: all fields @IsOptional(), only updatable fields included
- List endpoints: use DtoCollection with pagination, return createResponse(responses, pagination)

### Integration Points
- ScheduleModule: needs to register new services (ScheduleRetrieverService, ScheduleUpdaterService)
- ScheduleController: add GET (list), GET (by id), PATCH (update), POST (transition) endpoints
- OpenAPI spec: add new schemas and paths for all new endpoints
- Error codes: add transition-specific error codes to ErrorCodes enum and errors-map

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 04-schedule-status-and-crud-api*
*Context gathered: 2026-03-01*
