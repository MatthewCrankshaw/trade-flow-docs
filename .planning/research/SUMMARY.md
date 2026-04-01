# Project Research Summary

**Project:** Trade Flow v1.7 — Onboarding & Landing Page
**Domain:** SaaS onboarding overhaul, public landing page, no-card Stripe trial, hard paywall
**Researched:** 2026-03-31
**Confidence:** HIGH

## Executive Summary

Trade Flow v1.7 is a conversion and activation overhaul for an existing SaaS product. The milestone replaces a dismissible, optional onboarding experience with a mandatory wizard, introduces a public landing page as the primary acquisition surface, switches Stripe subscription creation from a card-required Checkout flow to a no-card trial via the direct Subscriptions API, and replaces the v1.6 soft paywall modal with a hard full-screen block. All four changes follow well-documented patterns from the trades software competitors (ServiceM8, Fergus, Tradify, Jobber) and from Stripe's official trial documentation. The research is high-confidence across all four dimensions.

The recommended approach is to build in five discrete workstreams with two parallel tracks available: the backend no-card trial endpoint can be built independently of the frontend route restructure, and the two can converge when the onboarding wizard pages wire up to the new API. The key architectural insight is to derive onboarding state from the existence of domain entities (user profile name, business record, subscription record) rather than storing separate progress state — this eliminates an entire Redux slice, middleware, context, and localStorage layer while also correctly handling existing users who already have businesses and subscriptions.

The primary risk is the transition for existing users: deploying a mandatory onboarding gate without verifying that existing users with businesses and subscriptions pass the guard cleanly will lock out paying customers. A secondary risk is the Stripe webhook flow change — the no-card trial fires `customer.subscription.created` (not `checkout.session.completed`), and if the existing webhook processor only creates local subscription records from the Checkout event, new trial users get no local record and hit the hard paywall immediately. Both risks are known and preventable with targeted testing before deploying the mandatory flow.

## Key Findings

### Recommended Stack

The stack changes for this milestone are minimal. Only one new frontend package is needed: `motion` (12.x, the rebranded Framer Motion) for landing page scroll animations. `react-intersection-observer` (10.x) is optional and should be evaluated during implementation. The backend requires no new packages — the Stripe SDK (v21.x) already installed in v1.6 handles direct subscription creation natively.

**Core technologies:**
- `motion` 12.x: Landing page scroll animations (stagger reveals, hero transitions) — successor to Framer Motion, React 19 compatible, 15KB gzipped, import from `motion/react`
- Stripe Subscriptions API (existing SDK): No-card trial creation via `stripe.subscriptions.create()` with `trial_period_days: 30` and `payment_behavior: "default_incomplete"` — bypasses Checkout entirely
- React Router 7.x layout routes (existing): Three-tier nested guards (ProtectedRoute > OnboardingGuard > HardPaywallGuard) — no new routing packages
- RTK Query (existing): Onboarding state derived from `useGetUserQuery()` + `useGetBusinessesQuery()` — replaces Redux slice approach

**What NOT to use:**
- `framer-motion` package (legacy name, no longer maintained — use `motion`)
- Stripe Checkout for no-card trial (unnecessary redirect to empty hosted page)
- Any stepper library (2-step wizard is trivial with React state)
- `@stripe/react-stripe-js` (no card elements needed — Billing Portal handles card collection later)

### Expected Features

**Must have (v1.7 table stakes):**
- Public landing page with hero, features section, pricing card, and single CTA — every competitor has this as primary acquisition surface
- Mandatory profile setup (display name) — feeds greeting, invoice identity, and personalisation
- Mandatory business setup (name + trade selection) — triggers auto-provisioned defaults that already exist
- No-card 30-day Stripe trial — industry standard; all four competitors offer no-card trials; Trade Flow's 30-day period undercuts the standard 14-day competitor offering
- Mandatory onboarding route guards — redirect incomplete users, make setup non-skippable
- Hard paywall full-screen block — replaces soft modal; industry standard for post-trial lockout
- Welcome dashboard with personalised greeting and getting-started widget — reduces blank-slate anxiety on first login

