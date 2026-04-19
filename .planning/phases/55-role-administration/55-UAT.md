---
status: diagnosed
phase: 55-role-administration
source: [55-01-SUMMARY.md, 55-02-SUMMARY.md]
started: 2026-04-19T09:00:00Z
updated: 2026-04-19T09:20:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Support User Detail Page Loads
expected: Navigate to /support/users/:userId (using a valid user ID). The page loads showing three cards: Account (email, creation time, last sign-in), Subscription (status badge, business name, trade), and Roles (role badges for assigned support roles).
result: pass

### 2. Grant Role Button Visibility (Super User)
expected: As a super user viewing a customer user's detail page (a user with no support roles), a "Grant Role" button appears in the Roles card. The button should not appear if the target user already has a support role.
result: pass

### 3. Grant Role Flow
expected: Click "Grant Role" on a customer user's detail page. A confirmation dialog appears with title "Grant Admin Role" and the target user's name in the description. Clicking "Grant Role" in the dialog shows a loading spinner, then a success toast "Admin role granted to {name}" appears and the Roles card updates to show the new role badge.
result: issue
reported: "Dialog appears but clicking Grant Role returns a 500 error. Logs show SubscriptionGuard.canActivate throwing ForbiddenError on POST /v1/users/:id/support-role"
severity: blocker

### 4. Revoke Role Button Visibility
expected: As a super user viewing a support_administrator user's detail page (not your own profile, not a super user), a "Revoke Role" button appears in the Roles card with destructive styling. The button should NOT appear when viewing your own profile or a super user's profile.
result: blocked
blocked_by: prior-phase
reason: "Depends on working grant flow to have a revocable user"

### 5. Revoke Role Flow
expected: Click "Revoke Role" on a support_administrator's detail page. A confirmation dialog appears with destructive styling. Clicking "Revoke Role" shows a loading spinner, then a success toast appears and the role badge is removed from the Roles card.
result: blocked
blocked_by: prior-phase
reason: "Depends on working grant flow"

### 6. Self-Revocation Prevention
expected: When viewing your own profile as a super user, no "Revoke Role" button should be visible. The backend also blocks self-revocation with a ForbiddenError if attempted directly.
result: pass

### 7. Role Actions Hidden for Non-Super-Users
expected: Log in as a support_administrator (not super_user). Navigate to any user's detail page. No Grant or Revoke role buttons should be visible — role management actions are restricted to super users only.
result: blocked
blocked_by: prior-phase
reason: "Need a working grant flow to have a support_administrator account to test with"

### 8. Permission Guard Enforcement
expected: If a non-admin user attempts to call POST /v1/users/:id/support-role or DELETE /v1/users/:id/support-role directly (e.g., via API tool), the request is rejected with a 403 Forbidden error. The manage_roles permission is required.
result: issue
reported: "ForbiddenError appears in logs but API returns 500 instead of 403. Error-to-HTTP-status mapping is not working for errors thrown from guards."
severity: major

## Summary

total: 8
passed: 3
issues: 2
pending: 0
skipped: 0
blocked: 3

## Gaps

- truth: "Grant role flow completes successfully — dialog confirmation triggers POST, returns success, UI updates"
  status: failed
  reason: "User reported: SubscriptionGuard.canActivate throws ForbiddenError on POST /v1/users/:id/support-role, returning 500"
  severity: blocker
  test: 3
  root_cause: "SubscriptionGuard is registered as APP_GUARD globally (app.module.ts:67). POST requests are not exempt. SupportRoleAdminController is missing @SkipSubscriptionCheck() decorator, so the guard rejects the request before it reaches the controller."
  artifacts:
    - path: "trade-flow-api/src/user/controllers/support-role-admin.controller.ts"
      issue: "Missing @SkipSubscriptionCheck() decorator on POST and DELETE methods"
    - path: "trade-flow-api/src/app.module.ts"
      issue: "SubscriptionGuard registered globally via APP_GUARD (line 67)"
  missing:
    - "Add @SkipSubscriptionCheck() to both POST and DELETE methods on SupportRoleAdminController"
  debug_session: ""

- truth: "Permission guard returns 403 Forbidden for unauthorized users"
  status: failed
  reason: "User reported: ForbiddenError in logs but API returns 500 instead of 403"
  severity: major
  test: 8
  root_cause: "ForbiddenError is a domain error that does not extend NestJS HttpException. Guards throw errors before controllers, so createHttpError() (used in controller catch blocks) never runs. No global exception filter exists to convert domain errors to HTTP exceptions. When a guard throws ForbiddenError, NestJS treats it as unhandled and returns 500."
  artifacts:
    - path: "trade-flow-api/src/core/errors/handle-error.utility.ts"
      issue: "createHttpError() maps ForbiddenError to 403 but only works in controller catch blocks, not guards"
    - path: "trade-flow-api/src/auth/guards/permission.guard.ts"
      issue: "Throws domain ForbiddenError instead of NestJS ForbiddenException"
  missing:
    - "PermissionGuard should throw NestJS ForbiddenException (from @nestjs/common) instead of domain ForbiddenError, OR implement a global exception filter to convert domain errors to HTTP exceptions"
  debug_session: ""
