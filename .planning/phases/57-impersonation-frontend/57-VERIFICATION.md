---
phase: 57-impersonation-frontend
verified: 2026-04-19T18:00:00Z
status: human_needed
score: 12/12 must-haves verified
overrides_applied: 0
re_verification: false
human_verification:
  - test: "Full impersonation flow end-to-end (requires Phase 56 backend)"
    expected: "Navigate to /support/users/{customerId}, click Impersonate User, enter reason, click Start Impersonation — amber banner appears at top with target user name and Return to Support button; sidebar shows customer navigation (Jobs, Customers, Quotes, etc.); banner stays fixed while scrolling and above modals"
    why_human: "Visual layout and DOM stacking (z-index behaviour, fixed positioning, scroll behaviour) cannot be verified programmatically. Requires Phase 56 backend to be running for API calls to succeed."
  - test: "Return to Support button flow"
    expected: "Click Return to Support during active impersonation — navigates to /support, banner disappears, toast shows 'Impersonation session ended'"
    why_human: "Navigation and toast rendering are runtime behaviours. Requires Phase 56 backend."
  - test: "Button visibility gating on user detail page"
    expected: "Impersonate User button is visible for customer users (no support roles), hidden entirely for support users. Button is also hidden when impersonation is already active."
    why_human: "Requires real user data from Phase 56 backend to confirm supportRoleIds check works correctly with live data."
  - test: "Empty reason validation"
    expected: "Clicking Start Impersonation with empty reason textarea shows inline error 'A reason is required before impersonating a user.' below the textarea; dialog stays open"
    why_human: "Form interaction and inline validation rendering must be confirmed visually."
  - test: "Page refresh during impersonation"
    expected: "Refreshing the page terminates the impersonation session (Redux state is in-memory only, not persisted); support user returns to their own context"
    why_human: "Redux-only session persistence behaviour must be confirmed via browser test."
---

# Phase 57: Impersonation Frontend Verification Report

**Phase Goal:** A support user can impersonate a customer and see exactly what that customer sees, with a persistent banner and a clean exit back to the support dashboard
**Verified:** 2026-04-19T18:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| #  | Truth | Status | Evidence |
|----|-------|--------|----------|
| 1  | When impersonation is active, all API calls use the impersonation token instead of the Firebase token | VERIFIED | `api.ts` prepareHeaders reads `state.impersonation.active && state.impersonation.token` and sets Bearer header from Redux state |
| 2  | When impersonation is not active, API calls use the Firebase token as before | VERIFIED | `api.ts` prepareHeaders falls through to `auth.currentUser.getIdToken()` when impersonation not active |
| 3  | When the backend returns 401 during active impersonation, the impersonation state is cleared and the RTK Query cache is reset | VERIFIED | `baseQueryWithImpersonationExpiry` dispatches `endImpersonation()` + `{ type: "api/resetApiState" }` on 401 |
| 4  | The impersonation slice stores token, sessionId, targetUser (id, name, email), and active flag | VERIFIED | `impersonationSlice.ts` defines `ImpersonationState` with all four fields; `startImpersonation` populates all, `endImpersonation` resets all |
| 5  | A fixed amber banner appears at the top of the viewport during impersonation showing the target user's name and a Return to Support button | VERIFIED | `ImpersonationBanner.tsx` renders with `fixed top-0 left-0 right-0 z-[60] bg-amber-500 h-10`; returns null when `!isActive` |
| 6  | The banner cannot be scrolled away, dismissed, or covered by modals | VERIFIED | `fixed` CSS positioning + `z-[60]` (above Radix z-50 overlays); no dismiss mechanism exists |
| 7  | Clicking Return to Support calls the backend end endpoint, clears Redux state, resets cache, navigates to /support, and shows a toast | VERIFIED | `useImpersonation.endSession`: calls `endMutation`, then `navigate("/support")`, then `dispatch(endImpersonation())`, then `dispatch(apiSlice.util.resetApiState())`, then `toast.success("Impersonation session ended")` |
| 8  | During impersonation, the sidebar shows customer navigation instead of support navigation | VERIFIED | `DashboardLayout.NavContent` calls `getBusinessNavigationItems()` when `isImpersonating` is true |
| 9  | During impersonation, OnboardingGuard and PaywallGuard allow the support user through without checking onboarding or subscription status | VERIFIED | Both guards check `state.impersonation.active` first and return `<Outlet />` immediately when true |
| 10 | When impersonation state transitions from active to inactive, the user is navigated to /support with a toast | VERIFIED | `DashboardLayout` has `wasImpersonatingRef` + `useEffect` watching `isImpersonating`; fires `navigate("/support")` + `toast.warning("Impersonation session expired")` on active-to-inactive transition |
| 11 | An Impersonate User button appears on the user detail page when the viewer has permission and the target is a customer user | VERIFIED | `SupportUserDetailPage` computes `canImpersonate = !isImpersonating && targetIsCustomer && viewerHasPermission` and conditionally renders button |
| 12 | Clicking Impersonate User opens a confirmation dialog with a required reason textarea, validation, and loading state | VERIFIED | `ImpersonateUserDialog` has `Textarea`, inline validation message on empty submit, "Start Impersonation" with Loader2 spinner, "Keep Reviewing" cancel button |