**Should have (competitive differentiators):**
- Trade-specific landing page copy mentioning plumbers, electricians, and builders by name — avoids generic "field service" positioning
- Single pricing card (GBP 6/month) prominently displayed — radically undercuts competitors charging GBP 29-75/month; simplicity is a selling point
- Contextual trial-ending nudges at 7, 3, and 1 day before trial end — converts better than email alone
- Differentiated paywall states (trial expired vs payment failed vs canceled) — reduces confusion and support tickets

**Defer to v1.x after validation:**
- Trial-ending email nudges (webhook infrastructure exists, just needs `customer.subscription.trial_will_end` handler)
- Enhanced trial chip with urgency visual states
- Landing page testimonials and social proof (needs real user quotes)

**Defer to v2+:**
- Multi-page marketing site (no SEO/content strategy yet)
- Product demo video
- Interactive product tour (validate checklist approach first)
- A/B testing on landing page (needs traffic volume)

### Architecture Approach

The architectural shift is primarily a routing and state management simplification. The v1.7 route tree introduces three nested layout guards replacing the single SubscriptionGatedLayout approach. Onboarding state moves from a Redux slice with middleware and localStorage persistence to a `useOnboardingStatus` hook that derives step from RTK Query cache — this eliminates six files of state management infrastructure while becoming more correct by using actual data as truth. The hard paywall replaces fourteen files worth of per-page `openPaywall()` dispatch calls with a single layout route component.

**Major components:**

1. **LandingPage** — public marketing page at `/`; redirects authenticated users to `/dashboard`; zero imports from app modules (store, features, auth hooks)
2. **OnboardingGuard** — layout route that checks `displayName` existence and business existence; redirects to appropriate onboarding step; derived entirely from RTK Query cache
3. **ProfileSetupPage + BusinessSetupPage** — mandatory wizard pages at `/onboarding/profile` and `/onboarding/business`; BusinessSetupPage calls `POST /v1/business` then `POST /v1/subscription/trial` sequentially
4. **HardPaywallGuard** — layout route replacing SubscriptionGatedLayout; renders `PaywallPage` full-screen for any subscription status outside `trialing` or `active`; settings route always passes
5. **SubscriptionCreator (API)** — new `createTrialWithoutCard` method calling `stripe.subscriptions.create()` with `trial_period_days: 30`, `payment_behavior: "default_incomplete"`, `trial_settings.end_behavior.missing_payment_method: "cancel"`; new `POST /v1/subscription/trial` endpoint decorated with `@SkipSubscriptionCheck()`
6. **useOnboardingStatus hook** — derives `profileComplete`, `businessComplete`, `currentStep` from existing RTK Query user and business queries; no stored state; replaces Redux slice, middleware, context, localStorage
7. **WelcomeDashboard variant** — conditional rendering on existing DashboardPage for users with no jobs; personalised greeting using `displayName` + GettingStartedWidget

**Files removed (cleanup):** 15 UI files across onboarding components, context, Redux slice, middleware, paywall modal, persistent CTA, dashboard banner, and SubscriptionGatedLayout.

### Critical Pitfalls

1. **Existing users locked out by mandatory onboarding guard** — Guard must check actual data state (has `displayName` + business + subscription), not a boolean flag. Existing users with businesses and subscriptions pass automatically. Deploy backend state evaluation before enabling frontend mandatory flow. Test against at least five user state variations (existing active, existing trialing, existing canceled, existing no-subscription, brand new).

2. **Stripe webhook flow change breaks new trial subscriptions** — Switching from Checkout to direct API changes the entry webhook event from `checkout.session.completed` to `customer.subscription.created`. If the webhook processor only creates local records from the Checkout event, new trial users get no local subscription record and hit the hard paywall immediately. Fix: ensure `customer.subscription.created` handler can create a fresh record (not just update), and also write the local record synchronously in the onboarding endpoint as belt-and-suspenders.

