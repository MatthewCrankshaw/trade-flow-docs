# Phase 53: Support Access & Routing - Research

**Researched:** 2026-04-18
**Domain:** React routing, route guards, role-based navigation, frontend access control
**Confidence:** HIGH

## Summary

Phase 53 is a frontend-only phase that restructures the React routing tree in `trade-flow-ui` to give support users a dedicated experience. The work touches six files and creates one new component (`SupportGuard`). After Phase 52 migrates all hardcoded role checks to the RBAC permission system on the backend, this phase ensures the frontend routes support users correctly: bypassing onboarding/paywall guards, redirecting to `/support` after login, protecting `/support/*` routes, and showing support-specific navigation items.

The existing codebase already has all the patterns needed. `ProtectedRoute` and `OnboardingGuard` demonstrate the route guard pattern (check condition, render `<Outlet />` or `<Navigate />`). The `navigation.ts` config already has a support section. `DashboardLayout` already renders navigation items from config. The `useGetCurrentUserQuery()` RTK Query hook provides user data with hydrated `supportRoles` arrays. No new libraries, no new API endpoints, no backend changes.

**Primary recommendation:** Create `SupportGuard` following the `OnboardingGuard` pattern exactly. Restructure `App.tsx` to place `/support/*` routes in a separate branch under `SupportGuard`, outside `OnboardingGuard` and `PaywallGuard`. Update `LoginPage` with role-aware redirect logic. Refactor `navigation.ts` to return support-specific or business-specific nav items based on role. Remove redundant per-page role checks from three support pages.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Smart redirect in LoginPage -- after Firebase auth succeeds, fetch user data via RTK Query, check `supportRoles`, and navigate to `/support` (if support) or `/dashboard` (if business). All routing logic in LoginPage.
- **D-02:** Support role takes priority for dual-role users -- `supportRoles.length > 0` always routes to `/support`, even if the user also has a business association.
- **D-03:** Honour saved location state -- check `location.state?.from` first (set by ProtectedRoute on session expiry redirect), fall back to role-based redirect if no saved location.
- **D-04:** Create a dedicated `SupportGuard` component (similar to OnboardingGuard) that checks `supportRoles.length > 0` and redirects non-support users to `/dashboard`. Move `/support/*` routes into their own branch under `AuthenticatedLayout`, completely outside `OnboardingGuard` and `PaywallGuard`.
- **D-05:** No OnboardingGuard bypass needed -- since support routes are in a separate branch, OnboardingGuard never executes for support users. Each guard handles its own concern only.
- **D-06:** Remove per-page manual role checks from SupportDashboardPage, SupportBusinessesPage, and SupportUsersPage -- SupportGuard handles access control, making these checks redundant.
- **D-07:** Support users see support items only -- Dashboard, Users, Businesses. Business items (Jobs, Customers, Quotes, Items, etc.) are hidden since support users don't have a business context.
- **D-08:** Reuse DashboardLayout with role-conditional navigation -- pass different nav items based on `supportRoles.length > 0`. Same layout (header, sidebar, content area), different menu items.
- **D-09:** Keep existing SupportDashboardPage content -- migration controls (super user only) and quick links to businesses/users. Phase 54 adds membership summary cards above the existing content.

