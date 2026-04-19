---
phase: 54-user-management
reviewed: 2026-04-19T12:00:00Z
depth: standard
files_reviewed: 38
files_reviewed_list:
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/core/data-transfer-objects/base-query-options.dto.ts
  - trade-flow-api/src/core/interfaces/query-filter.interface.ts
  - trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts
  - trade-flow-api/src/core/utilities/query-filter-parser.utility.ts
  - trade-flow-api/src/core/utilities/query-sort-parser.utility.ts
  - trade-flow-api/src/support/controllers/support-user.controller.ts
  - trade-flow-api/src/support/data-transfer-objects/dashboard-metrics.dto.ts
  - trade-flow-api/src/support/data-transfer-objects/support-user-detail.dto.ts
  - trade-flow-api/src/support/data-transfer-objects/support-user.dto.ts
  - trade-flow-api/src/support/repositories/support-user.repository.ts
  - trade-flow-api/src/support/responses/dashboard-metrics.response.ts
  - trade-flow-api/src/support/responses/support-user-detail.response.ts
  - trade-flow-api/src/support/responses/support-user.response.ts
  - trade-flow-api/src/support/services/support-dashboard-metrics.service.ts
  - trade-flow-api/src/support/services/support-user-retriever.service.ts
  - trade-flow-api/src/support/support.module.ts
  - trade-flow-api/src/support/test/mocks/support-user.mock.ts
  - trade-flow-api/src/support/test/services/support-dashboard-metrics.service.spec.ts
  - trade-flow-api/src/support/test/services/support-user-retriever.service.spec.ts
  - trade-flow-ui/src/App.tsx
  - trade-flow-ui/src/components/ui/badge.variants.ts
  - trade-flow-ui/src/features/support/api/index.ts
  - trade-flow-ui/src/features/support/api/supportApi.ts
  - trade-flow-ui/src/features/support/components/MembershipMetrics.tsx
  - trade-flow-ui/src/features/support/components/RoleBadge.tsx
  - trade-flow-ui/src/features/support/components/SubscriptionBadge.tsx
  - trade-flow-ui/src/features/support/components/UserListFilters.tsx
  - trade-flow-ui/src/features/support/components/UserListPagination.tsx
  - trade-flow-ui/src/features/support/components/UserListTable.tsx
  - trade-flow-ui/src/features/support/hooks/useSupportUsers.ts
  - trade-flow-ui/src/features/support/index.ts
  - trade-flow-ui/src/index.css
  - trade-flow-ui/src/pages/support/SupportDashboardPage.tsx
  - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx
  - trade-flow-ui/src/pages/support/SupportUsersPage.tsx
  - trade-flow-ui/src/services/api.ts
  - trade-flow-ui/src/types/api.types.ts
  - trade-flow-ui/src/types/index.ts
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 54: Code Review Report

**Reviewed:** 2026-04-19T12:00:00Z
**Depth:** standard
**Files Reviewed:** 38
**Status:** issues_found

## Summary

The Phase 54 user management feature adds a support module to the API (controller, service, repository, DTOs, responses) and a corresponding support feature in the UI (pages, components, hooks, RTK Query endpoints). The architecture follows project conventions well -- layered API design, feature-based UI modules, proper guards and permissions. However, there are a few issues worth addressing: a security concern around unvalidated filter fields enabling NoSQL injection via sort parameters, a bug where `createdAt` is always hardcoded to `null`, a duplicate RTK Query endpoint, and several minor quality items.

## Critical Issues

### CR-01: Sort field injection -- arbitrary MongoDB field names accepted without allowlist

**File:** `trade-flow-api/src/core/utilities/query-sort-parser.utility.ts:1-11`
**Issue:** The `parseQuerySort` utility accepts any user-supplied string as a MongoDB sort field name. In the controller at `trade-flow-api/src/support/controllers/support-user.controller.ts:34`, the raw `query.sort` value is parsed and passed directly into the aggregation pipeline sort stage at `trade-flow-api/src/support/repositories/support-user.repository.ts:118`. While MongoDB aggregation `$sort` does not support operator injection the way `$where` does, an attacker could sort on internal fields (e.g., `externalAuthUserId`, `supportRoleIds`, `businessRoleIds`) to infer data about other users by observing result ordering. The CLAUDE.md convention states: "Each endpoint validates filter field names against its own allowlist (not in the generic parser)." This same principle should apply to sort fields but is not implemented.
**Fix:** Add a sort field allowlist in the controller or service layer:
```typescript
const ALLOWED_SORT_FIELDS = new Set(["name", "email", "createdAt", "_id"]);
const sort = parseQuerySort(query.sort);
const sortField = Object.keys(sort)[0];
if (sortField && !ALLOWED_SORT_FIELDS.has(sortField)) {
  throw new InvalidRequestError(ErrorCodes.INVALID_REQUEST, `Invalid sort field: ${sortField}`);
}
```

## Warnings

### WR-01: `createdAt` always hardcoded to `null` in user list response

**File:** `trade-flow-api/src/support/controllers/support-user.controller.ts:114`
**Issue:** The `mapToResponse` method hardcodes `createdAt: null` for every user. The `ISupportUserResponse` interface declares `createdAt: string | null`, and the UI renders it via `formatCreatedAt` in `UserListTable.tsx:16-19` (which shows an em-dash for null). The "Created" column in the table will always display "---" for every user, making the column useless. Either the user creation date should be fetched from the database (e.g., from the MongoDB `_id` timestamp or a stored `createdAt` field), or the column should be removed from the UI.
**Fix:** Extract the creation timestamp from the MongoDB ObjectId in the repository's `mapToDto` method, or add a `createdAt` field to the aggregation pipeline projection. Then pass it through the DTO and response:
```typescript
// In mapToDto, extract from ObjectId:
createdAt: doc._id.getTimestamp().toISOString(),
```

