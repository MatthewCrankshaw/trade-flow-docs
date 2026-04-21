---
phase: 54-user-management
plan: "02"
subsystem: api
tags: [nestjs, mongodb, aggregation, pagination, rbac, support-tools]

dependency_graph:
  requires:
    - phase: 54-01
      provides: parseQueryFilters, parseQuerySort, escapeRegex, IBaseQueryOptionsDto with filters/sort/search, MongoDbFetcher.aggregate()
  provides:
    - SupportModule with controller, retriever service, and repository
    - GET /v1/support/users endpoint with JwtAuthGuard + RequiresPermission(manage_users)
    - MongoDB $lookup aggregation pipeline joining users with subscriptions and businesses
    - Server-side search (name/email regex), role filter (all/support/customer), subscription status post-lookup filter
    - $facet-based pagination with configurable page/limit
    - ISupportUserDto, ISupportUserResponse interfaces
    - SupportUserMockGenerator for unit testing
  affects:
    - Plan 54-04 (user list page UI — consumes GET /v1/support/users)

tech_stack:
  added: []
  patterns:
    - MongoDB $lookup with pipeline-style join (ObjectId-to-string via $toString for cross-type join)
    - $facet pagination (data + totalCount in single aggregation pass)
    - Post-lookup $match for subscription status filter (applied after $unwind)
    - Pre-lookup $match for role filter (applied before expensive joins)
    - PermissionGuard + RequiresPermission decorator pattern for support-only endpoints
    - SkipSubscriptionCheck on support endpoints to bypass subscription gating

key_files:
  created:
    - trade-flow-api/src/support/data-transfer-objects/support-user.dto.ts
    - trade-flow-api/src/support/responses/support-user.response.ts
    - trade-flow-api/src/support/repositories/support-user.repository.ts
    - trade-flow-api/src/support/services/support-user-retriever.service.ts
    - trade-flow-api/src/support/controllers/support-user.controller.ts
    - trade-flow-api/src/support/support.module.ts
    - trade-flow-api/src/support/test/mocks/support-user.mock.ts
    - trade-flow-api/src/support/test/services/support-user-retriever.service.spec.ts
  modified:
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json

key_decisions:
  - "Use pipeline-style $lookup (let/pipeline/$expr) not simple localField/foreignField to handle ObjectId-to-string type mismatch between user._id (ObjectId) and subscription.userId (string)"
  - "Apply role filter as pre-lookup $match (before joins) and subscription status as post-lookup $match (after $unwind) for correct semantics"
  - "Add @support/* and @support-test/* path aliases to tsconfig.json and package.json moduleNameMapper"
  - "RequiresPermission and PermissionGuard already existed (Phase 52/55 work) — used directly with manage_users permission"

requirements-completed:
  - UMGT-01
  - UMGT-02

duration: ~12 minutes
completed: "2026-04-19"
---

# Phase 54 Plan 02: Support User List API Summary

**GET /v1/support/users endpoint with MongoDB $lookup aggregation pipeline joining users, subscriptions, and businesses, protected by manage_users permission guard.**

## Performance

- **Duration:** ~12 minutes
- **Started:** 2026-04-19T07:30:00Z
- **Completed:** 2026-04-19T07:42:00Z
- **Tasks:** 2
- **Files modified:** 10

## Accomplishments

