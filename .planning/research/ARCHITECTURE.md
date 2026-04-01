# Architecture Research

**Domain:** Onboarding overhaul, public landing page, no-card Stripe trial, hard paywall
**Researched:** 2026-03-31
**Confidence:** HIGH

## System Overview

### Current Route Architecture (v1.6)

```
BrowserRouter
├── /login                          ← Public (LoginPage)
├── /quote/:token                   ← Public (CustomerQuotePage, no auth)
├── /subscribe                      ← Auth required, no subscription required
├── /subscribe/success              ← Auth required, no subscription required
├── /subscribe/cancel               ← Auth required, no subscription required
├── ProtectedRoute (auth guard)
│   └── DashboardLayout
│       └── SubscriptionGatedLayout (soft paywall: read-only + PaywallModal)
│           ├── /dashboard
│           ├── /customers
│           ├── /jobs, /jobs/:jobId
│           ├── /quotes, /quotes/:quoteId
│           ├── /items
│           ├── /business
│           └── /settings            ← Always accessible (even without subscription)
└── OnboardingProvider (dismissible dialog)
```

### Target Route Architecture (v1.7)

```
BrowserRouter
├── /                               ← NEW: Public landing page (LandingPage)
├── /login                          ← Public (LoginPage)
├── /quote/:token                   ← Public (CustomerQuotePage, no auth)
├── ProtectedRoute (auth guard)
│   ├── OnboardingGuard (mandatory)
│   │   ├── /onboarding/profile     ← NEW: Step 1 — name (ProfileSetupPage)
│   │   └── /onboarding/business    ← NEW: Step 2 — business + trial (BusinessSetupPage)
│   ├── HardPaywallGuard            ← NEW: replaces SubscriptionGatedLayout
│   │   └── DashboardLayout
│   │       ├── /dashboard          ← MODIFIED: Welcome screen with getting-started widget
│   │       ├── /customers
│   │       ├── /jobs, /jobs/:jobId
│   │       ├── /quotes, /quotes/:quoteId
│   │       ├── /items
│   │       ├── /business
│   │       └── /settings           ← Always accessible (even past paywall)
│   └── /subscribe                  ← Auth required, accessible past paywall
└── /subscribe/success              ← Auth required
```

### Component Responsibilities

| Component | Responsibility | New vs Modified |
|-----------|----------------|-----------------|
| LandingPage | Public marketing page at `/`, redirects authenticated users to `/dashboard` | NEW |
| OnboardingGuard | Layout route that checks profile + business completion, redirects to correct onboarding step | NEW |
| ProfileSetupPage | Collects user display name, calls PATCH /v1/user | NEW |
| BusinessSetupPage | Collects business name + trade, creates business + no-card trial subscription in single flow | NEW |
| HardPaywallGuard | Layout route replacing SubscriptionGatedLayout, renders blocking screen (not modal) for invalid subscriptions | NEW (replaces existing) |
| PaywallPage | Full-screen blocking page shown by HardPaywallGuard, with CTA to billing portal or subscribe | NEW (replaces PaywallModal) |
| WelcomeDashboard | Dashboard variant for new users with greeting + getting-started widget | MODIFIED (DashboardPage) |
| GettingStartedWidget | Checklist widget (create job, create quote) following existing onboarding widget pattern | NEW |
| SubscriptionCreator (API) | Extended to support no-card trial via Stripe Subscriptions API (not Checkout) | MODIFIED |
| DashboardLayout | Remove PaywallModal rendering, remove PersistentCta for inactive users | MODIFIED |
| App.tsx | Restructured route tree with new guards and public landing route | MODIFIED |

## Recommended Project Structure

### Frontend Changes

