---
phase: 38-hard-paywall-and-soft-paywall-removal
verified: 2026-04-02T11:51:25Z
status: gaps_found
score: 13/14 must-haves verified
gaps:
  - truth: "PAYWALL-06 marked complete in REQUIREMENTS.md"
    status: partial
    reason: "Implementation is fully complete but REQUIREMENTS.md still shows PAYWALL-06 as '[ ]' (unchecked) and 'Pending' in the tracking table. The code evidence is unambiguous -- soft paywall is gone -- but the requirements ledger is stale."
    artifacts:
      - path: ".planning/REQUIREMENTS.md"
        issue: "PAYWALL-06 checkbox is '- [ ]' and status table shows 'Pending' despite Phase 38 Plan 02 completing it"
    missing:
      - "Update PAYWALL-06 checkbox to '- [x]' in REQUIREMENTS.md"
      - "Update PAYWALL-06 status from 'Pending' to 'Complete' in the requirements tracking table"
---

# Phase 38: Hard Paywall and Soft Paywall Removal Verification Report

**Phase Goal:** Replace soft paywall (read-only mode + modal) with hard paywall (full-screen blocking page) and remove all soft paywall infrastructure
**Verified:** 2026-04-02T11:51:25Z
**Status:** gaps_found (1 documentation gap -- implementation fully complete)
**Re-verification:** No -- initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User with invalid subscription sees a full-screen paywall page instead of the app | VERIFIED | `PaywallGuard.tsx` returns `<PaywallPage variant={variant} />` when `!isActive`; wraps all business routes in App.tsx |
| 2 | Paywall displays variant-specific headline and body copy for trial expired, payment failed, and canceled states | VERIFIED | `PAYWALL_COPY` object in `PaywallPage.tsx` with three distinct `headline` and `body` values |
| 3 | User can click Subscribe to go directly to Stripe Checkout | VERIFIED | `handleSubscribe` calls `useCreateCheckoutSessionMutation`, sets `window.location.href = result.url` |
| 4 | User can click Manage Billing to open Stripe Billing Portal in a new tab | VERIFIED | `handleManageBilling` calls `useCreatePortalSessionMutation`, opens `window.open(result.url, "_blank")` |
| 5 | User sees "your data is safe" reassurance inline in body copy | VERIFIED | trial-expired body: "Your jobs, quotes, and customers are safe"; canceled body: "Your data is still here" |
| 6 | Support role users bypass the paywall and see the normal app | VERIFIED | `PaywallGuard.tsx` line 39: `if ((user?.supportRoles?.length ?? 0) > 0) return <Outlet />;` |
| 7 | Active/trialing users never see a paywall flash during loading | VERIFIED | `PaywallGuard.tsx` renders `Loader2` spinner when `isLoading` is true, before any paywall check |
| 8 | No soft paywall modal appears anywhere in the app | VERIFIED | `PaywallModal.tsx` deleted; grep returns zero matches for `PaywallModal` across entire `src/` |
| 9 | No PersistentCta button or banner appears in the header or on mobile | VERIFIED | `PersistentCta.tsx` deleted; `DashboardLayout.tsx` contains no `PersistentCta` import or render |
| 10 | No DashboardBanner appears on the dashboard | VERIFIED | `DashboardBanner.tsx` deleted; `DashboardPage.tsx` contains no `DashboardBanner` reference |
| 11 | No SubscriptionGatedLayout wraps business routes | VERIFIED | `SubscriptionGatedLayout.tsx` deleted; App.tsx uses `<PaywallGuard />` with no `SubscriptionGatedLayout` |
| 12 | No openPaywall dispatch calls exist in any feature page | VERIFIED | grep for `openPaywall` across `src/` returns zero matches |
| 13 | paywallSlice is not registered in the Redux store | VERIFIED | `paywallSlice.ts` deleted; `store/index.ts` contains no `paywall` reducer entry |
| 14 | PAYWALL-06 marked complete in REQUIREMENTS.md | FAILED | Implementation complete but REQUIREMENTS.md still shows `- [ ] **PAYWALL-06**` and "Pending" in status table |

