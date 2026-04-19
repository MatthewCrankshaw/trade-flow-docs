---
status: partial
phase: 57-impersonation-frontend
source: [57-VERIFICATION.md]
started: 2026-04-19T18:30:00Z
updated: 2026-04-19T18:30:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Full impersonation flow end-to-end
expected: Navigate to /support/users/{customerId} for a customer user, click Impersonate User, enter reason, click Start Impersonation — amber banner appears at top of viewport with target user name and Return to Support button; sidebar switches to customer navigation (Jobs, Customers, Quotes, etc.); banner stays fixed while scrolling and above modals
result: [pending]

### 2. Return to Support button flow
expected: Click Return to Support during active impersonation — navigates to /support, banner disappears, toast shows "Impersonation session ended"
result: [pending]

### 3. Button visibility gating on user detail page
expected: Impersonate User button is visible for customer users (no support roles), hidden entirely for support users (supportRoleIds.length > 0). Button is also hidden when impersonation is already active.
result: [pending]

### 4. Empty reason validation
expected: Clicking Start Impersonation with empty reason textarea shows inline error "A reason is required before impersonating a user." below the textarea; dialog stays open; no API call is made
result: [pending]

### 5. Page refresh during impersonation
expected: Refreshing the page terminates the impersonation session (Redux state is in-memory only, not persisted); support user returns to their own context without amber banner
result: [pending]

## Summary

total: 5
passed: 0
issues: 0
pending: 5
skipped: 0
blocked: 0

## Gaps
