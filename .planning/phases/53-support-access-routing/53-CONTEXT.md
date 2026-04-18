# Phase 53: Support Access & Routing - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the frontend routing and access control changes that allow support users to log in, bypass onboarding and subscription gating, and reach a dedicated `/support` dashboard. It introduces a `SupportGuard` route wrapper, restructures App.tsx routing to separate support and business route branches, updates the login flow with role-aware redirect logic, and configures the sidebar navigation to show support-only menu items for support users.

</domain>

<decisions>
## Implementation Decisions

### Post-Login Routing
- **D-01:** Smart redirect in LoginPage — after Firebase auth succeeds, fetch user data via RTK Query, check `supportRoles`, and navigate to `/support` (if support) or `/dashboard` (if business). All routing logic in LoginPage.
- **D-02:** Support role takes priority for dual-role users — `supportRoles.length > 0` always routes to `/support`, even if the user also has a business association.
- **D-03:** Honour saved location state — check `location.state?.from` first (set by ProtectedRoute on session expiry redirect), fall back to role-based redirect if no saved location.

### Route Structure
- **D-04:** Create a dedicated `SupportGuard` component (similar to OnboardingGuard) that checks `supportRoles.length > 0` and redirects non-support users to `/dashboard`. Move `/support/*` routes into their own branch under `AuthenticatedLayout`, completely outside `OnboardingGuard` and `PaywallGuard`.
- **D-05:** No OnboardingGuard bypass needed — since support routes are in a separate branch, OnboardingGuard never executes for support users. Each guard handles its own concern only.
- **D-06:** Remove per-page manual role checks from SupportDashboardPage, SupportBusinessesPage, and SupportUsersPage — SupportGuard handles access control, making these checks redundant.

### Sidebar Navigation
- **D-07:** Support users see support items only — Dashboard, Users, Businesses. Business items (Jobs, Customers, Quotes, Items, etc.) are hidden since support users don't have a business context.
- **D-08:** Reuse DashboardLayout with role-conditional navigation — pass different nav items based on `supportRoles.length > 0`. Same layout (header, sidebar, content area), different menu items.

### Shell Dashboard
- **D-09:** Keep existing SupportDashboardPage content — migration controls (super user only) and quick links to businesses/users. Phase 54 adds membership summary cards above the existing content.

### Claude's Discretion
- Whether the super_user-only migration controls use a permission-based check or keep the existing role-name check (depends on what Phase 52 delivers and which permissions are seeded in Phase 51)
- Frontend role check approach — direct `supportRoles.length > 0` on arrays, no frontend `hasPermission()` or `hasSupportRole()` utility function needed
- Exact SupportGuard implementation details (loading states, error handling)
- Whether to create a `getSupportNavigationItems()` function or inline the support nav config

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` — Permission/role data model, hydration strategy, seeding approach
- `.planning/phases/52-permission-guard-migration/52-CONTEXT.md` — Permission guard infrastructure, hasPermission() utility, migration of hardcoded role checks

### Frontend Routing & Guards
- `trade-flow-ui/src/App.tsx` — Route structure to restructure (support branch + business branch)
- `trade-flow-ui/src/features/auth/components/ProtectedRoute.tsx` — Auth guard pattern and location state for return URL
- `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` — Onboarding guard (support routes bypass by being in separate branch)
- `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` — Paywall guard with existing support bypass (migrated in Phase 52)

### Login & Auth
- `trade-flow-ui/src/pages/LoginPage.tsx` — Login page to update with smart redirect logic
- `trade-flow-ui/src/hooks/useAuth.ts` — Auth hook providing Firebase user
- `trade-flow-ui/src/providers/auth-provider.tsx` — Auth provider managing Firebase state

### Support Pages
- `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` — Existing dashboard with migration controls (remove per-page role check)
- `trade-flow-ui/src/pages/support/SupportBusinessesPage.tsx` — Existing businesses page (remove per-page role check)
- `trade-flow-ui/src/pages/support/SupportUsersPage.tsx` — Existing users page (remove per-page role check)

### Navigation & Layout
- `trade-flow-ui/src/config/navigation.ts` — Navigation config with existing support section (to be refactored for role-conditional rendering)
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` — Layout to update with role-conditional navigation items

### User Types
- `trade-flow-ui/src/types/api.types.ts` — User interface with supportRoles and businessRoles arrays
- `trade-flow-ui/src/hooks/useCurrentBusiness.ts` — Business context hook (used by business routes, not support routes)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProtectedRoute` — Pattern for SupportGuard (checks condition, renders Outlet or Navigate)
- `OnboardingGuard` — Route guard pattern to follow for SupportGuard implementation
- `DashboardLayout` — Reusable layout, just needs role-conditional nav items
- `navigation.ts` — Already has support nav section definition, needs refactoring to separate support vs business items

### Established Patterns
- Route guards use `<Outlet />` for pass-through and `<Navigate to="..." />` for redirect
- Navigation config uses `getNavigationItems(user)` function that returns sections based on user state
- RTK Query hooks used for data fetching (user data available via `useGetCurrentUserQuery()`)
- `supportRoles.length > 0` is the established frontend pattern for support user detection

### Integration Points
- `App.tsx` — Route tree restructured to add SupportGuard branch alongside OnboardingGuard branch
- `LoginPage.tsx` — handleSignInSuccess updated with role-aware redirect
- `DashboardLayout.tsx` — Navigation items conditional on support role
- `navigation.ts` — Split into support and business navigation item generators
- Three support pages — Remove redundant per-page role checks

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 53-support-access-routing*
*Context gathered: 2026-04-18*
