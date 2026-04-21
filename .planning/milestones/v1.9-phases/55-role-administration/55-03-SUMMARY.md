---
phase: 55-role-administration
plan: "03"
subsystem: trade-flow-api
tags: [rbac, permission-guard, subscription-guard, nestjs, gap-closure]
dependency_graph:
  requires:
    - phase: 55-01
      provides: SupportRoleAdminController, PermissionGuard
  provides:
    - SkipSubscriptionCheck on both grant/revoke endpoints
    - PermissionGuard throws ForbiddenException (403) instead of ForbiddenError (500)
  affects:
    - trade-flow-api/src/user/controllers/support-role-admin.controller.ts
    - trade-flow-api/src/auth/guards/permission.guard.ts
tech_stack:
  added: []
  patterns:
    - Guards must throw NestJS HttpException subclasses (not domain errors) since they execute before controller try/catch
    - Support admin endpoints use @SkipSubscriptionCheck() alongside @RequiresPermission() (no subscription context)
key_files:
  created: []
  modified:
    - trade-flow-api/src/user/controllers/support-role-admin.controller.ts
    - trade-flow-api/src/auth/guards/permission.guard.ts
    - trade-flow-api/src/auth/test/guards/permission.guard.spec.ts
key_decisions:
  - "PermissionGuard response body uses { errors: [{ code, message, details }] } shape matching createHttpError() output for frontend compatibility"
  - "Used string 'ACTION_FORBIDDEN' directly instead of ErrorCodes enum to avoid importing domain error infrastructure into guard"
patterns-established:
  - "Guards throw NestJS HttpException subclasses, not domain errors -- domain errors only work in controller catch blocks via createHttpError()"
requirements-completed: [RADM-01, RADM-02, RADM-05]
metrics:
  duration: 3min
  completed: 2026-04-19
---

# Phase 55 Plan 03: SubscriptionGuard and PermissionGuard Gap Closure Summary

**Fixed SubscriptionGuard blocking grant/revoke endpoints via @SkipSubscriptionCheck() and PermissionGuard returning 500 via ForbiddenException replacement**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-19T13:41:05Z
- **Completed:** 2026-04-19T13:44:00Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Grant/revoke role endpoints no longer blocked by globally-registered SubscriptionGuard
- PermissionGuard now returns proper 403 Forbidden instead of 500 Internal Server Error for unauthorized requests
- Response body shape preserved for frontend error parsing compatibility

## Task Commits

Each task was committed atomically:

1. **Task 1: Add @SkipSubscriptionCheck() to SupportRoleAdminController** - `28ccaa7` (fix)
2. **Task 2: Fix PermissionGuard to throw NestJS ForbiddenException** - `6da7c46` (fix)

## Files Created/Modified
- `trade-flow-api/src/user/controllers/support-role-admin.controller.ts` - Added @SkipSubscriptionCheck() to both grantRole() and revokeRole() methods
- `trade-flow-api/src/auth/guards/permission.guard.ts` - Replaced ForbiddenError with ForbiddenException, removed ErrorCodes import
- `trade-flow-api/src/auth/test/guards/permission.guard.spec.ts` - Updated tests to assert ForbiddenException with 403 status

## Decisions Made
- Used string literal `"ACTION_FORBIDDEN"` instead of `ErrorCodes.ACTION_FORBIDDEN` enum to avoid importing domain error infrastructure into the guard layer
- Preserved `{ errors: [{ code, message, details }] }` response body shape so frontend `RevokeRoleDialog` error parsing (`error.data.errors[0].details`) continues working

## Deviations from Plan

None - plan executed exactly as written.

## Verification

- `npm run test -- --testPathPatterns=support-role-admin.controller`: 5 tests passed
- `npm run test -- --testPathPatterns=permission.guard`: 9 tests passed
- `npm run ci`: all quality gates passed (tests, lint, format, typecheck)
- `grep -n "SkipSubscriptionCheck" support-role-admin.controller.ts`: lines 5, 20, 35
- `grep -n "ForbiddenException" permission.guard.ts`: lines 1, 25, 37
- `grep -c "ForbiddenError" permission.guard.ts`: 0

## Threat Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| T-55-10 Elevation of Privilege (SubscriptionGuard bypass) | @SkipSubscriptionCheck() only on endpoints already protected by PermissionGuard + @RequiresPermission("manage_roles") |
| T-55-11 Information Disclosure (error message) | Error details show only permission name (e.g., "manage_roles"), not internal IDs or user data |

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Both UAT failures from Phase 55 are now resolved
- Grant/revoke role flow should complete successfully end-to-end
- PermissionGuard returns proper HTTP status codes for all permission-protected endpoints

---
*Phase: 55-role-administration*
*Completed: 2026-04-19*