3. **Duplicate businesses and subscriptions on retry or refresh** — Onboarding state resets on page refresh. Business creation and Stripe subscription creation must be idempotent: return existing business if one already exists for the user; use Stripe idempotency keys tied to `userId` on `subscriptions.create()`. Wizard must resume at the first incomplete step on mount, not always from step 1.

4. **Hard paywall false positives for `past_due` subscriptions** — Stripe's payment retry window is 1-4 weeks during which status fluctuates. A hard block during `past_due` locks out users who are actively retrying payment. Define an explicit status-to-access mapping: `trialing` and `active` = full access; `past_due` = grace period; `canceled` and `incomplete` = hard paywall. Never render the hard paywall while subscription status is loading.

5. **Landing page loads the full 500KB+ app bundle** — Adding `/` as a standard React Router route pulls in React, Redux, RTK Query, and all feature modules. The landing page must import zero app-level dependencies. Use `React.lazy()` with bundle analysis verification as minimum viable; use `vite-plugin-prerender` for better SEO; consider separate Vite entry point if organic search is a day-one acquisition channel.

## Implications for Roadmap

Based on combined research, the following phase structure is recommended. The architecture research explicitly specifies a build order based on dependencies; this maps directly to phases.

### Phase 1: No-Card Trial API Endpoint

**Rationale:** Backend has no frontend dependency. Building it first means the onboarding wizard pages have a real endpoint to wire to. This is also the highest-risk technical change (Stripe API flow change) and benefits from being isolated and tested independently before the frontend mandatory flow is deployed.

**Delivers:** `POST /v1/subscription/trial` endpoint; `createTrialWithoutCard` service method; updated webhook handler ensuring `customer.subscription.created` creates local records; Stripe idempotency key on subscription creation; schema audit confirming Checkout-created and API-created subscriptions produce compatible local records.

**Addresses:** No-card trial feature, Stripe direct API pattern

**Avoids:** Pitfall 2 (webhook flow breaks new subscriptions), Pitfall 8 (schema incompatibility between Checkout and API paths)

**Research flag:** Standard patterns — skip research-phase.

---

### Phase 2: Public Landing Page and Route Restructure

**Rationale:** Can run in parallel with Phase 1 (separate repos). The route architecture decision must be locked before building the onboarding wizard pages — the guard nesting and page paths are dependencies for Phase 3. The landing page itself is independent of all Stripe changes.

**Delivers:** LandingPage at `/`; restructured App.tsx with three-tier guard nesting; OnboardingGuard and HardPaywallGuard as shells (redirect/block logic without full wizard pages wired yet); authenticated-user redirect from `/` to `/dashboard`; explicit auth loading state to prevent flash of landing content for authenticated users.

**Addresses:** Public landing page, trade-specific copy and single pricing display, routing architecture

**Avoids:** Pitfall 3 (root route conflict with auth-async state), Pitfall 7 (landing page bundle bloat — zero app imports enforced)

**Research flag:** Standard SPA routing patterns — skip research-phase. Bundle analysis verification is the main deliverable check.

---

### Phase 3: Onboarding Wizard Pages

**Rationale:** Depends on Phase 2 (route structure must exist) and Phase 1 (real trial endpoint must exist for wiring). This is the phase that makes onboarding mandatory. Must address existing user migration and wizard resumability before going live.

**Delivers:** ProfileSetupPage with display name form; BusinessSetupPage wired to `POST /v1/business` then `POST /v1/subscription/trial`; `useOnboardingStatus` hook deriving step from RTK Query cache; OnboardingGuard fully implemented; removal of old onboarding infrastructure (Redux slice, middleware, context, provider, dialog components — 10 files).

**Addresses:** Mandatory profile setup, mandatory business setup, mandatory onboarding route guards, getting-started widget

