---
status: partial
phase: 56-impersonation-backend-audit
source: [56-VERIFICATION.md]
started: 2026-04-19T20:30:00Z
updated: 2026-04-19T20:30:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. POST /v1/impersonation/start creates JWT and audit document
expected: Returns 200 with token, sessionId, expiresInMinutes: 30. An impersonation_audit document is created in MongoDB with status active.
result: [pending]

### 2. Impersonation token hydrates request.user as target user
expected: Subsequent endpoint call (e.g. GET /v1/business) responds as if the target customer user is authenticated. request.impersonator is set to the support user.
result: [pending]

### 3. Expired impersonation token returns descriptive 401
expected: Returns 401 with message "Impersonation session expired. Please start a new session."
result: [pending]

### 4. Ended session invalidates JWT immediately
expected: After POST /v1/impersonation/end, reusing the impersonation JWT returns 401 "Impersonation session is no longer active" because the guard checks audit entry status.
result: [pending]

## Summary

total: 4
passed: 0
issues: 0
pending: 4
skipped: 0
blocked: 0

## Gaps
