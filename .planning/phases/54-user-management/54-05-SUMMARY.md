---
phase: 54-user-management
plan: "05"
subsystem: ui
tags: [react, rtk-query, support-tools, user-detail, dashboard-metrics, routing]

dependency_graph:
  requires:
    - phase: 54-03
      provides: GET /v1/support/users/:id, GET /v1/support/dashboard/metrics
    - phase: 54-04
      provides: supportApi RTK Query base, SupportUsersPage, useSupportUsers hook
  provides:
    - SupportUserDetailPage with 4-card layout (Profile, Business, Subscription, Roles)
    - MembershipMetrics component with 5 clickable metric cards
    - RoleBadge component for role display
    - useGetSupportUserDetailQuery RTK Query endpoint
    - useGetDashboardMetricsQuery RTK Query endpoint
    - Route /support/users/:id registered in App.tsx
  affects:
    - Plan 55-02 (role actions integrated into user detail page)
    - SupportDashboardPage (metric cards added above quick links)

tech_stack:
  added: []
  patterns:
    - Clickable metric cards as Link components with filter query params
    - 4-card grid layout with md:grid-cols-2 for user detail sections
    - 5-card grid layout with md:grid-cols-5 for dashboard metrics

key_files:
  created:
    - trade-flow-ui/src/features/support/components/MembershipMetrics.tsx
    - trade-flow-ui/src/features/support/components/RoleBadge.tsx
  modified:
    - trade-flow-ui/src/features/support/api/supportApi.ts
    - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx
    - trade-flow-ui/src/pages/support/SupportDashboardPage.tsx
    - trade-flow-ui/src/App.tsx
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
    - trade-flow-ui/src/features/support/index.ts
    - trade-flow-ui/src/features/support/api/index.ts

decisions:
  - "MembershipMetrics filter links use the established query convention (filter:subscription.status:eq=value) for URL param state continuity with the user list page"
  - "Removed Firebase Admin SDK metadata (creationTime/lastSignInTime) during verification — credentials not configured and not needed"
  - "Expired metric counts incomplete subscriptions only (not incomplete_expired which is not a valid SubscriptionStatus)"
  - "Trade field converted from MongoDB object {primaryTrade, customTradeName} to display string in repository mapBusiness"
  - "Route path uses :id parameter to match NestJS controller @Param('id') convention"

metrics:
  duration: "~2 minutes"
  completed: "2026-04-19T11:12:00Z"
  tasks_completed: 1
  files_created: 2
  files_modified: 8
---

# Phase 54 Plan 05: User Detail Page, Dashboard Metrics, and Route Registration Summary

**RTK Query endpoints for user detail and dashboard metrics, MembershipMetrics and RoleBadge components, SupportUserDetailPage with 4-card layout, and route registration at /support/users/:id.**

## Tasks Completed

| # | Name | Commit (UI) | Status |
|---|------|------------|--------|
| 1 | RTK Query endpoints, types, components, pages, and route | 196ba7b | Done |
| 2 | Visual verification | -- | Checkpoint |

## What Was Built

### Task 1 -- Full UI Integration

**Types** (`src/types/api.types.ts`):
- `DashboardMetrics` interface with 5 count fields (totalUsers, activeTrials, activeSubscriptions, expiredSubscriptions, canceledSubscriptions)
- `SupportUserDetail` already existed from Plan 54-04

**RTK Query Endpoints** (`src/features/support/api/supportApi.ts`):
- `getSupportUserDetail` -- fetches single user detail, transforms StandardResponse
- `getDashboardMetrics` -- fetches membership counts, transforms StandardResponse

**RoleBadge** (`src/features/support/components/RoleBadge.tsx`):
- Maps role name to badge variant (super_user = default, others = secondary)
- Replaces underscores with spaces for display
- Null guard for safety (added in 54-07 gap closure)

**MembershipMetrics** (`src/features/support/components/MembershipMetrics.tsx`):
- 5 clickable cards in `grid gap-4 md:grid-cols-5`
- Each card links to `/support/users` with appropriate filter query param
- Icons: Users, Clock, CreditCard, XCircle, Ban from lucide-react
- Loading state: 5 Skeleton cards
- Error state: Alert with "Try Again" button

**SupportUserDetailPage** (`src/pages/support/SupportUserDetailPage.tsx`):
- Back button with ArrowLeft icon linking to `/support/users`
- Header with user name, email, and inline RoleBadge components
- Profile card: Display Name, Email
- Business card: Business Name, Trade Type, or "No business associated"
- Subscription card: SubscriptionBadge, Status, Trial Start, Trial End, Current Period End, Canceled At (conditional)
- Roles card: RoleBadge list or "No support roles assigned"
- Loading state: Skeleton placeholders for all 4 cards
- Error state: 404 redirects to user list with toast; other errors show Alert with "Try Again"

**SupportDashboardPage** (`src/pages/support/SupportDashboardPage.tsx`):
- MembershipMetrics added above Quick Links section

**App.tsx**:
- Route `/support/users/:id` registered inside SupportGuard branch

**Barrel exports** updated in `src/features/support/index.ts` and `src/features/support/api/index.ts`.

## Deviations from Plan

- Removed Firebase Admin SDK metadata (creationTime/lastSignInTime) from user detail -- credentials not configured and feature deemed unnecessary
- Fixed trade field rendering: MongoDB stores trade as object {primaryTrade, customTradeName}, repository now converts to display string
- Fixed expired metric: uses `incomplete` status (valid enum value) instead of nonexistent `expired` status
- Added `in` operator support to subscription status filter in repository

## Known Stubs

None. All data flows from real API endpoints:
- `GET /v1/support/users/:id` for user detail
- `GET /v1/support/dashboard/metrics` for membership counts

## Threat Surface Scan

No new network endpoints introduced by the frontend. Client-side changes consume existing API endpoints protected by JwtAuthGuard + PermissionGuard(manage_users) (T-54-11 mitigated). Filter params in MembershipMetrics links are hardcoded client-side (T-54-12 accepted).

## Self-Check: PASSED

| Item | Status |
|------|--------|
| MembershipMetrics.tsx | FOUND |
| RoleBadge.tsx | FOUND |
| supportApi.ts contains getSupportUserDetail | FOUND |
| supportApi.ts contains getDashboardMetrics | FOUND |
| SupportUserDetailPage.tsx | FOUND |
| SupportDashboardPage.tsx contains MembershipMetrics | FOUND |
| App.tsx contains support/users/:id | FOUND |
| api.types.ts contains DashboardMetrics | FOUND |
| Commit 196ba7b (Task 1) | FOUND |
| TypeScript: zero errors | PASSED |
| CI: passes | PASSED |
