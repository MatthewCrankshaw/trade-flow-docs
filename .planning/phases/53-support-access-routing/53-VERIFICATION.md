---
phase: 53-support-access-routing
verified: 2026-04-19T07:30:00Z
status: human_needed
score: 8/8 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Support user login redirects to /support"
    expected: "After signing in with a support-role account, browser lands on /support (not /dashboard or /onboarding)"
    why_human: "RTK Query post-login useEffect and Firebase auth callback sequence cannot be exercised without a running browser session"
  - test: "Non-support user navigating to /support is redirected to /dashboard"
    expected: "Visiting /support as a regular business user shows /dashboard"
    why_human: "SupportGuard redirect behaviour requires live auth + RTK Query response with supportRoles=[] to confirm the redirect fires"
  - test: "Support sidebar renders only Dashboard, Users, Businesses"
    expected: "Logged-in support user sees exactly three nav items; no Jobs, Customers, Quotes, My Business, etc."
    why_human: "NavContent role-conditional branch requires live user.supportRoles data in browser"
  - test: "Business user sidebar has no support items"
    expected: "Business user sees Main + Business sections; no Support section at all"
    why_human: "Same conditional rendering, requires live session"
  - test: "Logo link is role-aware"
    expected: "Support user logo links to /support; business user logo links to /dashboard"
    why_human: "Requires live sessions for both user types"
---

# Phase 53: Support Access Routing Verification Report

**Phase Goal:** Support users can log in and reach a dedicated /support dashboard without going through onboarding or needing a business association
**Verified:** 2026-04-19T07:30:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | SupportGuard component exists and checks supportRoles.length > 0 | VERIFIED | `SupportGuard.tsx` line 17: `if (!user \|\| user.supportRoles.length === 0)` |
| 2 | SupportGuard renders Outlet for support users and Navigate to /dashboard for non-support users | VERIFIED | Lines 18-21: `<Navigate to="/dashboard" replace />` and `<Outlet />` |
| 3 | SupportGuard shows loading spinner while user data loads | VERIFIED | Lines 9-15: isLoading renders full-screen Loader2 spinner with `min-h-screen` |
| 4 | Navigation config provides separate getSupportNavigationItems() and getBusinessNavigationItems() | VERIFIED | Both functions exported from `navigation.ts` lines 25 and 88 |
| 5 | Support nav items are Dashboard (/support), Users (/support/users), Businesses (/support/businesses) | VERIFIED | `navigation.ts` lines 93-95 with correct hrefs, titles, and icons (LayoutDashboard, Users, Building2) |
| 6 | Support user logging in is redirected to /support, not /dashboard or /onboarding | VERIFIED (code path) | `LoginPage.tsx` lines 39-41: `if (appUser.supportRoles.length > 0) navigate("/support", ...)` |
| 7 | Support routes are outside OnboardingGuard and PaywallGuard branches | VERIFIED | `App.tsx` lines 101-106: `<Route element={<SupportGuard />}>` is a sibling of `<Route element={<OnboardingGuard />}>`, not nested within it |
| 8 | Support pages no longer have per-page role checks | VERIFIED | `grep hasSupportRole` returns no matches in support pages; `Navigate to="/dashboard"` absent from support pages |

**Score:** 8/8 truths verified (automated code inspection)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/features/auth/components/SupportGuard.tsx` | Route guard for support routes | VERIFIED | 22 lines, substantive — loading/guard/redirect/outlet all present |
| `trade-flow-ui/src/features/auth/components/index.ts` | Barrel export including SupportGuard | VERIFIED | Line 5: `export { SupportGuard } from "./SupportGuard"` |
| `trade-flow-ui/src/config/navigation.ts` | Split navigation config | VERIFIED | Both `getSupportNavigationItems` and `getBusinessNavigationItems` exported; `getNavigationItems` removed |
| `trade-flow-ui/src/App.tsx` | Restructured route tree with SupportGuard branch | VERIFIED | Lines 101-106: SupportGuard wraps /support, /support/users, /support/businesses outside OnboardingGuard/PaywallGuard |
| `trade-flow-ui/src/pages/LoginPage.tsx` | Role-aware post-login redirect | VERIFIED | AlreadyAuthenticatedRedirect component + useEffect pattern with supportRoles check |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` | Role-conditional navigation rendering | VERIFIED | Line 20 imports both nav functions; line 49 applies conditional selection; lines 127/183 role-aware logoHref |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| SupportGuard.tsx | @/services | useGetCurrentUserQuery | WIRED | Import line 4, called line 7 |
| App.tsx | SupportGuard | route element import | WIRED | Line 9 imports from `@/features/auth`; line 101 used as `<Route element={<SupportGuard />}>` |
| LoginPage.tsx | @/services | useGetCurrentUserQuery for role check | WIRED | Line 10 import, line 23 used with skip pattern, line 39 reads supportRoles |
| DashboardLayout.tsx | navigation.ts | getSupportNavigationItems/getBusinessNavigationItems | WIRED | Line 20 imports both; line 49 conditional call |
| navigation.ts | lucide-react | icon imports | WIRED | Lines 1-11: LayoutDashboard, Users, Building2 imported and used in getSupportNavigationItems |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| SupportGuard.tsx | user (supportRoles) | useGetCurrentUserQuery → API /v1/users/me | Yes — RTK Query fetches from API | FLOWING |
| LoginPage.tsx | appUser (supportRoles) | useGetCurrentUserQuery with skip:!loginSucceeded | Yes — triggered post-login | FLOWING |
| DashboardLayout.tsx NavContent | sections | getSupportNavigationItems/getBusinessNavigationItems based on user.supportRoles | Static config (correct: nav items are static definitions, not DB data) | FLOWING |
| SupportBusinessesPage.tsx | businesses table | mockBusinesses (hardcoded array, 4 entries) | No — hardcoded mock data | HOLLOW (stub data, noted below) |