```
src/
├── pages/
│   ├── LandingPage.tsx              # NEW: Public marketing page
│   ├── onboarding/
│   │   ├── ProfileSetupPage.tsx     # NEW: Step 1
│   │   └── BusinessSetupPage.tsx    # NEW: Step 2
│   ├── PaywallPage.tsx              # NEW: Hard paywall full-screen
│   └── DashboardPage.tsx            # MODIFIED: Welcome variant
├── features/
│   ├── onboarding/                  # NEW: Replace components/onboarding/ and contexts/
│   │   ├── components/
│   │   │   ├── OnboardingGuard.tsx  # Layout route with redirect logic
│   │   │   ├── GettingStartedWidget.tsx
│   │   │   └── index.ts
│   │   ├── hooks/
│   │   │   └── useOnboardingStatus.ts  # Derives step from user + business state
│   │   └── index.ts
│   └── subscription/               # MODIFIED
│       ├── components/
│       │   ├── HardPaywallGuard.tsx # NEW: Replaces SubscriptionGatedLayout
│       │   ├── PricingCard.tsx      # KEEP
│       │   ├── TrialChip.tsx        # KEEP
│       │   └── index.ts
│       └── hooks/
│           └── useSubscription.ts   # KEEP
├── components/
│   └── onboarding/                  # DELETE: Move to features/onboarding/
├── contexts/
│   ├── OnboardingContext.tsx         # DELETE: Replace with useOnboardingStatus hook
│   └── OnboardingProvider.tsx       # DELETE: No longer needed
├── store/
│   └── slices/
│       └── onboardingSlice.ts       # DELETE: Replace with route-based onboarding
```

### Backend Changes

```
src/
├── subscription/
│   ├── services/
│   │   └── subscription-creator.service.ts  # MODIFIED: Add createTrialWithoutCard method
│   └── controllers/
│       └── subscription.controller.ts       # MODIFIED: Add POST /v1/subscription/trial endpoint
├── user/
│   ├── requests/
│   │   └── update-user.request.ts           # VERIFY: displayName field exists
│   └── services/
│       └── user-updater.service.ts          # VERIFY: Can update displayName
```

### Structure Rationale

- **features/onboarding/ replaces scattered onboarding code:** The current onboarding lives across `components/onboarding/`, `contexts/OnboardingContext*`, `store/slices/onboardingSlice`, and `store/middleware/onboardingMiddleware`. Consolidating into a feature module follows the established pattern for every other domain (customers, jobs, quotes, subscription).
- **Route-based onboarding replaces dialog-based:** The existing dismissible dialog approach (Redux slice controlling open/close) is being replaced with mandatory page routes. This eliminates the need for the onboarding Redux slice, middleware, context, and provider entirely. The onboarding state is derived from user/business data, not stored separately.
- **HardPaywallGuard as layout route:** Follows the same pattern as the existing SubscriptionGatedLayout (a React Router `<Outlet />` wrapper) but blocks the entire screen instead of showing a modal on write actions.

## Architectural Patterns

### Pattern 1: Three-Tier Route Guarding

**What:** Nested layout routes that enforce auth, onboarding completion, and subscription validity as progressive gates.
**When to use:** When authenticated users must complete setup before accessing the main app, and must maintain a valid subscription to continue using it.
**Trade-offs:** Clear separation of concerns per guard, but adds nesting depth in the route tree. Each guard is independently testable.

**Implementation:**

```typescript
// App.tsx route structure
<Routes>
  {/* Public routes - no auth required */}
  <Route path="/" element={<LandingPage />} />
  <Route path="/login" element={<LoginPage />} />
  <Route path="/quote/:token" element={<CustomerQuotePage />} />

  {/* Auth-required routes */}
  <Route element={<ProtectedRoute />}>
    {/* Onboarding routes - auth required, no business/subscription needed */}
    <Route element={<OnboardingGuard />}>
      <Route path="/onboarding/profile" element={<ProfileSetupPage />} />
      <Route path="/onboarding/business" element={<BusinessSetupPage />} />
    </Route>

    {/* Subscribe routes - auth required, accessible even without subscription */}
    <Route path="/subscribe" element={<SubscribePage />} />
    <Route path="/subscribe/success" element={<SubscribeSuccessPage />} />

    {/* Main app routes - auth + onboarding + subscription required */}
    <Route element={<HardPaywallGuard />}>
      <Route element={<DashboardLayout />}>
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/customers" element={<CustomersPage />} />
        {/* ... other business routes */}
        <Route path="/settings" element={<SettingsPage />} />
      </Route>
    </Route>
  </Route>
</Routes>
```

