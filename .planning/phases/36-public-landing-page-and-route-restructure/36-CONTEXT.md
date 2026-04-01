# Phase 36: Public Landing Page and Route Restructure - Context

**Gathered:** 2026-04-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Frontend-only phase (`trade-flow-ui`). Deliver a public marketing landing page at the root URL (`/`) with hero section, feature highlights, pricing card, and sign-up/login navigation. Restructure App.tsx route tree to support three nested guard layers (ProtectedRoute, OnboardingGuard shell, HardPaywallGuard shell) ready for Phase 37 and 38 to implement. Landing page must be bundle-isolated from the app's Redux store, feature modules, and auth hooks.

</domain>

<decisions>
## Implementation Decisions

### Landing Page Content
- **D-01:** Direct and practical tone — speak like a tradesperson thinks. Short sentences, no jargon, no corporate language. Example: "Run your trade business in one place. Jobs, quotes, schedules — sorted."
- **D-02:** Feature highlights showcase three core capabilities: **Jobs** (manage from first call to final payment), **Quotes** (send professional quotes, customers accept online), **Scheduling** (book visits, track progress). Maps to shipped features.
- **D-03:** Single pricing card: "£6/month — 30 days free, no card required." No feature comparison, no free tier, no complexity. One plan, one price, one CTA.
- **D-04:** Visual elements are Lucide icons only (already in the project). No screenshots, no illustrations. Clean, fast-loading, lightweight.

### Route Guard Architecture
- **D-05:** Three nested guard layers in App.tsx: `ProtectedRoute` (auth) > `OnboardingGuard` (profile/business check) > `PaywallGuard` (subscription check). Landing page and /login sit OUTSIDE all guards as public routes.
- **D-06:** OnboardingGuard and PaywallGuard are pass-through shells in Phase 36 — they render `<Outlet />` only. Phase 37 adds onboarding redirect logic. Phase 38 adds paywall blocking logic.
- **D-07:** Existing `SubscriptionGatedLayout` from Phase 32 stays as-is. Phase 38 replaces it with the new PaywallGuard and removes the soft paywall. Don't touch the existing soft paywall during route restructure.
- **D-08:** Route tree structure:
  ```
  <Route path="/" element={<LandingPage />} />          // public
  <Route path="/login" element={<LoginPage />} />        // public
  <Route element={<ProtectedRoute />}>                   // auth guard
    <Route element={<OnboardingGuard />}>                // Phase 37 implements
      <Route path="/onboarding/*" ... />
    </Route>
    <Route element={<OnboardingGuard />}>
      <Route element={<PaywallGuard />}>                 // Phase 38 implements
        <Route element={<DashboardLayout />}>
          <Route path="/dashboard" ... />
          <Route path="/customers" ... />
          ...
        </Route>
      </Route>
    </Route>
  </Route>
  ```

### Bundle Isolation
- **D-09:** Landing page is lazy-loaded via `React.lazy(() => import('./pages/LandingPage'))` with `<Suspense>` fallback. Vite automatically code-splits it into a separate chunk.
- **D-10:** LandingPage.tsx may only import from: its own directory, `@/components/ui/*` (Button, Card, etc.), `@/config/firebase` (for auth check). NO imports from `@/store`, `@/features/*`, `@/hooks/*`, `@/providers/*`.
- **D-11:** Bundle isolation verified via `vite build` + manual chunk inspection (or rollup-plugin-visualizer). One-time verification during development, not a CI gate.

### Navigation and CTA Flow
- **D-12:** "Start Free Trial" CTA navigates to `/login` with sign-up mode. After Firebase sign-up, user flows through ProtectedRoute > OnboardingGuard > onboarding wizard (Phase 37) > trial auto-starts (Phase 35 endpoint) > dashboard.
- **D-13:** Minimal landing page header: Trade Flow logo/name on left, "Log in" and "Start Free Trial" buttons on right. Separate component from DashboardLayout header — no sidebar, no app navigation.
- **D-14:** Authenticated user redirect (LAND-06): LandingPage component checks Firebase auth state via `onAuthStateChanged` directly (not useAuth hook — maintains bundle isolation). If authenticated, `navigate('/dashboard')`. If not, render landing page. Brief spinner during auth state check.

### Claude's Discretion
- Exact hero section copy and headline
- Feature highlight card layout (grid, stack, etc.)
- Pricing card visual treatment
- Landing page footer content (if any)
- Suspense fallback component design
- Mobile responsive breakpoints for landing page sections
- Whether to pass `?mode=signup` query param to /login or use a different mechanism for sign-up tab default

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.7 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.7 requirements; Phase 36 covers LAND-01 through LAND-06

### Roadmap
- `.planning/ROADMAP.md` §Phase 36 — Phase goal, success criteria, UI hint, parallel with Phase 35

### Prior Phase Context (relevant decisions)
- `.planning/milestones/v1.6-phases/32-subscription-gate-and-subscribe-pages/32-CONTEXT.md` — SubscriptionGatedLayout, soft paywall architecture, useSubscription hook, /subscribe page patterns
- `.planning/phases/35-no-card-trial-api-endpoint/35-CONTEXT.md` — Trial endpoint (POST /v1/subscription/trial) that Phase 37 will call after onboarding

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — Frontend file naming, component structure, styling patterns
- `.planning/codebase/STRUCTURE.md` — Frontend directory structure, existing pages, features, components

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProtectedRoute` (`src/features/auth/components/ProtectedRoute.tsx`): existing auth guard — wraps authenticated routes
- `DashboardLayout` (`src/components/layouts/DashboardLayout.tsx`): existing app shell with header + sidebar
- `Button`, `Card` and other shadcn/ui components (`src/components/ui/`): reusable for landing page sections
- `lucide-react`: 560+ icons already installed — use for feature highlight icons
- `AuthForm` (`src/features/auth/components/AuthForm.tsx`): existing login/signup toggle form
- `LoginPage` (`src/pages/LoginPage.tsx`): existing login page
- Firebase config (`src/config/firebase.ts`): Firebase app + auth already initialized

### Established Patterns
- Feature-based module structure: pages import from feature barrels
- RTK Query for server state, Redux slices for client state
- `cn()` utility for conditional Tailwind classes
- Mobile-first responsive design with `md:` breakpoint prefix
- Existing route structure in App.tsx with ProtectedRoute wrapper

### Integration Points
- `App.tsx`: Route tree restructure — add landing page route, add guard shells, reorganize existing routes
- `src/pages/LandingPage.tsx`: New page component (lazy-loaded)
- `src/features/auth/components/OnboardingGuard.tsx`: New pass-through shell (Phase 37 implements logic)
- `src/features/auth/components/PaywallGuard.tsx`: New pass-through shell (Phase 38 implements logic)

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing UI patterns. Direct, practical tone for tradesperson audience.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 36-public-landing-page-and-route-restructure*
*Context gathered: 2026-04-01*