**Avoids:** Pitfall 1 (existing users locked out — guard uses derived data state), Pitfall 5 (duplicate creation — idempotent business creation and Stripe idempotency keys), Pitfall 9 (old onboarding removed too early — recommend feature flag)

**Research flag:** The existing user migration edge case needs validation against a production data snapshot before this phase ships. Recommend `/gsd:research-phase` specifically for the migration test matrix (user states: existing active, existing trialing, existing canceled, existing no-subscription, brand new).

---

### Phase 4: Hard Paywall and Soft Paywall Removal

**Rationale:** Can run in parallel with Phase 3 (both depend on Phase 2 route structure but not on each other). Removing the soft paywall is a clean mechanical cut once HardPaywallGuard is in place — `openPaywall()` calls across eight business pages are straightforward removals.

**Delivers:** PaywallPage full-screen component with differentiated states (trial expired, payment failed, canceled); HardPaywallGuard fully implemented with explicit subscription status-to-access mapping; grace period logic for `past_due` status; "Verify subscription" button on paywall; removal of PaywallModal, PersistentCta, DashboardBanner, paywallSlice, SubscriptionGatedLayout, and all `openPaywall()` dispatch calls (5 files removed, 8 pages modified).

**Addresses:** Hard paywall, differentiated paywall states, data preservation messaging

**Avoids:** Pitfall 4 (false positives for `past_due` — grace period required), Pitfall 6 (no-card trial churn — trial-expired CTA pointing directly to Billing Portal)

**Research flag:** The `past_due` grace period duration is a product decision — flag for user input during planning. Standard guard pattern otherwise — skip research-phase.

---

### Phase 5: Welcome Dashboard and Final Cleanup

**Rationale:** Depends on Phase 3 (profile name from onboarding feeds the greeting; business existence drives the welcome variant). Last phase because it enhances rather than enables — the app is fully functional after Phase 4. Also the correct time for final cleanup after new flows have been validated.

**Delivers:** Welcome dashboard variant on DashboardPage (conditional on zero jobs); personalised greeting using `displayName`; GettingStartedWidget with "Create your first job" and "Send your first quote" checklist items; final removal of any remaining dead code from old onboarding and paywall systems.

**Addresses:** Welcome dashboard with personalised greeting, getting-started widget, removal of existing dismissible onboarding flow

**Avoids:** Pitfall 9 (old onboarding removed too early — cleanup happens after Phases 3 and 4 are production-validated)

**Research flag:** Standard patterns — skip research-phase.

---

### Phase Ordering Rationale

- **Phases 1 and 2 run in parallel** — different repos (API and UI), no dependencies between them. Maximises development throughput.
- **Phases 3 and 4 run in parallel** — both depend on Phase 2 route structure, but not on each other.
- **Phase 5 is last** — it depends on Phase 3 (displayName from onboarding) and cleans up after all other phases are validated in production.
- **Stripe webhook changes ship in Phase 1** — before the onboarding wizard calls the new endpoint, preventing the failure mode where onboarding succeeds but the user has no local subscription record.
- **Existing user migration must be validated before Phase 3 goes live** — the mandatory onboarding guard is the highest-risk user-facing change in this milestone.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Onboarding Wizard):** Existing user migration test matrix. Run the onboarding guard logic against a production data snapshot to verify all user states pass correctly. This is a verification task, not research into unknowns — but it must happen before the mandatory redirect guards are deployed.

