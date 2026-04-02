# Phase 38: Hard Paywall and Soft Paywall Removal - Research

**Researched:** 2026-04-02
**Domain:** React route guard implementation, subscription state detection, UI component removal
**Confidence:** HIGH

## Summary

Phase 38 is a frontend-only phase (`trade-flow-ui`) with two distinct workstreams: (1) implement the hard paywall blocking logic in the existing `PaywallGuard` shell (created by Phase 36) and create the `PaywallPage` component, and (2) remove all soft paywall infrastructure from the codebase. No backend changes are required -- all API endpoints (subscription status, Stripe Checkout, Stripe Billing Portal) already exist from v1.6 phases.

The implementation is straightforward because prior phases have laid all the groundwork: Phase 36 created the `PaywallGuard` shell in the route tree, Phase 32 built the `useSubscription` hook and `subscriptionApi` RTK Query endpoints, and Phase 35 defined the subscription entity fields (`status`, `trialEnd`, `canceledAt`) needed for state differentiation. The primary engineering challenge is the clean removal of soft paywall infrastructure across multiple feature files without breaking existing functionality.

**Primary recommendation:** Implement PaywallGuard blocking logic and PaywallPage first, verify the hard paywall works end-to-end, then perform the soft paywall removal as a separate step. This order ensures the replacement is in place before removing what it replaces.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Three paywall variants with specific headlines and body copy (trial expired, payment failed, canceled)
- D-02: "Your data is safe" reassurance is inline in body copy, not a separate callout
- D-03: Subscribe button goes directly to Stripe Checkout (POST /v1/subscription/checkout), no intermediate /subscribe page
- D-04: Manage Billing button opens Stripe Billing Portal in new tab (existing manage endpoint)
- D-05: Subscription state detection uses existing entity fields: status, trialEnd, canceledAt. Trial expired = status canceled + canceledAt close to trialEnd. Payment failed = status past_due. Canceled = status canceled + canceledAt not near trialEnd
- D-06: Centered card on clean/gradient background with Trade Flow logo above -- same pattern as onboarding wizard
- D-07: Full-width on mobile with padding, max-width ~480px on desktop
- D-08: PaywallGuard shows existing loading screen while subscription status is being fetched
- D-09: Subtle "Log out" text link below the card
- D-10: No access to Settings or any app routes through the paywall -- Billing Portal link is sufficient
- D-11: PaywallGuard blocks ALL routes inside DashboardLayout, no allowlisting
- D-12: Remove ALL soft paywall infrastructure (SubscriptionGatedLayout, PaywallModal, paywallSlice, PersistentCta, DashboardBanner, useSubscription openPaywall method, all per-page openPaywall dispatch calls)
- D-13: Remove /subscribe pricing page route entirely
- D-14: Keep /subscribe/success and /subscribe/cancel routes, move outside PaywallGuard (sibling to DashboardLayout, inside ProtectedRoute + OnboardingGuard but outside PaywallGuard)
- D-15: Support role bypass via user.supportRoles.length > 0 check in PaywallGuard

### Claude's Discretion
- Exact card dimensions, padding, and shadow treatment for paywall page
- Gradient background color values (should match onboarding wizard)
- Subscribe button icon and exact copy
- Manage Billing button styling (primary vs secondary/outline)
- Log out link exact positioning and styling
- How subscription state is determined in the guard (direct hook call vs context)
- Loading screen component reuse approach
- Order of removal operations during implementation

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PAYWALL-01 | User with invalid subscription sees a full-screen blocking page instead of the app | PaywallGuard blocking logic renders PaywallPage instead of Outlet when subscription is invalid. Guard pattern documented in Architecture research Pattern 4. |
| PAYWALL-02 | User sees differentiated messaging based on subscription state (trial expired, payment failed, canceled) | Subscription state detection logic using status + trialEnd + canceledAt fields (D-05). Three variant props passed to PaywallPage. |
| PAYWALL-03 | User can access Stripe Billing Portal from the paywall screen to manage subscription | Existing subscriptionApi manage endpoint returns portal URL. Manage Billing button opens in new tab. |
| PAYWALL-04 | User sees "your data is safe" messaging on the paywall screen | Inline in body copy per D-02. Copy contract defined in UI-SPEC. |
| PAYWALL-05 | Support role users bypass the hard paywall entirely | user.supportRoles.length > 0 check in PaywallGuard renders Outlet regardless of subscription status (D-15). |
| PAYWALL-06 | Existing soft paywall modal and write-action gating are removed | Full removal inventory documented in this research (7 components/files to delete, 8+ feature files to edit). |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| react | 19.2.0 | UI framework | Already installed, project standard |
| react-router-dom | 7.13.0 | Route guards (layout routes with Outlet) | Already installed, PaywallGuard is a layout route |
| @reduxjs/toolkit | 2.11.2 | RTK Query for subscription data fetching | Already installed, subscriptionApi endpoints exist |
| sonner | 2.0.7 | Error toast notifications | Already installed, used for checkout/billing errors |
| lucide-react | 0.563.0 | CreditCard, ExternalLink, Loader2 icons | Already installed, icons specified in UI-SPEC |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| shadcn Card/Button | Already installed | PaywallPage UI components | Card, CardHeader, CardContent, CardFooter, Button |
| firebase | 12.8.0 | Sign-out for log out link | Already installed, signOut function |

