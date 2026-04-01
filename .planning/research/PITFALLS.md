# Pitfalls Research

**Domain:** Onboarding overhaul, public landing page, no-card Stripe trial, hard paywall -- added to existing Trade Flow SaaS (v1.7)
**Researched:** 2026-03-31
**Confidence:** HIGH (based on existing codebase knowledge, Stripe docs, SPA routing patterns)

## Critical Pitfalls

### Pitfall 1: Existing Users Locked Out by Mandatory Onboarding

**What goes wrong:**
Deploying a mandatory onboarding wizard that gates app access breaks the experience for every existing user. Users who already have a business, profile name, and active subscription are forced through a setup flow they completed months ago. If the onboarding guard checks a boolean `onboardingCompleted` flag that doesn't exist on old user records, every existing user fails the check and gets trapped in the wizard.

**Why it happens:**
The new onboarding flow assumes all users start fresh. The current app has a dismissible onboarding experience -- there is no server-side record of "this user completed onboarding." Existing users have businesses, subscriptions, and customers, but nothing that explicitly marks them as "onboarded" in the new system's terms.

**How to avoid:**
1. Build the onboarding guard to check **actual data state**, not a flag. If user has a profile name + business + valid subscription, they are onboarded regardless of any flag. This is the primary gate logic.
2. Optionally add a migration script that evaluates existing users and sets an `onboardingCompletedAt` timestamp based on current data state -- but the guard must work without it as a safety net.
3. Deploy the backend onboarding state evaluation endpoint **before** the frontend mandatory flow. Test against a production data snapshot.
4. Test with at least these user states: (a) existing user with business + active subscription, (b) existing user with business + trialing subscription, (c) existing user with business + canceled subscription, (d) existing user with business but no subscription (edge case from before v1.6), (e) brand new user with nothing.

**Warning signs:**
- QA only tests with fresh accounts
- No migration script or data-state guard in the implementation plan
- Onboarding guard checks a single boolean flag instead of derived state
- No test matrix for existing user states

**Phase to address:**
Phase 1 -- backend onboarding state model and migration must ship before any frontend mandatory flow enforcement.

---

### Pitfall 2: Stripe Webhook Flow Change Breaks Subscription Creation for New Users

**What goes wrong:**
The current v1.6 flow uses Stripe Checkout (hosted page) to create subscriptions. The entry point is `checkout.session.completed` webhook, which creates the local MongoDB subscription record. Switching to no-card trials via `stripe.subscriptions.create()` (server-side API call) bypasses Checkout entirely -- `checkout.session.completed` **never fires**. If the webhook processor relies on that event as the sole path to create local subscription records, new trial users get no local record and hit the paywall immediately after completing onboarding.

**Why it happens:**
Developers add the new `stripe.subscriptions.create()` call for no-card trials but don't realize the existing webhook pipeline assumes Checkout as the only subscription creation path. The `customer.subscription.created` event fires for API-created subscriptions, but if the existing handler only treats it as a secondary/update path (not a creation path), the local record is never created.

**How to avoid:**
1. Map the full webhook event sequence for both flows side by side:
   - **Current (Checkout with card):** `checkout.session.completed` -> creates local record -> `customer.subscription.updated` events follow
   - **New (API no-card trial):** `customer.subscription.created` (status: `trialing`, no payment method) -> must create local record here
2. Ensure the `customer.subscription.created` handler can create a fresh local subscription record from scratch, not just update an existing one. This likely means refactoring from an update-only handler to an upsert handler.
3. After calling `stripe.subscriptions.create()` in the onboarding endpoint, also write the local subscription record synchronously (belt-and-suspenders). The webhook then upserts over it.
4. Add the `customer.subscription.trial_will_end` event handler (fires 3 days before trial end) -- this is new and critical for no-card trials where you need to prompt for payment method.
5. Keep both code paths working: Checkout flow for "add card later" (users going through Billing Portal) and API flow for initial no-card trial creation.

**Warning signs:**
- Webhook processor switch/case only creates records from `checkout.session.completed`
- No integration test for `customer.subscription.created` creating a record from scratch
- Local subscription records appear for existing Checkout users but not for new API-created trial users
- No `customer.subscription.trial_will_end` handler exists

