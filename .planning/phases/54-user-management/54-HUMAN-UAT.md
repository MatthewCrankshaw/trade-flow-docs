---
status: diagnosed
phase: 54-user-management
source: [54-VERIFICATION.md]
started: 2026-04-19T14:50:00.000Z
updated: 2026-04-19T14:55:00.000Z
---

## Current Test

[testing complete]

## Tests

### 1. User List 6-Column Layout
expected: All 6 columns render correctly — Name, Email, Role (badge for support users, dash otherwise), Subscription (badge), Business (name or dash), Created
result: failed — Created column always shows "---" because createdAt is hardcoded to null

### 2. Dashboard Metric Card Navigation
expected: Clicking each of the 5 metric cards navigates to /support/users with the correct pre-applied filter
result: passed

### 3. User Detail Data Accuracy
expected: Clicking into a user detail page shows Profile, Business, Subscription, and Roles cards with correct data
result: passed

## Summary

total: 3
passed: 2
issues: 1
pending: 0
skipped: 0
blocked: 0

## Gaps

### Gap 1: User createdAt always null
status: failed
severity: medium
description: The Created column in UserListTable always shows "---" because the user's createdAt is hardcoded to null in the controller's mapToResponse method. The users collection stores createdAt but the support aggregation pipeline, DTO, and response don't thread it through.
root_cause: ISupportUserDto has no createdAt field. SupportUserAggregateResult doesn't project it. Controller hardcodes createdAt: null.
fix_approach: Add createdAt to the aggregate result interface, project it in the pipeline, add to DTO, map in repository, and pass through controller response.