### Alternatives Considered
None -- this phase uses exclusively existing libraries. No new dependencies required.

**Installation:**
```bash
# No new packages needed. All dependencies already installed.
```

## Architecture Patterns

### Recommended Project Structure
```
src/
  pages/
    PaywallPage.tsx              # NEW: Full-screen blocking page with variant props
  features/
    auth/
      components/
        PaywallGuard.tsx         # MODIFIED: Replace pass-through with blocking logic
    subscription/
      components/
        SubscriptionGatedLayout.tsx  # DELETE
        PaywallModal.tsx             # DELETE
        PersistentCta.tsx            # DELETE
        DashboardBanner.tsx          # DELETE
        PricingCard.tsx              # DELETE (only used by PaywallModal and SubscribePage)
      hooks/
        useSubscription.ts           # MODIFIED: Remove openPaywall method
  store/
    slices/
      paywallSlice.ts               # DELETE
  pages/
    SubscribePage.tsx                # DELETE
```

### Pattern 1: Layout Route Guard with Inline Blocking Page
**What:** PaywallGuard as a React Router layout route that renders either `<Outlet />` (pass-through) or `<PaywallPage />` (blocking) based on subscription state. Not a redirect -- the paywall page renders in place.
**When to use:** When access must be completely blocked (no read-only mode) and the blocking page needs to be visually distinct from the app shell.

```typescript
// PaywallGuard.tsx -- conceptual structure
function PaywallGuard() {
  const { user } = useAuth();
  const { isActive, isLoading, subscription } = useSubscription();

  // Support role bypass (D-15)
  if (user?.supportRoles?.length > 0) {
    return <Outlet />;
  }

  // Loading state -- seamless transition (D-08)
  if (isLoading) {
    return <LoadingScreen />;
  }

  // Active subscription -- pass through
  if (isActive) {
    return <Outlet />;
  }

  // Determine paywall variant from subscription state (D-05)
  const variant = derivePaywallVariant(subscription);
  return <PaywallPage variant={variant} />;
}
```

### Pattern 2: Subscription State Variant Derivation
**What:** Determine which paywall variant to show based on subscription entity fields.
**When to use:** In PaywallGuard to compute the variant prop for PaywallPage.

```typescript
type PaywallVariant = "trial-expired" | "payment-failed" | "canceled";

function derivePaywallVariant(subscription: Subscription | null): PaywallVariant {
  if (!subscription) return "trial-expired"; // No subscription record = fallback

  if (subscription.status === "past_due") return "payment-failed";

  if (subscription.status === "canceled") {
    // Trial expired: canceledAt is close to trialEnd (auto-canceled by Stripe)
    // Canceled: canceledAt is NOT close to trialEnd (user or manual cancel)
    const trialEnd = new Date(subscription.trialEnd).getTime();
    const canceledAt = new Date(subscription.canceledAt).getTime();
    const threshold = 24 * 60 * 60 * 1000; // 24 hours tolerance
    if (Math.abs(canceledAt - trialEnd) <= threshold) {
      return "trial-expired";
    }
    return "canceled";
  }

  return "trial-expired"; // Fallback for any unexpected status
}
```

### Pattern 3: Route Tree Adjustment for /subscribe Routes (D-14)
**What:** Move `/subscribe/success` and `/subscribe/cancel` outside PaywallGuard but inside ProtectedRoute + OnboardingGuard.
**When to use:** These are Stripe Checkout redirect targets that must be accessible even when subscription is invalid.

```typescript
// App.tsx route structure adjustment
<Route element={<ProtectedRoute />}>
  <Route element={<OnboardingGuard />}>
    {/* Subscribe callback routes -- outside PaywallGuard */}
    <Route path="/subscribe/success" element={<SubscribeSuccessPage />} />
    <Route path="/subscribe/cancel" element={<SubscribeCancelPage />} />

    {/* Main app -- inside PaywallGuard */}
    <Route element={<PaywallGuard />}>
      <Route element={<DashboardLayout />}>
        {/* all business routes */}
      </Route>
    </Route>
  </Route>
</Route>
```