**Guard logic:**

```
ProtectedRoute:
  - No Firebase user -> redirect /login
  - Has user -> render <Outlet />

OnboardingGuard:
  - Profile incomplete (no displayName) -> redirect /onboarding/profile
  - Business incomplete (no business) -> redirect /onboarding/business
  - Both complete -> redirect /dashboard (user should not be on onboarding routes)
  - NOTE: Only renders onboarding pages, not business routes

HardPaywallGuard:
  - Onboarding incomplete -> redirect to appropriate onboarding step
  - No subscription or invalid status -> render <PaywallPage /> (full-screen block)
  - Support role -> bypass, render <Outlet />
  - Valid subscription (trialing/active) -> render <Outlet />
  - Settings route -> always accessible (exception within the guard)
```

### Pattern 2: Derived Onboarding State (No Stored State)

**What:** Onboarding step is computed from existing API data (user profile + business existence + subscription status), not stored as separate onboarding state.
**When to use:** When onboarding steps map 1:1 to the presence of existing domain entities.
**Trade-offs:** Eliminates an entire state management layer (Redux slice, middleware, context, localStorage). The "source of truth" is the actual data, not a separate progress tracker. Downside: requires RTK Query data to be loaded before determining step.

**Implementation:**

```typescript
// useOnboardingStatus.ts
function useOnboardingStatus() {
  const { user } = useAuth();
  const { data: userProfile, isLoading: userLoading } = useGetUserQuery();
  const { data: businesses, isLoading: bizLoading } = useGetBusinessesQuery();

  const isLoading = userLoading || bizLoading;

  // Derive step from data state
  const profileComplete = Boolean(userProfile?.displayName);
  const businessComplete = Boolean(businesses?.length);
  const onboardingComplete = profileComplete && businessComplete;

  const currentStep = !profileComplete
    ? "profile"
    : !businessComplete
    ? "business"
    : "complete";

  return { isLoading, onboardingComplete, currentStep, profileComplete, businessComplete };
}
```

**What this replaces:**
- `store/slices/onboardingSlice.ts` (Redux state for dialog open/close/step)
- `store/middleware/onboardingMiddleware.ts` (watches auth/business changes)
- `contexts/OnboardingContext.tsx` and `contexts/OnboardingProvider.tsx`
- `hooks/useOnboarding.ts`
- `store/utils/onboarding-storage.ts` (localStorage persistence)
- `components/onboarding/OnboardingDialogs.tsx` and `PrerequisiteAlert.tsx`

### Pattern 3: No-Card Trial via Stripe Subscriptions API (Not Checkout)

**What:** Create a Stripe Subscription directly via the API with `trial_period_days` and no payment method, bypassing the Checkout Session flow entirely.
**When to use:** When you want zero-friction trial activation without redirecting to Stripe's hosted checkout.
**Trade-offs:** No card on file means conversion requires the user to add payment later (via Billing Portal). Simpler UX at signup, but requires handling the "trial ending without payment method" case.

**Implementation (API side):**

```typescript
// subscription-creator.service.ts - NEW method
async createTrialWithoutCard(authUser: IUserDto): Promise<ISubscriptionDto> {
  // 1. Create or retrieve Stripe Customer
  const customer = await this.stripe.customers.create({
    email: authUser.email,
    metadata: { userId: authUser.id },
  });

  // 2. Create subscription with trial, no payment method required
  const subscription = await this.stripe.subscriptions.create({
    customer: customer.id,
    items: [{ price: this.priceId }],
    trial_period_days: 30,
    payment_behavior: "default_incomplete",
    trial_settings: {
      end_behavior: {
        missing_payment_method: "cancel",
      },
    },
    metadata: { userId: authUser.id },
  });

  // 3. Create local subscription record (same as webhook would)
  return this.subscriptionRepository.upsert({
    userId: authUser.id,
    stripeCustomerId: customer.id,
    stripeSubscriptionId: subscription.id,
    status: SubscriptionStatus.TRIALING,
    currentPeriodEnd: toDateTime(subscription.current_period_end),
    trialEnd: toDateTime(subscription.trial_end),
    cancelAtPeriodEnd: false,
  });
}
```