**Phase to address:**
Stripe integration phase -- must be implemented and tested before the onboarding flow calls the new subscription creation endpoint.

---

### Pitfall 3: Root Route Conflict -- Landing Page vs App Dashboard

**What goes wrong:**
The existing app serves the authenticated dashboard or a redirect at `/` (root path). Adding a public landing page at `/` creates a routing conflict. Three failure modes: (a) authenticated users see the marketing landing page every time they open the app, (b) unauthenticated users see a loading spinner then get redirected, or (c) the React Router config becomes a conditional tangle based on auth state at the root level.

**Why it happens:**
React Router treats routes as declarative. When you need the same path (`/`) to serve entirely different content based on auth state, the route tree doesn't naturally express this. The Firebase auth state is async (`onAuthStateChanged` hasn't fired yet on initial load), creating a third state -- "unknown" -- that neither the landing page nor the dashboard handles cleanly.

**How to avoid:**
1. **Best approach: Split URLs.** Landing page at `/` (public, no auth). Authenticated app at `/app/*` or `/dashboard`. This eliminates conditional root routing entirely. The tradeoff is existing bookmarks/URLs to `/customers`, `/jobs`, etc. need redirects to `/app/customers`, `/app/jobs`.
2. **If keeping single root:** Use a root layout component that checks auth state and renders either `<LandingPage />` or `<Navigate to="/dashboard" />`. Must handle the Firebase auth loading state explicitly (show a neutral loading screen, not the landing page) to avoid flash of marketing content for logged-in users.
3. Whichever approach: the landing page route must NOT be wrapped in `ProtectedRoute`, `SubscriptionGate`, or any auth guard. It must import zero auth/store dependencies.

**Warning signs:**
- Flash of landing page when authenticated user loads the app
- Flash of loading spinner when unauthenticated user visits `/`
- Landing page component importing from `@/store`, `@/hooks/useAuth`, or `@/features`
- No explicit handling of the "auth state loading" period (the 100-500ms before Firebase resolves)

**Phase to address:**
Phase 1 -- URL structure and routing architecture decision must come before any implementation.

---

### Pitfall 4: Hard Paywall Locks Out Users with Legitimate Subscriptions

**What goes wrong:**
Replacing the soft paywall modal (v1.6 -- blocks writes only) with a hard paywall (full blocking screen) increases the blast radius of subscription status errors from "annoying but usable" to "completely locked out." Any false negative in subscription status resolution (stale cache, webhook delay during renewal, Stripe API error, race condition during payment retry) now makes the app completely unusable instead of just read-only.

**Why it happens:**
The v1.6 soft paywall was forgiving by design -- users could still browse their data, which meant a brief status sync delay was invisible. A hard paywall means ANY moment where subscription status is unknown or stale results in complete lockout. Stripe's retry window for failed payments is 1-4 weeks, during which the subscription status fluctuates between `past_due` and `active`.

**How to avoid:**
1. Define an explicit state machine for subscription statuses and their UI consequences:
   - `trialing` -> full access
   - `active` -> full access
   - `past_due` -> **grace period: read-only for 7 days**, then hard paywall. Stripe is retrying the payment during this window.
   - `canceled` -> hard paywall (user explicitly canceled and period ended)
   - `incomplete` / `incomplete_expired` -> hard paywall (initial payment never completed)
   - `null` (no subscription) -> hard paywall with onboarding CTA
   - `loading` (status unknown) -> show app with loading indicator, NOT the paywall
2. Add a "Verify subscription" button on the paywall screen that calls the existing verify-session endpoint to force-refresh status from Stripe.
3. Never show the hard paywall while subscription status is still loading. Show a loading state instead.
4. Log paywall trigger events (userId, subscription status, timestamp) to detect false-positive lockouts.
5. Keep the `SubscriptionGuard` on API endpoints as the real enforcement -- the frontend paywall is a UX layer, not a security layer.

**Warning signs:**
- No grace period between `past_due` and full lockout
- Paywall renders during subscription status loading
- No manual refresh/verify option on the paywall screen
- Support tickets from users who clearly have valid subscriptions but got locked out during a payment retry

**Phase to address:**
Paywall phase -- implement with explicit status-to-access mapping and grace period logic.