### Claude's Discretion
- Whether the super_user-only migration controls use a permission-based check or keep the existing role-name check (depends on what Phase 52 delivers and which permissions are seeded in Phase 51)
- Frontend role check approach -- direct `supportRoles.length > 0` on arrays, no frontend `hasPermission()` or `hasSupportRole()` utility function needed
- Exact SupportGuard implementation details (loading states, error handling)
- Whether to create a `getSupportNavigationItems()` function or inline the support nav config

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SACC-01 | Support user can log in and bypass onboarding entirely (no business association required) | Support routes placed in separate App.tsx branch outside OnboardingGuard/PaywallGuard; SupportGuard only checks `supportRoles.length > 0` |
| SACC-02 | Support user is redirected to `/support` dashboard after login instead of the main app dashboard | LoginPage updated with role-aware redirect: fetch user data, check supportRoles, navigate to `/support` or `/dashboard` |
| SACC-03 | Support routes (`/support/*`) are protected and only accessible to users with a support role | SupportGuard component checks `supportRoles.length > 0`, redirects non-support users to `/dashboard` |
| SACC-04 | Support user bypasses subscription gating (existing behaviour, preserved through RBAC migration) | Support route branch is outside PaywallGuard entirely; backend bypass handled by Phase 52's permission migration |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Post-login redirect | Frontend (Client) | -- | Client-side navigation after Firebase auth; user data from RTK Query cache |
| Route guard (SupportGuard) | Frontend (Client) | -- | Client-side route protection using React Router Outlet/Navigate |
| Onboarding bypass | Frontend (Client) | -- | Structural bypass -- support routes in separate branch, OnboardingGuard never executes |
| Subscription gating bypass | Frontend (Client) | API / Backend | Frontend: support routes outside PaywallGuard branch. Backend: Phase 52 permission-based bypass |
| Navigation items | Frontend (Client) | -- | Role-conditional nav config rendering in DashboardLayout sidebar |
| Shell dashboard page | Frontend (Client) | -- | Static page component, no new data fetching |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| react | 19.2.0 | Component framework | Already installed [VERIFIED: CLAUDE.md] |
| react-router-dom | 7.13.0 | Client-side routing, `Outlet`, `Navigate`, `useNavigate`, `useLocation` | Already installed [VERIFIED: CLAUDE.md] |
| @reduxjs/toolkit | 2.11.2 | RTK Query for user data fetching | Already installed [VERIFIED: CLAUDE.md] |
| lucide-react | 0.563.0 | Icons for support nav items (LayoutDashboard, Users, Building2) | Already installed [VERIFIED: CLAUDE.md] |

### Supporting
No new dependencies required. This phase uses only existing packages.

**Installation:**
```bash
# No new packages needed
```

## Architecture Patterns

### System Architecture Diagram

```
User Login (Firebase Auth)
  |
  v
LoginPage.tsx
  |  Firebase auth succeeds
  |  Fetch user data via useGetCurrentUserQuery()
  |
  +-- location.state?.from exists? --> Navigate to saved location
  |
  +-- user.supportRoles.length > 0? --> Navigate to /support
  |
  +-- else --> Navigate to /dashboard

---

App.tsx Route Tree (post-restructure):

<ProtectedRoute>                    (checks Firebase auth)
  |
  <AuthenticatedLayout>
    |
    +-- <SupportGuard>              (NEW: checks supportRoles.length > 0)
    |     |
    |     +-- <DashboardLayout>     (with SUPPORT nav items)
    |           |
    |           +-- /support        --> SupportDashboardPage
    |           +-- /support/users  --> SupportUsersPage
    |           +-- /support/businesses --> SupportBusinessesPage
    |
    +-- <OnboardingGuard>           (existing: checks onboarding complete)
          |
          +-- <PaywallGuard>        (existing: checks subscription)
                |
                +-- <DashboardLayout> (with BUSINESS nav items)
                      |
                      +-- /dashboard  --> DashboardPage
                      +-- /jobs       --> JobsPage
                      +-- /customers  --> CustomersPage
                      +-- ... (all business routes)
```

### Recommended Project Structure
```
src/
├── features/auth/components/
│   ├── ProtectedRoute.tsx       # Existing auth guard
│   ├── OnboardingGuard.tsx      # Existing onboarding guard
│   ├── SupportGuard.tsx         # NEW: support role guard
│   └── index.ts                 # Updated barrel export
├── config/
│   └── navigation.ts            # MODIFIED: split into support/business nav items
├── components/layouts/
│   └── DashboardLayout.tsx      # MODIFIED: role-conditional nav items
├── pages/
│   ├── LoginPage.tsx            # MODIFIED: role-aware redirect
│   └── support/
│       ├── SupportDashboardPage.tsx  # MODIFIED: remove per-page role check
│       ├── SupportBusinessesPage.tsx # MODIFIED: remove per-page role check
│       └── SupportUsersPage.tsx      # MODIFIED: remove per-page role check
└── App.tsx                      # MODIFIED: restructured route tree
```

