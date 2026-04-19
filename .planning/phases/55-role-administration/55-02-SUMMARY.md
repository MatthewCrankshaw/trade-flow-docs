---
phase: 55-role-administration
plan: "02"
subsystem: trade-flow-ui
tags: [rbac, role-administration, rtk-query, alert-dialog, support-ui, react]
dependency_graph:
  requires: [55-01]
  provides:
    - grantSupportRole RTK Query mutation (POST /v1/users/:id/support-role)
    - revokeSupportRole RTK Query mutation (DELETE /v1/users/:id/support-role)
    - getSupportUserById RTK Query query (GET /v1/support/users/:id)
    - RoleActions component with super-user-only visibility and grant/revoke button logic
    - GrantRoleDialog AlertDialog with confirmation copy and loading state
    - RevokeRoleDialog AlertDialog with destructive styling and differentiated error messages
    - SupportUserDetailPage with account, subscription, and roles cards
    - /support/users/:userId route under SupportGuard
  affects:
    - trade-flow-ui/src/services/userApi.ts (no change — mutations in supportApi)
    - trade-flow-ui/src/features/support/api/supportApi.ts
    - trade-flow-ui/src/features/support/api/index.ts
    - trade-flow-ui/src/features/support/components/RoleActions.tsx
    - trade-flow-ui/src/features/support/components/GrantRoleDialog.tsx
    - trade-flow-ui/src/features/support/components/RevokeRoleDialog.tsx
    - trade-flow-ui/src/features/support/hooks/useSupportUsers.ts
    - trade-flow-ui/src/features/support/index.ts
    - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx
    - trade-flow-ui/src/App.tsx
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
tech_stack:
  added: []
  patterns:
    - RTK Query mutations with per-resource tag invalidation (SupportUser id + LIST)
    - Controlled AlertDialog (open/onOpenChange) for confirmation flows
    - Differentiated error message resolution from RTK Query error.data.errors[0].details
    - RoleActions visibility computed from viewer.supportRoles (User) and targetUser.supportRoleNames (SupportUserDetail)
    - SupportUserDetail type with supportRoleNames string[] matching backend ISupportUserDetailResponse
key_files:
  created:
    - trade-flow-ui/src/features/support/components/GrantRoleDialog.tsx
    - trade-flow-ui/src/features/support/components/RevokeRoleDialog.tsx
    - trade-flow-ui/src/features/support/components/RoleActions.tsx
    - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx
  modified:
    - trade-flow-ui/src/features/support/api/supportApi.ts
    - trade-flow-ui/src/features/support/api/index.ts
    - trade-flow-ui/src/features/support/hooks/useSupportUsers.ts
    - trade-flow-ui/src/features/support/index.ts
    - trade-flow-ui/src/App.tsx
    - trade-flow-ui/src/types/api.types.ts
    - trade-flow-ui/src/types/index.ts
decisions:
  - Mutations placed in supportApi (not userApi) since the support feature owns the context for these user admin operations
  - Cache invalidation uses SupportUser tag (not User/UserList) since the list view uses SupportUser tags
  - RoleActions receives User (viewer) and SupportUserDetail (target) as distinct types — avoids forcing a single shape onto two different API responses
  - SupportUserDetail.supportRoleNames string[] (not SupportRole[]) matches the GET /v1/support/users/:id response shape (ISupportUserDetailResponse)
  - SupportUserDetailPage created as Rule 3 fix — plan referenced it as existing but it was never built
  - Route parameter is :userId (not :id) to distinguish from other detail routes
metrics:
  duration: "~20 minutes"
  completed: "2026-04-19T08:30:00Z"
  tasks_completed: 2
  files_created: 4
  files_modified: 7
---

# Phase 55 Plan 02: Role Administration Frontend Summary

**One-liner:** RTK Query grant/revoke mutations with SupportUser cache invalidation, RoleActions component with super-user-only visibility, GrantRoleDialog and RevokeRoleDialog with UI-SPEC copy, and SupportUserDetailPage wiring them together under `/support/users/:userId`.

## What Was Built

### Task 1: RTK Query mutations and role action components

**`src/features/support/api/supportApi.ts`**
- Added `getSupportUserById` query: `GET /v1/support/users/:id` returning `SupportUserDetail`
- Added `grantSupportRole` mutation: `POST /v1/users/:id/support-role`, invalidates `[SupportUser:id, SupportUser:LIST]`
- Added `revokeSupportRole` mutation: `DELETE /v1/users/:id/support-role`, invalidates `[SupportUser:id, SupportUser:LIST]`
- Both mutations `transformResponse` to extract `response.data[0]`

**`src/types/api.types.ts`**
- Added `SupportUserDetail` interface with `supportRoleIds: string[]`, `supportRoleNames: string[]`, and `firebaseMetadata` — matches `ISupportUserDetailResponse` from the backend

**`src/features/support/components/GrantRoleDialog.tsx`**
- Controlled `AlertDialog` (open/onOpenChange) — title "Grant Admin Role", description with target user name
- Cancel: "Keep Current Role" | Action: "Grant Role" (default variant)
- Loading state disables both buttons and shows `Loader2` spinner in action button
- Success: `toast.success("Admin role granted to {name}")` | Error: generic error toast

