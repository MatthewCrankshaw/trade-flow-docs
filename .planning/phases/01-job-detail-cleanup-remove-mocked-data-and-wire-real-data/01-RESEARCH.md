# Phase 1: Job Detail Cleanup -- Remove mocked data and wire real data - Research

**Researched:** 2026-04-21
**Domain:** Full-stack data wiring (React RTK Query + NestJS module creation + MongoDB)
**Confidence:** HIGH

## Summary

This phase has two distinct workstreams: (1) removing mocked data from the job detail page and wiring real API data using existing RTK Query hooks, and (2) building a new `job-event` backend module with a dedicated `job_events` MongoDB collection to power a timeline/activity feed. The first workstream is straightforward -- the RTK Query hooks (`useGetCustomerQuery`, `useGetJobTypeQuery`, `useGetQuotesQuery`, `useGetSchedulesByJobQuery`) already exist and just need to be called from the job detail page instead of referencing mock constants. The second workstream follows the well-established NestJS module pattern (controller/service/repository/entity/DTO) that has been implemented dozens of times across this codebase.

The frontend work involves removing 7 mock data blocks, removing 4 UI sections (Invoices tab, Notes tab, access notes, Create Invoice button), wiring real data into remaining sections, updating the Job TypeScript type to include `jobTypeId` and `address`, and replacing the mock timeline with a real `jobEventApi` RTK Query slice. The backend work involves creating a new `job-event` module with a retriever endpoint and inline event-writing calls injected into existing services (quote-creator, schedule-updater, job-updater, etc.).

**Primary recommendation:** Structure the work as backend-first (create job-event module), then frontend cleanup (remove mocks + wire real data), then integration (inline event writes in existing services + timeline UI).

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Wire real API data for all sections that have existing backends: customer details (via `useGetCustomerQuery`), job type name (via `useGetJobTypeQuery`), quotes, schedules, and estimates.
- **D-02:** Remove all 7 mock data blocks: `MOCK_CUSTOMER`, `MOCK_JOB_TYPE`, `MOCK_ACCESS_NOTES`, `MOCK_COMMERCIAL`, `generateMockTimeline()`, `MOCK_INVOICES`, `MOCK_NOTES`.
- **D-03:** Surface `jobTypeId` and `address` in the frontend Job type -- both exist in the API response but aren't reflected in UI types. Wire job type name display and job address display.
- **D-04:** Create a new `job_events` MongoDB collection to store activity events for jobs. This is a dedicated events collection, not an embedded array on the job entity.
- **D-05:** Build a dedicated backend timeline endpoint that queries `job_events` for a given job and returns a chronological event list.
- **D-06:** Event types to support: job status changes, schedule events (created/status changes), quote events (created/sent/accepted/rejected), estimate events (created/sent/responded/converted/lost).
- **D-07:** Event writing is inline in existing services -- each service (quote-creator, schedule-updater, job-updater, etc.) writes a job event as part of its operation. No event listener/observer pattern.
- **D-08:** Forward-only event recording -- no backfill migration for existing jobs. New events are recorded from this point forward only.
- **D-09:** Remove the Invoices tab entirely from the job detail tabs.
- **D-10:** Remove the Notes tab entirely from the job detail tabs.
- **D-11:** Remove the access notes section (parking/pets/keys/gate code) entirely.
- **D-12:** Remove the "Create Invoice" placeholder action button.
- **D-13:** Keep the tab structure with Schedules and Quotes tabs.
- **D-14:** Replace the mock commercial summary with a quotes-only display showing total quoted amount aggregated from all non-deleted quotes for this job.
- **D-15:** Remove invoiced, paid, and outstanding fields from the commercial section.

### Claude's Discretion
- Component structure and file organization for the job events module (follow existing codebase patterns)
- Exact UI layout adjustments after removing sections (spacing, responsive behavior)
- Job event entity/DTO field design (event type enum values, metadata structure)
- Whether the timeline endpoint uses pagination or returns all events for a job

### Deferred Ideas (OUT OF SCOPE)
- Access notes / site info -- needs proper design as part of a site/property feature
- Invoices tab -- requires invoice module and API to be built first
- Notes tab -- requires notes/comments module and API to be built first
- Timeline backfill migration -- could retroactively create events from existing data timestamps
- Commercial summary: invoiced/paid/outstanding -- returns when invoice feature ships

