# Phase 38: Hard Paywall and Soft Paywall Removal - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace the soft paywall (read-only mode + modal on write actions) with a full-screen blocking page when subscription is invalid. Implement `PaywallGuard` blocking logic (Phase 36 created the pass-through shell). Build a `PaywallPage` with state-specific messaging for trial expired, payment failed, and canceled subscriptions. Remove all soft paywall infrastructure: `SubscriptionGatedLayout`, `PaywallModal`, `paywallSlice`, `PersistentCta`, `DashboardBanner`, `useSubscription.openPaywall()`, and all per-page `openPaywall()` dispatch calls. Support role users bypass the hard paywall entirely.

</domain>

<decisions>
## Implementation Decisions

### Paywall Messaging
- **D-01:** State-specific headlines with shared body copy pattern. Three variants:
  - **Trial expired:** "Your free trial has ended" + "Your jobs, quotes, and customers are safe -- pick up where you left off." + Subscribe CTA + Manage Billing
  - **Payment failed:** "There's a problem with your payment" + "Update your payment method to get back to work." + Manage Billing only (no Subscribe -- they already have a subscription)
  - **Canceled:** "Your subscription is canceled" + "Your data is still here. Resubscribe to pick up where you left off." + Subscribe CTA + Manage Billing
- **D-02:** "Your data is safe" reassurance is inline in the body copy -- natural part of the message, not a separate callout. Confident, not panicky. Matches Phase 36 D-01 casual tradesperson tone.
- **D-03:** Subscribe button goes directly to Stripe Checkout (calls POST /v1/subscription/checkout, redirects to Stripe). No intermediate /subscribe pricing page. Fewer clicks, user is already committed.
- **D-04:** Manage Billing button opens Stripe Billing Portal in a new tab (calls existing manage endpoint for portal URL). Available on all three states.
- **D-05:** Subscription state detection uses existing entity fields: `status`, `trialEnd`, `canceledAt` (Phase 35 D-13). Trial expired = status canceled + canceledAt close to trialEnd. Payment failed = status past_due. Canceled = status canceled + canceledAt not near trialEnd. No new schema fields needed.

### Paywall Page Layout
- **D-06:** Centered card on clean/gradient background with Trade Flow logo above -- same pattern as onboarding wizard (Phase 37 D-01/D-02/D-03). Consistent full-screen blocking experience across the app.
- **D-07:** Full-width on mobile (with padding), max-width ~480px on desktop -- matching onboarding wizard responsive pattern (Phase 37 D-07).
- **D-08:** PaywallGuard shows the existing loading screen while subscription status is being fetched -- seamless transition from app loading screen to either paywall or application. Active users never see a paywall flash.
- **D-09:** Subtle "Log out" text link below the card or at bottom of page. Unobtrusive but available for shared devices or account switching.

### Settings Access
- **D-10:** Billing Portal link on the paywall page is sufficient. No access to Settings or any app routes through the paywall. Keeps the guard simple and blocking complete.
- **D-11:** PaywallGuard blocks ALL routes inside DashboardLayout. No allowlisting, no exceptions (support users bypass at the guard level, not via route exceptions).

### Removal Scope -- Clean Sweep
- **D-12:** Remove ALL soft paywall infrastructure:
  - `SubscriptionGatedLayout` component
  - `PaywallModal` component
  - `paywallSlice` Redux slice
  - `PersistentCta` component (header button + mobile banner)
  - `DashboardBanner` component
  - `useSubscription` hook's `openPaywall` method (hook stays, loses openPaywall -- only exposes isActive/isLoading)
  - All per-page `openPaywall()` dispatch calls across features (customers, jobs, quotes, items, etc.)
- **D-13:** Remove the `/subscribe` pricing page route entirely -- redundant now that Subscribe goes directly to Stripe Checkout from the paywall page.
- **D-14:** Keep `/subscribe/success` and `/subscribe/cancel` routes as Stripe Checkout redirect targets. Move them OUTSIDE the PaywallGuard -- sibling routes to DashboardLayout, inside ProtectedRoute + OnboardingGuard but outside PaywallGuard. They're transient pages that redirect to dashboard on completion.
- **D-15:** Support role bypass: check `user.supportRoles.length > 0` in PaywallGuard (same check as Phase 32 D-03). If support user, render `<Outlet />` regardless of subscription status.