Phases with standard patterns (skip research-phase):
- **Phase 1 (No-Card Trial API):** Stripe API patterns fully documented; existing NestJS service/controller/repository structure is established.
- **Phase 2 (Landing Page + Routes):** React Router layout routes are standard; landing page is a straightforward public component.
- **Phase 4 (Hard Paywall):** Layout route guard pattern is identical to existing SubscriptionGatedLayout — just a stricter version.
- **Phase 5 (Welcome Dashboard):** Standard conditional rendering on an existing page; getting-started widget follows existing patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Stripe API verified against official docs; Motion v12.38.0 confirmed current; all existing technology reuse confirmed from installed versions |
| Features | HIGH | Competitor analysis conducted against live sites on 2026-03-31; Stripe trial conversion statistics from industry sources |
| Architecture | HIGH | Route structure based on existing codebase patterns; Stripe subscription creation verified against official API docs; file removal inventory based on confirmed existing codebase structure |
| Pitfalls | HIGH | Based on Stripe official docs, existing codebase knowledge (v1.6 implementation details), and established SPA routing patterns |

**Overall confidence:** HIGH

### Gaps to Address

- **`past_due` grace period duration:** Research recommends a grace period exists but does not prescribe duration (7 days? 14 days?). Flag for product decision during Phase 4 planning.

- **Landing page rendering strategy:** Three options with increasing complexity: `React.lazy()` with bundle analysis (minimum viable), `vite-plugin-prerender` (better SEO), separate Vite entry point (best SEO isolation). Right choice depends on whether organic search is a day-one acquisition channel — flag for decision during Phase 2 planning.

- **`react-intersection-observer` inclusion:** Marked optional in research. Defer decision to Phase 2 implementation — evaluate whether Motion's `whileInView` covers all landing page visibility needs before adding the package.

- **Old onboarding feature flag:** Research recommends an environment variable (`FEATURE_MANDATORY_ONBOARDING=true`) to allow rollback. Adds a small implementation cost in Phase 3. Flag for discussion during Phase 3 planning based on deployment risk tolerance.

## Sources

### Primary (HIGH confidence)
- [Stripe: Use free trial periods on subscriptions](https://docs.stripe.com/billing/subscriptions/trials/free-trials) — no-card trial API, `trial_period_days`, `trial_settings.end_behavior.missing_payment_method`
- [Stripe: Create a subscription API](https://docs.stripe.com/api/subscriptions/create) — `payment_behavior: "default_incomplete"`, idempotency keys
- [Stripe: Using webhooks with subscriptions](https://docs.stripe.com/billing/subscriptions/webhooks) — event sequence for API-created vs Checkout-created subscriptions
- [Motion official site + npm](https://motion.dev/) — v12.38.0 confirmed current, React 19 compatible, import from `motion/react`
- [react-intersection-observer npm](https://www.npmjs.com/package/react-intersection-observer) — v10.0.3 confirmed current
- Trade Flow existing codebase (ARCHITECTURE.md, STRUCTURE.md, PROJECT.md) — v1.6 implementation patterns, existing route structure, subscription schema

### Secondary (MEDIUM confidence)
- [ServiceM8](https://www.servicem8.com), [Fergus](https://fergus.com), [Tradify](https://www.tradifyhq.com), [Jobber](https://www.getjobber.com) — live competitor site analysis, 2026-03-31
- [Stripe: Configure free trials (Checkout)](https://docs.stripe.com/payments/checkout/free-trials) — Checkout approach considered and rejected for no-card flow
- [SaaS Free Trial Conversion Statistics 2025](https://www.amraandelma.com/free-trial-conversion-statistics/) — 18-25% opt-in vs ~49% opt-out conversion rates
- [Hard Paywall vs Soft Paywall (RevenueCat)](https://www.revenuecat.com/blog/growth/hard-paywall-vs-soft-paywall/) — conversion trade-off analysis

### Tertiary (LOW confidence — needs validation)
- [Airtable Onboarding Wizard case study](https://www.candu.ai/blog/airtables-best-wizard-onboarding-flow) — 20% activation lift from wizard approach (single source)
- [SaaS Onboarding Best Practices 2025](https://productled.com/blog/5-best-practices-for-better-saas-user-onboarding) — checklist patterns and time-to-value benchmarks

---
*Research completed: 2026-03-31*
*Ready for roadmap: yes*