</user_constraints>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Job event storage | Database / Storage | -- | New `job_events` collection in MongoDB |
| Job event writing | API / Backend | -- | Inline writes in existing NestJS services |
| Timeline retrieval endpoint | API / Backend | -- | New GET endpoint returning chronological events |
| Mock data removal | Browser / Client | -- | Remove hardcoded constants from React components |
| Real data wiring | Browser / Client | -- | Use existing RTK Query hooks to fetch API data |
| Job type display | Browser / Client | API / Backend | Frontend type update + existing API already returns data |
| Commercial summary (quotes total) | Browser / Client | -- | Aggregate from already-fetched quote data client-side |
| Tab/section removal | Browser / Client | -- | Remove React components/tabs for unbuilt features |

## Standard Stack

### Core (Already in Project)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.1.12 | Backend framework | Project standard [VERIFIED: CLAUDE.md] |
| React | 19.2.0 | Frontend framework | Project standard [VERIFIED: CLAUDE.md] |
| RTK Query | via @reduxjs/toolkit 2.11.2 | API data fetching | Project standard [VERIFIED: CLAUDE.md] |
| MongoDB native driver | 7.0.0 | Database access | Project standard [VERIFIED: CLAUDE.md] |
| class-validator | 0.14.1 | DTO validation | Project standard [VERIFIED: CLAUDE.md] |
| class-transformer | 0.5.1 | Object transformation | Project standard [VERIFIED: CLAUDE.md] |
| Luxon | 3.5.1 | Date/time handling | Project standard [VERIFIED: CLAUDE.md] |
| dinero.js | 1.9.1 (API) / 2.0.0-alpha.14 (UI) | Money handling | Project standard [VERIFIED: CLAUDE.md] |

### No New Dependencies Required

This phase uses only existing project dependencies. No new packages need to be installed.

## Architecture Patterns

### System Architecture Diagram

```
[Job Detail Page]
    |
    |--- useGetJobQuery(jobId) -----------> GET /v1/job/:id -------> jobs collection
    |--- useGetCustomerQuery(customerId) -> GET /v1/customer/:id --> customers collection
    |--- useGetJobTypeQuery(jobTypeId) ---> GET /v1/job-type/:id --> job_types collection
    |--- useGetQuotesQuery(jobId) --------> GET /v1/quote?jobId= --> quotes collection
    |--- useGetSchedulesByJobQuery(jobId) > GET /v1/schedule?jobId= > schedules collection
    |--- useGetJobEventsQuery(jobId) -----> GET /v1/job-event?jobId= > job_events collection [NEW]
    |
    v
[Render sections with real data]
    - Header: job name, status, customer name, job type name, address
    - Tabs: Schedules, Quotes (only)
    - Commercial: total quoted amount from quotes
    - Timeline: chronological events from job_events

--- Event Writing (Backend, inline in services) ---

[quote-creator.service] --writes--> job_events { type: "quote_created", jobId, ... }
[quote-transition.service] --writes--> job_events { type: "quote_sent/accepted/rejected", ... }
[schedule-creator.service] --writes--> job_events { type: "schedule_created", ... }
[schedule-updater.service] --writes--> job_events { type: "schedule_status_changed", ... }
[job-updater.service] --writes--> job_events { type: "job_status_changed", ... }
[estimate-creator.service] --writes--> job_events { type: "estimate_created", ... }
[estimate-transition.service] --writes--> job_events { type: "estimate_sent/responded", ... }
[estimate-to-quote-converter.service] --writes--> job_events { type: "estimate_converted", ... }
[estimate-lost-marker.service] --writes--> job_events { type: "estimate_lost", ... }
```

### Recommended Project Structure -- Backend (new files)

```
src/job-event/
  ├── job-event.module.ts              # Module declaration
  ├── controllers/
  │   └── job-event.controller.ts      # GET /v1/job-event?jobId=
  ├── services/
  │   ├── job-event-creator.service.ts # Write events (used by other services)
  │   └── job-event-retriever.service.ts # Read events for timeline
  ├── repositories/
  │   └── job-event.repository.ts      # MongoDB CRUD for job_events
  ├── entities/
  │   └── job-event.entity.ts          # MongoDB document shape
  ├── data-transfer-objects/
  │   └── job-event.dto.ts             # DTO interface
  ├── enums/
  │   └── job-event-type.enum.ts       # Event type enumeration
  ├── responses/
  │   └── job-event.response.ts        # API response shape
  ├── policies/
  │   └── job-event.policy.ts          # Authorization (via job's businessId)
  └── test/
      ├── services/
      ├── controllers/
      ├── repositories/
      └── mocks/
```