**Key Stripe parameters:**
- `trial_period_days: 30` -- 30-day trial
- `payment_behavior: "default_incomplete"` -- allows creation without payment method
- `trial_settings.end_behavior.missing_payment_method: "cancel"` -- auto-cancels if no card added by trial end (cleanest UX; user gets a clear "trial expired" state rather than a paused limbo)

**What this changes from v1.6:**
- The existing `POST /v1/subscription/checkout` flow (Stripe Checkout Session with card required) remains available but is no longer the primary trial path
- The `/subscribe` page with pricing card still exists for users who need to resubscribe after trial expiry
- The `/subscribe/success` polling page is no longer needed for the onboarding flow (trial is created synchronously)
- The existing checkout flow could be kept for users who want to subscribe with a card after trial expiry, or replaced with Billing Portal redirect

**What stays the same:**
- Webhook processing for all 5 event types (subscription status sync still works)
- `GET /v1/subscription` endpoint
- `DELETE /v1/subscription` endpoint (cancel at period end)
- `POST /v1/subscription/portal` endpoint (user adds card here later)
- The local MongoDB subscription record format
- SubscriptionGuard on API routes

**New endpoint:**

```
POST /v1/subscription/trial
Auth: Required (JwtAuthGuard)
SubscriptionCheck: Skipped (@SkipSubscriptionCheck)
Body: {} (empty -- user/business context comes from auth)
Response: { data: [{ status: "trialing", trialEnd: "...", ... }] }
```

This endpoint is called automatically during the business setup step, not by user action on a subscribe page.

### Pattern 4: Hard Paywall as Layout Route

**What:** Replace the soft paywall (PaywallModal on write actions + read-only browsing) with a full-screen blocking layout route that prevents access to all business routes.
**When to use:** When subscription enforcement must be absolute -- no browsing, no read access, no "try before you buy" for expired users.
**Trade-offs:** More aggressive gating, but simpler implementation (one guard vs. per-page `openPaywall` calls on every write action handler). Also removes the need for `paywallSlice` and `PaywallModal`.

**Implementation:**

```typescript
// HardPaywallGuard.tsx
function HardPaywallGuard() {
  const { isActive, isLoading, subscription } = useSubscription();
  const location = useLocation();

  // Settings always accessible (so users can manage billing)
  if (location.pathname.startsWith("/settings")) {
    return <Outlet />;
  }

  if (isLoading) {
    return <PaywallSkeleton />; // No flash of paywall for active users
  }

  if (!isActive) {
    return <PaywallPage subscription={subscription} />;
  }

  return <Outlet />;
}
```

**What this removes:**
- `PaywallModal.tsx` component
- `PersistentCta.tsx` component (header CTA for inactive users)
- `DashboardBanner.tsx` component (read-only mode message)
- `store/slices/paywallSlice.ts` (Redux modal state)
- All `openPaywall()` dispatch calls in every business page (CustomersPage, JobsPage, QuotesPage, ItemsPage, BusinessPage, DashboardPage, JobDetailPage, QuoteDetailPage)
- `SubscriptionGatedLayout.tsx` (replaced by HardPaywallGuard)

**What this adds:**
- `PaywallPage.tsx` -- full-screen page with subscription status message, "Add payment method" CTA (links to Billing Portal), and "Contact support" option

## Data Flow

### New User Registration to Dashboard (Complete Flow)