### Pattern 1: Route Guard (SupportGuard)
**What:** A component that checks a condition and either renders child routes (`<Outlet />`) or redirects.
**When to use:** Protecting a branch of routes that require a specific user state.
**Example:**
```typescript
// Source: follows OnboardingGuard / ProtectedRoute pattern from codebase [VERIFIED: STRUCTURE.md]
import { Navigate, Outlet } from "react-router-dom";
import { useGetCurrentUserQuery } from "@/services/userApi";

export function SupportGuard() {
  const { data: user, isLoading } = useGetCurrentUserQuery();

  if (isLoading) {
    // Render loading state (spinner or skeleton) -- no flash of content
    return <LoadingSpinner />;
  }

  if (!user || user.supportRoles.length === 0) {
    return <Navigate to="/dashboard" replace />;
  }

  return <Outlet />;
}
```

### Pattern 2: Role-Aware Login Redirect
**What:** After successful authentication, check user role and navigate to the appropriate dashboard.
**When to use:** LoginPage post-auth success handler.
**Example:**
```typescript
// Source: D-01, D-02, D-03 from CONTEXT.md [VERIFIED: CONTEXT.md]
const navigate = useNavigate();
const location = useLocation();

function handlePostLoginRedirect(user: UserResponse) {
  const savedLocation = location.state?.from;
  if (savedLocation) {
    navigate(savedLocation, { replace: true });
    return;
  }

  if (user.supportRoles.length > 0) {
    navigate("/support", { replace: true });
  } else {
    navigate("/dashboard", { replace: true });
  }
}
```

### Pattern 3: Role-Conditional Navigation
**What:** Return different navigation item arrays based on user role.
**When to use:** DashboardLayout rendering sidebar navigation.
**Example:**
```typescript
// Source: D-07, D-08 from CONTEXT.md [VERIFIED: CONTEXT.md]
// navigation.ts

export function getSupportNavigationItems(): NavigationSection[] {
  return [
    {
      title: "Support",
      items: [
        { title: "Dashboard", href: "/support", icon: LayoutDashboard },
        { title: "Users", href: "/support/users", icon: Users },
        { title: "Businesses", href: "/support/businesses", icon: Building2 },
      ],
    },
  ];
}

export function getBusinessNavigationItems(/* existing params */): NavigationSection[] {
  // existing business nav items
}
```

### Anti-Patterns to Avoid
- **Per-page role checks when guard exists:** D-06 explicitly removes redundant `supportRoles` checks from SupportDashboardPage, SupportBusinessesPage, and SupportUsersPage. The guard handles access control; pages should assume they are only rendered for authorized users.
- **Shared guard branch for support and business routes:** Support routes MUST be in a separate branch from OnboardingGuard/PaywallGuard. Putting them in the same branch and adding bypass logic creates coupling between unrelated concerns (D-05).
- **Creating a frontend `hasPermission()` utility:** D-discretion says direct `supportRoles.length > 0` is the pattern. No abstraction layer needed for this phase.
- **Hardcoded role name checks:** Do not check `supportRoles.some(r => r.name === "super_user")` for routing -- all support roles route to `/support`. Role-specific checks (like migration controls) are a separate concern within the support dashboard.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Route protection | Custom auth middleware | React Router `<Navigate />` + guard components | Established pattern in codebase; consistent with ProtectedRoute/OnboardingGuard |
| Loading states in guards | Custom loading logic | RTK Query `isLoading` state | Already used throughout the app; prevents flash of unauthorized content |
| Navigation config | Hardcoded JSX in sidebar | Config-driven `navigation.ts` | Already exists; role-conditional rendering is a natural extension |
| Post-login routing | Window.location manipulation | `useNavigate()` from react-router-dom | Client-side navigation preserves SPA state and Redux store |

