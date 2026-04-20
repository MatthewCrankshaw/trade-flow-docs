---
status: diagnosed
phase: 57-impersonation-frontend
source: [57-VERIFICATION.md]
started: 2026-04-20T08:20:00Z
updated: 2026-04-20T20:40:00Z
---

## Current Test

[awaiting re-test after fixes]

## Tests

### 1. Full impersonation flow (re-test after Plan 04 fix)
expected: Amber banner shows "Impersonating: {user name}" instead of white space. Sidebar switches to customer navigation. Banner stays fixed while scrolling and above modals.
result: issue — banner renders but POST/PATCH requests during impersonation fail with 500 (SubscriptionGuard rejects before JwtAuthGuard runs). Fixed: SubscriptionGuard now defers when no user present.

### 2. Return to Support button flow (re-test)
expected: Click "Return to Support" during active impersonation. Navigates to /support, banner disappears, toast shows "Impersonation session ended".
result: issue — toast shows "Session end request failed" because prepareHeaders attaches impersonation token (HS256) but endpoint requires Firebase token (RS256). Fixed: dispatch endImpersonation() before API call so prepareHeaders uses Firebase token.

## Summary

total: 2
passed: 0
issues: 2
pending: 2
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
