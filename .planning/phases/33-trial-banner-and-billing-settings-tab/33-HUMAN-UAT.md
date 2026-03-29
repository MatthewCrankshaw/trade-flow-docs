---
status: partial
phase: 33-trial-banner-and-billing-settings-tab
source: [33-VERIFICATION.md]
started: 2026-03-29T20:35:00Z
updated: 2026-03-29T20:35:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. TrialChip visual and data binding
expected: Amber pill with Clock icon and "X days left in trial" appears in header for trialing user
result: [pending]

### 2. TrialChip click navigation
expected: Click navigates to /settings?tab=billing with Billing tab pre-selected
result: [pending]

### 3. TrialChip absent for active users
expected: No TrialChip visible for non-trialing users; PersistentCta also absent for active users
result: [pending]

### 4. Billing tab status display (trialing)
expected: Amber "Trial" badge, "Trial -- X days remaining" status line, "Manage Billing" button
result: [pending]

### 5. cancelAtPeriodEnd display (BILL-05)
expected: Orange "Cancels on [date]" badge and status line for cancelAtPeriodEnd=true subscriptions
result: [pending]

### 6. Manage Billing portal redirect
expected: Click "Manage Billing" redirects to Stripe Billing Portal via POST /v1/subscription/portal
result: [pending]

### 7. Stripe Portal return URL
expected: Returning from Stripe Portal lands on /settings?tab=billing with Billing tab pre-selected
result: [pending]

## Summary

total: 7
passed: 0
issues: 0
pending: 7
skipped: 0
blocked: 0

## Gaps