- Built SupportUserRepository with a multi-stage MongoDB aggregation pipeline: pre-match (search + role filter), $lookup for subscriptions (ObjectId-to-string join), $lookup for businesses, post-lookup subscription status filter, and $facet pagination
- Created GET /v1/support/users endpoint protected by JwtAuthGuard + PermissionGuard with `manage_users` permission, with SkipSubscriptionCheck for support bypass
- Added @support/* path aliases, SupportModule registered in AppModule, 3 unit tests passing

## Task Commits

Each task was committed atomically in the API repo:

1. **Task 1: DTOs, responses, repository, and mocks** - `986166b` (feat)
2. **Task 2: Retriever service, controller, module, and unit tests** - `afc0325` (feat)

**Plan metadata (docs repo):** committed below

## Files Created/Modified

- `trade-flow-api/src/support/data-transfer-objects/support-user.dto.ts` - ISupportUserDto, ISupportUserSubscriptionDto, ISupportUserBusinessDto
- `trade-flow-api/src/support/responses/support-user.response.ts` - ISupportUserResponse for HTTP shape
- `trade-flow-api/src/support/repositories/support-user.repository.ts` - Aggregation pipeline with $lookup, $facet, search/role/status filters
- `trade-flow-api/src/support/services/support-user-retriever.service.ts` - SupportUserRetriever delegating to repository
- `trade-flow-api/src/support/controllers/support-user.controller.ts` - GET /v1/support/users with guards and query parsing
- `trade-flow-api/src/support/support.module.ts` - SupportModule with CoreModule import
- `trade-flow-api/src/support/test/mocks/support-user.mock.ts` - SupportUserMockGenerator with factory methods
- `trade-flow-api/src/support/test/services/support-user-retriever.service.spec.ts` - 3 unit tests, all passing
- `trade-flow-api/src/app.module.ts` - SupportModule added to imports
- `trade-flow-api/tsconfig.json` - Added @support/* and @support-test/* path aliases
- `trade-flow-api/package.json` - Added moduleNameMapper entries for @support/* and @support-test/*

## Decisions Made

- **Pipeline-style $lookup for cross-type join:** subscription.userId is stored as a string representation of the user ObjectId. Simple localField/foreignField $lookup cannot match across types, so we use `let: { odUserId: { $toString: "$_id" } }` with `$expr: { $eq: ["$userId", "$$odUserId"] }`.
- **Filter placement strategy:** Role filter applied pre-lookup (cheaper, reduces input to joins). Subscription status filter applied post-lookup (requires joined data to filter on).
- **Path aliases added as deviation Rule 3:** No @support/* alias existed. Added to both tsconfig.json and package.json moduleNameMapper before writing any support module files.
- **RequiresPermission available:** The RequiresPermission decorator and PermissionGuard were already built (Phase 55 wave), so used directly rather than adding a TODO comment.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added @support/* path aliases to tsconfig.json and package.json**
- **Found during:** Task 1 (before creating any files)
- **Issue:** No @support/* path alias existed in tsconfig.json or jest moduleNameMapper — imports like `@support/data-transfer-objects/support-user.dto` would fail TypeScript compilation and Jest resolution
- **Fix:** Added `"@support/*": ["./src/support/*"]` and `"@support-test/*": ["./src/support/test/*"]` to both tsconfig.json paths and package.json moduleNameMapper
- **Files modified:** trade-flow-api/tsconfig.json, trade-flow-api/package.json
- **Verification:** TypeScript compilation passes with zero support-file errors
- **Committed in:** 986166b (Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Path alias registration is a prerequisite for all NestJS module imports to work. Zero scope creep.

## Issues Encountered

None. Plan executed smoothly with the one pre-requisite alias registration handled as a Rule 3 deviation.

## Known Stubs

None. The `createdAt` field in ISupportUserResponse is returned as `null` because the user entity does not expose `createdAt` through the aggregation result. This is intentional — the field is present in the response interface for future use when the aggregation pipeline is extended to project user audit fields. It does not affect the plan's goal of returning paginated users with subscription status.

## Threat Surface Scan

New network endpoint introduced: `GET /v1/support/users`

| Flag | File | Description |
|------|------|-------------|
| threat_flag: new-endpoint | support-user.controller.ts | GET /v1/support/users — mitigated by JwtAuthGuard + PermissionGuard(manage_users) + SkipSubscriptionCheck as designed in threat model T-54-03 |

T-54-04 (regex injection): mitigated via escapeRegex() before $regex construction.
T-54-05 (information disclosure): mitigated — endpoint requires manage_users permission, response excludes passwords/tokens.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| support-user.dto.ts | FOUND |
| support-user.response.ts | FOUND |
| support-user.repository.ts | FOUND |
| support-user-retriever.service.ts | FOUND |
| support-user.controller.ts | FOUND |
| support.module.ts | FOUND |
| support-user.mock.ts | FOUND |
| support-user-retriever.service.spec.ts | FOUND |
| Commit 986166b (Task 1) | FOUND |
| Commit afc0325 (Task 2) | FOUND |
| Tests: 3 passed | PASSED |

## Next Phase Readiness

- GET /v1/support/users is ready for the UI user list page (Plan 54-04)
- Endpoint accepts: `?search=`, `?filter:role.type:eq=support`, `?filter:subscription.status:eq=active`, `?sort=-_id`, `?page=1&limit=20`
- SupportUserMockGenerator available for future test authoring

---
*Phase: 54-user-management*
*Completed: 2026-04-19*