### Recommended Project Structure -- Frontend (new/modified files)

```
src/features/jobs/
  ├── api/
  │   ├── jobApi.ts                    # EXISTING -- already has job endpoints
  │   └── jobEventApi.ts               # NEW -- RTK Query for job events
  ├── components/
  │   ├── JobDetailPage.tsx            # MODIFIED -- remove mocks, wire real data
  │   ├── JobDetailTabs.tsx            # MODIFIED -- remove Invoices/Notes tabs
  │   ├── JobOverviewSection.tsx       # MODIFIED -- remove mock interfaces, wire real data
  │   ├── JobTimeline.tsx              # NEW or MODIFIED -- real timeline from API
  │   └── JobCommercialSummary.tsx     # NEW or MODIFIED -- quotes-only display
  └── hooks/
      └── generate-mock-timeline.ts    # DELETE -- replaced by real API
```

### Pattern 1: New NestJS Module (job-event)

**What:** Follow the exact same Controller -> Service -> Repository pattern used by every other module in the project.
**When to use:** Always -- this is the only pattern used in this codebase.

```typescript
// Source: existing codebase pattern from STRUCTURE.md and ARCHITECTURE.md
// job-event.module.ts
@Module({
  imports: [CoreModule],
  controllers: [JobEventController],
  providers: [JobEventCreator, JobEventRetriever, JobEventRepository],
  exports: [JobEventCreator], // Exported so other modules can write events
})
export class JobEventModule {}
```

### Pattern 2: Inline Event Writing in Existing Services

**What:** Each service that performs an action on a job-related entity calls `JobEventCreator.create()` inline after the primary operation succeeds. [VERIFIED: D-07 locked decision]
**When to use:** Every time a quote, schedule, estimate, or job status change occurs.

```typescript
// Source: follows existing pattern from business-creator.service.ts (inline side effects)
// In quote-creator.service.ts (after successful quote creation):
async create(authUser: IUserDto, quoteDto: IQuoteDto): Promise<IQuoteDto> {
  const created = await this.authorizedCreator.createFor(/* ... */);
  // Inline event write (fire-and-forget or awaited)
  await this.jobEventCreator.create({
    jobId: created.jobId,
    businessId: created.businessId,
    type: JobEventType.QUOTE_CREATED,
    metadata: { quoteId: created.id },
    occurredAt: DateTime.now().toISO(),
  });
  return created;
}
```

### Pattern 3: RTK Query Data Wiring on Job Detail Page

**What:** Replace mock constants with RTK Query hook calls that fetch real data.
**When to use:** For every data section on the job detail page.

```typescript
// Source: follows existing RTK Query patterns in customerApi.ts, quoteApi.ts
// In JobDetailPage.tsx:
const { data: job } = useGetJobQuery(jobId);
const { data: customer } = useGetCustomerQuery(job?.customerId, { skip: !job?.customerId });
const { data: jobType } = useGetJobTypeQuery(job?.jobTypeId, { skip: !job?.jobTypeId });
const { data: quotes } = useGetQuotesQuery({ jobId });
const { data: schedules } = useGetSchedulesByJobQuery(jobId);
const { data: events } = useGetJobEventsQuery(jobId);
```

### Pattern 4: Job Event Entity/DTO Design (Claude's Discretion)

