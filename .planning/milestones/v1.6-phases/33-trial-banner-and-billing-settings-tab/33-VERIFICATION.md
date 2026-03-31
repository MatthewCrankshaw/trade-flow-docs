---
phase: 33-trial-banner-and-billing-settings-tab
verified: 2026-03-29T20:30:00Z
status: human_needed
score: 9/9 must-haves verified
human_verification:
  - test: "Navigate to the app as a trialing user and confirm the amber TrialChip appears in the header"
    expected: "An amber pill badge with a Clock icon and 'X days left in trial' (or 'Last day of trial') appears in the header, before PersistentCta"
    why_human: "Visual rendering and correct data binding require a live browser session with a trialing subscription"
  - test: "Click the TrialChip while on any page"
    expected: "Browser navigates to /settings?tab=billing and the Billing tab is pre-selected"
    why_human: "Navigation behavior and tab pre-selection require a live browser session"
  - test: "Navigate to /settings as an active (non-trialing) user"
    expected: "TrialChip is absent from the header; PersistentCta is also absent (isActive=true hides it)"
    why_human: "Mutual-exclusion between TrialChip and PersistentCta requires a live session"
  - test: "Navigate to /settings?tab=billing as a trialing user"
    expected: "Billing tab is active, SubscriptionStatusCard shows amber 'Trial' badge and 'Trial — X days remaining' status line, and 'Manage Billing' button is present"
    why_human: "Badge color, status text, and CTA label require a live browser with real subscription data"
  - test: "Navigate to /settings?tab=billing as an active user with cancelAtPeriodEnd=true"
    expected: "Orange 'Cancels on [date]' badge is shown and status line reads 'Cancels on [date]'"
    why_human: "Requires a subscription record with cancelAtPeriodEnd=true in the database"
  - test: "Click 'Manage Billing' on the Billing tab"
    expected: "Browser is redirected to the Stripe Billing Portal URL returned by POST /v1/subscription/portal"
    why_human: "Requires live Stripe test-mode API key and a real portal session to be created"
  - test: "Return from Stripe Portal via return_url (/settings?tab=billing)"
    expected: "Settings page opens with Billing tab pre-selected (URL param preserved)"
    why_human: "Requires completing a Stripe Portal redirect flow end-to-end"
---

# Phase 33: Trial Banner and Billing Settings Tab — Verification Report

**Phase Goal:** Trialing users see how many days remain in their trial, and any user can view and manage their subscription from Settings > Billing
**Verified:** 2026-03-29T20:30:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Trialing users see a persistent amber chip in the header showing days remaining | VERIFIED | `TrialChip.tsx` renders amber pill with Clock icon and `daysRemaining` label; wired at `DashboardLayout.tsx:217` |
| 2 | The chip is hidden when status is not trialing or subscription is null | VERIFIED | `TrialChip.tsx:11` — `if (isLoading \|\| subscription?.status !== "trialing") return null` |
| 3 | The chip is hidden during initial loading (no flash) | VERIFIED | `TrialChip.tsx:11` — guards on `isLoading === true` before any render |
| 4 | Clicking the chip navigates to /settings?tab=billing | VERIFIED | `TrialChip.tsx:29` — `onClick={() => navigate("/settings?tab=billing")}` |
| 5 | When trial ends today, chip shows "Last day of trial" | VERIFIED | `TrialChip.tsx:22-24` — `daysRemaining === 0` branch returns `"Last day of trial"` |
| 6 | When daysRemaining is negative, chip is hidden | VERIFIED | `TrialChip.tsx:17` — `if (daysRemaining === null \|\| daysRemaining < 0) return null` |
| 7 | Settings page has a Billing tab accessible via ?tab=billing | VERIFIED | `SettingsPage.tsx:22-23` — `useSearchParams`; `activeTab = searchParams.get("tab") \|\| "general"` |
| 8 | Billing tab shows subscription status with correct badge and CTA matrix | VERIFIED | `SubscriptionStatusCard.tsx:28-71` — `getStatusDisplay()` covers trialing/active/cancelling/inactive states |
| 9 | "Manage Billing" button calls POST /v1/subscription/portal and redirects | VERIFIED | `usePortalSession.ts:10-11` — `createPortal().unwrap()` then `window.location.href = result.portalUrl`; mutation targets `"/v1/subscription/portal"` at `subscriptionApi.ts:55-56` |

