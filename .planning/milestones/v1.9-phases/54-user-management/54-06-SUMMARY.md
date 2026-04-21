---
phase: 54-user-management
plan: 06
subsystem: api
tags: [mongodb, aggregation, lookup, firebase-admin, support-module]

# Dependency graph
requires:
  - phase: 54-02
    provides: Support user repository with $lookup aggregation pipelines
  - phase: 54-03
    provides: Firebase auth metadata service and dashboard metrics service
provides:
  - Corrected $lookup join keys using externalAuthUserId for subscription joins
  - Two-step businessusers-then-businesses join for user business data
  - Fixed supportroles projection using roleName field
  - Initialization logging for Firebase Admin SDK
affects: [54-user-management]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Two-step $lookup via businessusers join collection for user-to-business resolution"
    - "Use externalAuthUserId (Firebase UID) not MongoDB ObjectId for subscription joins"

key-files:
  created: []
  modified:
    - trade-flow-api/src/support/repositories/support-user.repository.ts
    - trade-flow-api/src/support/services/support-dashboard-metrics.service.ts
    - trade-flow-api/src/support/services/firebase-auth-metadata.service.ts
    - trade-flow-api/src/support/test/services/firebase-auth-metadata.service.spec.ts

key-decisions:
  - "Use externalAuthUserId for all subscription $lookup joins since subscription.userId stores Firebase UID strings"
  - "Two-step businessusers-then-businesses $lookup since UserEntity has no businessIds field"
  - "Changed Firebase metadata error logging from warn to debug since failures are expected in dev environments"

patterns-established:
  - "subscription joins must use externalAuthUserId, never $toString of _id"
  - "business data for users requires join through businessusers collection"

requirements-completed: [UMGT-01, UMGT-02, UMGT-03, UMGT-04]

# Metrics
duration: 6min
completed: 2026-04-19
---

# Phase 54 Plan 06: Gap Closure -- $lookup Join Key Fixes Summary

**Fixed all broken MongoDB $lookup join keys in the support module using externalAuthUserId for subscriptions, two-step businessusers join for business data, and roleName for role projection**

## Performance

- **Duration:** 6 min
- **Started:** 2026-04-19T12:24:57Z
- **Completed:** 2026-04-19T12:30:43Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Fixed subscription $lookup joins across findPaginated, findByIdWithDetails, and getMetrics to use externalAuthUserId instead of $toString of MongoDB _id
- Replaced broken businessIds $lookup with two-step join: businessusers collection (userId match) then businesses collection (businessId match)
- Fixed supportroles $lookup projection from name to roleName and updated the mapping function
- Added initialization warning logging when Firebase Admin SDK credentials are missing
- Changed getAuthMetadata error logging from warn to debug level for expected dev failures

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix $lookup join keys in support-user.repository.ts** - `1c170a4` (fix)
2. **Task 2: Fix dashboard metrics $lookup and improve Firebase metadata logging** - `536f032` (fix)

_Note: Commits are in the trade-flow-api sub-repo, not the docs repo._

## Files Created/Modified
- `trade-flow-api/src/support/repositories/support-user.repository.ts` - Fixed all three $lookup pipelines (subscription, business, roles) and role name mapping
- `trade-flow-api/src/support/services/support-dashboard-metrics.service.ts` - Fixed subscription $lookup let variable
- `trade-flow-api/src/support/services/firebase-auth-metadata.service.ts` - Added initialization logging, changed error log level
- `trade-flow-api/src/support/test/services/firebase-auth-metadata.service.spec.ts` - Added test for missing credentials initialization

## Decisions Made
- Used externalAuthUserId for all subscription $lookup joins since subscription.userId stores Firebase UID strings, not MongoDB ObjectIds
- Implemented two-step businessusers-then-businesses $lookup since UserEntity has no businessIds field (it is derived dynamically)
- Changed Firebase metadata error logging from warn to debug since these failures are expected in development environments without Firebase credentials

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed Prettier formatting in businessusers pipeline**
- **Found during:** Task 2 (CI verification)
- **Issue:** Multi-line pipeline array in businessusers $lookup exceeded Prettier formatting rules
- **Fix:** Collapsed two-element pipeline arrays to single lines per the 125-char print width
- **Files modified:** trade-flow-api/src/support/repositories/support-user.repository.ts
- **Verification:** `npm run format:check` passes
- **Committed in:** 536f032 (part of Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Minor formatting fix. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All $lookup join keys corrected, ready for UAT re-test of Tests 4, 7, 9, and 10
- CI passes with 0 errors

---
*Phase: 54-user-management*
*Completed: 2026-04-19*