### Anti-Patterns to Avoid
- **Settings allowlisting in the guard:** D-10/D-11 explicitly state NO route exceptions. The v1.6 Architecture research showed a `/settings` exception pattern -- do NOT use it. The paywall blocks ALL routes.
- **Storing paywall variant in Redux:** The variant is derived from subscription data already in RTK Query cache. No separate state needed.
- **Gradual removal:** Do not partially remove soft paywall components. Remove them all in one sweep to avoid dead code and confusing hybrid state.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Subscription data fetching | Custom fetch + state | Existing `useSubscription` hook + RTK Query `getSubscription` | Already cached, handles loading/error states, 404-as-null pattern |
| Stripe Checkout redirect | Custom form or Stripe Elements | Existing POST /v1/subscription/checkout endpoint + window.location.href | Stripe hosted Checkout handles PCI/SCA, no frontend Stripe SDK needed |
| Stripe Billing Portal | Custom billing UI | Existing manage endpoint returns portal URL | Portal handles all billing management, opens in new tab |
| Loading screen | New spinner component | Existing loading screen pattern (Loader2 + animate-spin) | Same pattern as OnboardingGuard and Phase 36 suspense fallback |
| Firebase sign-out | Custom logout flow | `signOut(auth)` from firebase/auth | One function call, AuthProvider handles state cleanup |

**Key insight:** Phase 38 creates only two new things (PaywallPage component and PaywallGuard logic). Everything else is removal or reuse of existing infrastructure.

## Common Pitfalls

### Pitfall 1: Flash of Paywall for Active Users
**What goes wrong:** Active subscribers briefly see the paywall page before subscription data loads.
**Why it happens:** PaywallGuard renders the blocking page during the RTK Query loading state instead of a neutral loading screen.
**How to avoid:** Check `isLoading` BEFORE checking `isActive`. Render the loading screen during data fetch (D-08). The loading screen matches OnboardingGuard's loading screen for seamless transition.
**Warning signs:** Active user sees paywall card flash for a fraction of a second on page load.

### Pitfall 2: Trial Expired vs Canceled Misclassification
**What goes wrong:** A manually canceled subscription is shown as "trial expired" or vice versa.
**Why it happens:** The threshold comparison between `canceledAt` and `trialEnd` is too tight or too loose.
**How to avoid:** Use a 24-hour tolerance window. Stripe auto-cancellation happens at or very near the `trialEnd` timestamp. Manual cancellation or payment failure cancellation will have a `canceledAt` well before `trialEnd`.
**Warning signs:** Copy mismatch -- user sees "Your free trial has ended" when they explicitly canceled their paid subscription.

### Pitfall 3: Stale Subscription Data After Stripe Action
**What goes wrong:** User completes Stripe Checkout or updates payment in Billing Portal, returns to app, still sees paywall.
**Why it happens:** RTK Query cache still holds the old subscription status. The /subscribe/success page polls verify-session, but if the user navigates directly back, no invalidation occurs.
**How to avoid:** The /subscribe/success page already handles polling and cache invalidation. The Billing Portal opens in a new tab, so the original tab retains its state. Consider adding a `refetchOnFocus` behavior or window focus listener that re-fetches subscription status.
**Warning signs:** User says "I just paid but I still see the paywall."

### Pitfall 4: Incomplete Removal Leaves Dead Imports
**What goes wrong:** Removing soft paywall components but missing import statements or type references in other files.
**Why it happens:** `paywallSlice` is imported in the store index, `openPaywall` is dispatched across 8+ feature files, `PaywallModal` is rendered in DashboardLayout.
**How to avoid:** Work through the removal inventory systematically. After removal, run TypeScript compilation (`tsc --noEmit`) to catch all broken imports. ESLint will flag unused imports.
**Warning signs:** TypeScript compilation errors after removal, or unused import warnings.

### Pitfall 5: Subscribe Route Still Accessible via Direct URL
**What goes wrong:** Users navigate to /subscribe directly (bookmarked or from old links) and see a 404 or broken page.
**Why it happens:** D-13 removes the /subscribe route entirely, but old bookmarks or external links may still point to it.
**How to avoid:** Add a redirect from /subscribe to / (landing page) or render a simple "page not found" that links to the landing page. Since this is inside ProtectedRoute, it could redirect to /dashboard which will then hit the PaywallGuard if subscription is invalid.
**Warning signs:** 404 errors in browser console when visiting /subscribe.

