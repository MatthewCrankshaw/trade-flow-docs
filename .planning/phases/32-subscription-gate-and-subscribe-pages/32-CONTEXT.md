# Phase 32: Subscription Gate and Subscribe Pages - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Frontend-only phase (`trade-flow-ui`). Deliver the subscription gate (soft — read access maintained, write actions trigger a paywall modal), the `/subscribe` pricing page, `/subscribe/success` (polling for activation), `/subscribe/cancel` (try again), and all supporting components: `useSubscription` hook, `PaywallModal`, persistent CTA layer, and `subscriptionApi` RTK Query endpoints.

**Requirements update:** GATE-01 as written in REQUIREMENTS.md (hard redirect to /subscribe) is superseded by the soft-gate model decided here. The gate allows read access on all business routes regardless of subscription status — only write actions (create/edit/delete) are blocked via paywall modal. GATE-02 through GATE-05 remain valid.

</domain>

<decisions>
## Implementation Decisions

### Gate Architecture — Soft Gate Model
- **D-01:** No hard redirect. All business routes (Dashboard, Customers, Jobs, Quotes, Items, Business) are accessible with read access regardless of subscription status. Write actions trigger a paywall modal instead of blocking navigation. This replaces the hard-redirect model specified in GATE-01.
- **D-02:** `SubscriptionGate` is a provider-style component (not a route guard) that wraps the `DashboardLayout`. It fetches subscription status, stores it in context/hook, and renders a `PaywallModal` at the layout level. Use a nested `<Route element={<SubscriptionGatedLayout />}>` in App.tsx for the gated business routes — the component itself has zero knowledge of which routes exist.
- **D-03:** Support role bypass: check `user.supportRoles.length > 0` from the existing `User` type (returned by `useGetCurrentUserQuery()`). If non-empty, subscription status is irrelevant — full write access granted.
- **D-04:** `trialing` + `active` statuses = full write access. `past_due`, `canceled`, `incomplete`, and no-subscription record (404 from API) = read-only mode with paywall modal on write actions.
- **D-05:** One unified loading state — if either `useGetCurrentUserQuery()` or `useGetSubscriptionQuery()` is loading, show the page content skeleton. CTA components hidden until status is confirmed.

### subscriptionApi RTK Query
- **D-06:** `subscriptionApi` injects endpoints on the existing `apiSlice` (same pattern as `userApi`, `businessApi`). Adds "Subscription" tag type to `apiSlice` tag list. Two endpoints needed for Phase 32: `getSubscription` (GET /v1/subscription) and `verifySession` (GET /v1/subscription/verify-session?sessionId=).
- **D-07:** `getSubscription` treats 404 as "no subscription" (not an error) — returns `null`. Phase 31 established: 404 = no record exists = unsubscribed state. RTK Query hook should handle 404 gracefully rather than throwing.

### useSubscription Hook
- **D-08:** `useSubscription()` returns `{ isActive: boolean, isLoading: boolean, openPaywall: (featureLabel?: string) => void }`. Components call `openPaywall('Create Jobs')` on button click. A single `PaywallModal` renders at `DashboardLayout` level driven by a context or lightweight Redux slice (`paywallSlice`).
- **D-09:** `isActive` is `true` when subscription status is `trialing` or `active`, OR when the user is a support user (`supportRoles.length > 0`).

### PaywallModal — Feature-Aware Conversion
- **D-10:** Feature-aware paywall modal — title copy matches the attempted action ("Create Jobs & Send Quotes", "Send This Quote", "Add a Customer"). Same modal component, different heading per `featureLabel` argument.
- **D-11:** Modal content: feature-aware title, "Start your 30-day free trial to unlock this feature.", plan name ("Trade Flow Pro"), "£6 / month – 30 days free", two CTAs: "Start free trial" (calls POST /v1/subscription/checkout → redirects to Stripe) and "Maybe later" (dismisses).
- **D-12:** Copy philosophy throughout: outcome-driven language ("Unlock Full Access", "Create Jobs & Send Quotes", "Continue Using Trade Flow") — not generic "Subscribe". Apply this to CTA buttons, banners, and modal headings.