### Claude's Discretion
- Exact card dimensions, padding, and shadow treatment for paywall page
- Gradient background color values (should match onboarding wizard)
- Subscribe button icon and exact copy ("Subscribe -- GBP6/month" vs "Start subscription" etc.)
- Manage Billing button styling (primary vs secondary/outline)
- Log out link exact positioning and styling
- How subscription state is determined in the guard (direct hook call vs context)
- Loading screen component reuse approach
- Order of removal operations during implementation

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.7 Requirements
- `.planning/REQUIREMENTS.md` -- Full v1.7 requirements; Phase 38 covers PAYWALL-01 through PAYWALL-06

### Roadmap
- `.planning/ROADMAP.md` §Phase 38 -- Phase goal, success criteria, dependency on Phase 36

### Prior Phase Context (REQUIRED -- decisions carry forward)
- `.planning/phases/35-no-card-trial-api-endpoint/35-CONTEXT.md` -- Trial endpoint, subscription entity fields (D-13: trialEnd/canceledAt for state detection), auto-cancel behavior
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` -- PaywallGuard shell (D-05 through D-08), route tree structure, bundle isolation
- `.planning/phases/37-onboarding-wizard-pages/37-CONTEXT.md` -- Centered card layout pattern (D-01/D-02/D-03), gradient background, loading screen style
- `.planning/milestones/v1.6-phases/32-subscription-gate-and-subscribe-pages/32-CONTEXT.md` -- Existing soft paywall architecture (SubscriptionGatedLayout, PaywallModal, paywallSlice, PersistentCta, DashboardBanner, useSubscription hook, openPaywall, /subscribe pages) -- ALL of this gets removed

### Research
- `.planning/research/ARCHITECTURE.md` -- HardPaywallGuard architecture, PaywallPage component, route restructure, removal inventory
- `.planning/research/STACK.md` -- Guard pattern, paywall screen rendering approach

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `PaywallGuard` shell (Phase 36): Pass-through `<Outlet />` component in route tree -- implement blocking logic here
- `useSubscription` hook (Phase 32): Already fetches subscription status via RTK Query -- simplify by removing `openPaywall`
- `subscriptionApi` RTK Query endpoints: `getSubscription` (GET /v1/subscription) and `verifySession` already exist
- Onboarding wizard layout (Phase 37): Centered card + gradient background + logo -- reuse same visual pattern for paywall page
- Existing loading screen: Reuse for PaywallGuard loading state during subscription fetch
- Stripe Billing Portal integration: Existing manage endpoint returns portal URL -- reuse for "Manage Billing" CTA
- Stripe Checkout integration: Existing POST /v1/subscription/checkout -- reuse for "Subscribe" CTA

### Established Patterns
- Route guards as layout routes: `ProtectedRoute` > `OnboardingGuard` > `PaywallGuard` > `DashboardLayout` (Phase 36 D-05)
- Support role bypass: `user.supportRoles.length > 0` check (Phase 32 D-03)
- Subscription status enum: trialing, active, past_due, canceled, incomplete (Phase 29)
- RTK Query 404 handling: `getSubscription` returns null for no subscription (Phase 32 D-07)

### Integration Points
- `App.tsx` route tree: PaywallGuard already positioned, needs blocking logic implementation
- `useSubscription` hook: Remove `openPaywall` method, keep `isActive`/`isLoading`
- All feature components with `openPaywall()` calls: Remove dispatch calls and related gating logic
- `/subscribe/success` and `/subscribe/cancel`: Relocate in route tree outside PaywallGuard

</code_context>

<specifics>
## Specific Ideas

- Loading screen should provide seamless transition -- user sees loading screen, then either the app (if active) or the paywall (if blocked). No flash of paywall for active users.
- Paywall page should feel like the onboarding wizard -- same visual language, same centered card, same background. Cohesive "Trade Flow blocking state" experience.
- Copy tone matches the rest of v1.7: direct, casual, tradesperson-friendly. No corporate language.

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 38-hard-paywall-and-soft-paywall-removal*
*Context gathered: 2026-04-02*
