# Phase 36: Public Landing Page and Route Restructure - Research

**Researched:** 2026-04-02
**Domain:** React SPA routing, code splitting, public marketing page, route guard architecture
**Confidence:** HIGH

## Summary

Phase 36 is a frontend-only phase in `trade-flow-ui` that delivers two distinct pieces: (1) a public-facing marketing landing page at the root URL, and (2) a restructured App.tsx route tree with three nested guard layers ready for Phases 37 and 38.

The landing page must be bundle-isolated from the app's Redux store, feature modules, and auth hooks. This is achieved via `React.lazy` dynamic import, which Vite automatically code-splits into a separate chunk. The auth check on the landing page uses Firebase's `onAuthStateChanged` directly (not the app's `useAuth` hook) to maintain bundle isolation.

The route restructure introduces `OnboardingGuard` and `PaywallGuard` as pass-through shell components (rendering only `<Outlet />`) that Phase 37 and Phase 38 will fill with logic. The existing `SubscriptionGatedLayout` from Phase 32 remains untouched.

**Primary recommendation:** Build the landing page as a self-contained page with its own sub-components in `src/pages/landing/`, lazy-load it at the route level, and restructure App.tsx routes using react-router-dom v7's nested layout route pattern with `<Outlet />` for guard shells.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Direct and practical tone -- speak like a tradesperson thinks. Short sentences, no jargon, no corporate language.
- **D-02:** Feature highlights showcase three core capabilities: Jobs, Quotes, Scheduling.
- **D-03:** Single pricing card: "GBP 6/month -- 30 days free, no card required." One plan, one price, one CTA.
- **D-04:** Visual elements are Lucide icons only. No screenshots, no illustrations.
- **D-05:** Three nested guard layers in App.tsx: ProtectedRoute > OnboardingGuard > PaywallGuard. Landing page and /login sit OUTSIDE all guards.
- **D-06:** OnboardingGuard and PaywallGuard are pass-through shells in Phase 36 -- render `<Outlet />` only.
- **D-07:** Existing SubscriptionGatedLayout from Phase 32 stays as-is. Don't touch the existing soft paywall.
- **D-08:** Route tree structure defined (public routes outside, auth-guarded routes nested with OnboardingGuard and PaywallGuard).
- **D-09:** Landing page lazy-loaded via `React.lazy(() => import('./pages/LandingPage'))` with `<Suspense>` fallback.
- **D-10:** LandingPage may only import from: own directory, `@/components/ui/*`, `@/config/firebase`, `lucide-react`. NO imports from `@/store`, `@/features/*`, `@/hooks/*`, `@/providers/*`.
- **D-11:** Bundle isolation verified via `vite build` + manual chunk inspection. One-time, not CI gate.
- **D-12:** "Start Free Trial" CTA navigates to `/login` with sign-up mode.
- **D-13:** Minimal landing page header: logo/name left, "Log in" + "Start Free Trial" right. Separate from DashboardLayout.
- **D-14:** Auth redirect uses `onAuthStateChanged` directly from `@/config/firebase` (not useAuth hook). If authenticated, `navigate('/dashboard')`.

### Claude's Discretion
- Exact hero section copy and headline
- Feature highlight card layout (grid, stack, etc.)
- Pricing card visual treatment
- Landing page footer content (if any)
- Suspense fallback component design
- Mobile responsive breakpoints for landing page sections
- Whether to pass `?mode=signup` query param to /login or use a different mechanism

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LAND-01 | User can view a public landing page at the root URL without authentication | React.lazy code splitting + public route outside guards in App.tsx |
| LAND-02 | User can see a hero section with trade-specific value proposition and "Start Free Trial" CTA | Static component with Button, ArrowRight icon, navigate to /login?mode=signup |
| LAND-03 | User can see feature highlights showing core capabilities (jobs, quotes, scheduling) | Card components with Lucide icons in 3-column grid |
| LAND-04 | User can see a pricing section displaying GBP 6/month with no-card trial messaging | Card with pricing layout, Check icons for feature list |
| LAND-05 | User can navigate to sign up or log in from the landing page | Header nav buttons + hero/pricing CTAs linking to /login and /login?mode=signup |
| LAND-06 | Authenticated user visiting the root URL is redirected to the dashboard | Direct onAuthStateChanged check in LandingPage, navigate('/dashboard') if authenticated |

</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Tech stack:** React 19 / Vite 7 / TypeScript 5.9, react-router-dom 7.13
- **Two repos:** This phase is `trade-flow-ui` only
- **Styling:** Tailwind CSS 4 with semantic color tokens, mobile-first, `cn()` for conditional classes
- **Components:** shadcn/ui with Radix UI primitives (new-york preset)
- **State management:** RTK Query for API, Redux slices for client state -- but landing page must NOT use either
- **shadcn skill rules:** No `space-y-*` (use `flex flex-col gap-*`), no raw colors (use semantic tokens), `size-*` not `w-* h-*`, icons use `data-icon` attribute in Button, no sizing classes on icons inside components
- **File naming:** Pages PascalCase, components PascalCase in feature directories
- **Import type:** Always use `import type` for type-only imports
- **No `as` type assertions:** Prefer type guards/validation

## Standard Stack

### Core (already installed -- no new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| react | 19.2.0 | UI framework | Already installed |
| react-router-dom | 7.13.0 | Client-side routing with nested layout routes | Already installed |
| lucide-react | 0.563.0 | Icons for feature cards, pricing checks, CTA arrows | Already installed |
| firebase | 12.8.0 | Direct `onAuthStateChanged` for bundle-isolated auth check | Already installed |

### Supporting (already installed)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @/components/ui/button | shadcn | CTA buttons, nav buttons | All interactive elements |
| @/components/ui/card | shadcn | Feature cards, pricing card | Content containers |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Direct onAuthStateChanged | useAuth hook | Hook pulls in Redux store and providers -- breaks bundle isolation. Decision D-14 locks this. |
| React.lazy | Eager import | No code splitting -- landing page would bundle with entire app. Decision D-09 locks this. |

**Installation:**
```bash
# No new packages needed -- all dependencies already installed
```

## Architecture Patterns

### Recommended Project Structure

```
src/
├── pages/
│   ├── LandingPage.tsx              # Lazy-loaded entry, auth check + redirect logic
│   └── landing/                     # Landing page sub-components (co-located)
│       ├── LandingHeader.tsx        # Sticky header with logo + nav buttons
│       ├── HeroSection.tsx          # Headline, subheadline, CTA
│       ├── FeaturesSection.tsx      # 3-column feature cards grid
│       ├── PricingSection.tsx       # Single pricing card
│       └── LandingFooter.tsx        # Copyright line
├── features/
│   └── auth/
│       └── components/
│           ├── ProtectedRoute.tsx   # EXISTING -- no changes
│           ├── OnboardingGuard.tsx  # NEW -- pass-through <Outlet />
│           └── PaywallGuard.tsx     # NEW -- pass-through <Outlet />
└── App.tsx                          # Route tree restructure
```

### Pattern 1: Lazy-Loaded Public Route

**What:** Use `React.lazy` + `<Suspense>` to code-split the landing page into its own chunk.
**When to use:** Any public-facing page that should not bundle with the authenticated app.
**Example:**
```typescript
// In App.tsx
const LandingPage = lazy(() => import("./pages/LandingPage"));

// In route tree
<Route
  path="/"
  element={
    <Suspense fallback={<PageSpinner />}>
      <LandingPage />
    </Suspense>
  }
/>
```

### Pattern 2: Pass-Through Guard Shell

**What:** A layout route component that renders only `<Outlet />`, ready for logic injection in a future phase.
**When to use:** When establishing route hierarchy now but deferring guard logic to later phases.
**Example:**
```typescript
// src/features/auth/components/OnboardingGuard.tsx
import { Outlet } from "react-router-dom";

export function OnboardingGuard() {
  return <Outlet />;
}
```

### Pattern 3: Direct Firebase Auth Check (Bundle Isolation)

**What:** Use `onAuthStateChanged` directly from the Firebase config module instead of the app's `useAuth` hook.
**When to use:** In bundle-isolated components that must not import from `@/store`, `@/features/*`, `@/hooks/*`, or `@/providers/*`.
**Example:**
```typescript
// In LandingPage.tsx
import { useEffect, useState } from "react";
import { onAuthStateChanged } from "firebase/auth";
import { auth } from "@/config/firebase";
import { useNavigate } from "react-router-dom";

export default function LandingPage() {
  const [checking, setChecking] = useState(true);
  const navigate = useNavigate();

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      if (user) {
        navigate("/dashboard", { replace: true });
      } else {
        setChecking(false);
      }
    });
    return unsubscribe;
  }, [navigate]);

  if (checking) {
    return <PageSpinner />;
  }

  return (
    // Landing page content
  );
}
```

### Pattern 4: Nested Route Tree with Guard Layers

**What:** React Router v7 nested `<Route element={}>` for layered guards.
**When to use:** Multi-tier access control (public > auth > onboarding > paywall).
**Example:**
```typescript
// App.tsx route tree (per decision D-08)
<Routes>
  {/* Public routes -- outside all guards */}
  <Route path="/" element={<Suspense fallback={<PageSpinner />}><LandingPage /></Suspense>} />
  <Route path="/login" element={<LoginPage />} />

  {/* Authenticated routes */}
  <Route element={<ProtectedRoute />}>
    {/* Onboarding guard wraps onboarding routes */}
    <Route element={<OnboardingGuard />}>
      <Route path="/onboarding/*" element={...} />
    </Route>

    {/* Onboarding + Paywall guards wrap business routes */}
    <Route element={<OnboardingGuard />}>
      <Route element={<PaywallGuard />}>
        <Route element={<DashboardLayout />}>
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/customers" element={<CustomersPage />} />
          {/* ... other business routes */}
        </Route>
      </Route>
    </Route>
  </Route>
</Routes>
```

### Anti-Patterns to Avoid

- **Importing useAuth in LandingPage:** Breaks bundle isolation. The `useAuth` hook pulls in AuthContext, which depends on the provider tree, which pulls in Redux store. Use `onAuthStateChanged` directly.
- **Eager-importing LandingPage:** Without `React.lazy`, Vite bundles the landing page with the main app chunk, defeating the purpose of bundle isolation.
- **Modifying SubscriptionGatedLayout:** Decision D-07 explicitly prohibits touching the existing soft paywall. Phase 38 handles replacement.
- **Using `space-y-*` in landing page components:** shadcn skill rule prohibits this. Use `flex flex-col gap-*` instead. Note: the UI-SPEC mentions `space-y-3` in the pricing card -- implementors must translate this to `flex flex-col gap-3`.
- **Adding `w-*` `h-*` separately when equal:** Use `size-*` shorthand (e.g., `size-8` not `h-8 w-8`).
- **Adding sizing classes to icons inside Button:** shadcn uses `data-icon` attribute. Icons inside Button should have `data-icon="inline-start"` or `data-icon="inline-end"` with no sizing classes.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Code splitting | Manual dynamic import with state management | `React.lazy` + `<Suspense>` | React handles loading states, error boundaries, and concurrent mode automatically |
| Auth state detection | Custom Firebase polling or token checks | `onAuthStateChanged` listener | Firebase SDK handles token refresh, state persistence, and edge cases |
| Route guarding | Manual redirect logic in every page component | Nested layout routes with `<Outlet />` | React Router v7 handles this natively with layout route pattern |
| Responsive grid | Custom media query hooks | Tailwind responsive classes (`md:grid-cols-3`) | Tailwind's mobile-first responsive system is already configured |
| Conditional CSS | Template literal ternaries | `cn()` utility from `@/lib/utils` | Project convention; merges Tailwind classes correctly |

## Common Pitfalls

### Pitfall 1: Firebase Auth State Race Condition
**What goes wrong:** Landing page content flashes briefly before redirect for authenticated users.
**Why it happens:** `onAuthStateChanged` fires asynchronously. If the component renders before the callback, users see the landing page momentarily.
**How to avoid:** Start with `checking = true` state. Only render landing page content after auth state resolves as unauthenticated. Show spinner during check.
**Warning signs:** Authenticated users report seeing the landing page briefly on page load.

### Pitfall 2: Import Leakage Breaking Bundle Isolation
**What goes wrong:** A single import from `@/store` or `@/features/*` in any landing page sub-component pulls the entire app bundle into the landing page chunk.
**Why it happens:** Vite/Rollup follows the import graph. One transitive dependency is enough to defeat code splitting.
**How to avoid:** Strict import allowlist per D-10: own directory, `@/components/ui/*`, `@/config/firebase`, `lucide-react` only. Verify with `vite build` + chunk inspection.
**Warning signs:** Landing page chunk is unexpectedly large (should be < 20KB gzipped for static content + UI components).

### Pitfall 3: LoginPage mode Parameter Not Handled
**What goes wrong:** "Start Free Trial" CTA navigates to `/login?mode=signup` but LoginPage/AuthForm doesn't read the query param, so users land on the login tab instead of sign-up.
**Why it happens:** The existing LoginPage was built without query param awareness.
**How to avoid:** Read `mode` from `useSearchParams()` in LoginPage and pass it to AuthForm as the default active tab.
**Warning signs:** Users click "Start Free Trial" but see the login form instead of sign-up.

### Pitfall 4: Duplicate OnboardingGuard in Route Tree
**What goes wrong:** The route tree in D-08 shows two separate `<OnboardingGuard />` wrappers -- one for `/onboarding/*` and one for the business routes. If Phase 37 adds state-dependent redirect logic, both instances would need to stay in sync.
**Why it happens:** Onboarding routes and business routes need the same guard but with different redirect behaviors (onboarding routes should be accessible during onboarding, business routes should redirect to onboarding).
**How to avoid:** This is by design per D-08. Each OnboardingGuard instance will receive different logic in Phase 37. Keep them as separate components or use props. For Phase 36, both are pass-through so no issue.
**Warning signs:** None in Phase 36 -- this is a Phase 37 concern.

### Pitfall 5: UI-SPEC space-y vs shadcn Rule Conflict
**What goes wrong:** UI-SPEC uses `space-y-3` in pricing card feature list, but shadcn skill rules prohibit `space-y-*`.
**Why it happens:** UI-SPEC was written with standard Tailwind patterns, shadcn skill enforces `gap-*` pattern.
**How to avoid:** Translate `space-y-3` to `flex flex-col gap-3` during implementation. The visual result is identical.
**Warning signs:** shadcn skill lint or review catches `space-y-*` usage.

## Code Examples

### LandingPage.tsx (Lazy-loaded root)
```typescript
// Source: Project conventions + decisions D-09, D-10, D-14
import { lazy, Suspense, useEffect, useState } from "react";
import { onAuthStateChanged } from "firebase/auth";
import { auth } from "@/config/firebase";
import { useNavigate } from "react-router-dom";
import { Loader2 } from "lucide-react";

import { LandingHeader } from "./landing/LandingHeader";
import { HeroSection } from "./landing/HeroSection";
import { FeaturesSection } from "./landing/FeaturesSection";
import { PricingSection } from "./landing/PricingSection";
import { LandingFooter } from "./landing/LandingFooter";

export default function LandingPage() {
  const [checking, setChecking] = useState(true);
  const navigate = useNavigate();

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      if (user) {
        navigate("/dashboard", { replace: true });
      } else {
        setChecking(false);
      }
    });
    return unsubscribe;
  }, [navigate]);

  if (checking) {
    return (
      <div className="flex items-center justify-center min-h-screen bg-background">
        <Loader2 className="size-8 animate-spin text-muted-foreground" aria-label="Loading" />
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-background">
      <LandingHeader />
      <main>
        <HeroSection />
        <FeaturesSection />
        <PricingSection />
      </main>
      <LandingFooter />
    </div>
  );
}
```

### OnboardingGuard.tsx (Pass-through shell)
```typescript
// Source: Decision D-06
import { Outlet } from "react-router-dom";

export function OnboardingGuard() {
  return <Outlet />;
}
```

### App.tsx Route Tree (restructured)
```typescript
// Source: Decision D-08
import { lazy, Suspense } from "react";
import { Route, Routes } from "react-router-dom";
import { Loader2 } from "lucide-react";

const LandingPage = lazy(() => import("./pages/LandingPage"));

function PageSpinner() {
  return (
    <div className="flex items-center justify-center min-h-screen bg-background">
      <Loader2 className="size-8 animate-spin text-muted-foreground" aria-label="Loading" />
    </div>
  );
}

// Inside router:
<Routes>
  <Route path="/" element={<Suspense fallback={<PageSpinner />}><LandingPage /></Suspense>} />
  <Route path="/login" element={<LoginPage />} />

  <Route element={<ProtectedRoute />}>
    <Route element={<OnboardingGuard />}>
      {/* Onboarding routes -- Phase 37 adds content */}
    </Route>
    <Route element={<OnboardingGuard />}>
      <Route element={<PaywallGuard />}>
        <Route element={<DashboardLayout />}>
          {/* All existing business routes move here */}
        </Route>
      </Route>
    </Route>
  </Route>
</Routes>
```

### HeroSection.tsx (Example sub-component)
```typescript
// Source: UI-SPEC layout specification + shadcn icon rules
import { useNavigate } from "react-router-dom";
import { ArrowRight } from "lucide-react";
import { Button } from "@/components/ui/button";

export function HeroSection() {
  const navigate = useNavigate();

  return (
    <section className="py-16 md:py-24">
      <div className="mx-auto max-w-3xl px-4 text-center">
        <h1 className="text-4xl font-semibold tracking-tight md:text-5xl">
          Run your trade business in one place
        </h1>
        <p className="mx-auto mt-4 max-w-2xl text-lg text-muted-foreground">
          Jobs, quotes, schedules -- sorted. Built for plumbers, electricians,
          builders, and every trade in between.
        </p>
        <Button
          size="lg"
          className="mt-8"
          onClick={() => navigate("/login?mode=signup")}
        >
          Start Free Trial
          <ArrowRight data-icon="inline-end" />
        </Button>
      </div>
    </section>
  );
}
```

### LoginPage Modification (mode query param)
```typescript
// Source: Decision D-12
import { useSearchParams } from "react-router-dom";

// Inside LoginPage component:
const [searchParams] = useSearchParams();
const defaultMode = searchParams.get("mode") === "signup" ? "signup" : "login";

// Pass defaultMode to AuthForm as initial tab
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Route-level auth redirects in each page | Nested layout route guards with `<Outlet />` | React Router v6+ (2022+) | Centralized auth logic, cleaner route tree |
| Webpack dynamic imports | Vite native code splitting via `React.lazy` | Vite 2+ (2021+) | Zero-config code splitting, faster builds |
| `componentDidMount` auth checks | `useEffect` + `onAuthStateChanged` listener | React 16.8+ (2019+) | Cleanup via unsubscribe, no memory leaks |

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | No frontend test framework configured (UI repo has ESLint + TypeScript only) |
| Config file | None |
| Quick run command | `npx tsc --noEmit` (type checking) |
| Full suite command | `npx tsc --noEmit && npx eslint .` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| LAND-01 | Landing page renders at root URL without auth | manual | Navigate to `/` in browser, verify content | N/A |
| LAND-02 | Hero section with value proposition and CTA | manual | Visual inspection in browser | N/A |
| LAND-03 | Feature highlights for jobs, quotes, scheduling | manual | Visual inspection in browser | N/A |
| LAND-04 | Pricing section with GBP 6/month | manual | Visual inspection in browser | N/A |
| LAND-05 | Navigation to sign up or log in | manual | Click CTAs, verify /login and /login?mode=signup navigation | N/A |
| LAND-06 | Authenticated user redirected to dashboard | manual | Log in, visit `/`, verify redirect to `/dashboard` | N/A |
| -- | Bundle isolation | smoke | `npx vite build` + inspect chunk sizes in `dist/assets/` | N/A |
| -- | TypeScript compilation | static | `npx tsc --noEmit` | N/A |

### Sampling Rate
- **Per task commit:** `npx tsc --noEmit`
- **Per wave merge:** `npx tsc --noEmit && npx eslint .`
- **Phase gate:** Full build (`npx vite build`) + manual chunk inspection + browser verification

### Wave 0 Gaps
None -- no automated test infrastructure for frontend UI. All verification is manual + type checking + linting. This is consistent with project conventions (frontend has static analysis only per CLAUDE.md).

## Sources

### Primary (HIGH confidence)
- CONTEXT.md (36-CONTEXT.md) -- Locked decisions D-01 through D-14
- UI-SPEC (36-UI-SPEC.md) -- Visual contract, component inventory, layout specifications
- REQUIREMENTS.md -- LAND-01 through LAND-06 requirement definitions
- STRUCTURE.md -- Existing frontend directory structure, existing components
- Phase 32 CONTEXT.md -- SubscriptionGatedLayout pattern, existing route structure

### Secondary (MEDIUM confidence)
- [React Router v7 Protected Routes](https://www.robinwieruch.de/react-router-private-routes/) -- Layout route guard pattern with Outlet
- [React Code Splitting docs](https://legacy.reactjs.org/docs/code-splitting.html) -- React.lazy + Suspense pattern
- [Vite Code Splitting](https://sambitsahoo.com/blog/vite-code-splitting-that-works.html) -- Vite automatic chunk splitting behavior

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and in use
- Architecture: HIGH -- patterns directly specified in locked decisions with code examples
- Pitfalls: HIGH -- based on known React/Firebase/Vite behaviors and project-specific constraints

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable -- no fast-moving dependencies)