### 3-Layer CTA Architecture
- **D-13:** Persistent CTA: Desktop = top-right button in the header ("Unlock Full Access" or similar). Mobile = banner at the top of the page + highlighted nav item. Hidden entirely during subscription fetch; shown only once status is confirmed inactive/no-subscription. Support users and active/trialing users never see it.
- **D-14:** Passive reminder: A dashboard banner shown to users with no active subscription ("You're in read-only mode — subscribe to create jobs, quotes, and more."). This is Phase 32's banner. The trial countdown chip near the avatar (for trialing users) is deferred to Phase 33 (same component family as TrialBanner).
- **D-15:** Active conversion trigger (MOST IMPORTANT): `PaywallModal` opens when user hits a locked write action anywhere in the app. This is the primary conversion mechanism.

### /subscribe Page
- **D-16:** Renders inside `DashboardLayout` (with sidebar). Users can reach it while browsing in read-only mode.
- **D-17:** Minimal pricing card: "Trade Flow Pro", "£6 / month", "✓ 30-day free trial", "✓ No charge until trial ends", "Start free trial" CTA. No feature list needed — the app itself demonstrates value.
- **D-18:** "Start free trial" calls `POST /v1/subscription/checkout` → receives Stripe-hosted checkout URL → `window.location.href` redirect to Stripe. No Stripe.js, no custom card input (hosted checkout handles PCI/SCA).
- **D-19:** Primary entry point from within the app: Settings > Billing tab (Phase 33). Secondary: persistent CTA button/banner (Phase 32 D-13).

### /subscribe/success Page — Polling
- **D-20:** Poll `GET /v1/subscription/verify-session?sessionId=` every 3 seconds, maximum 10 attempts (30 seconds total). `sessionId` comes from the `?session_id=` query param appended by Stripe's `success_url`.
- **D-21:** Timeout state (10 attempts exhausted) = soft success: show the confirmed state with an additional note: "Your subscription is being activated — you'll receive a confirmation email shortly." User proceeds to app. Avoids blocking on rare infrastructure delay.
- **D-22:** Confirmed success state: checkmark icon, "You're all set!", "Your 30-day free trial has started.", "Go to Dashboard" button (manual — no auto-redirect). Clean, explicit transition into the app.

### /subscribe/cancel Page
- **D-23:** Minimal — explain checkout was abandoned, offer a "Try again" button that returns to `/subscribe`. No complex state needed.

### Loading Skeleton (GATE-05)
- **D-24:** No special subscription skeleton needed with the soft-gate model — the page renders normally (read-only) while subscription status loads. The persistent CTA (D-13) is hidden during loading to prevent flash. For pages that already have their own loading skeletons, those render naturally.

### Claude's Discretion
- `PaywallModal` state management implementation (context vs Redux slice `paywallSlice`)
- Feature label strings for each write action across the app (researcher to enumerate from feature components)
- Exact copy for persistent CTA button and dashboard banner
- `/subscribe/success` loading state animation (spinner vs skeleton)
- `subscriptionApi` feature module location (`src/features/subscription/` following existing feature structure)
- App.tsx nested route structure for `SubscriptionGatedLayout`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.6 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.6 requirements; Phase 32 covers GATE-01 (soft-gate supersedes redirect model), GATE-02, GATE-03, GATE-04, GATE-05