**Score:** 13/14 truths verified

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-ui/src/pages/PaywallPage.tsx` | Full-screen blocking page with three variant modes | VERIFIED | 134 lines; contains all three variants, Subscribe/Manage Billing CTAs, Log out link, `data-icon` convention |
| `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` | Route guard that blocks invalid subscriptions | VERIFIED | 64 lines; `derivePaywallVariant` function, support bypass, loading state, `<PaywallPage variant=` render |
| `trade-flow-ui/src/store/index.ts` | Redux store without paywallSlice | VERIFIED | No `paywall` key in reducer; only `apiSlice`, `publicQuoteApi`, `onboarding` registered |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` | Dashboard layout without soft paywall components | VERIFIED | No `PaywallModal`, `PersistentCta`, or `DashboardBanner` imports or renders |

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `PaywallGuard.tsx` | `PaywallPage.tsx` | `<PaywallPage variant={variant} />` when subscription is invalid | WIRED | Line 62: `return <PaywallPage variant={variant} />;` |
| `PaywallPage.tsx` | `subscriptionApi` checkout endpoint | `useCreateCheckoutSessionMutation` | WIRED | Lines 42-43: mutation hook called in `handleSubscribe` |
| `PaywallPage.tsx` | `subscriptionApi` portal endpoint | `useCreatePortalSessionMutation` | WIRED | Line 44: mutation hook called in `handleManageBilling` |
| `App.tsx` | `PaywallGuard.tsx` | `<Route element={<PaywallGuard />}>` wrapping business routes | WIRED | Line 127: `<Route element={<PaywallGuard />}>` wraps dashboard, customers, jobs, quotes, items, business |
| `DashboardLayout.tsx` | `PaywallGuard.tsx` | PaywallGuard wraps DashboardLayout routes (replaced SubscriptionGatedLayout) | WIRED | App.tsx confirms PaywallGuard is the outer layout route; DashboardLayout contains no soft paywall components |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|-------------------|--------|
| `PaywallGuard.tsx` | `subscription` (from `useSubscription`) | `SubscriptionProvider` via `useGetSubscriptionQuery` → `GET /v1/subscription` | Yes -- RTK Query fetches from API | FLOWING |
| `PaywallGuard.tsx` | `user` (from `useGetCurrentUserQuery`) | RTK Query → `GET /v1/users/me` or equivalent | Yes -- real API query | FLOWING |
| `PaywallPage.tsx` | `variant` prop | `derivePaywallVariant(subscription)` in PaywallGuard | Yes -- derived from real subscription data | FLOWING |

---

### Behavioral Spot-Checks

Step 7b skipped -- no runnable entry points available without a running dev server. Key behavioral logic verified through code inspection:

- PaywallGuard support bypass: condition `(user?.supportRoles?.length ?? 0) > 0` executes first in guard body
- Loading state: `isLoading` check precedes paywall render -- no flash possible
- Subscribe CTA conditionality: `copy.showSubscribe` false for `payment-failed` variant only, true for `trial-expired` and `canceled`
- `/subscribe/success` and `/subscribe/cancel` routes appear at lines 109 and 113 in App.tsx, before `<Route element={<PaywallGuard />}>` at line 127 -- correctly outside paywall scope

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| PAYWALL-01 | 38-01 | User with invalid subscription sees a full-screen blocking page | SATISFIED | PaywallGuard + PaywallPage fully implemented and wired |
| PAYWALL-02 | 38-01 | Differentiated messaging based on subscription state | SATISFIED | `PAYWALL_COPY` with three distinct variant objects |
| PAYWALL-03 | 38-01 | User can access Stripe Billing Portal from paywall screen | SATISFIED | Manage Billing CTA calls portal mutation, opens in new tab |
| PAYWALL-04 | 38-01 | User sees "your data is safe" messaging | SATISFIED | Inline in both trial-expired and canceled body copy |
| PAYWALL-05 | 38-01 | Support role users bypass the hard paywall | SATISFIED | `supportRoles` check in PaywallGuard returns `<Outlet />` immediately |
| PAYWALL-06 | 38-02 | Existing soft paywall modal and write-action gating are removed | SATISFIED (code) / STALE (docs) | All 7 files deleted, all `openPaywall` calls removed, grep confirms zero references -- but REQUIREMENTS.md checkbox/status not updated |

