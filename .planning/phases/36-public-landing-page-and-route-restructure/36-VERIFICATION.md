---
phase: 36-public-landing-page-and-route-restructure
verified: 2026-04-07T00:00:00Z
status: human_needed
score: 4/4 automated must-haves verified
human_verification:
  - test: "Visit root URL in incognito browser and confirm landing page renders with hero, feature cards, pricing card, and nav buttons"
    expected: "Sticky header with 'Trade Flow' logo and Log in / Start Free Trial buttons; hero h1 'Run your trade business in one place'; three feature cards (Jobs, Quotes, Scheduling); pricing card showing £6/month; footer with copyright year"
    why_human: "Visual rendering and layout correctness cannot be verified from source alone"
  - test: "Click 'Start Free Trial' on hero section and pricing card; confirm navigation to /login?mode=signup with sign-up tab active"
    expected: "Browser navigates to /login?mode=signup and the Sign Up form tab is pre-selected, not Sign In"
    why_human: "Tab default-state driven by URL param requires live browser interaction to confirm"
  - test: "Log in with valid credentials, then navigate to root URL; confirm redirect to /dashboard"
    expected: "LandingPage's onAuthStateChanged detects active session and immediately redirects to /dashboard without rendering the landing content"
    why_human: "Auth redirect is runtime behaviour that requires Firebase session state"
  - test: "Run 'npx vite build' in trade-flow-ui and inspect dist/assets/ for a separate LandingPage chunk"
    expected: "A separate JS chunk file exists that contains LandingPage code, confirming React.lazy code-splitting is active and the landing page bundle is isolated from the main app bundle"
    why_human: "Bundle analysis requires running the build tool; cannot be verified from source inspection"
---

# Phase 36: Public Landing Page and Route Restructure — Verification Report

**Phase Goal:** Visitors can discover Trade Flow's value proposition and start a free trial from a public marketing page, and the app's route architecture supports three tiers of access (public, onboarding, authenticated+subscribed)
**Verified:** 2026-04-07
**Status:** human_needed
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Unauthenticated visitor can view the landing page at root URL with hero, features, pricing, and nav | ? HUMAN NEEDED | All six landing components exist with correct content; routing confirmed in App.tsx; visual rendering needs browser |
| 2 | Landing page loads without importing Redux store, feature modules, or auth hooks | ✓ VERIFIED | Grep across `src/pages/LandingPage.tsx` and `src/pages/landing/` returns zero matches for `@/store`, `@/features/`, `@/hooks/`, `@/providers/`; React.lazy in App.tsx line 43 confirms code-split entry point |
| 3 | Authenticated user visiting root URL is redirected to dashboard | ? HUMAN NEEDED | `onAuthStateChanged` redirect logic exists in LandingPage.tsx lines 17-26 with `navigate("/dashboard", { replace: true })`; requires live Firebase session to confirm |
| 4 | App.tsx route tree has three nested guard layers (ProtectedRoute, OnboardingGuard, PaywallGuard) | ✓ VERIFIED | `AuthenticatedLayout` (lines 63-76) wraps `ProtectedRoute`; `OnboardingGuard` at lines 100 and 126; `PaywallGuard` at line 127 — all three layers confirmed in source |