**Score:** 12/12 truths verified

### Required Artifacts

| Artifact | Status | Details |
|----------|--------|---------|
| `trade-flow-ui/src/store/slices/impersonationSlice.ts` | VERIFIED | Exports `impersonationSlice`, `startImpersonation`, `endImpersonation`, `selectImpersonation`; all fields present |
| `trade-flow-ui/src/features/support/api/impersonationApi.ts` | VERIFIED | Exports `useStartImpersonationMutation`, `useEndImpersonationMutation`; uses `injectEndpoints` pattern; correct POST endpoints |
| `trade-flow-ui/src/services/api.ts` | VERIFIED | Contains `rawBaseQuery` with `state.impersonation.active` + token check; `baseQueryWithImpersonationExpiry` wrapper dispatches on 401; `apiSlice` uses wrapper |
| `trade-flow-ui/src/store/index.ts` | VERIFIED | Contains `impersonation: impersonationSlice.reducer` in reducer map |
| `trade-flow-ui/src/features/support/hooks/useImpersonation.ts` | VERIFIED | Exports `useImpersonation`; startSession and endSession wired to API mutations, Redux actions, cache reset, navigation, toasts |
| `trade-flow-ui/src/features/support/components/ImpersonationBanner.tsx` | VERIFIED | Exports `ImpersonationBanner`; `role="alert"`, `aria-live="assertive"`, `z-[60]`, `bg-amber-500`, Return to Support button, loading state |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` | VERIFIED | Imports and renders `ImpersonationBanner`; applies `pt-10` when `isImpersonating`; passes `isImpersonating` to `NavContent`; contains `wasImpersonatingRef` expired session watcher |
| `trade-flow-ui/src/config/navigation.ts` | VERIFIED | Contains `getNavigationItems(user, isImpersonating)` wrapper; `getBusinessNavigationItems()` and `getSupportNavigationItems()` split functions |
| `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` | VERIFIED | Contains `state.impersonation.active` check returning `<Outlet />` before onboarding logic |
| `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` | VERIFIED | Contains `state.impersonation.active` check returning `<Outlet />` before paywall logic |
| `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx` | VERIFIED | Exports `ImpersonateUserDialog`; DialogTitle "Impersonate User", Label "Reason", Textarea with placeholder, validation message, "Start Impersonation" and "Keep Reviewing" buttons |
| `trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx` | VERIFIED | Imports `ImpersonateUserDialog`; imports `UserCheck`; `canImpersonate` boolean gates button and dialog; `aria-label` containing "Impersonate" |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `api.ts` | `impersonationSlice.ts` | `getState()` reads `state.impersonation.active` and `state.impersonation.token` | WIRED | Lines 13-16 of api.ts |
| `api.ts` | `impersonationSlice.ts` | `baseQuery` wrapper dispatches `endImpersonation` on 401 | WIRED | Line 36 of api.ts |
| `ImpersonationBanner.tsx` | `useImpersonation.ts` | `endSession` callback from `useImpersonation` hook | WIRED | `const { isActive, targetUser, endSession, isEnding } = useImpersonation()` |
| `DashboardLayout.tsx` | `ImpersonationBanner.tsx` | Conditional rendering above header | WIRED | `<ImpersonationBanner />` as first child of outermost div |
| `DashboardLayout.tsx` | `navigation.ts` | `isImpersonating` prop to `NavContent` which calls `getBusinessNavigationItems()` | WIRED | NavContent uses `isImpersonating` to pick nav set (equivalent to `getNavigationItems` wrapper call) |
| `SupportUserDetailPage.tsx` | `ImpersonateUserDialog.tsx` | Dialog opened by Impersonate User button | WIRED | `canImpersonate &&  <ImpersonateUserDialog .../>` |
| `ImpersonateUserDialog.tsx` | `useImpersonation.ts` | `startSession` called on form submission | WIRED | `const { startSession, isStarting } = useImpersonation()` |

### Wiring Note

The plan's key_link for `DashboardLayout -> navigation.ts` specifies pattern `getNavigationItems.*isImpersonating`. The actual implementation passes `isImpersonating` as a prop to `NavContent` which directly calls `getBusinessNavigationItems()` or `getSupportNavigationItems()` conditionally — bypassing the `getNavigationItems` wrapper. The wrapper exists in `navigation.ts` and correctly delegates to the same functions. The SUMMARY documents this as an intentional adaptation to the pre-existing split-function architecture. Behavioural outcome is identical: customer navigation items are shown during impersonation.

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `ImpersonationBanner.tsx` | `isActive`, `targetUser` | `useImpersonation` → `useAppSelector(selectImpersonation)` → Redux store populated by `startImpersonation` dispatch | Yes — Redux state set only after backend API success | FLOWING |
| `OnboardingGuard.tsx` | `isImpersonating` | `useAppSelector((state) => state.impersonation.active)` | Yes — live Redux state | FLOWING |
| `PaywallGuard.tsx` | `isImpersonating` | `useAppSelector((state) => state.impersonation.active)` | Yes — live Redux state | FLOWING |
| `SupportUserDetailPage.tsx` | `canImpersonate` | `state.impersonation.active` (Redux) + `targetUser.supportRoleIds` (API) + `currentUser.supportRoles` (API) | Yes — all three sources are live | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED (no runnable entry points available without a running dev server and Phase 56 backend)

### Requirements Coverage

| Requirement | Source Plan(s) | Description | Status | Evidence |
|-------------|---------------|-------------|--------|----------|
| IMP-03 | 57-01, 57-02, 57-03 | During impersonation, the app renders exactly what the customer sees (same data, same subscription state, same permissions) | SATISFIED | Token switching via prepareHeaders ensures all API calls use impersonation token; guard bypasses (OnboardingGuard, PaywallGuard) allow customer routes; navigation shows customer items |
| IMP-04 | 57-02 | A fixed impersonation banner is visible at all times during an impersonation session showing the impersonated user's name and a "Return to Support" button | SATISFIED | `ImpersonationBanner.tsx` renders fixed amber banner with `z-[60]`; integrated into `DashboardLayout` as first child |
| IMP-05 | 57-02, 57-03 | Support user can terminate the impersonation session and return to their support dashboard cleanly | SATISFIED | `useImpersonation.endSession` calls backend, clears Redux, resets cache, navigates to `/support`, shows success toast |

**REQUIREMENTS.md traceability check:** IMP-03, IMP-04, IMP-05 are all mapped to Phase 57 in the traceability table. No orphaned requirements for this phase.

**Requirements out of scope for this phase:** IMP-01 (Phase 56), IMP-02 (Phase 56), IMP-06 (Phase 56), IAUD-01/02/03 (Phase 56) — all backend concerns excluded from Phase 57's scope.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `ImpersonateUserDialog.tsx` | 75 | `placeholder="Why are you impersonating this user?"` | Info | UI textarea placeholder text — not a stub. This is proper UX placeholder text for the input field, not placeholder implementation. |

No blockers or warnings found. The placeholder match is a false positive — it is a legitimate HTML `placeholder` attribute on a `<Textarea>` element.

**Commits verified:** All 5 phase commits exist in trade-flow-ui git history:
- `28f03a0` — feat(57-01): impersonation Redux slice and RTK Query endpoints
- `a6d06c5` — feat(57-01): api.ts token switching and expired session detection
- `57b5e7e` — feat(57-02): useImpersonation hook, guard bypasses, navigation switching
- `03120f5` — feat(57-02): ImpersonationBanner and DashboardLayout integration
- `8c8b4d9` — feat(57-03): ImpersonateUserDialog and Impersonate User button

### Human Verification Required

The plan includes a blocking `checkpoint:human-verify` task in Plan 03. All automated checks pass, but the following require human confirmation:

#### 1. Full Impersonation Flow (requires Phase 56 backend)

**Test:** Navigate to `/support/users/{customerId}` for a customer user. Click "Impersonate User". Enter a reason. Click "Start Impersonation".
**Expected:** Amber banner appears at top of viewport with "Impersonating: {user name}" and "Return to Support" button. Sidebar changes to show customer navigation (Dashboard, Jobs, Customers, Quotes, Estimates, Items). Banner stays visible while scrolling. Banner stays visible when opening dropdowns or dialogs.
**Why human:** Visual layout, DOM stacking (z-index, fixed positioning, scroll behaviour) cannot be verified programmatically. Requires Phase 56 backend running.

#### 2. Return to Support Button

**Test:** During active impersonation, click "Return to Support".
**Expected:** Navigates to /support, amber banner disappears, toast notification shows "Impersonation session ended".
**Why human:** Navigation and toast rendering are runtime behaviours requiring browser interaction.

#### 3. Button Visibility Gating

**Test:** Navigate to a support user's detail page (a user with `supportRoleIds.length > 0`).
**Expected:** "Impersonate User" button is NOT visible at all (completely absent from DOM, not disabled or greyed out).
**Why human:** Requires real user data from live backend to confirm `supportRoleIds` check works correctly.

#### 4. Empty Reason Validation

**Test:** Click "Impersonate User" to open dialog. Click "Start Impersonation" without entering any text.
**Expected:** Inline validation message "A reason is required before impersonating a user." appears below the textarea. Dialog stays open. No API call is made.
**Why human:** Form interaction and inline validation rendering must be confirmed visually.

#### 5. Page Refresh During Impersonation

**Test:** Start impersonation. Refresh the browser page.
**Expected:** Impersonation session ends (Redux state is in-memory only, no persistence). Support user returns to their own context without amber banner.
**Why human:** Redux-only session persistence behaviour requires manual browser test (refresh clears in-memory state).

### Gaps Summary

No gaps blocking goal achievement. All 12 must-haves are verified in the codebase. The 5 human verification items are interactive/visual behaviours that require a running browser + Phase 56 backend to confirm.

---

_Verified: 2026-04-19T18:00:00Z_
_Verifier: Claude (gsd-verifier)_
