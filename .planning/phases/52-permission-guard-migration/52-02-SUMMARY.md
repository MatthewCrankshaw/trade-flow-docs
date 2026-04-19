---
phase: 52-permission-guard-migration
plan: "02"
subsystem: trade-flow-api
tags: [auth, rbac, permissions, policy, guard, migration]
dependency_graph:
  requires: [52-01]
  provides: [permission-based policy checks, permission-based subscription bypass]
  affects:
    - trade-flow-api/src/subscription/guards/subscription.guard.ts
    - trade-flow-api/src/business/policies/business.policy.ts
    - trade-flow-api/src/customer/policies/customer.policy.ts
    - trade-flow-api/src/job/policies/job.policy.ts
    - trade-flow-api/src/job/policies/job-type.policy.ts
    - trade-flow-api/src/quote/policies/quote.policy.ts
    - trade-flow-api/src/quote/policies/quote-line-item.policy.ts
    - trade-flow-api/src/quote-settings/policies/quote-settings.policy.ts
    - trade-flow-api/src/schedule/policies/schedule.policy.ts
    - trade-flow-api/src/visit-type/policies/visit-type.policy.ts
    - trade-flow-api/src/tax-rate/policies/tax-rate.policy.ts
    - trade-flow-api/src/item/policies/item.policy.ts
    - trade-flow-api/src/estimate/policies/estimate.policy.ts
    - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
    - trade-flow-api/src/estimate-settings/policies/estimate-settings.policy.ts
    - trade-flow-api/src/user/policies/support-role.policy.ts
    - trade-flow-api/src/migration/policies/migration.policy.ts
tech_stack:
  added: []
  patterns: [hasPermission permission check, permission-carrying role mocks in tests]
key_files:
  created: []
  modified:
    - trade-flow-api/src/subscription/guards/subscription.guard.ts
    - trade-flow-api/src/business/policies/business.policy.ts
    - trade-flow-api/src/customer/policies/customer.policy.ts
    - trade-flow-api/src/job/policies/job.policy.ts
    - trade-flow-api/src/job/policies/job-type.policy.ts
    - trade-flow-api/src/quote/policies/quote.policy.ts
    - trade-flow-api/src/quote/policies/quote-line-item.policy.ts
    - trade-flow-api/src/quote-settings/policies/quote-settings.policy.ts
    - trade-flow-api/src/schedule/policies/schedule.policy.ts
    - trade-flow-api/src/visit-type/policies/visit-type.policy.ts
    - trade-flow-api/src/tax-rate/policies/tax-rate.policy.ts
    - trade-flow-api/src/item/policies/item.policy.ts
    - trade-flow-api/src/estimate/policies/estimate.policy.ts
    - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
    - trade-flow-api/src/estimate-settings/policies/estimate-settings.policy.ts
    - trade-flow-api/src/user/policies/support-role.policy.ts
    - trade-flow-api/src/migration/policies/migration.policy.ts
    - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
    - trade-flow-api/src/business/test/policies/business.policy.spec.ts
    - trade-flow-api/src/schedule/test/policies/schedule.policy.spec.ts
    - trade-flow-api/src/visit-type/test/policies/visit-type.policy.spec.ts
  deleted:
    - trade-flow-api/src/user/utilities/is-support-user.utility.ts
    - trade-flow-api/src/user/utilities/is-super-user.utility.ts
decisions:
  - All 15 isSupportUser() call sites replaced with hasPermission(authUser, "view_support_dashboard") -- semantically equivalent because both Super User and Support Administrator roles have this permission
  - MigrationPolicy isSuperUser() replaced with hasPermission(authUser, "manage_migrations") -- only Super User role has this permission, preserving restricted access
  - SubscriptionGuard supportRoles.length > 0 replaced with hasPermission(user, "bypass_subscription") -- fail-closed because hasPermission returns false when permissions field missing or empty
  - Old utility files deleted after all call sites migrated; TypeScript compilation would immediately catch any missed import
  - Negative test added to SubscriptionGuard spec: user with supportRoles but without bypass_subscription permission does NOT bypass subscription check
metrics:
  duration: 15 minutes
  completed: "2026-04-19"
  tasks_completed: 2
  files_changed: 21
---

# Phase 52 Plan 02: Permission Guard Migration Summary

Complete RBAC migration: all 17 hardcoded role-name checks replaced with `hasPermission()` calls (`bypass_subscription`, `view_support_dashboard`, `manage_migrations`), `isSupportUser` and `isSuperUser` utilities deleted, and all policy test mocks updated to carry permission-bearing role objects.

## What Was Built

### Task 1: Migrate SubscriptionGuard and 15 isSupportUser policy files

**`src/subscription/guards/subscription.guard.ts`**
- Replaced `user.supportRoles && user.supportRoles.length > 0` with `hasPermission(user, "bypass_subscription")`
- Added import from `@auth/utilities/has-permission.utility`
- Guard is now fail-closed: if a support role has no hydrated permissions, bypass is denied

