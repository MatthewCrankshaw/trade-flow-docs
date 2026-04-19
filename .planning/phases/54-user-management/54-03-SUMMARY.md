---
phase: 54-user-management
plan: "03"
subsystem: api
tags: [nestjs, firebase-admin, mongodb-aggregation, support-tools, rbac]

dependency_graph:
  requires:
    - phase: 54-01
      provides: MongoDbFetcher.aggregate(), IBaseQueryOptionsDto
    - phase: 54-02
      provides: SupportUserRepository, SupportUserRetriever, SupportUserController, SupportModule
  provides:
    - GET /v1/support/users/:id — user detail with Firebase auth metadata
    - GET /v1/support/dashboard/metrics — membership summary counts
    - FirebaseAuthMetadataService wrapping firebase-admin SDK
    - SupportDashboardMetrics computing membership counts via aggregation
    - ISupportUserDetailDto, ISupportUserDetailResponse
    - IDashboardMetricsDto, IDashboardMetricsResponse
  affects:
    - Plan 54-05 (user detail page UI — consumes GET /v1/support/users/:id)
    - Plan 54-04 (dashboard page UI — consumes GET /v1/support/dashboard/metrics)

tech_stack:
  added:
    - firebase-admin (npm package, server-side Firebase Admin SDK)
  patterns:
    - Singleton Firebase Admin SDK initialization (admin.apps.length === 0 guard)
    - Graceful metadata degradation (null on failure, never throws)
    - MongoDB $lookup for role name resolution via supportroles collection
    - $group aggregation for membership count metrics

key_files:
  created:
    - trade-flow-api/src/support/services/firebase-auth-metadata.service.ts
    - trade-flow-api/src/support/services/support-dashboard-metrics.service.ts
    - trade-flow-api/src/support/data-transfer-objects/support-user-detail.dto.ts
    - trade-flow-api/src/support/data-transfer-objects/dashboard-metrics.dto.ts
    - trade-flow-api/src/support/responses/support-user-detail.response.ts
    - trade-flow-api/src/support/responses/dashboard-metrics.response.ts
    - trade-flow-api/src/support/test/services/firebase-auth-metadata.service.spec.ts
    - trade-flow-api/src/support/test/services/support-dashboard-metrics.service.spec.ts
  modified:
    - trade-flow-api/src/support/repositories/support-user.repository.ts
    - trade-flow-api/src/support/services/support-user-retriever.service.ts
    - trade-flow-api/src/support/controllers/support-user.controller.ts
    - trade-flow-api/src/support/support.module.ts
    - trade-flow-api/package.json
    - trade-flow-api/.env.example

decisions:
  - "Initialize Firebase Admin SDK once using admin.apps.length === 0 guard; support both service account credentials and Application Default Credentials"
  - "FirebaseAuthMetadataService returns nulls on failure — metadata is supplementary, not critical to the response"
  - "findByIdWithDetails adds supportroles $lookup to resolve role names, kept separate from findPaginated to avoid performance impact on list queries"
  - "firebaseMetadata field populated in controller (not repository) to keep repository layer free of external service dependencies"

requirements-completed:
  - UMGT-03
  - UMGT-04

metrics:
  duration: ~20 minutes
  completed: "2026-04-19T07:20:00Z"
  tasks_completed: 2
  files_created: 8
  files_modified: 6
---

# Phase 54 Plan 03: User Detail and Dashboard Metrics API Summary

**Firebase Admin SDK integration for auth metadata plus GET /v1/support/users/:id and GET /v1/support/dashboard/metrics endpoints backed by MongoDB aggregation pipelines.**

## Performance

- **Duration:** ~20 minutes
- **Started:** 2026-04-19T07:00:00Z
- **Completed:** 2026-04-19T07:20:00Z
- **Tasks:** 2
- **Files created:** 8
- **Files modified:** 6

## Accomplishments

- Installed `firebase-admin` and wrapped it in `FirebaseAuthMetadataService` with singleton initialization and graceful null fallback on any Firebase error
- Built `SupportDashboardMetrics` computing membership counts (totalUsers, activeTrials, activeSubscriptions, expiredSubscriptions, canceledSubscriptions) via a single `$group` aggregation pipeline over the users+subscriptions join
- Extended `SupportUserRepository` with `findByIdWithDetails` — matches single user by ObjectId, joins subscriptions, businesses, and supportroles collections, maps to `ISupportUserDetailDto` with role names
- Extended `SupportUserRetriever` with `findByIdWithDetails` delegate method
- Added `GET /v1/support/users/:id` and `GET /v1/support/dashboard/metrics` endpoints to `SupportUserController`, both protected by JwtAuthGuard + PermissionGuard(manage_users) + SkipSubscriptionCheck
- Registered `FirebaseAuthMetadataService`, `SupportDashboardMetrics`, and `ConfigModule` in `SupportModule`
- Updated `.env.example` with `FIREBASE_CLIENT_EMAIL` and `FIREBASE_PRIVATE_KEY` variables

## Task Commits

Each task was committed atomically in the API repo:

1. **Task 1: Firebase auth metadata service, dashboard metrics service, DTOs, responses, and tests** - `5cbc837` (feat)
2. **Task 2: User detail and dashboard metrics controller endpoints with module registration** - `01ddef7` (feat)

## Files Created/Modified

