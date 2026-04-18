---
status: resolved
trigger: "Stuck on onboarding screens (user name or business details) with no way to logout or go back. Refreshing doesn't help. Manually typing login URL doesn't work either. Intermittent -- seems tied to older datasets in testing environment."
created: 2026-04-17
updated: 2026-04-17
---

# Debug: Onboarding Stuck - No Escape

## Symptoms

- **Expected:** A logout button or back-to-login option should always be available on onboarding screens
- **Actual:** Page loads but there is no way to navigate away from onboarding (no logout, no back button)
- **Errors:** Haven't checked browser console
- **Timeline:** Only seems to be an issue on older datasets in the testing environment
- **Reproduction:** Intermittent -- not reliably reproducible, possibly triggered by stale/incomplete user or business data

## Current Focus

- hypothesis: OnboardingWizard has no logout button; OnboardingGuard traps authenticated users whose name or businessRoles is missing/empty
- test: Verified OnboardingGuard redirects to /onboarding when !hasDisplayName || !hasBusiness; OnboardingWizard renders no logout/escape UI
- expecting: Confirmed
- result: Confirmed -- two root causes identified and fixed
- next_action: none (resolved)
- reasoning_checkpoint: Both fixes applied, lint + typecheck + 34 tests passing

## Evidence

- timestamp: 2026-04-17 OnboardingGuard.tsx lines 28-42: redirects to /onboarding if !hasDisplayName || !hasBusiness; allows stay if already on /onboarding path
- timestamp: 2026-04-17 OnboardingWizard.tsx: renders only ProfileStep and BusinessStep inside a Card -- no logout button, no "back to login" link, no escape mechanism
- timestamp: 2026-04-17 useCurrentBusiness.ts line 42: hasBusiness = Boolean(user?.businessRoles[0]?.businessId) -- if businessRoles is empty array or missing, user is trapped in onboarding
- timestamp: 2026-04-17 DashboardLayout.tsx has signOut functionality; PaywallPage has logout handler; OnboardingWizard has neither
- timestamp: 2026-04-17 App.tsx route structure: /login is outside ProtectedRoute, but catch-all redirects to /dashboard which goes through OnboardingGuard -> redirect to /onboarding
- timestamp: 2026-04-17 OnboardingWizard useState(initialStep) bug: initialStep is always "profile" on first render because isLoading=true returns "profile" default; useState captures only initial value

## Root Cause

**PRIMARY: No logout/escape mechanism on onboarding screens.** The `OnboardingWizard` component renders only the onboarding steps with no logout button, no "back to login" link, and no way to exit. The `OnboardingGuard` correctly identifies incomplete users and redirects them to `/onboarding`, but once there, the user is trapped because:

1. There is no logout button on the onboarding page
2. The ProtectedRoute wraps the onboarding route, so the user is authenticated
3. Navigating to `/login` hits the catch-all `*` route which redirects to `/dashboard`, which hits OnboardingGuard, which redirects back to `/onboarding`
4. Refreshing reloads the same trapped state

**SECONDARY: `useState(initialStep)` stale initialization.** In `OnboardingWizard`, `useState<WizardStep>(initialStep)` captures the value at first render. Since `useOnboardingStep` returns `isLoading: true` initially (with `initialStep: "profile"` as default), the wizard always starts at "profile" even if the user already has a name and should start at "business". This causes users who completed the profile step to re-see it.

**TRIGGER FOR OLDER DATASETS:** Users in older test environments may have Firebase auth accounts but incomplete user records in MongoDB (missing `name` field or empty `businessRoles` array). The OnboardingGuard correctly detects this as "needs onboarding", but the lack of a logout button means these users cannot escape.

## Resolution

- **root_cause:** OnboardingWizard had no logout button, trapping authenticated users with incomplete profiles in an inescapable onboarding loop
- **fix:** Added "Sign out" button to OnboardingWizard; refactored to split into OnboardingWizard (loading gate) and OnboardingWizardContent (stateful wizard) so useState(initialStep) receives the correct value after data loads
- **files_changed:** trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx
- **verification:** TypeScript compiles clean, ESLint passes, all 34 onboarding tests pass

## Eliminated
