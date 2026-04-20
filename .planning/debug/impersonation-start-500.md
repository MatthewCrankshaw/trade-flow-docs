---
status: resolved
trigger: "POST /v1/impersonation/start returns 500 - SubscriptionGuard throws ForbiddenError"
created: 2026-04-20
updated: 2026-04-20
---

## Symptoms

- **Expected:** Clicking impersonate user button switches admin to target user's view
- **Actual:** 500 error returned from POST /v1/impersonation/start
- **Error:** ForbiddenError thrown at SubscriptionGuard.canActivate (subscription.guard.ts:46) with message "Action is forbidden", details "Authentication required", code CORE_7
- **Timeline:** First time testing - new phase 56 functionality
- **Reproduction:** Click impersonate user button on support user detail page
- **Notes:** GET /v1/support/users/{id} succeeds (200) just before the POST fails. The subscription guard is blocking the impersonation endpoint.

## Stack Trace

```
ForbiddenError: Action is forbidden
    at SubscriptionGuard.canActivate (/app/src/subscription/guards/subscription.guard.ts:46:13)
    at GuardsConsumer.tryActivate
    at canActivateFn
```

## Current Focus

- hypothesis: CONFIRMED - SubscriptionGuard (global APP_GUARD) runs before JwtAuthGuard (method-level), so request.user is undefined for POST requests to impersonation endpoints
- test: Verified impersonation controller lacked @SkipSubscriptionCheck() decorator
- expecting: Adding @SkipSubscriptionCheck() at controller class level bypasses the guard
- next_action: none - fix applied

## Evidence

- timestamp: 2026-04-20 - SubscriptionGuard is registered as APP_GUARD in app.module.ts (line 69), making it global
- timestamp: 2026-04-20 - SubscriptionGuard line 37-39 allows GET/HEAD unconditionally, line 46 throws ForbiddenError when request.user is undefined for mutation requests
- timestamp: 2026-04-20 - Phase 52 commit c55bb25 changed the !user check from `return true` to `throw ForbiddenError` (defense-in-depth)
- timestamp: 2026-04-20 - ImpersonationController had @UseGuards(JwtAuthGuard) at method level but no @SkipSubscriptionCheck()
- timestamp: 2026-04-20 - SupportUserController, SupportRoleAdminController, SubscriptionController all use @SkipSubscriptionCheck() for their mutation endpoints
- timestamp: 2026-04-20 - All 35 impersonation tests pass after adding @SkipSubscriptionCheck() at class level
- timestamp: 2026-04-20 - All 15 subscription guard tests pass (no regression)

## Eliminated

- Missing bypass_subscription permission: Not reached because request.user is undefined before the permission check at line 50
- JwtAuthGuard failure: GET /v1/support/users/{id} succeeds with same auth token, confirming JWT is valid

## Resolution

- **root_cause:** ImpersonationController POST endpoints missing @SkipSubscriptionCheck() decorator. The global SubscriptionGuard (APP_GUARD) runs before method-level JwtAuthGuard, so request.user is undefined when the guard checks mutation requests. The guard's fail-closed behavior (added in phase 52) throws ForbiddenError for undefined users.
- **fix:** Added @SkipSubscriptionCheck() at class level on ImpersonationController. Impersonation endpoints are support-admin-only and already enforce their own support-role check in the controller body.
- **file:** trade-flow-api/src/impersonation/controllers/impersonation.controller.ts
