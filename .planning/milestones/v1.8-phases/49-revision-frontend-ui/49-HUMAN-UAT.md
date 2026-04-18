---
status: partial
phase: 49-revision-frontend-ui
source: [49-VERIFICATION.md]
started: 2026-04-16T00:00:00Z
updated: 2026-04-16T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Edit and resend button triggers navigation to new Draft revision
expected: After clicking the button, a new Draft revision is created via POST /v1/estimates/:id/revisions and the browser navigates to /estimates/{newId}
result: [pending]

### 2. History section collapses and expands, shows correct revision data
expected: Clicking the ChevronsUpDown trigger reveals the revision list with revision number, sent date, and viewed date; collapsed by default
result: [pending]

### 3. Edit and resend button hidden on Draft estimate
expected: Draft status shows Send Estimate and Delete buttons; no Edit and resend button visible
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