**15 policy files** (business, customer, job, job-type, quote, quote-line-item, quote-settings, schedule, visit-type, tax-rate, item, estimate, estimate-line-item, estimate-settings, support-role):
- Each: removed `import { isSupportUser } from "@user/utilities/is-support-user.utility"`
- Each: added `import { hasPermission } from "@auth/utilities/has-permission.utility"`
- Each: replaced all `isSupportUser(authUser)` calls with `hasPermission(authUser, "view_support_dashboard")`
- Zero other logic changes — surrounding if-block and return patterns unchanged

**Verification:** `grep -r "is-support-user" src/ --include="*.ts" | grep -v ".spec.ts" | grep -v "test/"` returned zero results. `npx tsc --noEmit` exited 0.

### Task 2: Migrate MigrationPolicy, delete old utilities, update all tests

**`src/migration/policies/migration.policy.ts`**
- Replaced `import { isSuperUser } from "@user/utilities/is-super-user.utility"` with `import { hasPermission } from "@auth/utilities/has-permission.utility"`
- Replaced all 4 `isSuperUser(authUser)` calls with `hasPermission(authUser, "manage_migrations")` in canCreate, canRead, canUpdate, canDelete

**Deleted files:**
- `src/user/utilities/is-support-user.utility.ts` — removed entirely
- `src/user/utilities/is-super-user.utility.ts` — removed entirely

**`src/subscription/test/guards/subscription.guard.spec.ts`**
- Updated support user mock to carry `bypass_subscription` permission on the support role's `permissions: DtoCollection.create([...])`
- Added negative test: support user WITHOUT `bypass_subscription` permission does NOT bypass subscription check and receives ForbiddenError

**`src/business/test/policies/business.policy.spec.ts`**
- Updated support user mock in canRead test to carry `view_support_dashboard` permission

**`src/schedule/test/policies/schedule.policy.spec.ts`**
- Updated 3 support user mocks (canCreate, canRead, canUpdate) to carry `view_support_dashboard` permission

**`src/visit-type/test/policies/visit-type.policy.spec.ts`**
- Updated 3 support user mocks (canCreate, canRead, canUpdate) to carry `view_support_dashboard` permission

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated policy test mocks missing permission field**
- **Found during:** Task 2 — analysis before running tests
- **Issue:** business.policy.spec.ts, schedule.policy.spec.ts, and visit-type.policy.spec.ts all created support user mocks with `DtoCollection.create([{ id: "support-role-id", roleName: "super_user" }])` — no `permissions` field. After migration, `hasPermission` checks `role.permissions` which would return `false` for these mocks, breaking all "support user can access resource" test assertions.
- **Fix:** Added `permissions: DtoCollection.create<IPermissionDto>([{ id: "perm-1", name: "view_support_dashboard", ... }])` to the support role in each affected mock. Added `import type { IPermissionDto }` to each spec file.
- **Files modified:** business.policy.spec.ts, schedule.policy.spec.ts, visit-type.policy.spec.ts
- **Commit:** 6dbd7c2

## Verification

```
grep -r "is-support-user.utility|is-super-user.utility|isSupportUser|isSuperUser" src/ --include="*.ts"
# Result: zero matches

grep -r "hasPermission" src/ --include="*.ts" | grep -v "test/" | grep -v ".spec.ts" | wc -l
# Result: 63

npm run ci
# Test Suites: 100 passed, 100 total
# Tests: 855 passed, 855 total
# 0 ESLint errors, 0 Prettier issues, 0 TypeScript errors
```

## Commits

| Task | Commit | Message |
|------|--------|---------|
| Task 1 | f6901f8 | feat(52-02): migrate SubscriptionGuard and 15 policy files to hasPermission |
| Task 2 | 6dbd7c2 | feat(52-02): migrate MigrationPolicy, delete old utilities, update tests |

## Threat Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| T-52-05 Elevation of Privilege | All 15 isSupportUser() calls replaced with hasPermission(user, "view_support_dashboard") — fail-closed when permissions not hydrated |
| T-52-06 Elevation of Privilege | isSuperUser() in MigrationPolicy replaced with hasPermission(user, "manage_migrations") — Admin role does NOT have this permission |
| T-52-07 Denial of Service | Subscription bypass now permission-checked; fail-closed behavior verified by negative test |
| T-52-08 Elevation of Privilege | Old utilities deleted after all call sites migrated; TypeScript compilation catches any missed import at build time |

## Self-Check: PASSED

Files deleted:
- `trade-flow-api/src/user/utilities/is-support-user.utility.ts` — DELETED (confirmed via git log: delete mode 100644)
- `trade-flow-api/src/user/utilities/is-super-user.utility.ts` — DELETED (confirmed via git log: delete mode 100644)

Commits exist:
- f6901f8: feat(52-02): migrate SubscriptionGuard and 15 policy files to hasPermission — FOUND
- 6dbd7c2: feat(52-02): migrate MigrationPolicy, delete old utilities, update tests — FOUND

CI: 855 tests passed, 0 errors — PASSED
