---
status: complete
phase: 03-schedule-data-model-and-create-api
source: 03-01-SUMMARY.md, 03-02-SUMMARY.md
started: 2026-03-01T13:20:00Z
updated: 2026-03-01T13:28:00Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

[testing complete]

## Tests

### 1. Create a schedule with all fields
expected: POST /v1/business/:businessId/job/:jobId/schedule with startDateTime (ISO8601 UTC), durationMinutes, assigneeId, visitTypeId, and notes returns 201 with a ScheduleResponse containing all provided values and status "scheduled".
result: pass

### 2. Assignee defaults to authenticated user (SCHED-06)
expected: POST create schedule WITHOUT assigneeId in body. Response shows assigneeId equal to the authenticated user's ID.
result: pass

### 3. Duration defaults to 60 minutes (SCHED-07)
expected: POST create schedule WITHOUT durationMinutes in body. Response shows durationMinutes as 60.
result: pass

### 4. Invalid startDateTime rejected
expected: POST with invalid startDateTime (e.g., "not-a-date" or "March 1st") returns 400 validation error. Only valid ISO8601 strings are accepted.
result: pass

### 5. Duration must be divisible by 15
expected: POST with durationMinutes set to 37 (not divisible by 15) returns 400 validation error.
result: pass

### 6. Job must exist
expected: POST to /v1/business/:businessId/job/:nonExistentJobId/schedule returns error indicating job not found (SCHEDULE_0 error code).
result: pass

### 7. Visit type validation
expected: POST with a non-existent visitTypeId returns error indicating visit type not found (SCHEDULE_1 error code). POST without visitTypeId succeeds (visit type is optional).
result: pass

### 8. Unit tests pass
expected: Running npm test in trade-flow-api shows all schedule tests passing with 0 failures.
result: pass

## Summary

total: 8
passed: 8
issues: 0
pending: 0
skipped: 0

## Gaps

[none yet]