**Score:** 9/9 truths verified

### Required Artifacts

#### Plan 01 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/features/subscription/utils/subscription-helpers.ts` | calculateDaysRemaining and formatBillingDate utilities | VERIFIED | Both functions present and exported; uses `Intl.DateTimeFormat("en-GB")` |
| `trade-flow-ui/src/features/subscription/components/TrialChip.tsx` | TrialChip component for trialing users | VERIFIED | Substantive: 37 lines; handles all guard conditions; amber styling; Clock icon |
| `trade-flow-ui/src/features/subscription/index.ts` | Updated barrel export with TrialChip and helpers | VERIFIED | Lines 8, 14 export `TrialChip`, `calculateDaysRemaining`, `formatBillingDate` |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` | TrialChip imported and rendered in header | VERIFIED | Import at line 29; render at line 217 inside `flex items-center gap-2` header div |

#### Plan 02 Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/features/subscription/api/subscriptionApi.ts` | createPortalSession mutation | VERIFIED | Lines 54-67; targets `POST /v1/subscription/portal`; exports `useCreatePortalSessionMutation` |
| `trade-flow-ui/src/features/subscription/hooks/usePortalSession.ts` | usePortalSession hook | VERIFIED | Mirrors `useCheckout.ts` pattern; `window.location.href = result.portalUrl`; error toast |
| `trade-flow-ui/src/features/subscription/components/SubscriptionStatusCard.tsx` | Full CTA matrix status card | VERIFIED | 132 lines; handles trialing/active/cancelAtPeriodEnd/inactive; skeleton loading; Manage Billing + Start Subscription CTAs |
| `trade-flow-ui/src/features/subscription/components/BillingTab.tsx` | BillingTab composition wrapper | VERIFIED | 9 lines; imports and renders `SubscriptionStatusCard` |
| `trade-flow-ui/src/pages/SettingsPage.tsx` | Settings page with Billing tab + URL routing | VERIFIED | `useSearchParams`, `Tabs`, `TabsContent value="billing"`, `BillingTab` all present |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `TrialChip.tsx` | `useSubscription()` | Hook call | WIRED | `import { useSubscription }` + `const { isLoading, subscription } = useSubscription()` |
| `DashboardLayout.tsx` | `TrialChip` | Component render | WIRED | `import { TrialChip } from "@/features/subscription"` + `<TrialChip />` at line 217 |
| `usePortalSession.ts` | `POST /v1/subscription/portal` | RTK Query mutation | WIRED | `useCreatePortalSessionMutation` → `url: "/v1/subscription/portal"` in `subscriptionApi.ts` |
| `SubscriptionStatusCard.tsx` | `useSubscription()` | Hook call | WIRED | `const { subscription, isLoading } = useSubscription()` |
| `SubscriptionStatusCard.tsx` | `usePortalSession()` | Hook call | WIRED | `const { openPortal, isLoading: isPortalLoading } = usePortalSession()` |
| `SettingsPage.tsx` | `BillingTab` | shadcn TabsContent | WIRED | `<TabsContent value="billing"><BillingTab /></TabsContent>` at line 99-101 |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `TrialChip.tsx` | `subscription` (from `useSubscription`) | `SubscriptionProvider` → `useGetSubscriptionQuery` → `GET /v1/subscription` | RTK Query fetches from API; `subscription.trialEnd` used for `calculateDaysRemaining` | FLOWING |
| `SubscriptionStatusCard.tsx` | `subscription` (from `useSubscription`) | Same `SubscriptionProvider` context — same RTK Query subscription | `getStatusDisplay(subscription)` computes badge/statusLine/cta from real data | FLOWING |
| `SettingsPage.tsx` | `activeTab` | `useSearchParams()` — URL query param `?tab=billing` | React Router search params read from live URL | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED — all artifacts are UI components requiring a browser runtime; no runnable CLI entry points or standalone API endpoints to check without a live server.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| TRIAL-01 | 33-01-PLAN.md | A persistent banner is shown to trialing users displaying days remaining and a "Subscribe" CTA | SATISFIED* | `TrialChip.tsx` renders amber pill with days remaining; click navigates to `/settings?tab=billing`. Note: CTA links to billing settings rather than checkout — trialing users already have a subscription; plan explicitly designed chip to link to billing rather than checkout |
| TRIAL-02 | 33-01-PLAN.md | The trial banner is not shown when subscription status is `active` | SATISFIED | `TrialChip.tsx:11` — returns `null` unless `subscription.status === "trialing"` |
| BILL-04 | 33-02-PLAN.md | Settings page has a Billing tab showing current subscription status, next billing date, and appropriate CTAs | SATISFIED | `SettingsPage.tsx` has `<TabsContent value="billing">` with `BillingTab`; `SubscriptionStatusCard` shows all states |
| BILL-05 | 33-02-PLAN.md | When `cancelAtPeriodEnd` is true, the Billing tab shows "Cancels on [date]" rather than "Active" | SATISFIED | `SubscriptionStatusCard.tsx:54-59` — `if (subscription.cancelAtPeriodEnd)` returns `Cancels on ${date}` with orange badge |

