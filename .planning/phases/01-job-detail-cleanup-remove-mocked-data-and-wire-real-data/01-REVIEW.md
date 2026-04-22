---
phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
reviewed: 2026-04-22T00:00:00Z
depth: standard
files_reviewed: 31
files_reviewed_list:
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/job-event/job-event.module.ts
  - trade-flow-api/src/job-event/controllers/job-event.controller.ts
  - trade-flow-api/src/job-event/services/job-event-creator.service.ts
  - trade-flow-api/src/job-event/services/job-event-retriever.service.ts
  - trade-flow-api/src/job-event/repositories/job-event.repository.ts
  - trade-flow-api/src/job-event/entities/job-event.entity.ts
  - trade-flow-api/src/job-event/data-transfer-objects/job-event.dto.ts
  - trade-flow-api/src/job-event/enums/job-event-type.enum.ts
  - trade-flow-api/src/job-event/responses/job-event.response.ts
  - trade-flow-api/src/job-event/policies/job-event.policy.ts
  - trade-flow-api/src/estimate/services/estimate-creator.service.ts
  - trade-flow-api/src/estimate/services/estimate-lost-marker.service.ts
  - trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts
  - trade-flow-api/src/estimate/services/estimate-transition.service.ts
  - trade-flow-api/src/job/services/job-updater.service.ts
  - trade-flow-api/src/quote/services/quote-creator.service.ts
  - trade-flow-api/src/quote/services/quote-transition.service.ts
  - trade-flow-api/src/schedule/services/schedule-creator.service.ts
  - trade-flow-api/src/schedule/services/schedule-updater.service.ts
  - trade-flow-api/src/schedule/services/schedule-transition.service.ts
  - trade-flow-ui/src/features/jobs/api/jobEventApi.ts
  - trade-flow-ui/src/features/jobs/components/JobTimeline.tsx
  - trade-flow-ui/src/features/jobs/components/JobOverviewSection.tsx
  - trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx
  - trade-flow-ui/src/features/jobs/components/JobActionStrip.tsx
  - trade-flow-ui/src/features/quotes/api/quoteApi.ts
  - trade-flow-ui/src/features/schedules/api/scheduleApi.ts
  - trade-flow-ui/src/features/estimates/api/estimateApi.ts
  - trade-flow-ui/src/pages/JobDetailPage.tsx
  - trade-flow-ui/src/services/api.ts
  - trade-flow-ui/src/types/api.types.ts
findings:
  critical: 0
  warning: 5
  info: 4
  total: 9
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-04-22T00:00:00Z
**Depth:** standard
**Files Reviewed:** 31
**Status:** issues_found

## Summary

This phase wires real data into the Job Detail page — replacing mocked content with live API calls — and introduces the `job-event` module end-to-end. The overall implementation is solid: the new module follows project conventions correctly, the event emission points across estimate, quote, schedule, and job services are consistent, and the RTK Query cache invalidation strategy is coherent.

Five warnings and four info items were found. No critical security or data-loss issues exist. The most impactful issues are: a silent authorization gap in the event retriever when a job has no events, a non-null assertion on a potentially undefined `jobId` in `JobDetailPage`, and a quotes fetch strategy that over-fetches all business quotes on a page scoped to a single job.

## Warnings

### WR-01: Authorization check silently bypassed when job has no events