**Note on SupportBusinessesPage:** The businesses table renders hardcoded mock data (4 UK businesses). This is a pre-existing stub for the businesses list feature — the actual API-backed businesses list is Phase 54 scope (UMGT requirements). The routing and guard wiring are real; only the content is placeholder. This does not block the Phase 53 goal (routing and access control).

### Behavioral Spot-Checks

Step 7b: SKIPPED — requires running browser session for Firebase auth flows. Static code analysis is sufficient for routing structure verification.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| SACC-01 | 53-01, 53-02 | Support user bypasses onboarding entirely | SATISFIED | Support routes sit inside `<Route element={<SupportGuard />}>` which is a sibling (not child) of both OnboardingGuard branches in App.tsx |
| SACC-02 | 53-02 | Support user redirected to /support after login | SATISFIED (code path) | LoginPage useEffect navigates to /support when `appUser.supportRoles.length > 0`; AlreadyAuthenticatedRedirect does the same for already-auth users |
| SACC-03 | 53-01, 53-02 | /support/* routes protected, only accessible to support roles | SATISFIED | SupportGuard checks `user.supportRoles.length === 0` and redirects to /dashboard; wraps all three support routes in App.tsx |
| SACC-04 | 53-02 | Support user bypasses subscription gating | SATISFIED | Support routes are structurally outside the `<Route element={<PaywallGuard />}>` branch; no PaywallGuard in the support path |

All four requirements are structurally satisfied in code. SACC-02 live redirect behaviour needs human confirmation (listed in Human Verification Required).

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| SupportBusinessesPage.tsx | 12-54 | `mockBusinesses` hardcoded array | Info | Table shows fake data — this is placeholder content for Phase 54 (UMGT-01); does not affect routing or access control for Phase 53 |

No TODOs, FIXME comments, or empty implementations found in Phase 53 modified files. The mock data in SupportBusinessesPage predates Phase 53 and is outside its scope.

### Human Verification Required

#### 1. Post-Login Redirect to /support

**Test:** Log in as a support user (an account with `supportRoles` populated) using the sign-in form.
**Expected:** After Firebase auth succeeds and RTK Query resolves user data, the browser navigates to `/support` — not `/dashboard` or `/onboarding`.
**Why human:** The LoginPage useEffect fires after RTK Query's async resolution; cannot verify timing without a live browser session.

#### 2. Non-Support User Blocked from /support

**Test:** Log in as a regular business user, then manually navigate to `/support` in the URL bar.
**Expected:** Browser immediately redirects to `/dashboard` (SupportGuard fires).
**Why human:** Requires a live session with an authenticated non-support user.

#### 3. Support Sidebar Content

**Test:** As a support user, observe the sidebar navigation.
**Expected:** Exactly three items visible: "Dashboard", "Users", "Businesses". No Jobs, Customers, Quotes, My Business, Settings, or other business items.
**Why human:** NavContent's conditional branch requires live user data in React state.

#### 4. Business User Sidebar

**Test:** As a business user, observe the sidebar navigation.
**Expected:** Main section (Dashboard, Jobs, Customers, Quotes, Estimates, Items) and Business section (My Business, Settings). No "Support" section.
**Why human:** Same conditional rendering, opposite branch.

#### 5. Role-Aware Logo Link

**Test:** As support user, click the Trade Flow logo. As business user, click the logo.
**Expected:** Support user logo navigates to `/support`; business user logo navigates to `/dashboard`.
**Why human:** Requires live sessions for both roles.

### Gaps Summary

No gaps blocking the Phase 53 goal. All routing structure, guard logic, and navigation wiring are verified in code. The phase goal — "support users can log in and reach a dedicated /support dashboard without going through onboarding or needing a business association" — is structurally implemented.

Status is `human_needed` because the end-to-end login flow (SACC-02 live behaviour) and visual confirmation of role-conditional navigation require browser testing. The SUMMARY.md documents human checkpoint APPROVED from original execution, but a fresh verification is needed to confirm the implementation in the current state of the codebase.

---

_Verified: 2026-04-19T07:30:00Z_
_Verifier: Claude (gsd-verifier)_
