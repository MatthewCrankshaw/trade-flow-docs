---
phase: 54-user-management
plan: "04"
subsystem: ui
tags: [react, rtk-query, support-tools, pagination, filtering, url-state]

dependency_graph:
  requires:
    - phase: 54-02
      provides: GET /v1/support/users endpoint with search, role filter, subscription status filter, pagination
  provides:
    - useGetSupportUsersQuery RTK Query endpoint for support users
    - useSupportUsers hook with URL search param state management
    - UserListFilters, UserListTable, UserListPagination, SubscriptionBadge components
    - SupportUsersPage rewritten with real API data
  affects:
    - Plan 54-05 (dashboard metrics - SupportUsersPage already wired, ready for metric cards)

tech_stack:
  added: []
  patterns:
    - URL search params as filter state source of truth (useSearchParams)
    - Debounced search (300ms) via useRef timer
    - RTK Query injectEndpoints pattern (apiSlice.injectEndpoints)
    - CSS custom properties for semantic status badge colors
    - shadcn Badge variants for subscription status indicators

key_files:
  created:
    - trade-flow-ui/src/features/support/api/supportApi.ts
    - trade-flow-ui/src/features/support/api/index.ts
    - trade-flow-ui/src/features/support/hooks/useSupportUsers.ts
    - trade-flow-ui/src/features/support/components/SubscriptionBadge.tsx
    - trade-flow-ui/src/features/support/components/UserListFilters.tsx
    - trade-flow-ui/src/features/support/components/UserListTable.tsx
    - trade-flow-ui/src/features/support/components/UserListPagination.tsx
    - trade-flow-ui/src/features/support/index.ts
  modified:
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
    - trade-flow-ui/src/services/api.ts
    - trade-flow-ui/src/index.css
    - trade-flow-ui/src/components/ui/badge.variants.ts
    - trade-flow-ui/src/pages/support/SupportUsersPage.tsx

decisions:
  - "URL search params are the single source of truth for all filter state — browser back/forward preserves filter context per D-04 convention"
  - "Badge status colors use CSS custom properties (--status-trial, --status-active, --status-warning) with new badge variants, not raw Tailwind color classes, per shadcn skill rule"
  - "Debounce timer stored in useRef (not useState) to avoid re-renders while typing; cleanup in useEffect return"
  - "UserListPagination renders null when totalPages <= 1 to avoid showing unnecessary controls for small datasets"

metrics:
  duration: "~3 minutes"
  completed: "2026-04-19T07:21:03Z"
  tasks_completed: 2
  files_created: 8
  files_modified: 6
---

# Phase 54 Plan 04: Support User List Frontend Summary

**One-liner:** RTK Query support users endpoint, URL-param-driven filter/search/pagination hook, and full SupportUsersPage rewrite replacing mock data with real server-side filtered results.

## Tasks Completed

| # | Name | Commit (UI) | Status |
|---|------|------------|--------|
| 1 | Frontend types, RTK Query endpoints, and useSupportUsers hook | 5dc906d | Done |
| 2 | User list UI components and SupportUsersPage rewrite | ceddb5e | Done |

## What Was Built

### Task 1 — Types, API, and Hook

**PaginatedResponse<T>** (`src/types/api.types.ts`):
- Generic extension of `StandardResponse<T>` adding optional pagination metadata
- Exported from types barrel index

**SupportUser type** (`src/types/api.types.ts`):
- Full shape matching `ISupportUserResponse` from Plan 54-02 API
- Includes nested `subscription`, `business`, and `supportRoleIds` fields

**supportApi** (`src/features/support/api/supportApi.ts`):
- Injects `getSupportUsers` query into `apiSlice`
- Constructs URL with `page`, `limit`, `search`, `filter:role.type:eq`, `filter:subscription.status:eq` params
- `transformResponse` normalises `PaginatedResponse<SupportUser>` to `{ users, pagination }`
- Provides `SupportUser` and `LIST` cache tags

**useSupportUsers hook** (`src/features/support/hooks/useSupportUsers.ts`):
- All filter state sourced from URL search params via `useSearchParams`
- `searchInput` local state + 300ms debounce updates URL `search` param
- `setRoleFilter`, `setSubscriptionStatus`, `setPage` each update URL params and reset to page 1
- Exposes `totalPages`, `isLoading`, `isFetching`, `error`, `refetch`

### Task 2 — UI Components and Page

**SubscriptionBadge** (`src/features/support/components/SubscriptionBadge.tsx`):
- Uses shadcn `Badge` with `trial`, `active`, `warning`, `secondary` variants
- No raw Tailwind color classes — backed by CSS custom properties

**UserListFilters** (`src/features/support/components/UserListFilters.tsx`):
- Search `Input` with absolute-positioned `Search` icon (`pl-10` spacing)
- `Tabs` with "All", "Support", "Customer" role triggers
- `Select` with "All statuses", "Trialing", "Active", "Past Due", "Canceled", "Expired"
- Responsive: `flex-col md:flex-row` layout

**UserListTable** (`src/features/support/components/UserListTable.tsx`):
- Columns: Name, Email, Subscription (SubscriptionBadge), Created (Luxon DATE_MED)
- Loading state: 10 Skeleton rows
- Empty state: "No users found" heading + descriptive body in full-width cell
- Clickable rows: `onClick` navigates to `/support/users/:id`, `cursor-pointer hover:bg-muted/50`

**UserListPagination** (`src/features/support/components/UserListPagination.tsx`):
- Right-aligned "Previous" / "Page X of Y" / "Next" layout
- Hides when `totalPages <= 1`
- Boundary-disabled buttons

**SupportUsersPage** (`src/pages/support/SupportUsersPage.tsx`):
- All mock data arrays removed
- Wires `useSupportUsers` hook to `UserListFilters`, `UserListTable`, `UserListPagination`
- Error state: `Alert` with "Unable to load users. Please try again." and "Try Again" refetch button
- Table wrapped in `Card` with `p-0` `CardContent` for flush table edges

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. All data flows from the real `GET /v1/support/users` API endpoint. The clickable row navigation (`/support/users/:id`) links to a detail page that will be built in Plan 54-05.

## Threat Surface Scan

No new network endpoints introduced. The `supportApi` query constructs URL params from controlled filter values — server-side validation at the API layer (T-54-09, accepted). Page is protected by `SupportGuard` from Phase 53 (T-54-10, accepted).

## Self-Check

| Item | Status |
|------|--------|
| src/features/support/api/supportApi.ts | FOUND |
| src/features/support/hooks/useSupportUsers.ts | FOUND |
| src/features/support/components/SubscriptionBadge.tsx | FOUND |
| src/features/support/components/UserListFilters.tsx | FOUND |
| src/features/support/components/UserListTable.tsx | FOUND |
| src/features/support/components/UserListPagination.tsx | FOUND |
| src/pages/support/SupportUsersPage.tsx | FOUND (rewritten) |
| src/index.css contains --status-trial | FOUND |
| src/components/ui/badge.variants.ts contains trial/active/warning | FOUND |
| Commit 5dc906d (Task 1) | FOUND |
| Commit ceddb5e (Task 2) | FOUND |
| TypeScript: zero errors | PASSED |
| ESLint: zero errors | PASSED |

## Self-Check: PASSED
