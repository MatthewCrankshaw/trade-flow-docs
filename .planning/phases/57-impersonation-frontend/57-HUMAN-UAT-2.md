---
status: passed
phase: 57-impersonation-frontend
source: [57-VERIFICATION.md]
started: 2026-04-20T08:20:00Z
updated: 2026-04-20T21:00:00Z
---

## Current Test

[complete]

## Tests

### 1. Full impersonation flow (re-test after Plan 04 fix)
expected: Amber banner shows "Impersonating: {user name}" instead of white space. Sidebar switches to customer navigation. Banner stays fixed while scrolling and above modals.
result: pass — banner renders correctly, POST/PATCH requests work after SubscriptionGuard fix.

### 2. Return to Support button flow (re-test)
expected: Click "Return to Support" during active impersonation. Navigates to /support, banner disappears, toast shows "Impersonation session ended".
result: pass — toast shows success message, navigation works correctly after token fix.

## Summary

total: 2
passed: 2
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

### 1. SubscriptionGuard blocks impersonated POST requests
status: fixed
root_cause: Global APP_GUARD executes before route-level JwtAuthGuard. request.user is undefined at SubscriptionGuard execution time.
fix: Guard now returns true when user is undefined, deferring auth to JwtAuthGuard.
commit: f40c221 (trade-flow-api)

### 2. End impersonation uses wrong token type
status: fixed
root_cause: prepareHeaders attaches HS256 impersonation token but /v1/impersonation/end requires RS256 Firebase JWT via JwtAuthGuard.
fix: Dispatch endImpersonation() before calling endMutation so prepareHeaders falls through to Firebase token path.
commit: 5e6afc9 (trade-flow-ui)