### Roadmap
- `.planning/ROADMAP.md` §Phase 32 — Phase goal, success criteria, dependency on Phase 31

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/phases/29-subscription-module-foundation/29-CONTEXT.md` — Subscription entity schema, STRIPE_CLIENT token, env vars (FRONTEND_URL), Stripe customer creation
- `.planning/phases/30-stripe-checkout-and-webhooks/30-CONTEXT.md` — verify-session endpoint (D-12/D-13), `stripeLatestCheckoutSessionId` field
- `.planning/phases/31-subscription-api-endpoints-and-tests/31-CONTEXT.md` — GET /v1/subscription returns 404 for no subscription (D-01), portal return URL `${FRONTEND_URL}/settings?tab=billing` (D-05)

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — React naming patterns, hook conventions, RTK Query endpoint structure
- `.planning/codebase/ARCHITECTURE.md` — Frontend feature module structure, RTK Query patterns, Redux store setup

### Existing Frontend Patterns
- `trade-flow-ui/src/features/auth/components/ProtectedRoute.tsx` — Layout-route guard pattern this phase mirrors
- `trade-flow-ui/src/services/api.ts` — `apiSlice` base; `subscriptionApi` injects here (add "Subscription" to tagTypes)
- `trade-flow-ui/src/services/userApi.ts` — `injectEndpoints` pattern to follow
- `trade-flow-ui/src/App.tsx` — Current route tree; Phase 32 adds nested layout route + 3 new routes

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProtectedRoute` (`src/features/auth/components/ProtectedRoute.tsx`): layout-route pattern for auth guard — `SubscriptionGatedLayout` follows the same structure (fetch → loading → render children or redirect)
- `useAuth()` + `AuthContext` pattern: `useSubscription()` mirrors this — context-based hook with loading state
- `useGetCurrentUserQuery()`: already fetches `User` with `supportRoles: SupportRole[]` — support bypass check reads this without extra API call
- `apiSlice` with `injectEndpoints`: `subscriptionApi` follows `userApi.ts` pattern exactly
- `LoadingScreen` component: existing full-page loading — success/subscribe pages may use this during polling
- `DashboardLayout`: all new subscribe pages render inside this layout

### Established Patterns
- RTK Query: `injectEndpoints` on `apiSlice`, `transformResponse` for `StandardResponse<T>`, tag invalidation
- Feature modules: `src/features/{feature}/` with `api/`, `components/`, `hooks/`, `index.ts` barrel
- Soft 404 handling: `transformResponse` returns `null` for empty/missing data (see `userApi` onboarding progress endpoint)

### Integration Points
- `trade-flow-ui/src/App.tsx` — add nested layout route for SubscriptionGatedLayout + 3 new routes (/subscribe, /subscribe/success, /subscribe/cancel) inside `AuthenticatedLayout` but outside the gate
- `trade-flow-ui/src/services/api.ts` — add "Subscription" to `tagTypes` array
- `trade-flow-ui/src/store/index.ts` — add paywallSlice reducer if Redux approach used for modal state
- `DashboardLayout` component — render `PaywallModal` and persistent CTA layer here
- Each feature's create/edit/delete handlers — call `openPaywall('Feature label')` instead of opening their form dialogs

</code_context>

<specifics>
## Specific Ideas

- Paywall modal CTA pattern: feature-aware titles like "Create Jobs & Send Quotes", "Send This Quote", "Add a Customer" — outcome-driven, not generic "Subscribe"
- Soft-gate copy principle: avoid "Subscribe"; use "Unlock Full Access", "Start Your Subscription", "Continue Using Trade Flow"
- Trial countdown chip (near avatar, clickable → /subscribe) is DEFERRED to Phase 33 — same component family as TrialBanner
- Settings > Billing tab with pricing card is DEFERRED to Phase 33
- Stripe-hosted checkout redirect: `window.location.href = checkoutUrl` — no Stripe.js, no React Stripe packages
- `success_url` carries `?session_id={CHECKOUT_SESSION_ID}` (Stripe substitution) — success page reads this for the polling call

</specifics>

<deferred>
## Deferred Ideas

- **Trial countdown chip near avatar** — for trialing users (status=trialing), clickable → /subscribe. Deferred to Phase 33 (TrialBanner family).
- **Settings > Billing tab** — pricing card embedded in Settings > Billing for direct access from within settings. Deferred to Phase 33.
- **TrialBanner** (days remaining for trialing users) — Phase 33 TRIAL-01/TRIAL-02.

</deferred>

---

*Phase: 32-subscription-gate-and-subscribe-pages*
*Context gathered: 2026-03-29*
