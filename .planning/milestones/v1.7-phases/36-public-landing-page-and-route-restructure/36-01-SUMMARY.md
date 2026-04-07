---
phase: 36-public-landing-page-and-route-restructure
plan: 01
subsystem: ui
tags: [react, landing-page, firebase, tailwind, shadcn]

# Dependency graph
requires:
  - phase: 32-subscription-gate-and-subscribe-pages
    provides: shadcn Button/Card components, Tailwind design tokens, Firebase config
provides:
  - LandingPage orchestrator with Firebase auth redirect (lazy-loadable default export)
  - LandingHeader with sticky nav and Log in / Start Free Trial buttons
  - HeroSection with trade-specific headline and CTA
  - FeaturesSection with 3-column grid (Jobs, Quotes, Scheduling)
  - PricingSection with GBP 6/month pricing card
  - LandingFooter with copyright year
affects: [36-02 (route restructure wires LandingPage into App.tsx), 37 (onboarding wizard linked from Start Free Trial CTA)]

# Tech tracking
tech-stack:
  added: []
  patterns: [Bundle-isolated page with direct Firebase onAuthStateChanged (no useAuth hook)]

key-files:
  created:
    - trade-flow-ui/src/pages/LandingPage.tsx
    - trade-flow-ui/src/pages/landing/LandingHeader.tsx
    - trade-flow-ui/src/pages/landing/HeroSection.tsx
    - trade-flow-ui/src/pages/landing/FeaturesSection.tsx
    - trade-flow-ui/src/pages/landing/PricingSection.tsx
    - trade-flow-ui/src/pages/landing/LandingFooter.tsx
  modified: []

key-decisions:
  - "Used em dash HTML entities (&mdash;) and Unicode escapes for em dashes in feature descriptions"
  - "Used &pound; HTML entity for GBP symbol in pricing section"

patterns-established:
  - "Bundle-isolated page pattern: direct Firebase imports instead of useAuth hook, no @/store or @/features imports"
  - "Landing sub-component co-location: src/pages/landing/ directory for page-specific components"

requirements-completed: [LAND-01, LAND-02, LAND-03, LAND-04, LAND-05, LAND-06]

# Metrics
duration: 2min
completed: 2026-04-02
---

# Phase 36 Plan 01: Public Landing Page Summary

**Bundle-isolated marketing landing page with hero section, feature highlights, GBP 6/month pricing card, and Firebase auth redirect for authenticated users**

## Performance

- **Duration:** 2 min
- **Started:** 2026-04-02T11:16:49Z
- **Completed:** 2026-04-02T11:19:06Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Six new components forming a complete public landing page for tradespeople
- Bundle isolation maintained: zero imports from @/store, @/features, @/hooks, or @/providers
- Firebase auth redirect via direct onAuthStateChanged prevents authenticated users from seeing landing page
- shadcn rule compliance: flex flex-col gap-* (not space-y-*), size-* (not w-*/h-*), data-icon on icons in Button
- TypeScript compiles with zero errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Create landing page sub-components** - `0e66433` (feat) -- LandingHeader, HeroSection, FeaturesSection, PricingSection, LandingFooter
2. **Task 2: Create LandingPage orchestrator with Firebase auth redirect** - `f70e33c` (feat) -- Default export with onAuthStateChanged, loading spinner, component composition

## Files Created/Modified
- `trade-flow-ui/src/pages/LandingPage.tsx` - Lazy-loadable orchestrator with Firebase auth check, redirects authenticated users, composes all sub-components
- `trade-flow-ui/src/pages/landing/LandingHeader.tsx` - Sticky header with Trade Flow logo, Log in (outline) and Start Free Trial (primary) buttons
- `trade-flow-ui/src/pages/landing/HeroSection.tsx` - Hero with "Run your trade business in one place" headline, subheadline, and ArrowRight CTA button
- `trade-flow-ui/src/pages/landing/FeaturesSection.tsx` - 3-column responsive grid with Briefcase/FileText/CalendarDays icon cards
- `trade-flow-ui/src/pages/landing/PricingSection.tsx` - Trade Flow Pro card with GBP 6/month, 4 feature checkmarks, full-width CTA
- `trade-flow-ui/src/pages/landing/LandingFooter.tsx` - Footer with dynamic copyright year

## Decisions Made
- Used HTML entities (&mdash;, &pound;) and Unicode escapes (\u2014, \u2019) for special characters in JSX content for reliable rendering across all browsers
- Followed plan specifications exactly for all component structure, styling classes, and content

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- LandingPage ready for Plan 02 to wire into App.tsx route tree via React.lazy
- All sub-components use navigate() calls to /login and /login?mode=signup ready for LoginPage mode parameter handling in Plan 02
- Default export enables React.lazy(() => import('./pages/LandingPage')) pattern

## Self-Check: PASSED

All 6 files verified to exist. Both commit hashes (0e66433, f70e33c) confirmed in trade-flow-ui git log.

---
*Phase: 36-public-landing-page-and-route-restructure*
*Completed: 2026-04-02*