**File:** `trade-flow-api/src/job-event/services/job-event-retriever.service.ts:19-23`
**Issue:** `getByJobId` returns an empty array immediately when `events.length === 0` without performing any authorization check. An authenticated user belonging to a different business can call `GET /v1/job-event?jobId=<any-id>` and receive a 200 with empty data for any job in the system. The policy check only runs when there is at least one event to read. Because new jobs have zero events, this is a real attack surface.
**Fix:**
```typescript
// Fetch the job itself and check ownership before querying events,
// or require the caller to pass the businessId as a query param and
// validate it against the auth user before querying.
//
// Simplest fix: load the job to check ownership, throw ForbiddenError if
// the auth user's businessIds do not include the job's businessId.
// If you don't want to cross-module here, add businessId as a mandatory
// query param and validate it directly:

public async getByJobId(authUser: IUserDto, jobId: string): Promise<IJobEventDto[]> {
  const events = await this.jobEventRepository.findByJobId(jobId);

  // Always enforce: if the result is non-empty, use the first event's businessId.
  // If empty, the caller must supply a businessId to validate against.
  // Until that is wired, at minimum remove the early return:
  if (events.length === 0) {
    // Without a business context we cannot authorise — throw or require businessId param.
    return [];  // <-- replace with an authorisation check
  }

  const canRead = this.jobEventPolicy.canRead(authUser, events[0]);
  if (!canRead) {
    throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "You do not have permission to view events for this job");
  }

  return events;
}
```
The cleanest fix is to accept `businessId` as an additional validated query param (already present on the job) and check `authUser.businessIds.includes(businessId)` before the repository call.

---

### WR-02: `canCreate` defined on `JobEventPolicy` but never called

**File:** `trade-flow-api/src/job-event/policies/job-event.policy.ts:9-12`
**Issue:** `JobEventPolicy.canCreate` is defined but `JobEventCreator.create` does not call it. Every service that emits a job event calls `JobEventCreator.create` directly with no authorization check. This means any service that has a reference to `JobEventCreator` can write events for any job/business combination without a policy gate. The risk is bounded because `JobEventCreator` is only injected into trusted internal services (not controllers), but the dead policy method is misleading and the pattern diverges from the rest of the codebase where `AuthorizedCreatorFactory` enforces the policy on creation.
**Fix:** Either remove `canCreate` from `JobEventPolicy` (since internal services bypass it by design) and add a comment explaining the intentional trust model, or wire the policy check inside `JobEventCreator.create`:
```typescript
public async create(dto: Omit<IJobEventDto, "id" | "createdAt" | "updatedAt">): Promise<IJobEventDto> {
  // Job events are written by trusted internal services only.
  // No external authUser context is available here by design.
  return this.jobEventRepository.create(dto as IJobEventDto);
}
```
Removing `canCreate` from the policy is the cleaner option given the intent.

---

### WR-03: Non-null assertion on `jobId` may throw at runtime

**File:** `trade-flow-ui/src/pages/JobDetailPage.tsx:89`
**Issue:** `useGetJobEventsQuery(jobId!, { skip: !jobId })` uses a non-null assertion. The `skip` guard prevents the query from running when `jobId` is falsy, but TypeScript's `!` cast is applied at the call site before RTK Query evaluates `skip`. If RTK Query ever evaluates the `query` argument eagerly (e.g. in a future version, or in tests), this will produce a runtime error. This pattern appears twice more on lines 83-85 for schedules. The project's CLAUDE.md explicitly warns against `as` assertions; non-null assertions (`!`) carry the same risk.
**Fix:**
```typescript
// Provide a safe fallback instead of a non-null assertion:
const { data: events = [], isLoading: eventsLoading } = useGetJobEventsQuery(jobId ?? "", { skip: !jobId });
const { data: schedules, isLoading: schedulesLoading } = useGetSchedulesByJobQuery(
  { businessId: businessId ?? "", jobId: jobId ?? "" },
  { skip: !businessId || !jobId },
);
```

---

### WR-04: All business quotes fetched to filter by jobId on the client

**File:** `trade-flow-ui/src/pages/JobDetailPage.tsx:87, 91`
**Issue:** `useGetQuotesQuery(businessId!)` fetches all quotes for the business, and `jobQuotes` is derived by filtering on the client (`allQuotes.filter((q) => q.jobId === jobId)`). For businesses with many quotes this is wasteful — all quotes are loaded into memory and re-fetched whenever any quote cache invalidation fires. The API has job-scoped endpoints for other resources (e.g. schedules use `/v1/business/:businessId/job/:jobId/schedule`); quotes should similarly support a job-scoped query.
**Fix:** Add a `jobId` query param to the quotes API or create a dedicated job-scoped quotes endpoint. In the interim, if changing the API is out of scope for this phase, document the limitation with a `TODO`:
```typescript
// TODO: Replace with a job-scoped quote query once the API supports it (e.g. GET /v1/business/:businessId/job/:jobId/quotes)
const { data: allQuotes = [], isLoading: quotesLoading } = useGetQuotesQuery(businessId!, { skip: !businessId });
```

