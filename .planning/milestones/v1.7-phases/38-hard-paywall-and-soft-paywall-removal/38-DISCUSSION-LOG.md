# Phase 38: Hard Paywall and Soft Paywall Removal - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-02
**Phase:** 38-hard-paywall-and-soft-paywall-removal
**Areas discussed:** Paywall messaging, Paywall page layout, Settings access, Removal scope

---

## Paywall Messaging

### Q1: How should the paywall differentiate between subscription states?

| Option | Description | Selected |
|--------|-------------|----------|
| State-specific headlines | Different headline per state: "Your trial has ended" / "Payment issue" / "Subscription canceled" -- with shared body copy and CTA below | ✓ |
| Single generic message | One message for all states: "Your account needs attention" with a Billing Portal link | |
| State-specific with urgency tiers | Like option 1 but adds urgency styling -- yellow for payment failed, red for canceled | |

**User's choice:** State-specific headlines
**Notes:** Clear, honest, no confusion. Each state gets its own headline with appropriate body copy and CTAs.

### Q2: Should Subscribe button go to /subscribe page or directly to Stripe Checkout?

| Option | Description | Selected |
|--------|-------------|----------|
| Direct to Stripe Checkout | Subscribe button calls POST /v1/subscription/checkout and redirects straight to Stripe | ✓ |
| Navigate to /subscribe page | Subscribe button goes to /subscribe pricing card page | |

**User's choice:** Direct to Stripe Checkout
**Notes:** Fewer clicks, user is already committed to subscribing.

### Q3: How prominent should the "your data is safe" reassurance be?

| Option | Description | Selected |
|--------|-------------|----------|
| Inline in body copy | Natural part of the message. Feels confident, not panicky | ✓ |
| Icon callout below CTAs | Separate callout section with shield/lock icon | |

**User's choice:** Inline in body copy
**Notes:** Matches the casual tone established in Phase 36.

---

## Paywall Page Layout

### Q1: How should the full-screen blocking page be laid out?

| Option | Description | Selected |
|--------|-------------|----------|
| Centered card | Same pattern as onboarding wizard -- centered card on clean/gradient background with Trade Flow logo | ✓ |
| Full-width hero section | Marketing-style layout with big heading and illustration area | |
| Split layout | Left side messaging, right side illustration | |

**User's choice:** Centered card
**Notes:** Consistent full-screen experience across blocking states (onboarding, paywall).

### Q2: Should PaywallGuard show a loading skeleton while subscription status loads?

| Option | Description | Selected |
|--------|-------------|----------|
| Skeleton loader | Brief skeleton/spinner while subscription data loads | |
| No skeleton, trust cache | RTK Query cache should have data, only first-load might flash | |

**User's choice:** Other -- Use the existing loading screen
**Notes:** User specified: "Use the existing loading screen so it is a seamless transition from the loading screen to either the paywall or application."

---

## Settings Access

### Q1: Should users behind the hard paywall access Settings > Billing?

| Option | Description | Selected |
|--------|-------------|----------|
| Billing Portal link only | Paywall page has Manage Billing button that opens Stripe Billing Portal | ✓ |
| Allow Settings access | Add Go to Settings link, PaywallGuard allowlists /settings routes | |
| Log out link only | Just add Log out alongside billing CTA | |

**User's choice:** Billing Portal link only
**Notes:** Keeps the guard simple and blocking complete.

### Q2: Should there be a Log out option on the paywall page?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, subtle link | Small Log out text link below the card or at bottom of page | ✓ |
| No log out | Only Subscribe and Manage Billing | |

**User's choice:** Yes, subtle link
**Notes:** Unobtrusive but available for shared devices or account switching.

---

## Removal Scope

### Q1: Anything to keep from the soft paywall infrastructure?

| Option | Description | Selected |
|--------|-------------|----------|
| Remove everything listed | Clean sweep -- hard paywall replaces all of it | ✓ |
| Keep DashboardBanner | Repurpose for different message | |
| Keep PersistentCta | Keep header button for upsell | |

**User's choice:** Remove everything listed
**Notes:** Clean sweep. useSubscription hook stays but loses openPaywall.

### Q2: Should /subscribe page route be kept or removed?

| Option | Description | Selected |
|--------|-------------|----------|
| Remove /subscribe routes | Subscribe goes straight to Stripe Checkout from paywall page | ✓ |
| Keep /subscribe outside paywall | Move outside PaywallGuard so it's still accessible | |
| Keep all /subscribe routes as-is | Don't touch /subscribe pages | |

**User's choice:** Remove /subscribe routes
**Notes:** Redundant now that Subscribe goes directly to Stripe Checkout.

### Q3: Where should /subscribe/success and /subscribe/cancel sit in the route tree?

| Option | Description | Selected |
|--------|-------------|----------|
| Outside PaywallGuard | Sibling routes to DashboardLayout, inside ProtectedRoute but outside PaywallGuard | ✓ |
| PaywallGuard allowlist | Guard checks path and skips blocking for /subscribe/* | |

**User's choice:** Outside PaywallGuard
**Notes:** Clean -- transient pages that redirect to dashboard on completion.

---

## Claude's Discretion

- Exact card dimensions, padding, shadow treatment
- Gradient background color values
- Subscribe button icon and exact copy
- Manage Billing button styling
- Log out link positioning and styling
- Subscription state detection approach in guard
- Loading screen component reuse approach
- Order of removal operations

## Deferred Ideas

None -- discussion stayed within phase scope
