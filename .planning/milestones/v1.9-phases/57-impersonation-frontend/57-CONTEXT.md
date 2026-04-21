# Phase 57: Impersonation Frontend - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

A support user can impersonate a customer and see exactly what that customer sees — same data, same subscription state, same permissions — with a persistent banner and a clean exit back to the support dashboard. This phase delivers the frontend impersonation experience: token switching in RTK Query, an impersonation banner, the entry point on the user detail page, and session termination flow.

</domain>

<decisions>
## Implementation Decisions

### Auth Token Switching
- **D-01:** Replace the Bearer token during impersonation — store the impersonation token (returned by Phase 56's `POST /v1/impersonation/start`) in Redux state and use it instead of the Firebase token in RTK Query's `prepareHeaders`. When impersonation is active, `prepareHeaders` reads from the impersonation slice; otherwise, it uses `auth.currentUser.getIdToken()` as normal.
- **D-02:** Dedicated `impersonationSlice` in the Redux store holding: impersonation token, target user info (name, email, ID), session ID, and active flag. No sessionStorage persistence.
- **D-03:** Page refresh ends impersonation — Redux state is lost, support user returns to their own dashboard via normal Firebase auth. No persistence complexity, aligns with security best practice for sensitive sessions.

### Banner Design & Placement
- **D-04:** Fixed banner at the very top of the viewport, above the DashboardLayout header. Pushes all content down. Position fixed, z-50 (above header's z-20). Cannot be scrolled away, dismissed, or covered by modals/dropdowns.
- **D-05:** Warning amber visual style — amber/yellow background with dark text. Matches the conventional impersonation bar pattern used by Shopify, Stripe, and AWS.
- **D-06:** Banner content: warning icon, "Impersonating: [user name]", and a "Return to Support" button. Compact single-line bar.
- **D-07:** During impersonation, the sidebar shows the customer's business navigation (Jobs, Customers, Quotes, Items, etc.) — not the support navigation. The support user sees exactly what the customer sees. The banner provides the escape hatch back to support.

### Impersonation Entry Point
- **D-08:** "Impersonate" button on the user detail page only (from Phase 54). No quick-action in the users list table. Keeps the action deliberate — support user must navigate to a customer and review their info before impersonating.
- **D-09:** Confirmation dialog with a required reason text field before starting impersonation. Maps directly to Phase 56's audit log `reason` field. Support users must document why they're impersonating.
- **D-10:** The "Impersonate" button is hidden entirely for support users who lack the `impersonate_user` permission. No disabled/greyed-out state — if you can't use it, you don't see it.
- **D-11:** Cannot impersonate other support users — the button should not appear when viewing a support user's detail page (backend also enforces this with 403).

### Session Termination
- **D-12:** "Return to Support" calls a backend endpoint (e.g., `POST /v1/impersonation/end` from Phase 56) to record the end timestamp in the audit log, then clears Redux impersonation state and resets the RTK Query cache. Ensures complete audit trail.
- **D-13:** After ending impersonation, navigate to `/support` (support dashboard). Clean break from the customer context.
- **D-14:** Expired session handling: when the backend rejects the impersonation token (expired), show a toast notification ("Impersonation session expired"), clear impersonation state, and automatically navigate back to `/support`. No modal or user acknowledgment required.

### Claude's Discretion
- RTK Query cache reset strategy during impersonation start/end (full reset vs selective tag invalidation)
- Exact impersonation banner component structure and Tailwind classes
- How to detect expired tokens in RTK Query (baseQuery error interceptor vs per-endpoint handling)
- Whether to show a loading state during impersonation start (while calling backend + switching token)
- Mobile responsive behavior of the impersonation banner

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` — Permission data model, `impersonate_user` permission, role hydration on user DTOs
- `.planning/phases/52-permission-guard-migration/52-CONTEXT.md` — `@RequiresPermission` decorator, `hasPermission()` utility, permission checking patterns
- `.planning/phases/53-support-access-routing/53-CONTEXT.md` — SupportGuard, support route branch, role-conditional navigation, smart redirect in LoginPage

### Phase 56 Backend Contract
- `.planning/ROADMAP.md` §Phase 56 — `POST /v1/impersonation/start` returns time-limited impersonation token, `POST /v1/impersonation/end` records end timestamp, `impersonation_audit` collection

### Frontend Auth & State
- `trade-flow-ui/src/services/api.ts` — RTK Query base query with `prepareHeaders` (Firebase token injection point)
- `trade-flow-ui/src/providers/auth-provider.tsx` — AuthProvider with `apiSlice.util.resetApiState()` cache reset pattern
- `trade-flow-ui/src/store/` — Redux store configuration, existing slice patterns

### Layout & Navigation
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` — Header (z-20), sidebar, TrialBadge, navigation rendering
- `trade-flow-ui/src/config/navigation.ts` — `getNavigationItems(user)` with role-conditional sections
- `trade-flow-ui/src/App.tsx` — Route structure with guard branches (support vs business)

### Support Pages
- `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` — Return destination after impersonation ends
- `trade-flow-ui/src/pages/support/SupportUsersPage.tsx` — Users list (no impersonate action here)

### User Types
- `trade-flow-ui/src/types/api.types.ts` — User interface with `supportRoles` and `businessRoles` arrays

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `DashboardLayout` — Layout component to wrap with impersonation banner; header z-20 establishes z-index baseline
- `TrialBadge` — Pattern for temporary status display in header area
- `apiSlice.util.resetApiState()` — Cache reset pattern from AuthProvider, reuse for impersonation start/end
- `Alert` component (`src/components/ui/alert.tsx`) — Alert styling patterns, though banner needs custom implementation
- `getNavigationItems(user)` — Already role-conditional, needs impersonation-aware branch

### Established Patterns
- RTK Query `prepareHeaders` reads Firebase token via `auth.currentUser.getIdToken()` — impersonation overrides this
- Redux slices in `src/store/slices/` — pattern for new `impersonationSlice`
- Route guards (`ProtectedRoute`, `OnboardingGuard`, `SupportGuard`) — may need impersonation-aware routing
- Toast notifications via Sonner — use for expired session notification
- Confirmation dialogs via `AlertDialog` component — use for impersonation confirmation with reason field

### Integration Points
- `prepareHeaders` in `src/services/api.ts` — Must check impersonation state before falling back to Firebase token
- `DashboardLayout` — Banner injected above or wrapping existing header
- `navigation.ts` — Must return customer nav items during impersonation instead of support items
- `App.tsx` routing — During impersonation, support user must access business routes (Jobs, Customers, etc.) that are normally behind OnboardingGuard/PaywallGuard
- User detail page (Phase 54) — "Impersonate" button added here

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase patterns and conventional impersonation UIs (Shopify, Stripe admin panels).

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 57-impersonation-frontend*
*Context gathered: 2026-04-18*