### WR-02: Duplicate RTK Query endpoints for fetching a single support user

**File:** `trade-flow-ui/src/features/support/api/supportApi.ts:45-53` and `trade-flow-ui/src/features/support/api/supportApi.ts:87-96`
**Issue:** `getSupportUserById` (line 45) and `getSupportUserDetail` (line 87) are nearly identical endpoints -- both call `/v1/support/users/${id}`, both transform the response the same way, and both return `SupportUserDetail`. This creates two separate cache entries for the same data, wasting memory and risking stale data if one is invalidated but the other is not. The barrel export at `trade-flow-ui/src/features/support/api/index.ts:4-6` exports both hooks.
**Fix:** Remove the duplicate `getSupportUserById` endpoint and its hook. Update any consumers to use `useGetSupportUserDetailQuery` exclusively.

### WR-03: Missing `incomplete` status in the UI subscription filter dropdown

**File:** `trade-flow-ui/src/features/support/components/UserListFilters.tsx:49-56`
**Issue:** The backend dashboard metrics service counts subscriptions with status `"incomplete"` as "expired" (`support-dashboard-metrics.service.ts:54`). The MembershipMetrics card links to `?filter:subscription.status:eq=incomplete` (line 97 of `MembershipMetrics.tsx`). However, the subscription status dropdown in `UserListFilters.tsx` does not include `"incomplete"` as an option -- it has `"expired"` instead. When a user clicks the "Expired" metric card, they land on the users page with a filter value (`incomplete`) that does not match any dropdown option, so the dropdown will show "All statuses" while the filter is actually active. Similarly, `SubscriptionBadge.tsx` maps `"expired"` but not `"incomplete"`.
**Fix:** Align the terminology. Either:
1. Add `<SelectItem value="incomplete">Expired</SelectItem>` to the dropdown and update `SubscriptionBadge` to map `"incomplete"` to "Expired", or
2. Change the metrics service and metric card link to use `"expired"` consistently.

### WR-04: `findByIdWithDetails` does not validate ObjectId format before construction

**File:** `trade-flow-api/src/support/repositories/support-user.repository.ts:143`
**Issue:** `new ObjectId(id)` will throw a generic `BSONError` if `id` is not a valid 24-character hex string. This error is not a `ResourceNotFoundError`, so it will propagate as an unhandled error and likely become an HTTP 500 instead of a 404 or 422. Other repositories in the codebase likely have the same pattern, but for this new code the fix is straightforward.
**Fix:** Validate the id before constructing the ObjectId:
```typescript
if (!ObjectId.isValid(id)) {
  throw new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, `User with id ${id} not found`);
}
const objectId = new ObjectId(id);
```

### WR-05: `getDashboardMetrics` response can crash on empty `data` array

**File:** `trade-flow-ui/src/features/support/api/supportApi.ts:99`
**Issue:** The `transformResponse` for `getDashboardMetrics` accesses `response.data[0]` without checking if `response.data` exists or has elements. If the API returns `{ data: [] }`, this will return `undefined`, and downstream components accessing `metrics.totalUsers` etc. will throw.
**Fix:** Add a guard:
```typescript
transformResponse: (response: StandardResponse<DashboardMetrics>) => {
  if (!response.data || response.data.length === 0) {
    return { totalUsers: 0, activeTrials: 0, activeSubscriptions: 0, expiredSubscriptions: 0, canceledSubscriptions: 0 };
  }
  return response.data[0];
},
```

## Info

### IN-01: Test coverage gap -- `findByIdWithDetails` not tested in retriever spec

**File:** `trade-flow-api/src/support/test/services/support-user-retriever.service.spec.ts`
**Issue:** The `SupportUserRetriever` service has two public methods (`findPaginated` and `findByIdWithDetails`), but the test only covers `findPaginated`. Additionally, the mock for `SupportUserRepository` at line 9 only stubs `findPaginated`, so `findByIdWithDetails` would fail if tested.
**Fix:** Add test cases for `findByIdWithDetails` and include it in the mock setup:
```typescript
useValue: {
  findPaginated: jest.fn(),
  findByIdWithDetails: jest.fn(),
},
```

### IN-02: `"expired"` label mismatch with actual backend status `"incomplete"`

**File:** `trade-flow-api/src/support/services/support-dashboard-metrics.service.ts:53-55`
**Issue:** The `expiredSubscriptions` metric counts documents where `subscriptionDocs.status === "incomplete"`, but the metric name says "expired". This is a semantic mismatch that could confuse developers. Consider renaming the metric to `incompleteSubscriptions` or adding a code comment explaining the business mapping.
**Fix:** Consider aligning naming to match the actual status value, or document the mapping clearly.

### IN-03: Magic number `20` repeated across UI files

**File:** `trade-flow-ui/src/features/support/hooks/useSupportUsers.ts:21,99` and `trade-flow-ui/src/features/support/api/supportApi.ts:34`
**Issue:** The page size `20` is repeated in multiple places without a shared constant. If the page size needs to change, multiple files must be updated.
**Fix:** Extract to a shared constant:
```typescript
export const SUPPORT_USERS_PAGE_SIZE = 20;
```

---

_Reviewed: 2026-04-19T12:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
