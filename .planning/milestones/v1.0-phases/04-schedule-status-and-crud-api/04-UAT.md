---
status: resolved
phase: 04-schedule-status-and-crud-api
source: 04-01-SUMMARY.md, 04-02-SUMMARY.md
started: 2026-03-01T16:00:00Z
updated: 2026-03-01T17:00:00Z
---

## Current Test
<!-- OVERWRITE each test - shows where we are -->

[testing complete]

## Tests

### 1. Valid Status Transition
expected: POST transition endpoint with `{ "status": "Confirmed" }` on a Scheduled entry succeeds (200) and returns the schedule with status "Confirmed"
result: pass

### 2. Invalid Status Transition Rejected
expected: POST transition endpoint with `{ "status": "Completed" }` on a Scheduled entry returns an error response including the current status ("Scheduled") and valid next statuses ("Confirmed", "Canceled", "No-show")
result: pass

### 3. Terminal State is Final
expected: POST transition endpoint on a Completed, Canceled, or No-show schedule returns an error indicating no transitions are available (terminal state)
result: pass

### 4. Update Schedule Fields
expected: PATCH /v1/business/:businessId/job/:jobId/schedule/:scheduleId with fields like `{ "notes": "Updated note" }` succeeds on a Scheduled entry and returns the updated schedule
result: pass

### 5. Update Rejected on Terminal Status
expected: PATCH update endpoint on a Completed or Canceled schedule returns an error indicating updates are not allowed in that status
result: pass

### 6. Confirmed Auto-Reset on Time Change
expected: PATCH update with a new startDateTime or durationMinutes on a Confirmed schedule succeeds AND the returned schedule status is reset to "Scheduled"
result: pass

### 7. List Schedules by Job
expected: GET /v1/business/:businessId/job/:jobId/schedule returns a paginated list of schedules for that job, sorted chronologically (oldest first)
result: pass

### 8. List Schedules by Business
expected: GET /v1/business/:businessId/schedule returns a paginated list of all schedules across jobs for that business
result: pass

### 9. Filter by Status and Date Range
expected: GET list endpoint with query params `?status=Scheduled,Confirmed&from=2026-03-01&to=2026-03-31` returns only matching schedules
result: issue
reported: "The from and to filters appear to be working, but the status filter doesn't appear to be working. Additionally we need to use a structured and formal approach to filtering. We need to document the api standards for filtering. I want to use a colon separated approach that structures filters like so ?filter:<field>:<operation>=value. Where field is the field being filtered and if nested it will be . separated. The operator is a 2 character operator (eq, nq, lt, gt, bt, in). Where the bt operator has two comma separated values, and the in operator has one or more comma separated values."
severity: major

### 10. Get Schedule by ID
expected: GET /v1/business/:businessId/job/:jobId/schedule/:scheduleId returns the full schedule object with all fields including status
result: pass

## Summary

total: 10
passed: 9
issues: 1
pending: 0
skipped: 0

## Gaps

- truth: "List endpoints filter by status using query parameters"
  status: resolved
  reason: "User reported: Status filter doesn't work. Additionally, filtering needs a structured formal approach using colon-separated format: ?filter:<field>:<operation>=value with operators eq, nq, lt, gt, bt, in. This replaces the current ad-hoc ?status=...&from=...&to=... approach across all list endpoints."
  severity: major
  test: 9
  root_cause: "parseScheduleFilters in schedule.controller.ts validates status values case-sensitively against lowercase enum values (scheduled, no_show, etc.). Client-sent values like 'Scheduled' silently fail validation, resulting in an empty array that causes the status filter to be dropped entirely. Additionally, the entire ad-hoc filter approach (?status=...&from=...&to=...) needs to be replaced with a structured format: ?filter:<field>:<op>=value"
  artifacts:
    - path: "trade-flow-api/src/schedule/controllers/schedule.controller.ts"
      issue: "parseScheduleFilters case-sensitive validation silently drops mismatched values; entire ad-hoc approach needs replacement with structured filter format"
    - path: "trade-flow-api/src/schedule/data-transfer-objects/schedule-filter.dto.ts"
      issue: "Flat IScheduleFilterDto needs replacement with structured filter spec type"
    - path: "trade-flow-api/src/schedule/repositories/schedule.repository.ts"
      issue: "buildFilter per-field if-blocks need replacement with operator-mapped loop over structured filter specs"
  missing:
    - "Generic structured filter parser supporting filter:<field>:<op>=value query param format"
    - "Operator mapping: eq->$eq, nq->$ne, lt->$lt, gt->$gt, bt->$gte/$lte range, in->$in"
    - "Case-insensitive or normalized status value handling"
    - "API filtering standards documentation"