**ORPHANED requirements check:** No requirements mapped to Phase 38 in REQUIREMENTS.md that are absent from the plans.

**Note on PAYWALL-06 documentation gap:** The implementation satisfies the requirement completely. The gap is in `.planning/REQUIREMENTS.md` where the checkbox remains `- [ ]` and the tracking table still shows "Pending". This does not indicate a code gap.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `PaywallPage.tsx` | 59-64 | `handleManageBilling` checks `!result.url` and throws -- but `createPortalSessionMutation` already throws on API error via `unwrap()`. The extra check is defensive over-engineering, not a stub. | Info | Harmless double-guard; no user impact |

No blockers or warnings found. The `return null`, `return []` patterns are absent. No `openPaywall`, `paywallSlice`, or soft paywall component references remain anywhere in the source tree.

---

### Human Verification Required

#### 1. PaywallPage visual rendering

**Test:** Log in with a user whose subscription is expired. Observe the screen that appears.
**Expected:** Full-screen centered card with correct headline for the subscription state, Subscribe button (if applicable), Manage Billing button, and "Log out" link below the card.
**Why human:** Visual layout, centering, gradient background quality, and CTA prominence require visual inspection.

#### 2. Stripe Checkout redirect

**Test:** On the paywall screen, click "Subscribe -- GBP 6/month".
**Expected:** Button shows "Redirecting..." with a spinner, then browser navigates to Stripe hosted checkout page.
**Why human:** Requires a live Stripe integration and valid test subscription state.

#### 3. Manage Billing portal

**Test:** On the paywall screen, click "Manage Billing".
**Expected:** Stripe Billing Portal opens in a new browser tab; paywall screen remains in original tab.
**Why human:** Requires live Stripe portal session and verifying new-tab behavior.

#### 4. Payment-failed variant (no Subscribe button)

**Test:** Log in with a user in `past_due` subscription state.
**Expected:** Paywall shows "There's a problem with your payment" headline and only "Manage Billing" button (no Subscribe button).
**Why human:** Requires a `past_due` subscription state in the test environment.

---

### Gaps Summary

One gap found -- documentation tracking only, not a code gap:

**REQUIREMENTS.md not updated for PAYWALL-06:** The implementation is provably complete. All 7 soft paywall files are deleted. All `openPaywall` calls are removed from 10+ feature files. `paywallSlice` is absent from the Redux store. Zero grep matches for any soft paywall symbol in `src/`. Despite this, the REQUIREMENTS.md checkbox for PAYWALL-06 still reads `- [ ]` and the status table entry still reads "Pending" rather than "Complete". This needs a one-line fix to the requirements file.

---

## Summary

Phase 38 goal is **achieved in the codebase**. The hard paywall is fully functional:

- `PaywallPage.tsx` renders three distinct variants with correct copy, CTAs, and Firebase sign-out
- `PaywallGuard.tsx` correctly blocks invalid subscriptions, bypasses support users, prevents paywall flash during loading, and derives the correct variant from subscription state
- All `/subscribe/*` callback routes are correctly positioned outside `PaywallGuard` in the route tree
- All 7 soft paywall files are deleted with zero references remaining anywhere in `src/`
- `paywallSlice` is removed from the Redux store
- `DashboardLayout.tsx` is clean of all soft paywall renders

The single gap is a stale checkbox in `.planning/REQUIREMENTS.md` for PAYWALL-06. It does not affect runtime behavior.

---

_Verified: 2026-04-02T11:51:25Z_
_Verifier: Claude (gsd-verifier)_