- `src/support/services/firebase-auth-metadata.service.ts` — Firebase Admin SDK wrapper, singleton init, graceful null on failure
- `src/support/services/support-dashboard-metrics.service.ts` — getMetrics() via $lookup + $group aggregation
- `src/support/data-transfer-objects/support-user-detail.dto.ts` — ISupportUserDetailDto with firebaseMetadata and supportRoleNames
- `src/support/data-transfer-objects/dashboard-metrics.dto.ts` — IDashboardMetricsDto
- `src/support/responses/support-user-detail.response.ts` — ISupportUserDetailResponse for HTTP shape
- `src/support/responses/dashboard-metrics.response.ts` — IDashboardMetricsResponse for HTTP shape
- `src/support/test/services/firebase-auth-metadata.service.spec.ts` — 3 tests: success, Firebase error, undefined metadata
- `src/support/test/services/support-dashboard-metrics.service.spec.ts` — 3 tests: counts, empty/zeros, collection name
- `src/support/repositories/support-user.repository.ts` — Added findByIdWithDetails with supportroles $lookup
- `src/support/services/support-user-retriever.service.ts` — Added findByIdWithDetails delegate
- `src/support/controllers/support-user.controller.ts` — Added GET users/:id and GET dashboard/metrics endpoints
- `src/support/support.module.ts` — Registered FirebaseAuthMetadataService, SupportDashboardMetrics, ConfigModule
- `trade-flow-api/package.json` — Added firebase-admin dependency
- `trade-flow-api/.env.example` — Added FIREBASE_CLIENT_EMAIL and FIREBASE_PRIVATE_KEY

## Decisions Made

- **Singleton Firebase Admin SDK:** `admin.apps.length === 0` guard prevents re-initialization errors across NestJS module restarts. Supports both full service account credentials and Application Default Credentials (projectId only) for flexibility in different environments.
- **Graceful metadata degradation:** `getAuthMetadata` catches all errors and returns `{ creationTime: null, lastSignInTime: null }`. Firebase metadata is supplementary context — it should never block a support user detail page from loading.
- **Repository stays free of external services:** `firebaseMetadata` is populated in the controller after `findByIdWithDetails` returns. This keeps the repository layer concerned only with MongoDB and avoids coupling the repository to Firebase.
- **Separate findByIdWithDetails from findPaginated:** The detail method adds a `supportroles` $lookup for role name resolution. This is intentionally separate from the list method to avoid the performance cost of an extra join on every paginated row.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed jest.mock hoisting reference error in firebase-auth-metadata test**
- **Found during:** Task 1 test verification
- **Issue:** `jest.mock("firebase-admin", ...)` factory referenced `mockInitializeApp` variable before it was initialized — Jest hoists `jest.mock` calls above variable declarations, causing `ReferenceError: Cannot access 'mockInitializeApp' before initialization`
- **Fix:** Used `jest.fn()` inline inside the mock factory instead of referencing hoisted variables; kept `mockGetUser` as a module-level variable since it's used in test bodies (not in the factory itself)
- **Files modified:** `src/support/test/services/firebase-auth-metadata.service.spec.ts`
- **Commit:** 5cbc837 (test file rewritten before Task 1 commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Zero — test fix resolves hoisting issue; behavior under test is unchanged.

## Known Stubs

None. Both endpoints are fully implemented with real data sources:
- `GET /v1/support/users/:id` fetches from MongoDB and Firebase Admin SDK
- `GET /v1/support/dashboard/metrics` fetches from MongoDB aggregation

## Threat Surface Scan

Two new network endpoints introduced:

| Flag | File | Description |
|------|------|-------------|
| threat_flag: new-endpoint | support-user.controller.ts | GET /v1/support/users/:id — mitigated by JwtAuthGuard + PermissionGuard(manage_users) + SkipSubscriptionCheck per T-54-06 |
| threat_flag: new-endpoint | support-user.controller.ts | GET /v1/support/dashboard/metrics — mitigated by JwtAuthGuard + PermissionGuard(manage_users) + SkipSubscriptionCheck |
| threat_flag: external-service | firebase-auth-metadata.service.ts | Server-side Firebase Admin SDK call — mitigated: credentials server-side only, never user-supplied (T-54-07) |

T-54-08 (tampering via :id): mitigated — `new ObjectId(id)` in repository throws on invalid ObjectId format, caught by `createHttpError` and returned as 500; ResourceNotFoundError thrown if valid ObjectId but no user found (404).

## Self-Check: PASSED

| Item | Status |
|------|--------|
| firebase-auth-metadata.service.ts | FOUND |
| support-dashboard-metrics.service.ts | FOUND |
| support-user-detail.dto.ts | FOUND |
| dashboard-metrics.dto.ts | FOUND |
| support-user-detail.response.ts | FOUND |
| dashboard-metrics.response.ts | FOUND |
| firebase-auth-metadata.service.spec.ts | FOUND |
| support-dashboard-metrics.service.spec.ts | FOUND |
| Commit 5cbc837 (Task 1) | FOUND |
| Commit 01ddef7 (Task 2) | FOUND |
| Tests: 25 passed (5 suites) | PASSED |
| TypeScript: no errors in support/ | PASSED |

## Next Phase Readiness

- GET /v1/support/users/:id is ready for the UI user detail page (Plan 54-05)
- GET /v1/support/dashboard/metrics is ready for the UI dashboard metric cards (Plan 54-04)
- Response includes: id, email, name, supportRoleIds, supportRoleNames, subscription, business, firebaseMetadata (creationTime, lastSignInTime)
- Metrics response includes: totalUsers, activeTrials, activeSubscriptions, expiredSubscriptions, canceledSubscriptions

---
*Phase: 54-user-management*
*Completed: 2026-04-19*