**What:** Event entity stored in `job_events` collection with type-safe metadata. [ASSUMED: specific field design is at Claude's discretion per CONTEXT.md]

```typescript
// Recommended event type enum
enum JobEventType {
  JOB_STATUS_CHANGED = "job_status_changed",
  SCHEDULE_CREATED = "schedule_created",
  SCHEDULE_STATUS_CHANGED = "schedule_status_changed",
  QUOTE_CREATED = "quote_created",
  QUOTE_SENT = "quote_sent",
  QUOTE_ACCEPTED = "quote_accepted",
  QUOTE_REJECTED = "quote_rejected",
  ESTIMATE_CREATED = "estimate_created",
  ESTIMATE_SENT = "estimate_sent",
  ESTIMATE_RESPONDED = "estimate_responded",
  ESTIMATE_CONVERTED = "estimate_converted",
  ESTIMATE_LOST = "estimate_lost",
}

// Recommended entity shape
interface IJobEventEntity {
  _id: ObjectId;
  jobId: string;
  businessId: string;
  type: JobEventType;
  metadata: Record<string, unknown>; // Flexible per event type
  occurredAt: string; // ISO 8601 datetime
  createdAt: string;
  updatedAt: string;
}
```

### Pattern 5: Commercial Summary from Quote Aggregation

**What:** Calculate total quoted amount client-side from the quotes already fetched for the Quotes tab. No separate endpoint needed. [VERIFIED: D-14]

```typescript
// Aggregate total from fetched quotes, filtering out deleted ones
const totalQuotedAmount = quotes
  ?.filter((q) => q.status !== "deleted")
  .reduce((sum, q) => sum + q.totalAmount, 0) ?? 0;
```

### Anti-Patterns to Avoid

- **Embedding events in the job document:** D-04 explicitly mandates a separate `job_events` collection. Do not add an events array to the job entity.
- **Event listeners/observers:** D-07 explicitly mandates inline event writing. Do not use NestJS event emitters or observer patterns.
- **Backfill migration:** D-08 explicitly states forward-only. Do not create migration scripts for historical events.
- **Placeholder/coming-soon stubs:** User wants removed features to be completely gone, not disabled or hidden behind flags.
- **Shared event-writing utility across all services:** Instead, inject `JobEventCreator` directly into each service that needs it. This follows the project's dependency injection pattern.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| MongoDB queries | Raw MongoDB driver calls | Existing MongoDbFetcher/MongoDbWriter services | Project pattern; handles connection, logging, error handling |
| API data fetching | fetch() or axios calls | RTK Query injectEndpoints | Project pattern; handles caching, auth token injection, loading states |
| Date/time formatting | `new Date()` or string manipulation | Luxon DateTime | Project standard; handles timezone-safe ISO strings |
| Money display | Manual division/formatting | dinero.js + useCurrency hook | Project standard; handles minor units, currency formatting |
| Request validation | Manual field checks | class-validator decorators | Project standard; automatic 400 responses |
| Authorization | Manual user/business checks | Policy classes + AccessControllerFactory | Project standard; consistent authorization pattern |

## Common Pitfalls

### Pitfall 1: RTK Query Skip Conditions
**What goes wrong:** Calling `useGetCustomerQuery(undefined)` triggers an API call with `undefined` as the parameter, causing a 400 or 404 error.
**Why it happens:** The job data loads asynchronously, so `job.customerId` is undefined on first render.
**How to avoid:** Always use the `skip` option: `useGetCustomerQuery(job?.customerId, { skip: !job?.customerId })`.
**Warning signs:** Console errors showing failed API calls on page load.

### Pitfall 2: Circular Module Dependencies
**What goes wrong:** Importing `JobEventModule` into `QuoteModule` and `QuoteModule` into `JobEventModule` creates a circular dependency that NestJS cannot resolve.
**Why it happens:** The job-event module needs to be used by quote/schedule/job/estimate modules, but if it also depends on them, circularity occurs.
**How to avoid:** The `JobEventModule` should export `JobEventCreator`. Other modules import `JobEventModule` and inject `JobEventCreator`. The `JobEventModule` itself should NOT import other feature modules. It only depends on `CoreModule`.
**Warning signs:** NestJS startup errors about unresolved dependencies.

### Pitfall 3: Frontend Job Type Missing Fields
**What goes wrong:** The frontend `Job` type does not include `jobTypeId` or `address`, so TypeScript will error when trying to access these fields even though the API returns them.
**Why it happens:** The UI type definition was created before these fields were surfaced (D-03).
**How to avoid:** Update the frontend Job type/interface in the types file AND in the RTK Query response transformation to include these fields.
**Warning signs:** TypeScript compilation errors when accessing `job.jobTypeId` or `job.address`.

### Pitfall 4: Cache Invalidation for Events
**What goes wrong:** After creating a quote or schedule, the timeline on the job detail page does not update to show the new event.
**Why it happens:** RTK Query caches the job events query, and creating a quote does not automatically invalidate the events cache.
**How to avoid:** Add `"JobEvent"` as a tag type in the base API slice. In quote/schedule/estimate mutations, add `invalidatesTags: ["JobEvent"]` alongside existing invalidation tags.
**Warning signs:** Timeline shows stale data after performing actions.

### Pitfall 5: Money Aggregation with Minor Units
**What goes wrong:** Summing quote amounts incorrectly because values are in minor units (cents) but displayed as major units (dollars).
**Why it happens:** dinero.js stores amounts in minor units. Raw summation of minor units is correct, but displaying without conversion shows wrong values.
**How to avoid:** Sum the raw amounts (which are already in minor units), then use the existing `useCurrency` hook or dinero.js formatting to display the total.
**Warning signs:** Commercial summary shows a number 100x too large or too small.

### Pitfall 6: Estimate Module Existence
**What goes wrong:** Planner assumes estimate module does not exist and plans to create it.
**Why it happens:** The codebase reference files (STRUCTURE.md, INTEGRATIONS.md) are from February 2026 and predate the v1.8 Estimates milestone (shipped April 2026).
**How to avoid:** The estimate module was built in Phases 41-50 (v1.8). It exists with full CRUD. Event writes need to be added to existing estimate services, not new ones.
**Warning signs:** Planning tasks to create estimate endpoints that already exist.

## Code Examples

### RTK Query Endpoint for Job Events

```typescript
// Source: follows existing pattern in quoteApi.ts, scheduleApi.ts [VERIFIED: ARCHITECTURE.md]
// src/features/jobs/api/jobEventApi.ts
export const jobEventApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getJobEvents: builder.query({
      query: (jobId: string) => `/v1/job-event?jobId=${jobId}`,
      transformResponse: (response: IResponse<JobEventResponse>) => response.data,
      providesTags: (result) =>
        result
          ? [...result.map((e) => ({ type: "JobEvent" as const, id: e.id })), "JobEvent"]
          : ["JobEvent"],
    }),
  }),
});

export const { useGetJobEventsQuery } = jobEventApi;
```

### Job Event Controller

```typescript
// Source: follows pattern from job.controller.ts [VERIFIED: ARCHITECTURE.md]
@Controller("v1")
@UseGuards(JwtAuthGuard)
export class JobEventController {
  constructor(
    private readonly jobEventRetriever: JobEventRetriever,
  ) {}

  @Get("job-event")
  async getByJobId(@Req() request: Request, @Query("jobId") jobId: string) {
    try {
      const authUser = request.user as IUserDto;
      const events = await this.jobEventRetriever.getByJobId(authUser, jobId);
      const responses = events.map(mapToJobEventResponse);
      return createResponse(responses);
    } catch (error) {
      throw createHttpError(error);
    }
  }
}
```

### Removing Mock Data (Frontend)

```typescript
// BEFORE (current state with mocks):
const MOCK_CUSTOMER = { name: "John Smith", email: "john@example.com", ... };
// ... used directly in JSX

// AFTER (wired to real data):
const { data: job } = useGetJobQuery(jobId);
const { data: customer } = useGetCustomerQuery(job?.customerId, { skip: !job?.customerId });
// ... customer?.name used in JSX with loading/empty states
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mock data constants in components | RTK Query hooks with real API | This phase | Job detail page shows real, live data |
| `generateMockTimeline()` function | `job_events` collection + API endpoint | This phase | Timeline reflects actual activity history |
| Mock commercial summary | Quote aggregation from real data | This phase | Commercial section shows real financials |
| Invoices/Notes tabs with mock data | Tabs removed entirely | This phase | Clean UI with only functional features |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Job event entity uses `metadata: Record<string, unknown>` for flexible per-type data | Pattern 4 | May need typed metadata per event type -- but Record is extensible |
| A2 | Timeline endpoint returns all events (no pagination) for initial implementation | Claude's Discretion | Could be slow for jobs with many events -- but unlikely for sole traders in near term |
| A3 | Commercial summary aggregates quotes client-side rather than via a dedicated API endpoint | Pattern 5 | Could be wrong if quote data is not fully loaded -- but quotes are already fetched for the tab |
| A4 | Schedule module exists with `useGetSchedulesByJobQuery` hook | Code Context | CONTEXT.md references it as "Already used on job detail page" -- but codebase docs are from Feb |
| A5 | Estimate module exists with full CRUD from v1.8 | Pitfall 6 | If module structure differs from expected, event-writing injection points may differ |

## Open Questions (RESOLVED)

1. **Estimate module's exact service file names** (RESOLVED 2026-04-21)
   - **Answer:** Confirmed from v1.8 milestone artifacts (Phases 41-50). The estimate module has these services:
     - `estimate-creator.service.ts` (class: `EstimateCreator`) -- creates estimates
     - `estimate-transition.service.ts` (class: `EstimateTransitionService`) -- handles status transitions (SENT, RESPONDED via ALLOWED_TRANSITIONS map)
     - `estimate-to-quote-converter.service.ts` (class: `EstimateToQuoteConverter`) -- converts estimate to quote (transitions to Converted)
     - `estimate-lost-marker.service.ts` (class: `EstimateLostMarker`) -- marks estimate as lost (transitions to Lost)
   - No separate "estimate-sender" service; the email-sending service (`estimate-email-sender`) calls `EstimateTransitionService` for the SENT transition.
   - Module file: `estimate.module.ts` (class: `EstimateModule`)

2. **Quote filtering by jobId** (RESOLVED via PATTERNS.md)
   - **Answer:** The existing `useGetQuotesQuery` fetches by `businessId`, then filtering by jobId is done client-side. PATTERNS.md confirms this pattern at `JobDetailPage.tsx` line 93-103 where `allQuotes` is fetched by businessId and filtered in the component.

3. **RTK Query tag types registration** (RESOLVED via PATTERNS.md)
   - **Answer:** Tag types are registered in `trade-flow-ui/src/services/api.ts` in the `createApi` call's `tagTypes` array. PATTERNS.md confirms the exact location at lines 46-62 of that file.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 |
| Framework (UI) | Vitest 4.1.3 |
| API config | jest.config in package.json |
| UI config | vitest.config.ts (extends vite.config) |
| API quick run | `npm run test -- --testPathPattern=job-event` |
| API full suite | `npm run test` |
| UI quick run | `npm run test -- --testPathPattern=job` |
| UI full suite | `npm run test` |

### Phase Requirements to Test Map

| Req | Behavior | Test Type | Automated Command | File Exists? |
|-----|----------|-----------|-------------------|-------------|
| D-04 | job_events collection CRUD | unit | `npm run test -- job-event-creator` | Wave 0 |
| D-05 | Timeline endpoint returns events | unit | `npm run test -- job-event.controller` | Wave 0 |
| D-07 | Inline event writes | unit | `npm run test -- quote-creator` (updated) | Existing (needs update) |
| D-14 | Commercial summary aggregation | unit (UI) | `npm run test -- JobCommercialSummary` | Wave 0 |

### Wave 0 Gaps
- [ ] `src/job-event/test/services/job-event-creator.service.spec.ts` -- covers D-04
- [ ] `src/job-event/test/services/job-event-retriever.service.spec.ts` -- covers D-05
- [ ] `src/job-event/test/controllers/job-event.controller.spec.ts` -- covers D-05
- [ ] `src/job-event/test/repositories/job-event.repository.spec.ts` -- covers D-04
- [ ] `src/job-event/test/mocks/job-event-mock-generator.ts` -- shared mocks
- [ ] Updated mocks for existing service specs that now call JobEventCreator

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | Existing JwtAuthGuard (Firebase JWT) -- no changes needed |
| V3 Session Management | no | -- |
| V4 Access Control | yes | JobEventPolicy checks businessId ownership via job lookup |
| V5 Input Validation | yes | class-validator on query params (jobId validation) |
| V6 Cryptography | no | -- |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Cross-business event access | Information Disclosure | JobEventPolicy verifies requesting user owns the job's business |
| Event injection (fake events) | Tampering | Events only written server-side by services; no public write endpoint |
| Timeline data leakage | Information Disclosure | Events scoped to jobId + businessId; policy enforced at retrieval |

## Sources

### Primary (HIGH confidence)
- `.planning/codebase/ARCHITECTURE.md` -- Layer patterns, data flow, module structure
- `.planning/codebase/STRUCTURE.md` -- File naming, directory organization, "where to add new code"
- `.planning/codebase/INTEGRATIONS.md` -- MongoDB collections, API patterns
- `CLAUDE.md` -- Technology stack versions, conventions, testing setup
- `01-CONTEXT.md` -- All locked decisions (D-01 through D-15)

### Secondary (MEDIUM confidence)
- `.planning/ROADMAP.md` -- Confirmed estimate module exists (v1.8 shipped)
- `.planning/STATE.md` -- Project history and current position
- v1.8 milestone artifacts -- Confirmed estimate service file names (Phases 41-50)

### Tertiary (LOW confidence)
- None -- all claims derived from project documentation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in project, no new dependencies
- Architecture: HIGH -- follows exact same module pattern used 15+ times in codebase
- Pitfalls: HIGH -- based on known RTK Query patterns and NestJS DI behavior
- Event system design: MEDIUM -- entity design is discretionary, but follows established patterns
- Estimate service names: HIGH -- confirmed from v1.8 milestone artifacts

**Research date:** 2026-04-21
**Valid until:** 2026-05-21 (stable -- no external dependency changes)