**`src/features/support/components/RevokeRoleDialog.tsx`**
- Same controlled pattern as GrantRoleDialog
- Cancel: "Keep Admin Role" | Action: "Revoke Role" (`buttonVariants({ variant: "destructive" })`)
- Error parsing via `error.data.errors[0].details` — differentiates `self_revocation`, `last_admin_protection`, and generic errors
- Three distinct error toasts per UI-SPEC copywriting contract

**`src/features/support/components/RoleActions.tsx`**
- Returns `null` if viewer is not a super user (reads `viewer.supportRoles[].roleName === "super_user"`)
- Shows Grant button when `targetUser.supportRoleNames` is empty (customer user)
- Shows Revoke button when target is `"support_administrator"` AND not viewing self AND not a super user
- No button when viewing a super user profile or own profile (defense in depth per D-10, D-12)
- Both buttons have `aria-label` with the target user's name for accessibility

### Task 2: SupportUserDetailPage integration

**`src/pages/support/SupportUserDetailPage.tsx`** (new — Rule 3 fix)
- Account card: email, Firebase creation time, last sign-in time
- Subscription card: status badge, business name and trade
- Roles card: role badges for each `supportRoleNames` entry, `RoleActions` below (shows for super users only)
- Uses `useGetSupportUserByIdQuery` for target user, `useGetCurrentUserQuery` for viewer
- Loading skeleton and error state with retry button

**`src/App.tsx`**
- Added `/support/users/:userId` route under `SupportGuard` — accessible only to support users

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking Issue] Created SupportUserDetailPage — page referenced but never built**
- **Found during:** Task 2
- **Issue:** The plan described integrating `RoleActions` into "the existing Roles card on SupportUserDetailPage" but `SupportUserDetailPage.tsx` did not exist. Phase 54 (in this project) only built API query utilities; the UI detail page was never created.
- **Fix:** Created `SupportUserDetailPage.tsx` with account, subscription, and roles cards. Added `/support/users/:userId` route to `App.tsx`.
- **Files created:** `SupportUserDetailPage.tsx`
- **Files modified:** `App.tsx`

**2. [Rule 1 - Bug] Fixed three pre-existing TypeScript errors in support hooks**
- **Found during:** Task 2 CI gate
- **Issue:** `useSupportUsers.ts` used `useRef<ReturnType<typeof setTimeout>>()` (0 args, needs 1 with strict mode). `setRoleFilter` and `setSubscriptionStatus` returned callbacks typed as `(tab: RoleTab) => void` / `(status: SubscriptionFilter) => void` — incompatible with `UserListFilters` props that expect `(tab: string) => void`.
- **Fix:** Added `undefined` initial value to `useRef`. Widened callback parameter types to `string` with internal cast to the narrower union type.
- **Files modified:** `src/features/support/hooks/useSupportUsers.ts`

**3. [Rule 3 - Blocking Issue] Fixed missing hook exports in api/index.ts**
- **Found during:** Task 2 CI typecheck
- **Issue:** `src/features/support/api/index.ts` only exported `useGetSupportUsersQuery`. The new hooks were unreachable from the feature barrel, causing typecheck failures in `index.ts`.
- **Fix:** Added all four hook exports to `api/index.ts`.
- **Files modified:** `src/features/support/api/index.ts`

**4. [Design adaptation] Used SupportUser tags (not User/UserList) for cache invalidation**
- **Issue:** Plan specified `invalidatesTags: ["User", "UserList"]` but these tag types don't exist in `apiSlice`. The support API uses `"SupportUser"` tags.
- **Fix:** Used `[{ type: "SupportUser", id: userId }, { type: "SupportUser", id: "LIST" }]` — invalidates both the detail view cache and the list page cache.

**5. [Design adaptation] Mutations placed in supportApi (not userApi)**
- **Issue:** Plan suggested checking "which API slice handles support user endpoints" — both slices inject into the shared `apiSlice` but the support context owns these operations.
- **Fix:** Added mutations to `supportApi.ts` alongside the existing support user queries for consistency.

## Threat Mitigations Applied

| Threat | Mitigation |
|--------|-----------|
| T-55-07 Tampering (mutations) | Mutations send only `userId` in URL path — no request body to tamper with |
| T-55-08 Spoofing (viewer role) | `RoleActions` hides buttons for non-super-users (defense in depth) — backend PermissionGuard is the real enforcement |
| T-55-09 Information Disclosure (error toasts) | Error toasts only visible to the authenticated user performing the action |

## Known Stubs

None. All data is wired to real API endpoints (`/v1/support/users/:id`, `/v1/users/:id/support-role`).

## Threat Surface Scan

No new network endpoints introduced in the frontend. The grant/revoke mutations call existing backend endpoints established in Plan 55-01. The new `/support/users/:userId` route is protected by the existing `SupportGuard` which validates support user access.

## Self-Check: PASSED

| Item | Status |
|------|--------|
| `src/features/support/components/GrantRoleDialog.tsx` | FOUND |
| `src/features/support/components/RevokeRoleDialog.tsx` | FOUND |
| `src/features/support/components/RoleActions.tsx` | FOUND |
| `src/pages/support/SupportUserDetailPage.tsx` | FOUND |
| Commit be4382d (Task 1) | FOUND |
| Commit 7f503f2 (Task 2) | FOUND |
| `npm run ci` exits 0 | PASSED |