```
1. User visits / (LandingPage)
   -> Marketing page with "Get Started" CTA

2. User clicks "Get Started" -> /login
   -> AuthForm (existing) for signup via Firebase
   -> Firebase creates user, AuthProvider receives onAuthStateChanged

3. ProtectedRoute passes (user authenticated)
   -> OnboardingGuard checks: no displayName -> redirect /onboarding/profile

4. ProfileSetupPage
   -> User enters display name
   -> PATCH /v1/user/{id} with { displayName }
   -> RTK Query invalidates "User" tag
   -> useOnboardingStatus re-evaluates: profileComplete=true, businessComplete=false
   -> Redirect /onboarding/business

5. BusinessSetupPage
   -> User enters business name + selects trade
   -> Two sequential API calls:
     a. POST /v1/business { name, primaryTrade, country: "GB", currency: "GBP" }
        -> Creates business + defaults (items, tax rates, job types, visit types)
     b. POST /v1/subscription/trial {}
        -> Creates Stripe Customer + Subscription with 30-day trial (no card)
        -> Creates local subscription record with status: trialing
   -> RTK Query invalidates "Business" and "Subscription" tags
   -> useOnboardingStatus: onboardingComplete=true
   -> Redirect /dashboard

6. DashboardPage (Welcome variant)
   -> "Welcome, {displayName}!" greeting
   -> GettingStartedWidget: "Create your first job" + "Send your first quote"
   -> TrialChip in header: "28 days left in trial"
   -> Full app access (all routes accessible)
```

### Returning User with Expired Trial

```
1. User visits / (LandingPage)
   -> Already authenticated -> redirect /dashboard

2. HardPaywallGuard evaluates:
   -> useSubscription returns isActive=false (status: canceled, trial expired)
   -> Renders PaywallPage (full-screen block)
   -> "Your trial has ended. Add a payment method to continue."
   -> CTA: "Add Payment Method" -> POST /v1/subscription/portal -> Stripe Billing Portal
   -> Alternative: "Go to Settings" -> /settings (always accessible for billing management)
```

### Authenticated User Visiting Landing Page

```
1. User visits / (LandingPage)
   -> useAuth() returns user (Firebase auth state in memory)
   -> LandingPage renders <Navigate to="/dashboard" replace /> immediately
   -> No API call needed for redirect decision
```

### State Management Changes

```
Redux Store (BEFORE - v1.6):
{
  [apiSlice.reducerPath]: { queries, mutations },
  onboarding: { isOpen, currentStep },     <- REMOVE
  paywall: { isOpen, featureLabel },        <- REMOVE
}

Redux Store (AFTER - v1.7):
{
  [apiSlice.reducerPath]: { queries, mutations },
  // Onboarding state derived from RTK Query cache (user + business queries)
  // Paywall state derived from RTK Query cache (subscription query)
  // No new Redux slices needed
}
```

### Key Data Flows

1. **Onboarding status derivation:** `useGetUserQuery()` + `useGetBusinessesQuery()` -> `useOnboardingStatus()` -> OnboardingGuard redirect decision. No stored state, no middleware, no context provider.

2. **Trial creation:** BusinessSetupPage -> `POST /v1/subscription/trial` -> Stripe API -> local MongoDB record -> RTK Query cache update -> HardPaywallGuard sees `isActive=true`.

3. **Landing page redirect:** LandingPage checks `useAuth()` -> if authenticated, `<Navigate to="/dashboard" />`. No API call needed, Firebase auth state is in memory.

## Integration Points

### External Services

| Service | Integration Pattern | Changes in v1.7 |
|---------|---------------------|------------------|
| Stripe | `stripe.subscriptions.create()` for no-card trial | NEW method in SubscriptionCreator, new endpoint |
| Stripe Billing Portal | `stripe.billingPortal.sessions.create()` | No change -- used for "add payment method" CTA on paywall |
| Firebase Auth | `onAuthStateChanged()` in AuthProvider | No change |

### Internal Boundaries

| Boundary | Communication | Changes in v1.7 |
|----------|---------------|------------------|
| OnboardingGuard <-> RTK Query | Reads user + business queries to derive step | NEW guard, existing API endpoints |
| HardPaywallGuard <-> useSubscription | Reads subscription status from context/RTK | NEW guard, existing hook |
| BusinessSetupPage <-> Subscription API | Calls POST /v1/subscription/trial after business creation | NEW endpoint call |
| BusinessSetupPage <-> Business API | Calls POST /v1/business (existing) | No change to API |
| App.tsx <-> All Guards | Route tree nesting order determines guard evaluation | MODIFIED route structure |
| DashboardPage <-> GettingStartedWidget | Renders widget when user is newly onboarded | NEW component, existing page modified |
| LandingPage <-> AuthContext | Checks auth state for redirect | NEW page, existing context |

## Anti-Patterns