---

### Pitfall 5: Onboarding Creates Duplicate Businesses or Stripe Subscriptions on Retry/Refresh

**What goes wrong:**
User starts onboarding, creates a business at step 2, refreshes the page, and the wizard starts from step 1 again. They complete step 2 again and now have two businesses. Or they reach the trial activation step, the Stripe API call fires, they hit back/refresh, and a second subscription is created. Now they have duplicate Stripe subscriptions and potentially duplicate charges when the trial ends.

**Why it happens:**
Onboarding wizard state is stored in React component state or Redux, which resets on page refresh. Without server-side tracking of onboarding progress, the wizard has no memory of what steps already completed. Business creation and Stripe subscription creation endpoints are not idempotent.

**How to avoid:**
1. Track onboarding progress on the server (user document or dedicated onboarding state document). Each step completion updates a server-side record with a timestamp.
2. On wizard mount, check what already exists: if business exists, skip that step. If subscription exists, skip that step. The wizard must be **resumable** -- on load, it jumps to the first incomplete step.
3. Make creation endpoints idempotent:
   - Business creation: if user already has a business, return the existing one (don't create a duplicate).
   - Stripe subscription: use an idempotency key on `stripe.subscriptions.create()` tied to the user ID. Stripe will return the same subscription on retry within 24 hours.
4. Add a unique index on `businesses` collection for `userId` (one business per user for now -- the solo operator constraint).

**Warning signs:**
- Onboarding state stored only in frontend (Redux/useState) with no server persistence
- No idempotency check on business creation endpoint
- No idempotency key on `stripe.subscriptions.create()` call
- Wizard always starts at step 1 regardless of existing data
- No unique index on `{ userId: 1 }` in businesses collection

**Phase to address:**
Onboarding implementation phase -- idempotency and resumability must be designed in from the start, not added after duplicate bugs surface.

---

### Pitfall 6: No-Card Trial Users Silently Churn -- No Payment Method Prompts

**What goes wrong:**
No-card trials have significantly lower conversion rates than card-upfront trials (industry data: 20-40% vs 60-80%). Without proactive in-app and email prompts to add a payment method before the trial ends, users simply forget, the trial expires, subscription goes to `past_due` then `canceled`, and the user is locked out with no smooth recovery path. The hard paywall compounds this: users who would have been nudged by the soft modal are now completely blocked.

**Why it happens:**
Developers build the trial creation flow but treat payment method collection as "they'll find the Billing Portal in settings." The Billing Portal link is buried, and users who haven't been reminded they're on a trial won't seek it out.

**How to avoid:**
1. Handle `customer.subscription.trial_will_end` webhook (fires 3 days before trial ends). Trigger both: (a) an in-app banner with a direct "Add payment method" CTA linking to Billing Portal, and (b) an email reminder via Resend.
2. Keep the trial days-remaining chip (from v1.6) visible at all times during trial.
3. When trial expires with no card: show a dedicated screen with clear messaging ("Your 30-day trial ended -- add a payment method to continue") with a prominent Billing Portal link. This is different from the generic "subscription canceled" paywall.
4. Differentiate paywall states in the UI:
   - Trial expired, no card -> "Add payment method to continue" (encouraging, recovery-focused)
   - Payment failed -> "Update your payment method" (action-focused)
   - Subscription canceled -> "Resubscribe to continue" (different flow entirely)

**Warning signs:**
- No `customer.subscription.trial_will_end` handler in webhook processor
- Trial expiry and subscription cancellation produce identical UI
- No email notification before trial ends
- Billing Portal is the only way to add a payment method (buried in Settings > Billing)
- All paywall states show the same generic message

**Phase to address:**
Stripe integration phase (webhook handling) + paywall phase (trial-specific UI states).

---

### Pitfall 7: Landing Page Loads the Full 500KB+ App Bundle

**What goes wrong:**
The landing page is a marketing page -- the first thing prospects see. If it's just another route inside the React SPA, the browser must download React, Redux, RTK Query, all feature modules, and the auth system before rendering a static page with a heading and a "Sign Up" button. This means 500KB+ of JavaScript for what should be a 10-50KB page. First Contentful Paint exceeds 3 seconds. Google's crawler may not execute the JS at all, killing SEO.

**Why it happens:**
Adding a `/` route to the existing React Router config is the path of least resistance. Even with `React.lazy()`, the React runtime + Router + Redux store must load before the lazy component can even begin loading. The landing page becomes a victim of the app's bundle size.

**How to avoid:**
1. **Minimum viable approach:** The landing page route uses `React.lazy()` and imports ZERO app-level dependencies -- no Redux, no RTK Query, no auth hooks, no feature components. It's a pure component with its own CSS. Verify with bundle analyzer that the landing chunk is under 50KB (excluding React runtime).
2. **Better:** Pre-render the landing page at build time. Use `vite-plugin-prerender` to generate static HTML for `/`. Visitors get instant HTML, then React hydrates. Bots get full HTML for SEO.
3. **Best (if SEO is critical):** Serve the landing page as a separate HTML entry point in Vite's multi-page config (`landing.html` for `/`, `index.html` for `/app/*`). Completely separate bundle. This is more build config complexity but guarantees zero app code leakage.

**Warning signs:**
- Landing page component imports from `@/store`, `@/features`, or `@/hooks/useAuth`
- Lighthouse performance score below 80 on landing page
- Time to First Contentful Paint exceeds 2 seconds on landing page
- Bundle analyzer shows app code in the landing page chunk
- Google Search Console shows landing page as "Discovered - currently not indexed"

**Phase to address:**
Landing page phase -- rendering strategy decision must precede implementation.

---

### Pitfall 8: Existing Checkout-Created Subscriptions Incompatible with New Schema/Logic

**What goes wrong:**
Existing users created subscriptions via Stripe Checkout (v1.6). The local MongoDB subscription record was populated from `checkout.session.completed` event data. The new no-card trial creates subscriptions via `stripe.subscriptions.create()`, which fires `customer.subscription.created` with a different event payload. If the new paywall logic or the new onboarding guard reads fields that only Checkout-created records have (e.g., `checkoutSessionId`, `paymentIntentId`), existing records work fine but new trial records are missing those fields, causing null reference errors or incorrect paywall behavior.

**Why it happens:**
The two subscription creation paths populate the local record from different webhook event shapes. Developers build and test the new path without verifying backward compatibility with existing records, or vice versa. The local subscription schema implicitly assumes one creation path.

**How to avoid:**
1. Audit the current local subscription schema. List every field and which webhook event populates it.
2. Design the schema so it works for BOTH creation paths. Fields specific to Checkout (like `checkoutSessionId`) should be nullable/optional.
3. The paywall logic and subscription status checks must ONLY depend on fields present in both paths: `status`, `currentPeriodStart`, `currentPeriodEnd`, `cancelAtPeriodEnd`, `trialEnd`, `stripeSubscriptionId`, `stripeCustomerId`.
4. Write integration tests that create subscriptions via both paths and verify the paywall logic produces identical results.

**Warning signs:**
- Local subscription document has required fields only populated by Checkout events
- Paywall logic reads fields like `paymentIntentId` or `checkoutSessionId` for status decisions
- Tests only use one subscription creation path

**Phase to address:**
Stripe integration phase -- schema audit must happen before building the new creation path.

---

### Pitfall 9: Removing Dismissible Onboarding Too Early

**What goes wrong:**
The existing dismissible onboarding is deleted as "cleanup" in the same deployment as the new mandatory flow. The new flow has a bug (wrong guard logic, missing step, broken redirect), and now new users have NO onboarding guidance and existing users who hadn't finished setup lose their prompts. There's no rollback that doesn't also restore the old code.

**Why it happens:**
Developers treat old and new onboarding as mutually exclusive and want clean code. The old code feels like dead weight once the new flow exists.

**How to avoid:**
1. Keep old onboarding code in place (but inactive) until the new flow has been verified with real users for at least 1 week in production.
2. Deploy the new mandatory flow with a simple feature flag (environment variable `FEATURE_MANDATORY_ONBOARDING=true`). If issues arise, flip to false and the old behavior returns.
3. Remove old onboarding code in a separate, later cleanup phase -- not in the same deployment as the new flow.

**Warning signs:**
- Old onboarding code deleted in the same PR/phase as new onboarding
- No feature flag or toggle mechanism
- No rollback plan documented
- New flow tested only with fresh accounts

**Phase to address:**
Final cleanup phase -- old code removal is its own task, after new flow is validated in production.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Onboarding state only in Redux (no server persistence) | Faster to build, no API changes | Resets on refresh, duplicate creation risk, no resumability | Never -- server-side state is essential for a mandatory multi-step flow |
| Skipping landing page code splitting/prerendering | Ship faster, simpler build | Poor SEO, slow FCP, bad first impression for prospects | MVP only -- must optimize before any marketing spend |
| Single webhook handler for both Checkout and API-created subscriptions | Less code | Fragile if payloads differ, hard to debug creation-path-specific issues | Acceptable if handler normalizes both payload shapes explicitly with clear comments |
| Hardcoding 30-day trial duration instead of reading from Stripe response | Simpler frontend code | Stripe config and app disagree if trial duration changes | Acceptable short-term -- trial chip can read `trialEnd` from Stripe for accurate countdown |
| No grace period on hard paywall | Simpler status-to-UI mapping | Users locked out by Stripe webhook delays, payment retries, or transient API errors | Never -- always allow grace period for `past_due` status |
| Keeping soft paywall as fallback for `past_due` and hard paywall for `canceled` only | Incremental change, less risk | Two paywall implementations to maintain | Acceptable as a transitional approach -- converge to one after validation |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Stripe `subscriptions.create()` no-card trial | Omitting `payment_behavior: 'default_incomplete'` -- causes "No attached payment source" error | Always set `payment_behavior: 'default_incomplete'` when creating subscriptions without a payment method |
| Stripe `subscriptions.create()` no-card trial | Not setting `trial_settings.end_behavior.missing_payment_method` | Set to `'cancel'` or `'pause'` to control what happens when trial ends with no card |
| Stripe trial end webhook | Not handling `customer.subscription.trial_will_end` | Add handler -- this is the 3-day warning to prompt for payment method collection |
| Stripe trial-to-paid transition | Assuming `active` means card is on file | After trial, if no card: status goes `past_due` (not `active`). Handle this explicitly. |
| Stripe Billing Portal | Assuming users will find it in Settings | Surface "Add payment method" CTA prominently in trial banner, link directly to portal session URL |
| Stripe idempotency | Not using idempotency keys on `subscriptions.create()` | Pass `idempotencyKey` tied to userId to prevent duplicate subscriptions on retry/refresh |
| Firebase Auth + public routes | Checking auth state synchronously before `onAuthStateChanged` fires | Auth state is async -- show neutral loading state during initialization, not landing page or dashboard |
| React Router + auth-conditional root | Redirecting to paywall/login while auth is still loading | Use a three-state guard: loading -> show spinner; authenticated -> show app; unauthenticated -> show landing |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Landing page inside main SPA bundle | 500KB+ initial download, 3s+ FCP, poor Lighthouse | Code splitting with zero app imports, or prerendering, or separate entry point | Immediately -- every first-time visitor is affected |
| Subscription status API call on every route change | Visible latency on navigation, unnecessary API load | Cache in Redux store, refresh on app mount + webhook push events | At 100+ concurrent users; but UX impact is immediate |
| Loading full onboarding wizard component tree for completed users | Unnecessary JS parse/execute, flash of wizard before redirect | Route-level guard checks server state BEFORE mounting wizard component | Immediately for all existing users post-deployment |
| Landing page images not optimized | Slow load on mobile, layout shift | Use WebP/AVIF, proper `width`/`height` attributes, lazy load below-fold images | Immediately -- landing page is first impression |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Landing page accidentally importing Firebase config via shared module | API keys and Firebase project ID exposed in a public-facing, cached bundle chunk | Landing page must have zero imports from auth/store/config modules; verify with bundle analyzer |
| Subscription status checked only on frontend (removing API guard) | Users bypass hard paywall by calling API directly via curl/Postman | Keep `SubscriptionGuard` on API endpoints (exists in v1.6) -- never remove it when adding frontend paywall |
| Onboarding endpoint creates Stripe subscription without verifying user identity | Attacker could create subscriptions tied to other users' Stripe customers | Validate authenticated user from JWT matches the business owner before calling Stripe API |
| Trial abuse -- same person creates multiple accounts | Unlimited free access via new email addresses | Tie Stripe customer to Firebase UID; log and monitor trial creation rate; accept this risk for v1.7 (not worth solving pre-PMF) |
| Public landing page route accidentally serving authenticated API responses | Data leak if SSR/prerender includes auth-gated data | Landing page must be 100% static -- no API calls, no user data, no dynamic content |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Mandatory onboarding for existing users who already have everything set up | Frustration, churn of paying customers | Guard checks data state (has business + profile + subscription); skip onboarding if all present |
| All paywall states show same generic "Subscribe" screen | User confusion -- "did my payment fail or did I cancel?" | Distinct screens: trial expired (add card), payment failed (update card), canceled (resubscribe) |
| Landing page and app look like different products | Jarring transition, loss of trust | Share design tokens (colors, typography, spacing) between landing and app |
| Onboarding wizard with no progress indicator | Users abandon because they don't know how many steps remain | Show step indicator ("Step 2 of 3") with estimated time ("Takes ~2 minutes") |
| Trial banner disappears after dismissal with no way to check status | User forgets about trial, surprised by lockout | Trial status always visible in Settings > Billing; banner can be dismissed but reappears at 7 days, 3 days, 1 day |
| Hard paywall with no access to existing data | User feels trapped, can't even view their customers/jobs | Show read-only access for a grace period (past_due); hard paywall only for definitively ended subscriptions |
| Onboarding asks for info the system already has | Friction for users who already have a Firebase profile with displayName | Pre-fill onboarding fields from Firebase user profile (displayName, email) |

## "Looks Done But Isn't" Checklist

- [ ] **Existing user migration:** Users with business + subscription pass the onboarding guard without seeing the wizard -- verified against production data snapshot
- [ ] **Webhook dual-path:** Both Checkout-created (existing) and API-created (new trial) subscriptions produce functional local records -- verified with integration tests for both paths
- [ ] **Landing page isolation:** Landing page chunk contains zero bytes from app modules -- verified with `rollup-plugin-visualizer` or `npx vite-bundle-visualizer`
- [ ] **Hard paywall grace period:** `past_due` status shows read-only mode (not hard paywall) for configurable grace period -- verified by simulating failed payment in Stripe test mode
- [ ] **Onboarding resumability:** Refreshing mid-wizard resumes at correct step, not step 1 -- verified by creating business, refreshing, confirming wizard advances
- [ ] **Stripe idempotency:** Calling onboarding completion twice for same user does NOT create duplicate subscription -- verified by calling endpoint twice
- [ ] **Trial-will-end handling:** 3 days before trial end, user sees in-app banner AND receives email -- verified with Stripe test clock
- [ ] **Route guards both directions:** Unauthenticated user at `/app/*` redirects to landing page; authenticated user at `/` redirects to dashboard -- verified with both directions
- [ ] **Auth loading state:** No flash of landing page for authenticated users during Firebase initialization -- verified with throttled network in DevTools
- [ ] **Paywall state differentiation:** Trial expired, payment failed, and canceled show different messaging and CTAs -- verified visually for all three states
- [ ] **`payment_behavior: 'default_incomplete'` set:** Stripe subscription creation for no-card trial doesn't throw "no payment source" error -- verified in Stripe test mode

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Existing users locked out (P1) | LOW | Hotfix: change guard to check data state (has business + subscription) to skip onboarding; no data loss, purely logic fix |
| Webhook flow breaks new subscriptions (P2) | MEDIUM | Backfill script: query Stripe for all subscriptions, create missing local records; fix webhook handler to handle `customer.subscription.created` as creation event |
| Route conflict / flash of wrong content (P3) | LOW | Routing config fix; no data impact; may require URL migration if switching to `/app/*` prefix |
| Hard paywall false positives (P4) | HIGH (user trust) | Add "Verify subscription" button immediately; add grace period for `past_due`; contact affected users; investigate webhook delivery gaps |
| Duplicate businesses/subscriptions (P5) | MEDIUM | Identify duplicates, merge/delete extras; add unique indexes and idempotency keys; refund duplicate Stripe charges |
| No-card trial churn (P6) | LOW (revenue, not technical) | Add trial_will_end webhook handler and email reminder; add prominent CTA; purely additive change |
| Landing page bundle bloat (P7) | LOW | Add code splitting or prerendering; no data impact, build config change |
| Schema incompatibility (P8) | MEDIUM | Migrate existing records to add missing fields as null; update paywall logic to handle nullable fields; integration tests for both paths |
| Old onboarding removed too early (P9) | MEDIUM | Git revert to restore old onboarding; new flow continues on feature branch; requires coordinated deployment |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Existing users locked out (P1) | Phase 1: Backend onboarding state + migration | Run migration against production snapshot; all existing users with business+subscription pass guard |
| Webhook flow change (P2) | Phase 2: Stripe no-card trial integration | Integration tests: create subscription via API (no card), verify local record exists with correct status |
| Root route conflict (P3) | Phase 1: Routing architecture decision | Manual test: unauth -> landing, auth -> dashboard, no flash during auth loading |
| Hard paywall false positives (P4) | Phase 3: Hard paywall implementation | Simulate all subscription states (trialing, active, past_due, canceled, incomplete, null, loading) |
| Duplicate creation (P5) | Phase 2: Onboarding wizard + Stripe | Double-submit test: call onboarding endpoints twice, verify single business + single subscription |
| No-card trial churn (P6) | Phase 2: Stripe webhooks + Phase 3: Trial UI | Verify trial_will_end handler fires; verify trial banner + "add payment" CTA visible |
| Landing page bundle bloat (P7) | Phase 1: Landing page implementation | Bundle analysis confirms landing chunk has no app-module imports |
| Schema incompatibility (P8) | Phase 2: Stripe integration | Create subscription via both Checkout and API paths; verify paywall logic works identically for both |
| Old onboarding removal (P9) | Final phase: Cleanup (after production validation) | Old code removed only after new flow verified in production for 1+ week |

## Sources

- [Stripe: Configure trial offers on subscriptions](https://docs.stripe.com/billing/subscriptions/trials) -- trial_period_days, trial lifecycle events
- [Stripe: Configure free trials (Checkout)](https://docs.stripe.com/payments/checkout/free-trials) -- payment_method_collection: if_required, trial_settings
- [Stripe: Use free trial periods on subscriptions](https://docs.stripe.com/billing/subscriptions/trials/free-trials) -- no-card trial lifecycle, trial_will_end event
- [Stripe: Using webhooks with subscriptions](https://docs.stripe.com/billing/subscriptions/webhooks) -- event sequence, recommended events to handle
- [Stripe: How subscriptions work](https://docs.stripe.com/billing/subscriptions/overview) -- subscription status state machine, payment_behavior parameter
- [Stripe: Create a subscription API](https://docs.stripe.com/api/subscriptions/create) -- payment_behavior: default_incomplete for no-card trials
- [React Router: Pre-Rendering](https://reactrouter.com/how-to/pre-rendering) -- hybrid SPA + prerendered routes approach
- [w3tutorials: Stripe Subscriptions Without Payment Method](https://www.w3tutorials.net/blog/stripe-allow-subscription-without-payment-method/) -- payment_behavior gotcha and error
- [cjav_dev: Free trials without upfront payment](https://www.cjav.dev/articles/offer-free-trials-without-an-upfront-payment-method-using-stripe-checkout) -- Checkout no-card approach
- [UserPilot: Onboarding Wizard pitfalls](https://userpilot.com/blog/onboarding-wizard/) -- why traditional wizards fail, personalization needs
- [DevTechInsights: SPAs and SEO challenges](https://devtechinsights.com/spas-seo-challenges-2025/) -- SPA SEO limitations and hybrid rendering
- Trade Flow PROJECT.md -- v1.6 Stripe Checkout implementation, existing webhook events, soft paywall, SubscriptionGuard
- Trade Flow CLAUDE.md -- codebase conventions, architecture patterns, existing routing and auth structure

---
*Pitfalls research for: Trade Flow v1.7 -- Onboarding overhaul, public landing page, no-card Stripe trial, hard paywall*
*Researched: 2026-03-31*
