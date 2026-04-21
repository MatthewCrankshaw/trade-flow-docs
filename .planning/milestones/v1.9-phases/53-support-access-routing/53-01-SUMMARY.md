---
phase: 53-support-access-routing
plan: 01
subsystem: frontend-auth
tags: [route-guard, navigation, support-access]
dependency_graph:
  requires: []
  provides: [SupportGuard, getSupportNavigationItems, getBusinessNavigationItems]
  affects: [DashboardLayout, App.tsx]
tech_stack:
  added: []
  patterns: [route-guard, role-conditional-navigation]
key_files:
  created:
    - trade-flow-ui/src/features/auth/components/SupportGuard.tsx
  modified:
    - trade-flow-ui/src/features/auth/components/index.ts
    - trade-flow-ui/src/config/navigation.ts
decisions:
  - Kept getNavigationItems as backward-compatible alias delegating to getBusinessNavigationItems to avoid breaking DashboardLayout before Plan 02 migrates it
  - Used underscore prefix on unused user parameter in getBusinessNavigationItems since the function no longer needs user data but maintains compatible signature
metrics:
  duration: 100s
  completed: 2026-04-19T06:50:06Z
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 2
---

# Phase 53 Plan 01: SupportGuard and Navigation Config Summary

SupportGuard route guard with loading/redirect states following OnboardingGuard pattern, plus navigation config split into getSupportNavigationItems() and getBusinessNavigationItems() for role-conditional sidebar rendering.

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create SupportGuard component and update barrel export | bb65550 (trade-flow-ui) | SupportGuard.tsx, index.ts |
| 2 | Refactor navigation.ts into role-specific functions | 4b39774 (trade-flow-ui) | navigation.ts |

## What Was Built

### SupportGuard Component
- Checks `user.supportRoles.length === 0` to determine support access
- Shows full-screen centered Loader2 spinner while `useGetCurrentUserQuery` loads
- Redirects non-support users to `/dashboard` via `<Navigate to="/dashboard" replace />`
- Renders `<Outlet />` for authenticated support users
- Follows exact OnboardingGuard loading/guard/redirect pattern per D-04

### Navigation Config Refactor
- `getSupportNavigationItems()` returns a single Support section with Dashboard (/support), Users (/support/users), Businesses (/support/businesses)
- `getBusinessNavigationItems()` returns Main and Business sections (existing business nav without support items)
- `getNavigationItems()` retained as backward-compatible alias delegating to `getBusinessNavigationItems()` -- Plan 02 will remove it when updating DashboardLayout
- Icons per UI-SPEC: LayoutDashboard, Users, Building2 (removed HelpCircle)
- Titles per UI-SPEC copywriting: "Dashboard", "Users", "Businesses" (not prefixed)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Kept getNavigationItems as backward-compatible alias**
- **Found during:** Task 2
- **Issue:** Removing getNavigationItems would break DashboardLayout TypeScript compilation, violating CI Gate Policy
- **Fix:** Kept getNavigationItems as a thin wrapper calling getBusinessNavigationItems, allowing clean compilation
- **Files modified:** trade-flow-ui/src/config/navigation.ts
- **Commit:** 4b39774

## Verification

- TypeScript compilation passes (`npx tsc --noEmit` -- zero errors)
- SupportGuard exports named function component
- navigation.ts exports both getSupportNavigationItems and getBusinessNavigationItems
- Barrel export includes SupportGuard