### Anti-Pattern 1: Storing Onboarding Progress Separately from Domain Data

**What people do:** Create a separate `onboarding_progress` collection or Redux slice that tracks which steps are complete, synced via middleware when domain data changes.
**Why it's wrong:** Creates a dual source of truth. The "has the user created a business?" question is already answered by querying businesses. Storing it separately means the progress record can get out of sync with reality (e.g., business creation succeeds but progress update fails).
**Do this instead:** Derive onboarding status from the existence of domain entities (user profile has displayName, business collection is non-empty). The existing API already knows these things.

### Anti-Pattern 2: Using Stripe Checkout for No-Card Trial

**What people do:** Keep using `stripe.checkout.sessions.create()` with `payment_method_collection: "if_required"` to create a trial without a card.
**Why it's wrong:** Stripe Checkout redirects the user to a hosted page, which is unnecessary friction when no card is being collected. The user leaves the app, sees a Stripe-branded page that says "$0.00", and clicks a button to "start trial." This is confusing UX for a no-card flow.
**Do this instead:** Use `stripe.subscriptions.create()` directly with `trial_period_days` and `payment_behavior: "default_incomplete"`. The trial starts instantly on the backend with no redirect. The user never leaves the app.

### Anti-Pattern 3: Soft Paywall with Per-Page Wiring

**What people do:** Keep the PaywallModal approach and add `openPaywall()` calls to every write-action handler on every business page.
**Why it's wrong:** It requires modifying every page when the paywall strategy changes, creates inconsistency risk (miss one handler and users bypass the paywall), and the "read-only browsing" it enables has limited value for a fresh product without data.
**Do this instead:** A single HardPaywallGuard layout route that blocks all access. One component, one place to maintain, zero per-page wiring.

### Anti-Pattern 4: Onboarding as Modal Dialog

**What people do:** Show onboarding as a dismissible dialog that can be reopened from the UI.
**Why it's wrong:** Users dismiss it and never complete setup, then encounter errors when they try to use features that require a business. The dialog approach makes setup feel optional when it is actually mandatory.
**Do this instead:** Route-based onboarding with a guard that redirects. Users cannot access the main app until setup is complete. The back button takes them to the previous step, not around the guard.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-1k users | Current approach is fine. Direct Stripe API call in subscription creator is fast enough. |
| 1k-10k users | Consider rate limiting on POST /v1/subscription/trial to prevent abuse. Monitor Stripe API call latency. |
| 10k+ users | Move trial creation to BullMQ queue if Stripe API latency causes timeout on business setup page. |

### Scaling Priorities

1. **First bottleneck:** Stripe API latency during onboarding. If subscription creation takes >3s, the business setup page feels slow. Mitigation: create business first (optimistic), then trial in background. User lands on dashboard immediately, trial creation completes async.
2. **Second bottleneck:** RTK Query cache staleness after onboarding. If `useOnboardingStatus` reads stale cache, user gets stuck on onboarding routes. Mitigation: explicitly invalidate "User", "Business", and "Subscription" tags after each onboarding mutation.

## Suggested Build Order

The build order is dictated by dependencies between the five changes:

1. **API: No-card trial endpoint** (no frontend dependency)
   - Add `POST /v1/subscription/trial` to SubscriptionController
   - Implement `createTrialWithoutCard` in SubscriptionCreator
   - Add `@SkipSubscriptionCheck()` decorator (user has no subscription yet)
   - Unit tests for new service method

2. **UI: Landing page + route restructure** (independent of API changes)
   - Create LandingPage at `/`
   - Restructure App.tsx route tree (public / auth / onboarding / paywall tiers)
   - Add redirect logic for authenticated users on landing page
   - Create OnboardingGuard shell (just checks and redirects)
   - Create HardPaywallGuard shell (replaces SubscriptionGatedLayout)

3. **UI: Onboarding wizard pages** (depends on route structure from step 2)
   - Create ProfileSetupPage with name form
   - Create BusinessSetupPage with business name + trade form
   - Wire BusinessSetupPage to call existing POST /v1/business then new POST /v1/subscription/trial
   - Implement `useOnboardingStatus` hook
   - Remove old onboarding infrastructure (Redux slice, middleware, context, provider, dialog components)