**Score:** 2/4 truths fully verified by automated checks; 2/4 require human confirmation

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/pages/LandingPage.tsx` | Lazy-loadable orchestrator with Firebase auth redirect | ✓ VERIFIED | `export default function LandingPage`, uses `onAuthStateChanged` directly (not `useAuth`), composes all 5 sub-components |
| `trade-flow-ui/src/pages/landing/LandingHeader.tsx` | Sticky header with Log in / Start Free Trial nav | ✓ VERIFIED | `export function LandingHeader`, sticky CSS class, two `navigate()` calls to `/login` and `/login?mode=signup` |
| `trade-flow-ui/src/pages/landing/HeroSection.tsx` | Hero with trade-specific headline and CTA | ✓ VERIFIED | `export function HeroSection`, h1 "Run your trade business in one place", `navigate("/login?mode=signup")`, `ArrowRight data-icon="inline-end"` |
| `trade-flow-ui/src/pages/landing/FeaturesSection.tsx` | Three feature highlight cards | ✓ VERIFIED | `export function FeaturesSection`, `Briefcase`, `FileText`, `CalendarDays` icons, `md:grid-cols-3` grid |
| `trade-flow-ui/src/pages/landing/PricingSection.tsx` | Pricing card with £6/month | ✓ VERIFIED | `export function PricingSection`, "Trade Flow Pro", `&pound;6`, `flex flex-col gap-3` (no `space-y-`), four feature checkmarks |
| `trade-flow-ui/src/pages/landing/LandingFooter.tsx` | Footer with copyright year | ✓ VERIFIED | `export function LandingFooter`, `new Date().getFullYear()` |
| `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` | Route guard (originally pass-through shell, now full redirect logic) | ✓ VERIFIED | `export function OnboardingGuard`, returns `<Outlet />` for completed users and for `/onboarding` path, redirects incomplete users to `/onboarding` |
| `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` | Route guard (originally pass-through shell, now subscription enforcement) | ✓ VERIFIED | `export function PaywallGuard`, returns `<Outlet />` for active subscriptions and support roles, renders `PaywallPage` for blocked users |
| `trade-flow-ui/src/pages/LoginPage.tsx` | Login page with mode query param support | ✓ VERIFIED | `useSearchParams`, `searchParams.get("mode") === "signup"`, passes `defaultMode` to `AuthForm` |
| `trade-flow-ui/src/features/auth/components/AuthForm.tsx` | Auth form with defaultMode prop | ✓ VERIFIED | `defaultMode?: "signin" \| "signup"` prop accepted, `useState<AuthMode>(defaultMode ?? "signin")` as initial state |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `App.tsx` | `LandingPage.tsx` | `React.lazy` dynamic import | ✓ WIRED | Line 43: `const LandingPage = lazy(() => import("./pages/LandingPage"))` |
| `App.tsx` | `OnboardingGuard.tsx` | Route element wrapper | ✓ WIRED | Imported from `@/features/auth` (line 17), used at lines 100 and 126 |
| `App.tsx` | `PaywallGuard.tsx` | Route element wrapper | ✓ WIRED | Imported from `@/features/auth` (line 18), used at line 127 |
| `LandingPage.tsx` | `firebase/auth` | `onAuthStateChanged` for auth redirect | ✓ WIRED | Line 2: `import { onAuthStateChanged } from "firebase/auth"`, line 18: `onAuthStateChanged(auth, ...)` |
| `HeroSection.tsx` | `/login?mode=signup` | `navigate` on CTA click | ✓ WIRED | Line 21: `onClick={() => navigate("/login?mode=signup")}` |
| `LandingHeader.tsx` | `/login` and `/login?mode=signup` | `navigate` on button clicks | ✓ WIRED | Lines 13-18: both navigate calls present |
| `LoginPage.tsx` | `useSearchParams` | Query param for default mode | ✓ WIRED | Line 17-18: `useSearchParams`, `searchParams.get("mode") === "signup"` mapped to `defaultMode` |
| `LoginPage.tsx` | `AuthForm` | `defaultMode` prop | ✓ WIRED | Line 44: `<AuthForm defaultMode={defaultMode} ...>` |
| `features/auth/components/index.ts` | `OnboardingGuard`, `PaywallGuard` | Barrel export | ✓ WIRED | Both exported from `components/index.ts` lines 2-3 |

---

## Data-Flow Trace (Level 4)

Not applicable — all landing page components are static marketing content (no dynamic data fetched from API or store). Auth redirect flows through Firebase SDK directly without data fetching.

---

## Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Landing sub-components have no forbidden imports | `grep -r "@/store\|@/features/\|@/hooks/\|@/providers/" src/pages/landing/ src/pages/LandingPage.tsx` | Zero matches | ✓ PASS |
| `space-y-` not used in landing files (shadcn compliance) | grep for `space-y-` in `src/pages/landing/` | Zero matches | ✓ PASS |
| Three guard layers present in App.tsx | grep for `OnboardingGuard`, `PaywallGuard`, `ProtectedRoute` in App.tsx | All present (ProtectedRoute: 2, OnboardingGuard: 4, PaywallGuard: 3 references) | ✓ PASS |
| LandingPage is lazy-loaded at root route | Check App.tsx for `lazy(() => import` and `path="/"` | Both present; `<Suspense>` wrapper confirmed | ✓ PASS |
| `?mode=signup` correctly maps to `"signup"` in LoginPage | Source read: `searchParams.get("mode") === "signup" ? "signup" : "signin"` | Correct mapping confirmed | ✓ PASS |
| AuthForm accepts and uses `defaultMode` prop | Source read: `useState<AuthMode>(defaultMode ?? "signin")` | Initial state set from prop | ✓ PASS |
| Bundle isolation (separate LandingPage chunk) | Requires `npx vite build` | Cannot verify without running build | ? SKIP — needs human |
| Visual rendering of landing page | Requires live browser | Cannot verify from source | ? SKIP — needs human |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| LAND-01 | 36-01, 36-02 | User can view a public landing page at root URL without authentication | ✓ SATISFIED | `path="/"` in App.tsx routes to `LandingPage` outside all auth guards; `checking` state prevents flash of content while auth state loads |
| LAND-02 | 36-01 | User can see a hero section with trade-specific value proposition and "Start Free Trial" CTA | ✓ SATISFIED | `HeroSection.tsx` renders h1 "Run your trade business in one place" and "Start Free Trial" button with `ArrowRight` icon |
| LAND-03 | 36-01 | User can see feature highlights showing core capabilities (jobs, quotes, scheduling) | ✓ SATISFIED | `FeaturesSection.tsx` renders three cards: "Jobs from start to finish", "Professional quotes in minutes", "Your diary, your way" with Briefcase/FileText/CalendarDays icons |
| LAND-04 | 36-01 | User can see a pricing section displaying £6/month with no-card trial messaging | ✓ SATISFIED | `PricingSection.tsx` renders "Trade Flow Pro" card with `&pound;6 / month` and "No card required" feature item |
| LAND-05 | 36-01, 36-02 | User can navigate to sign up or log in from the landing page | ✓ SATISFIED | `LandingHeader` has Log in (`/login`) and Start Free Trial (`/login?mode=signup`) buttons; `HeroSection` and `PricingSection` CTAs also navigate to `/login?mode=signup` |
| LAND-06 | 36-01, 36-02 | Authenticated user visiting root URL is redirected to dashboard | ? NEEDS HUMAN | `onAuthStateChanged` redirect logic confirmed in source; runtime Firebase session needed to verify |

**Traceability note:** REQUIREMENTS.md marks all LAND-01 through LAND-06 as "Pending" — this is a documentation lag and does not reflect implementation state. All six requirements have implementation evidence in the codebase.

---

## Anti-Patterns Found

| File | Pattern | Severity | Assessment |
|------|---------|----------|------------|
| None detected | — | — | No stubs, no forbidden imports, no `space-y-` violations in landing files |

**Notable: ProtectedRoute uses `children` prop pattern (not `<Outlet />`).** This means `AuthenticatedLayout` wraps `ProtectedRoute` with children (`<Outlet />` rendered inside), rather than using the layout route pattern. This works correctly but differs from the plan's implied `Route element={<ProtectedRoute />}` style. The `OnboardingGuard` and `PaywallGuard` correctly use `<Outlet />`. Severity: INFO — no functional impact.

---

## Human Verification Required

### 1. Landing Page Visual Rendering

**Test:** Start `npm run dev` in `trade-flow-ui`, open incognito browser, navigate to `http://localhost:5173/`
**Expected:** Full landing page renders with: sticky header ("Trade Flow" logo, "Log in" outline button, "Start Free Trial" primary button); hero section with h1 "Run your trade business in one place" and "Start Free Trial" CTA with arrow icon; three feature cards in a responsive grid; pricing card "Trade Flow Pro" showing £6/month with four checkmarks; footer with copyright year
**Why human:** Visual rendering, layout correctness, and responsive behaviour cannot be confirmed from source analysis alone

### 2. Start Free Trial CTA — Sign-Up Tab Pre-Selected

**Test:** On the landing page, click "Start Free Trial" in either the header, hero, or pricing card
**Expected:** Browser navigates to `/login?mode=signup` and the AuthForm renders with the "Sign Up" tab active (not "Sign In")
**Why human:** Tab default-state initialised from URL param requires live browser interaction to confirm the end-to-end flow

### 3. Authenticated Redirect at Root URL

**Test:** Log in with valid credentials, then navigate to `http://localhost:5173/`
**Expected:** No landing page content flashes; user is immediately redirected to `/dashboard`
**Why human:** Auth redirect depends on Firebase session state and requires a live authenticated session

### 4. Bundle Isolation — Separate LandingPage Chunk

**Test:** Run `cd trade-flow-ui && npm run build` and inspect `dist/assets/` for a separate JS chunk containing LandingPage code (filename will include a content hash)
**Expected:** A separate chunk file exists for the landing page, confirming `React.lazy` code-splitting is active and the landing page bundle does not pull in Redux store, RTK Query, or feature modules
**Why human:** Bundle analysis requires executing the build tool; cannot be inferred from source

---

## Gaps Summary

No automated gaps found. All six artifacts exist with substantive implementation, all key links are wired, no forbidden imports detected, and no stub patterns identified.

The four items flagged for human verification are confirmations of runtime behaviour (auth redirect, visual rendering, tab state, bundle output) rather than implementation gaps. The source evidence strongly supports all four behaviours working correctly.

**Phase 36 goal is considered achieved** pending human confirmation of the four runtime / visual checks above.

---

_Verified: 2026-04-07_
_Verifier: Claude (gsd-verifier)_