## Common Pitfalls

### Pitfall 1: Flash of Business Dashboard for Support Users
**What goes wrong:** Support user briefly sees the business dashboard or onboarding wizard before being redirected to `/support`.
**Why it happens:** Route guards render before user data is loaded; default route renders business content.
**How to avoid:** SupportGuard must show a loading state (spinner) until `useGetCurrentUserQuery()` resolves. The guard renders nothing (or a spinner) during loading -- never `<Outlet />` and never `<Navigate />`.
**Warning signs:** Support user sees a brief flash of the onboarding wizard when navigating to `/support`.

### Pitfall 2: Infinite Redirect Loop Between Guards
**What goes wrong:** SupportGuard redirects to `/dashboard`, which triggers OnboardingGuard, which redirects to `/onboarding`, creating a loop.
**Why it happens:** Route branches overlap or guards don't account for the other branch's redirect targets.
**How to avoid:** Support routes and business routes are in completely separate branches (D-04). SupportGuard redirects to `/dashboard` (inside the business branch), and OnboardingGuard handles that path independently. A support user who also has a business will never reach SupportGuard when accessing `/dashboard` -- they go through the business branch. A non-support, non-onboarded user hits OnboardingGuard normally.
**Warning signs:** Browser shows "too many redirects" error.

### Pitfall 3: RTK Query Cache Stale After Login
**What goes wrong:** LoginPage tries to check `user.supportRoles` but the RTK Query cache has stale or no data.
**Why it happens:** Login creates a new Firebase auth token, but the user query may not have been triggered yet or may still have the unauthenticated cache state.
**How to avoid:** After Firebase auth succeeds, explicitly trigger or wait for `useGetCurrentUserQuery()` to complete with fresh data before making the redirect decision. Use the `isSuccess` flag, not just check for `data` existence.
**Warning signs:** User always redirects to `/dashboard` regardless of role, or sees "Unable to determine your account type" error toast.

### Pitfall 4: Location State Not Cleared After Use
**What goes wrong:** User keeps being redirected to a stale `location.state.from` on subsequent logins.
**Why it happens:** `location.state` persists in browser history.
**How to avoid:** Use `navigate(savedLocation, { replace: true })` which replaces the current history entry, preventing the state from being reused on back navigation.
**Warning signs:** User always returns to the same page after login instead of their role-appropriate dashboard.

### Pitfall 5: DashboardLayout Receives Wrong Nav Items
**What goes wrong:** Support user sees business navigation items, or business user sees support items.
**Why it happens:** The role check in DashboardLayout uses a different data source or timing than the route guards.
**How to avoid:** DashboardLayout must use the same `useGetCurrentUserQuery()` hook and the same `supportRoles.length > 0` check as SupportGuard. Since the guard has already resolved by the time the layout renders, the data will be available.
**Warning signs:** Sidebar shows "Jobs", "Customers" etc. for support users.

## Code Examples

### SupportGuard Component
```typescript
// Source: follows existing OnboardingGuard pattern [VERIFIED: STRUCTURE.md]
// Location: src/features/auth/components/SupportGuard.tsx

import { Navigate, Outlet } from "react-router-dom";
import { useGetCurrentUserQuery } from "@/services/userApi";

export function SupportGuard() {
  const { data: user, isLoading } = useGetCurrentUserQuery();

  if (isLoading) {
    return (
      <div className="flex h-screen items-center justify-center">
        {/* Use existing spinner/loading component */}
      </div>
    );
  }

  if (!user || user.supportRoles.length === 0) {
    return <Navigate to="/dashboard" replace />;
  }

  return <Outlet />;
}
```

