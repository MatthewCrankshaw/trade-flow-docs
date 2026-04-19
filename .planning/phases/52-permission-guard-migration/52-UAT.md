---
status: complete
phase: 52-permission-guard-migration
source:
  - 52-01-SUMMARY.md
  - 52-02-SUMMARY.md
started: 2026-04-19T00:00:00Z
updated: 2026-04-19T12:00:00Z
---

## Current Test

[testing complete]

## Tests

### 1. CI passes with 855 tests
expected: Run `npm run ci` inside trade-flow-api. All tests pass (855 tests), zero ESLint errors, zero Prettier issues, zero TypeScript errors.
result: pass
note: warnings present in lint output (expected — warnings are not errors)

### 2. Old utility files are gone
expected: Neither `is-support-user.utility.ts` nor `is-super-user.utility.ts` exist in `trade-flow-api/src/user/utilities/`. Running `grep -r "isSupportUser\|isSuperUser" src/ --include="*.ts"` returns zero results.
result: issue
reported: "Both the files are present and the grep command returns results."
severity: major

### 3. hasPermission utility is in place
expected: `trade-flow-api/src/auth/utilities/has-permission.utility.ts` exists and exports `hasPermission`. `trade-flow-api/src/auth/utilities/has-any-permission.utility.ts` exists and exports `hasAnyPermission`.
result: pass

### 4. PermissionGuard and decorator are registered
expected: `trade-flow-api/src/auth/guards/permission.guard.ts` exists. `trade-flow-api/src/auth/decorators/requires-permission.decorator.ts` exists. Both are listed in AuthModule `providers` and `exports`.
result: pass

### 5. All policies use hasPermission (no isSupportUser/isSuperUser calls)
expected: Running `grep -r "isSupportUser\|isSuperUser\|is-support-user\|is-super-user" src/ --include="*.ts"` inside trade-flow-api returns zero results. Running `grep -r "hasPermission" src/ --include="*.ts" | grep -v "test/" | grep -v ".spec.ts" | wc -l` returns a count > 0 (confirming the migration happened).
result: pass

### 6. SubscriptionGuard uses permission-based bypass
expected: Opening `trade-flow-api/src/subscription/guards/subscription.guard.ts` shows `hasPermission(user, "bypass_subscription")` where the old `supportRoles.length > 0` check used to be. No `supportRoles.length` check remains in this file.
result: pass

## Summary

total: 6
passed: 5
issues: 1
pending: 0
skipped: 0

## Gaps

- truth: "Neither is-support-user.utility.ts nor is-super-user.utility.ts exist in trade-flow-api/src/user/utilities/ and no references to isSupportUser/isSuperUser remain in the codebase"
  status: failed
  reason: "User reported: Both the files are present and the grep command returns results."
  severity: major
  test: 2
  artifacts: []
  missing: []