## Code Examples

### PaywallPage Component Structure
```typescript
// src/pages/PaywallPage.tsx
// Source: UI-SPEC layout specification + D-01 through D-09

import { Card, CardHeader, CardContent, CardFooter } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { CreditCard, ExternalLink, Loader2 } from "lucide-react";

type PaywallVariant = "trial-expired" | "payment-failed" | "canceled";

interface PaywallPageProps {
  variant: PaywallVariant;
}

const PAYWALL_COPY = {
  "trial-expired": {
    headline: "Your free trial has ended",
    body: "Your jobs, quotes, and customers are safe \u2014 pick up where you left off.",
    showSubscribe: true,
  },
  "payment-failed": {
    headline: "There\u2019s a problem with your payment",
    body: "Update your payment method to get back to work.",
    showSubscribe: false,
  },
  canceled: {
    headline: "Your subscription is canceled",
    body: "Your data is still here. Resubscribe to pick up where you left off.",
    showSubscribe: true,
  },
};
```

### Removing openPaywall Dispatch Calls
```typescript
// BEFORE (soft paywall pattern in feature pages):
const { openPaywall, isActive } = useSubscription();
const handleCreate = () => {
  if (!isActive) {
    openPaywall("Create a Customer");
    return;
  }
  // ... actual create logic
};

// AFTER (hard paywall -- no gating needed, user can't reach this page without subscription):
const handleCreate = () => {
  // ... actual create logic (no subscription check)
};
```

### useSubscription Hook Simplification
```typescript
// BEFORE:
// Returns { isActive, isLoading, openPaywall }
// openPaywall dispatches to paywallSlice to open PaywallModal

// AFTER:
// Returns { isActive, isLoading, subscription }
// Remove openPaywall entirely. Add subscription data for variant derivation.
// The hook stays because PaywallGuard and other components need isActive/isLoading.
```

## Removal Inventory

Complete list of files and code to remove, derived from Phase 32 CONTEXT.md and Architecture research:

### Files to Delete
| File | Reason |
|------|--------|
| `src/features/subscription/components/SubscriptionGatedLayout.tsx` | Replaced by PaywallGuard blocking |
| `src/features/subscription/components/PaywallModal.tsx` | Replaced by full-screen PaywallPage |
| `src/features/subscription/components/PersistentCta.tsx` | No longer needed -- hard paywall blocks entirely |
| `src/features/subscription/components/DashboardBanner.tsx` | No longer needed -- hard paywall blocks entirely |
| `src/features/subscription/components/PricingCard.tsx` | Only used by SubscribePage and PaywallModal, both removed |
| `src/store/slices/paywallSlice.ts` | No modal open/close state needed |
| `src/pages/SubscribePage.tsx` | Subscribe goes directly to Stripe Checkout from PaywallPage |

### Files to Modify
| File | Modification |
|------|-------------|
| `src/features/auth/components/PaywallGuard.tsx` | Replace pass-through Outlet with subscription status check + blocking logic |
| `src/features/subscription/hooks/useSubscription.ts` | Remove `openPaywall` method, keep `isActive`/`isLoading`, add `subscription` data |
| `src/App.tsx` | Remove /subscribe route, move /subscribe/success and /subscribe/cancel outside PaywallGuard, remove SubscriptionGatedLayout from route tree |
| `src/store/index.ts` | Remove paywallSlice reducer import and registration |
| `src/components/layouts/DashboardLayout.tsx` | Remove PaywallModal rendering, remove PersistentCta rendering, remove DashboardBanner rendering |
| `src/features/subscription/components/index.ts` | Remove exports for deleted components |
| Multiple feature pages (8+) | Remove all `openPaywall()` dispatch calls and `isActive` gating from write action handlers |

### Feature Pages Requiring openPaywall Removal
Based on Phase 32 architecture (D-15 active conversion triggers on write actions):
- `CustomersPage` / customer create/edit handlers
- `JobsPage` / job create handler
- `JobDetailPage` / job edit, schedule create handlers
- `QuotesPage` / quote create handler
- `QuoteDetailPage` / quote edit, send handlers
- `ItemsPage` / item create/edit handlers
- `BusinessPage` / business edit handler
- `DashboardPage` / any write-action shortcuts

## State of the Art