### App.tsx Route Structure (Restructured)
```typescript
// Source: D-04, D-05 from CONTEXT.md [VERIFIED: CONTEXT.md]
// Conceptual structure -- exact JSX depends on current App.tsx implementation

<Route element={<ProtectedRoute />}>
  {/* Support branch -- outside OnboardingGuard and PaywallGuard */}
  <Route element={<SupportGuard />}>
    <Route element={<DashboardLayout />}>
      <Route path="/support" element={<SupportDashboardPage />} />
      <Route path="/support/users" element={<SupportUsersPage />} />
      <Route path="/support/businesses" element={<SupportBusinessesPage />} />
    </Route>
  </Route>

  {/* Business branch -- existing guard chain */}
  <Route element={<OnboardingGuard />}>
    <Route element={<PaywallGuard />}>
      <Route element={<DashboardLayout />}>
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/jobs" element={<JobsPage />} />
        {/* ... all business routes ... */}
      </Route>
    </Route>
  </Route>
</Route>
```

### LoginPage Redirect Logic
```typescript
// Source: D-01, D-02, D-03 from CONTEXT.md [VERIFIED: CONTEXT.md]
// Added to LoginPage.tsx handleSignInSuccess or similar handler

const navigate = useNavigate();
const location = useLocation();
const { data: user, isSuccess } = useGetCurrentUserQuery(undefined, {
  skip: !isAuthenticated, // Only fetch after Firebase auth succeeds
});

useEffect(() => {
  if (!isSuccess || !user) return;

  const savedLocation = location.state?.from;
  if (savedLocation) {
    navigate(savedLocation, { replace: true });
    return;
  }

  if (user.supportRoles.length > 0) {
    navigate("/support", { replace: true });
  } else {
    navigate("/dashboard", { replace: true });
  }
}, [isSuccess, user, navigate, location.state]);
```