4. **UI: Hard paywall + cleanup** (depends on route structure from step 2)
   - Create PaywallPage component (full-screen block with CTA)
   - Wire HardPaywallGuard to render PaywallPage for invalid subscriptions
   - Remove PaywallModal, PersistentCta, DashboardBanner, paywallSlice
   - Remove all `openPaywall()` dispatch calls from business pages (8 files)
   - Verify Settings page remains accessible

5. **UI: Welcome dashboard + getting-started widget** (depends on onboarding from step 3)
   - Modify DashboardPage to show welcome variant for newly onboarded users
   - Create GettingStartedWidget with "create job" and "create quote" checklist items
   - Wire widget to existing job/quote counts from RTK Query cache

Steps 1 and 2 can run in parallel (API and UI repos are independent). Steps 3 and 4 can also run in parallel (they depend on step 2 but not on each other). Step 5 depends on step 3.

## Files Removed (Cleanup Inventory)

These files/modules are replaced by the new architecture and should be deleted:

| File | Reason for Removal |
|------|-------------------|
| `src/components/onboarding/OnboardingDialogs.tsx` | Replaced by route-based onboarding pages |
| `src/components/onboarding/PrerequisiteAlert.tsx` | Replaced by OnboardingGuard redirect |
| `src/contexts/OnboardingContext.tsx` | Replaced by useOnboardingStatus hook |
| `src/contexts/OnboardingContext.ts` | Replaced by useOnboardingStatus hook |
| `src/contexts/OnboardingContext.types.ts` | Replaced by useOnboardingStatus hook |
| `src/contexts/OnboardingProvider.tsx` | Replaced by OnboardingGuard layout route |
| `src/hooks/useOnboarding.ts` | Replaced by useOnboardingStatus hook |
| `src/store/slices/onboardingSlice.ts` | Onboarding state now derived, not stored |
| `src/store/middleware/onboardingMiddleware.ts` | No longer needed without onboarding slice |
| `src/store/utils/onboarding-storage.ts` | No localStorage persistence needed |
| `src/store/slices/paywallSlice.ts` | Replaced by HardPaywallGuard (no modal state) |
| `src/features/subscription/components/PaywallModal.tsx` | Replaced by PaywallPage |
| `src/features/subscription/components/PersistentCta.tsx` | Removed (hard paywall blocks entirely) |
| `src/features/subscription/components/DashboardBanner.tsx` | Removed (hard paywall blocks entirely) |
| `src/features/subscription/components/SubscriptionGatedLayout.tsx` | Replaced by HardPaywallGuard |

## Sources

- [Stripe: Configure trial offers on subscriptions](https://docs.stripe.com/billing/subscriptions/trials) -- trial_settings.end_behavior.missing_payment_method options
- [Stripe: Create a subscription API](https://docs.stripe.com/api/subscriptions/create) -- payment_behavior and trial_period_days parameters
- [Stripe: Free trials without payment details upfront](https://docs.stripe.com/payments/checkout/free-trials) -- payment_method_collection for Checkout approach (not recommended for this use case)
- [Stripe: How subscriptions work](https://docs.stripe.com/billing/subscriptions/overview) -- subscription lifecycle and status transitions
- Existing v1.6 plans: `.planning/milestones/v1.6-phases/29-02-PLAN.md` -- current checkout endpoint pattern
- Existing v1.6 plans: `.planning/milestones/v1.6-phases/32-01-PLAN.md` -- current subscription feature foundation
- Existing v1.6 plans: `.planning/milestones/v1.6-phases/32-02-PLAN.md` -- current SubscriptionGatedLayout and PaywallModal
- Existing v1.6 plans: `.planning/milestones/v1.6-phases/32-03-PLAN.md` -- current per-page paywall wiring
- Existing codebase: `.planning/codebase/ARCHITECTURE.md` -- route structure, RTK Query patterns, feature module organization
- Existing codebase: `.planning/codebase/STRUCTURE.md` -- file structure, onboarding components/contexts/store

---
*Architecture research for: v1.7 Onboarding & Landing Page*
*Researched: 2026-03-31*
