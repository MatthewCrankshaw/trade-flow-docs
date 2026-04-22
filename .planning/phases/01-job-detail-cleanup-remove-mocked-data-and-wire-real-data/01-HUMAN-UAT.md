---
status: partial
phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
source: [01-VERIFICATION.md]
started: 2026-04-22T08:00:00.000Z
updated: 2026-04-22T08:00:00.000Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Customer name rendering
expected: Navigate to a job detail page; confirm the customer name is real (from the API), not a hardcoded value
result: [pending]

### 2. Timeline shows events
expected: Create a new quote on a job; confirm the timeline shows "Quote created" with a relative timestamp
result: [pending]

### 3. Timeline auto-refresh
expected: Send a quote; confirm the "Quote sent" event appears in the timeline without a page reload (RTK Query cache invalidation)
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