### Navigation Config Split
```typescript
// Source: D-07 from CONTEXT.md, UI-SPEC [VERIFIED: CONTEXT.md, UI-SPEC.md]
// Location: src/config/navigation.ts

import { LayoutDashboard, Users, Building2 } from "lucide-react";

export function getSupportNavigationItems(): NavigationSection[] {
  return [
    {
      title: "Support",
      items: [
        { title: "Dashboard", href: "/support", icon: LayoutDashboard },
        { title: "Users", href: "/support/users", icon: Users },
        { title: "Businesses", href: "/support/businesses", icon: Building2 },
      ],
    },
  ];
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Per-page role checks in each support page | Route-level SupportGuard | This phase | Eliminates redundant checks; single point of access control |
| Flat route tree with all routes in one guard chain | Branched route tree (support branch + business branch) | This phase | Guards only execute for their relevant routes; no bypass logic needed |
| Single `getNavigationItems()` for all users | Role-conditional `getSupportNavigationItems()` / `getBusinessNavigationItems()` | This phase | Support users see only support nav items |
| Hardcoded `supportRoles.length > 0` scattered in pages | Centralized check in SupportGuard | This phase | Consistent access control, easier to maintain |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `useGetCurrentUserQuery()` is the RTK Query hook name for fetching current user data | Code Examples | LOW -- hook name may differ slightly; implementer will verify against actual codebase |
| A2 | DashboardLayout accepts navigation items as props or reads them from a config function | Pattern 3 | LOW -- may use context or direct import; implementer will check actual DashboardLayout implementation |
| A3 | OnboardingGuard and PaywallGuard are separate wrapper components in the route tree (not combined) | Architecture Diagram | LOW -- if combined, the restructuring is simpler, not harder |
| A4 | `location.state?.from` is the pattern used by ProtectedRoute for saving redirect location | Pitfall 4 | LOW -- standard React Router pattern; may use a different key name |

## Open Questions

1. **How does DashboardLayout currently receive its navigation items?**
   - What we know: `navigation.ts` has a `getNavigationItems()` function. DashboardLayout renders the sidebar.
   - What's unclear: Whether nav items are passed as props, imported directly, or provided via context.
   - Recommendation: Implementer should read DashboardLayout to determine the integration point. The approach (function call vs props) affects how role-conditional rendering is wired.

2. **What loading component does the app use for guard loading states?**
   - What we know: Guards need to show a loading state while user data loads.
   - What's unclear: Whether the app has a shared loading spinner component or if each guard creates its own.
   - Recommendation: Implementer should check existing guards (OnboardingGuard, ProtectedRoute) for their loading state pattern and replicate.

3. **How does LoginPage currently handle post-auth navigation?**
   - What we know: LoginPage uses Firebase auth and navigates after success.
   - What's unclear: Whether it uses `useNavigate()` in an effect, a callback, or the Firebase `onAuthStateChanged` listener.
   - Recommendation: Implementer should read LoginPage to understand the current flow and add role-aware redirect at the right point in the lifecycle.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest 4.1.3 |
| Config file | `trade-flow-ui/vite.config.ts` (merged via mergeConfig in vitest setup) |
| Quick run command | `npm run test -- --run` |
| Full suite command | `npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SACC-01 | SupportGuard renders Outlet for support users, bypasses onboarding | unit | `npx vitest run src/features/auth/components/__tests__/SupportGuard.test.tsx` | Wave 0 |
| SACC-02 | LoginPage redirects support users to /support | unit | `npx vitest run src/pages/__tests__/LoginPage.test.tsx` | Wave 0 |
| SACC-03 | SupportGuard redirects non-support users to /dashboard | unit | `npx vitest run src/features/auth/components/__tests__/SupportGuard.test.tsx` | Wave 0 |
| SACC-04 | Support routes outside PaywallGuard branch | unit | Verified by App.tsx route structure test (structural) | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --run`
- **Per wave merge:** `npm run ci`
- **Phase gate:** Full CI suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `src/features/auth/components/__tests__/SupportGuard.test.tsx` -- covers SACC-01, SACC-03
- [ ] Test for LoginPage redirect logic -- covers SACC-02 (may extend existing LoginPage tests)
- [ ] Mocks for `useGetCurrentUserQuery` returning support user vs business user

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | Handled by existing Firebase auth (ProtectedRoute) |
| V3 Session Management | no | Handled by existing Firebase session management |
| V4 Access Control | yes | SupportGuard checks `supportRoles.length > 0`; backend permission guards (Phase 52) are the authoritative enforcement |
| V5 Input Validation | no | No new user inputs in this phase |
| V6 Cryptography | no | No cryptographic operations |

### Known Threat Patterns for Frontend Route Guards

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Client-side guard bypass via URL manipulation | Tampering | Frontend guards are UX convenience only; backend API enforces permissions via Phase 52 PermissionGuard. A user navigating directly to `/support` without a support role sees a redirect, not data. |
| Support role spoofing in client state | Elevation of Privilege | Role data comes from API response (server-authoritative); RTK Query cache reflects server state. Client cannot fabricate `supportRoles` because API endpoints still enforce permissions server-side. |
| Stale role data after role revocation | Information Disclosure | RTK Query refetches user on page load / token refresh. RADM-04 (Phase 55) addresses immediate effect of role changes. Acceptable risk for this phase. |

## Sources

### Primary (HIGH confidence)
- Phase 53 CONTEXT.md -- all locked decisions (D-01 through D-09)
- Phase 53 UI-SPEC.md -- component inventory, interaction contracts, navigation config
- Phase 51 CONTEXT.md -- RBAC data model, role hydration, supportRoles array structure
- Phase 52 CONTEXT.md -- permission guard migration, hasPermission utility, subscription bypass
- CLAUDE.md -- tech stack versions, conventions, testing setup
- `.planning/codebase/STRUCTURE.md` -- file locations, directory structure, existing patterns

### Secondary (MEDIUM confidence)
- React Router 7.x documentation for `Outlet`, `Navigate`, `useNavigate`, `useLocation` patterns [ASSUMED: based on react-router-dom 7.13.0 in CLAUDE.md]

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new libraries; all dependencies already installed and documented in CLAUDE.md
- Architecture: HIGH -- route guard pattern is well-established in codebase; CONTEXT.md decisions are explicit and detailed
- Pitfalls: HIGH -- pitfalls derive from standard React Router/RTK Query patterns and are well-documented in the ecosystem

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable -- no fast-moving dependencies)