| Old Approach (v1.6) | New Approach (v1.7 Phase 38) | Impact |
|---------------------|------------------------------|--------|
| Soft paywall: read-only mode + PaywallModal on write actions | Hard paywall: full-screen blocking page, no app access | Simpler, removes per-page wiring, stronger conversion signal |
| SubscriptionGatedLayout wrapping business routes | PaywallGuard layout route rendering PaywallPage inline | Single guard component, no modal state management |
| paywallSlice Redux state for modal open/close | No Redux state -- variant derived from RTK Query subscription cache | Less state to manage, fewer potential sync bugs |
| Per-page openPaywall() calls in every feature | Zero per-page code -- guard handles everything | No risk of missed gating on new features |
| /subscribe pricing page as intermediate step | Direct Stripe Checkout from paywall page | Fewer clicks to conversion |

## Open Questions

1. **Window focus refetch after Billing Portal**
   - What we know: Billing Portal opens in new tab (D-04). When user returns to original tab, subscription cache may be stale.
   - What's unclear: Whether RTK Query's `refetchOnFocus` is already configured on the subscription endpoint.
   - Recommendation: Check if `refetchOnFocus: true` is set on `getSubscription` query. If not, add it or add a `visibilitychange` listener in PaywallGuard.

2. **Exact threshold for trial expired vs canceled detection**
   - What we know: D-05 says "canceledAt close to trialEnd" distinguishes trial expired from user-canceled.
   - What's unclear: Exact timing of Stripe auto-cancellation relative to trialEnd timestamp.
   - Recommendation: Use 24-hour tolerance. Stripe processes trial-end cancellations within minutes of the trial end time. A 24-hour window is generous and safe.

3. **Redirect for removed /subscribe route**
   - What we know: D-13 removes /subscribe entirely.
   - What's unclear: Whether to add a redirect or let React Router show the catch-all/404.
   - Recommendation: Add a `<Navigate to="/dashboard" replace />` for `/subscribe` path. Dashboard will hit PaywallGuard, which shows the paywall if needed.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | No frontend test framework currently configured (static analysis only -- ESLint, TypeScript) |
| Config file | None |
| Quick run command | `cd trade-flow-ui && npx tsc --noEmit` |
| Full suite command | `cd trade-flow-ui && npx tsc --noEmit && npx eslint src/` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PAYWALL-01 | Invalid subscription sees blocking page | manual | Visual verification in browser | N/A |
| PAYWALL-02 | Differentiated messaging per state | manual | Visual verification of 3 variants | N/A |
| PAYWALL-03 | Stripe Billing Portal accessible from paywall | manual | Click Manage Billing, verify portal opens | N/A |
| PAYWALL-04 | "Your data is safe" messaging visible | manual | Visual verification of copy | N/A |
| PAYWALL-05 | Support role bypasses paywall | manual | Login as support user, verify app access | N/A |
| PAYWALL-06 | Soft paywall removed | unit | `npx tsc --noEmit` (compilation verifies no broken imports) | N/A |

### Sampling Rate
- **Per task commit:** `cd trade-flow-ui && npx tsc --noEmit`
- **Per wave merge:** `cd trade-flow-ui && npx tsc --noEmit && npx eslint src/`
- **Phase gate:** Full TypeScript compilation + ESLint clean + manual verification of all 3 paywall variants

### Wave 0 Gaps
None -- existing TypeScript compilation serves as the primary automated verification for this frontend-only phase. Manual testing covers the visual/behavioral requirements.

## Sources

### Primary (HIGH confidence)
- `.planning/phases/38-hard-paywall-and-soft-paywall-removal/38-CONTEXT.md` -- All locked decisions (D-01 through D-15)
- `.planning/phases/38-hard-paywall-and-soft-paywall-removal/38-UI-SPEC.md` -- Full visual/interaction contract
- `.planning/research/ARCHITECTURE.md` -- Route architecture, PaywallGuard pattern, removal inventory
- `.planning/milestones/v1.6-phases/32-subscription-gate-and-subscribe-pages/32-CONTEXT.md` -- Original soft paywall architecture being removed

### Secondary (MEDIUM confidence)
- `.planning/phases/35-no-card-trial-api-endpoint/35-CONTEXT.md` -- Subscription entity fields for state detection (D-13)
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` -- PaywallGuard shell placement in route tree (D-05 through D-08)
- `.planning/phases/37-onboarding-wizard-pages/37-CONTEXT.md` -- Onboarding wizard visual pattern to match
- `.planning/research/STACK.md` -- Guard pattern, paywall rendering approach

### Tertiary (LOW confidence)
- None -- all findings are from project documentation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies, all libraries already in project
- Architecture: HIGH -- guard pattern, route structure, and component structure all documented in prior phases
- Pitfalls: HIGH -- derived from existing soft paywall implementation experience and known RTK Query caching behaviors

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable -- no external dependency changes expected)