---

### WR-05: `jobStatus` and `quotesLoading` props accepted but immediately voided

**File:** `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx:40-41`
**Issue:** `JobDetailTabs` accepts `jobStatus` and `quotesLoading` as props in its interface (lines 18 and 24), but the component body immediately discards them with `void jobStatus` and `void quotesLoading`. This indicates incomplete wiring: the loading state for quotes is never surfaced (the tab renders data or empty state without a skeleton), and `jobStatus`-dependent logic (e.g. disabling tabs for closed jobs) was deferred without removal from the interface. Dead props in a component interface create maintenance confusion.
**Fix:** Either implement the intended behavior or remove the props from the interface and the call site until they are needed:
```typescript
// Remove from interface:
interface JobDetailTabsProps {
  jobStatus: string;      // <-- remove if not used
  quotesLoading?: boolean; // <-- remove if not used
  // ...
}

// And remove void casts from body.
```
If `quotesLoading` will be used to show a quote tab skeleton, implement it now rather than suppressing it.

---

## Info

### IN-01: `JobEventCreator.create` uses an unsafe `as` cast

**File:** `trade-flow-api/src/job-event/services/job-event-creator.service.ts:10`
**Issue:** `return this.jobEventRepository.create(dto as IJobEventDto)` casts the `Omit<IJobEventDto, "id" | "createdAt" | "updatedAt">` argument to `IJobEventDto`. The project's conventions explicitly say to avoid `as` type assertions and prefer type guards or mapping functions. The cast is safe at runtime because the repository generates `id`, `createdAt`, and `updatedAt` itself, but the type system is misled.
**Fix:** Adjust the repository's `create` method signature to accept the `Omit` type, or define a dedicated input type:
```typescript
// In job-event.repository.ts, accept the narrower type:
public async create(dto: Omit<IJobEventDto, "id" | "createdAt" | "updatedAt">): Promise<IJobEventDto>
```
Then remove the cast in `JobEventCreator`.

---

### IN-02: Inline comment in `schedule-creator.service.ts` restates the code

**File:** `trade-flow-api/src/schedule/services/schedule-creator.service.ts:31-32`
**Issue:** `// Validate job exists and user has access` and `// Validate visit type if provided` are comments that restate exactly what the next few lines do. Per project coding standards, comments that restate the code should not be committed.
**Fix:** Remove both comments. The method calls (`jobRetriever.findByIdOrFail`, `visitTypeRetriever.findByIdOrFail`) and the `if (schedule.visitTypeId)` guard are self-documenting.

---

### IN-03: `ScheduleUpdaterService` has a cross-module comment with similar issue

**File:** `trade-flow-api/src/schedule/services/schedule-updater.service.ts:41`
**Issue:** `// Cross-module validation for visitTypeId if changed` restates the code. Same standard applies.
**Fix:** Remove the comment.

---

### IN-04: `JobDetailTabs` passes a non-null assertion on `schedules` after a truthiness check

**File:** `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx:88`
**Issue:** `schedules!` is used inside a branch guarded by `hasSchedules` (which checks `schedules?.length ?? 0 > 0`). While safe, the non-null assertion is unnecessary — `schedules` is guaranteed to be truthy inside that branch. Using `schedules ?? []` is cleaner.
**Fix:**
```tsx
{hasSchedules ? (
  <ScheduleList
    schedules={schedules ?? []}
    businessId={businessId}
    onSelect={handleSelectSchedule}
    isLoading={schedulesLoading ?? false}
  />
```

---

_Reviewed: 2026-04-22T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