*TRIAL-01: The REQUIREMENTS.md uses "Subscribe CTA" loosely — for trialing users the correct action is billing management, not initiating a new checkout. The plan doc and UI-SPEC are explicit that the chip navigates to `/settings?tab=billing`. This is the correct design choice.

### Anti-Patterns Found

No anti-patterns detected across any phase 33 files:
- No TODO/FIXME/PLACEHOLDER comments
- No empty implementations or stub returns
- No hardcoded empty arrays or objects passed to render-critical props
- No console.log-only handlers

### Human Verification Required

#### 1. TrialChip Visual and Data Binding

**Test:** Log in as a trialing user; navigate to any dashboard page
**Expected:** Amber pill with Clock icon and "X days left in trial" text appears in the top-right header area, before any other header controls; text matches actual days until `trialEnd` date
**Why human:** React component rendering and real-time date math require a live browser session with a real trialing subscription record

#### 2. TrialChip Click Navigation

**Test:** Click the TrialChip while on any authenticated page
**Expected:** Browser navigates to `/settings?tab=billing`; Settings page opens with the Billing tab pre-selected (not the General tab)
**Why human:** React Router navigation and tab pre-selection via URL param require a live browser

#### 3. TrialChip Absent for Active Users

**Test:** Log in as an active (non-trialing) user; inspect the header
**Expected:** No TrialChip; PersistentCta is also absent (trialing is `isActive=true` so PersistentCta is hidden, and for active status TrialChip returns null)
**Why human:** Requires two distinct subscription states in test environment

#### 4. Billing Tab Status Display

**Test:** Navigate to `/settings?tab=billing` as a trialing user
**Expected:** Amber "Trial" badge; status line "Trial — X days remaining"; "Manage Billing" button with ExternalLink icon
**Why human:** Visual badge colors and live subscription data required

#### 5. cancelAtPeriodEnd Display (BILL-05)

**Test:** Navigate to `/settings?tab=billing` as a user whose subscription has `cancelAtPeriodEnd=true`
**Expected:** Orange "Cancels on [date]" badge; status line "Cancels on [date]"; "Manage Billing" button
**Why human:** Requires a subscription with `cancelAtPeriodEnd=true` in the database (set via Stripe webhook or direct DB edit)

#### 6. Manage Billing Portal Redirect

**Test:** Click "Manage Billing" on the Billing tab
**Expected:** Loading spinner appears briefly; browser redirects to the Stripe Billing Portal URL; user can manage subscription from the portal
**Why human:** Requires live Stripe test-mode credentials and a real portal session; window.location redirect cannot be verified statically

#### 7. Stripe Portal Return URL

**Test:** After completing an action in the Stripe Portal, observe the return destination
**Expected:** Browser lands on `/settings?tab=billing`; the Billing tab is pre-selected
**Why human:** Requires completing a Stripe Portal redirect cycle with a real `return_url` configured in Phase 31 (POST /v1/subscription/portal)

### Gaps Summary

No gaps found. All 9 truths are verified against the actual codebase. All 9 artifacts exist and are substantive. All 6 key links are wired. Data flows from real API queries through the subscription context to the components. No anti-patterns detected.

The 7 human verification items cover visual rendering, live navigation behavior, and Stripe Portal integration — none are programmatically verifiable without a running application and real subscription data.

---

_Verified: 2026-03-29T20:30:00Z_
_Verifier: Claude (gsd-verifier)_
