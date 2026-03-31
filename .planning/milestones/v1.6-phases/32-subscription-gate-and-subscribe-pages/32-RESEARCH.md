# Phase 32: Subscription Gate and Subscribe Pages - Research

**Researched:** 2026-03-29
**Domain:** React frontend -- subscription gating, paywall modal, RTK Query subscription endpoints, route guards
**Confidence:** HIGH

## Summary

Phase 32 is a frontend-only phase in `trade-flow-ui` that adds a soft subscription gate (read access always permitted, write actions blocked via paywall modal), three new pages (`/subscribe`, `/subscribe/success`, `/subscribe/cancel`), RTK Query subscription endpoints, a `useSubscription` hook, and a 3-layer CTA architecture (persistent CTA, dashboard banner, paywall modal).

The backend API is fully built (Phases 29-31). The frontend needs to consume `GET /v1/subscription` (returns `ISubscriptionResponse` or 404), `POST /v1/subscription/checkout` (returns `{ url: string }`), and `GET /v1/subscription/verify-session?sessionId=` (returns `{ status: string, verified: boolean }`). No Stripe.js or React Stripe packages are needed -- hosted checkout via `window.location.href` redirect.

The architecture follows existing patterns exactly: `subscriptionApi` uses `apiSlice.injectEndpoints()` (like `userApi`), `SubscriptionGatedLayout` wraps business routes as a layout element (like `AuthenticatedLayout`), and `useSubscription` follows the context-based hook pattern (like `useAuth`). Paywall modal state is managed via a Redux slice (`paywallSlice`) following the `onboardingSlice` pattern.

**Primary recommendation:** Build incrementally -- subscriptionApi + types first, then useSubscription hook + SubscriptionContext, then SubscriptionGatedLayout + PaywallModal, then the three pages, and finally wire paywall triggers into existing page write actions.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** No hard redirect. All business routes accessible with read access regardless of subscription status. Write actions trigger paywall modal instead of blocking navigation. Supersedes GATE-01 hard-redirect model.
- **D-02:** `SubscriptionGate` is a provider-style component wrapping `DashboardLayout`. Uses nested `<Route element={<SubscriptionGatedLayout />}>` in App.tsx.
- **D-03:** Support role bypass: check `user.supportRoles.length > 0` from `useGetCurrentUserQuery()`.
- **D-04:** `trialing` + `active` = full write access. `past_due`, `canceled`, `incomplete`, no-subscription (404) = read-only with paywall.
- **D-05:** Unified loading state -- if either user or subscription query loading, show content skeleton. CTA hidden until confirmed.
- **D-06:** `subscriptionApi` injects on existing `apiSlice`. Adds "Subscription" tag. Two endpoints: `getSubscription`, `verifySession`.
- **D-07:** `getSubscription` treats 404 as "no subscription" (returns `null`), not an error.
- **D-08:** `useSubscription()` returns `{ isActive, isLoading, openPaywall }`. Single `PaywallModal` at DashboardLayout level.
- **D-09:** `isActive` true when `trialing`/`active` OR support user.
- **D-10:** Feature-aware paywall modal with `featureLabel` argument for custom titles.
- **D-11:** Modal content: feature title, trial description, plan info, two CTAs ("Start free trial" + "Maybe later").
- **D-12:** Outcome-driven copy -- "Unlock Full Access", not "Subscribe".
- **D-13:** Persistent CTA: Desktop = header button, Mobile = banner + nav item. Hidden during fetch, hidden for active/support users.
- **D-14:** Dashboard banner for inactive users ("read-only mode"). Trial countdown chip deferred to Phase 33.
- **D-15:** PaywallModal is primary conversion mechanism -- opens on locked write actions.
- **D-16:** /subscribe renders inside DashboardLayout (with sidebar).
- **D-17:** Minimal pricing card: "Trade Flow Pro", GBP 6/month, 30-day trial, CTA.
- **D-18:** "Start free trial" calls POST /v1/subscription/checkout -> window.location.href redirect to Stripe.
- **D-19:** Primary entry from Settings > Billing (Phase 33). Secondary from persistent CTA (Phase 32).
- **D-20:** Success page polls verify-session every 3s, max 10 attempts.
- **D-21:** Timeout = soft success with "being activated" message.
- **D-22:** Confirmed success: checkmark, "You're all set!", "Go to Dashboard" button.
- **D-23:** Cancel page: explain checkout abandoned, "Try again" button -> /subscribe.
- **D-24:** No special subscription skeleton needed with soft-gate. Persistent CTA hidden during loading.

### Claude's Discretion
- PaywallModal state management (context vs Redux slice `paywallSlice`)
- Feature label strings for each write action across the app
- Exact copy for persistent CTA button and dashboard banner
- /subscribe/success loading state animation (spinner vs skeleton)
- subscriptionApi feature module location
- App.tsx nested route structure for SubscriptionGatedLayout

### Deferred Ideas (OUT OF SCOPE)
- Trial countdown chip near avatar (Phase 33)
- Settings > Billing tab (Phase 33)
- TrialBanner for days remaining (Phase 33)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| GATE-01 | Business routes wrapped by SubscriptionGate (soft-gate model per D-01) | SubscriptionGatedLayout wraps business routes in App.tsx; PaywallModal blocks write actions; read access unrestricted |
| GATE-02 | Settings page always accessible regardless of subscription status | Settings route placed outside SubscriptionGatedLayout's gated section in App.tsx |
| GATE-03 | /subscribe, /subscribe/success, /subscribe/cancel accessible without active subscription | Subscribe routes placed inside AuthenticatedLayout but outside SubscriptionGatedLayout |
| GATE-04 | Support role users bypass SubscriptionGate entirely | useSubscription checks `user.supportRoles.length > 0`; returns `isActive: true` for support users |
| GATE-05 | Loading skeleton during subscription status fetch (no flash) | Unified loading: CTA hidden during fetch, page renders normally in read-only, no subscribe flash |
| ACQ-04 | /subscribe/cancel page with "Try again" option | Minimal page with explanation and Link to /subscribe |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @reduxjs/toolkit | 2.11.2 | RTK Query for subscriptionApi + paywallSlice | Already used for all API endpoints and client state |
| react-router-dom | 7.13.0 | Routing for /subscribe pages + layout routes | Already the project router |
| lucide-react | 0.563.0 | Icons (CheckCircle, Lock, Sparkles) | Already the icon library |

### Supporting (already installed)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| sonner | 2.0.7 | Toast for checkout errors | Error feedback on checkout API failure |
| clsx + tailwind-merge | via cn() | Conditional classes | All component styling |

### No New Dependencies
No new packages required. Stripe hosted checkout uses `window.location.href` -- no Stripe.js, no `@stripe/stripe-js`, no `@stripe/react-stripe-js`.

## Architecture Patterns

### Recommended Project Structure
```
src/
  features/
    subscription/
      api/
        subscriptionApi.ts          # RTK Query endpoints (injectEndpoints)
      components/
        PaywallModal.tsx             # Feature-aware paywall dialog
        PersistentCta.tsx            # Desktop header button + mobile banner
        DashboardBanner.tsx          # Read-only mode reminder banner
        PricingCard.tsx              # Shared pricing card (used in /subscribe + PaywallModal)
        SubscriptionGatedLayout.tsx  # Layout route component (wraps business routes)
      contexts/
        SubscriptionContext.tsx      # SubscriptionProvider + SubscriptionContext
      hooks/
        useSubscription.ts           # Main hook consuming context
        useCheckout.ts               # Encapsulates POST checkout + redirect
        useVerifySession.ts          # Polling logic for success page
      types/
        subscription.ts              # Subscription, VerifySessionResponse types
      index.ts                       # Barrel export
  store/
    slices/
      paywallSlice.ts                # PaywallModal open/close + featureLabel state
  pages/
    SubscribePage.tsx                 # /subscribe
    SubscribeSuccessPage.tsx          # /subscribe/success
    SubscribeCancelPage.tsx           # /subscribe/cancel
```

### Pattern 1: subscriptionApi (RTK Query inject)
**What:** RTK Query endpoints injected on the existing `apiSlice`, following `userApi.ts` pattern exactly.
**When to use:** All subscription API calls.
**Example:**
```typescript
// src/features/subscription/api/subscriptionApi.ts
import { apiSlice } from "@/services/api";
import type { Subscription, VerifySessionResponse, StandardResponse } from "@/types";

export const subscriptionApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getSubscription: builder.query<Subscription | null, void>({
      query: () => "/v1/subscription",
      transformResponse: (response: StandardResponse<Subscription>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        return null;
      },
      // Treat 404 as "no subscription" rather than error
      transformErrorResponse: (response) => {
        if (response.status === 404) {
          return null;
        }
        return response;
      },
      providesTags: ["Subscription"],
    }),
    verifySession: builder.query<VerifySessionResponse, string>({
      query: (sessionId) => `/v1/subscription/verify-session?sessionId=${sessionId}`,
      transformResponse: (response: StandardResponse<VerifySessionResponse>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No verification data returned");
      },
    }),
    createCheckoutSession: builder.mutation<{ url: string }, void>({
      query: () => ({
        url: "/v1/subscription/checkout",
        method: "POST",
      }),
      transformResponse: (response: StandardResponse<{ url: string }>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No checkout URL returned");
      },
    }),
  }),
});
```

**Important:** The 404-as-null pattern requires special handling. RTK Query treats non-2xx responses as errors by default. Use `queryFn` instead of `query` for `getSubscription` to intercept the 404:

```typescript
getSubscription: builder.query<Subscription | null, void>({
  queryFn: async (_arg, _queryApi, _extraOptions, baseQuery) => {
    const result = await baseQuery("/v1/subscription");
    if (result.error) {
      if (result.error.status === 404) {
        return { data: null };
      }
      return { error: result.error };
    }
    const response = result.data as StandardResponse<Subscription>;
    return { data: response.data?.[0] ?? null };
  },
  providesTags: ["Subscription"],
}),
```

### Pattern 2: SubscriptionContext + useSubscription
**What:** React Context providing subscription state app-wide, with a convenience hook.
**When to use:** Any component needing subscription status or paywall trigger.
**Example:**
```typescript
// src/features/subscription/contexts/SubscriptionContext.tsx
interface SubscriptionContextValue {
  isActive: boolean;
  isLoading: boolean;
  subscription: Subscription | null;
  openPaywall: (featureLabel?: string) => void;
}

// Provider wraps DashboardLayout content, fetches subscription + user, derives isActive
```

### Pattern 3: SubscriptionGatedLayout (layout route)
**What:** A layout route component that provides subscription context to child routes and renders PaywallModal.
**When to use:** Wraps all business routes that need the soft gate.
**Example in App.tsx:**
```typescript
function App() {
  return (
    <ErrorBoundary>
      <Provider store={store}>
        <AuthProvider>
          <BrowserRouter>
            <Routes>
              <Route path="/login" element={<LoginPage />} />
              <Route path="/quote/view/:token" element={<PublicQuotePage />} />

              <Route element={<AuthenticatedLayout />}>
                {/* Subscribe pages -- inside auth, outside gate */}
                <Route path="/subscribe" element={<SubscribePage />} />
                <Route path="/subscribe/success" element={<SubscribeSuccessPage />} />
                <Route path="/subscribe/cancel" element={<SubscribeCancelPage />} />

                {/* Settings -- inside auth, outside gate (GATE-02) */}
                <Route path="/settings" element={<SettingsPage />} />

                {/* Gated business routes */}
                <Route element={<SubscriptionGatedLayout />}>
                  <Route path="/dashboard" element={<DashboardPage />} />
                  <Route path="/customers" element={<CustomersPage />} />
                  <Route path="/jobs" element={<JobsPage />} />
                  <Route path="/jobs/:jobId" element={<JobDetailPage />} />
                  <Route path="/quotes" element={<QuotesPage />} />
                  <Route path="/quotes/:quoteId" element={<QuoteDetailPage />} />
                  <Route path="/items" element={<ItemsPage />} />
                  <Route path="/business" element={<BusinessPage />} />

                  {/* Support Routes */}
                  <Route path="/support" element={<SupportDashboardPage />} />
                  <Route path="/support/businesses" element={<SupportBusinessesPage />} />
                  <Route path="/support/users" element={<SupportUsersPage />} />

                  <Route index element={<Navigate to="/dashboard" replace />} />
                </Route>
              </Route>

              <Route path="*" element={<Navigate to="/dashboard" replace />} />
            </Routes>
            <Toaster />
          </BrowserRouter>
        </AuthProvider>
      </Provider>
    </ErrorBoundary>
  );
}
```

**Key insight:** Settings and subscribe routes are INSIDE `AuthenticatedLayout` (requires login) but OUTSIDE `SubscriptionGatedLayout` (no subscription required). This satisfies GATE-02 and GATE-03.

### Pattern 4: paywallSlice (Redux state for modal)
**What:** A Redux slice managing PaywallModal open/close state and the current feature label.
**Why Redux over Context:** The paywall can be triggered from deeply nested components across different features. A Redux slice + `useAppDispatch` is cleaner than prop-drilling or context. Follows `onboardingSlice` pattern exactly.
**Example:**
```typescript
// src/store/slices/paywallSlice.ts
interface PaywallState {
  isOpen: boolean;
  featureLabel: string | null;
}

const paywallSlice = createSlice({
  name: "paywall",
  initialState: { isOpen: false, featureLabel: null } as PaywallState,
  reducers: {
    openPaywall: (state, action: PayloadAction<string | undefined>) => {
      state.isOpen = true;
      state.featureLabel = action.payload ?? null;
    },
    closePaywall: (state) => {
      state.isOpen = false;
      state.featureLabel = null;
    },
  },
});
```

### Pattern 5: Paywall triggers on write actions
**What:** Each page's create/edit/delete handlers check `isActive` before proceeding. If inactive, call `openPaywall('Feature Label')` instead.
**Example:**
```typescript
// In CustomersPage.tsx
const { isActive, openPaywall } = useSubscription();

const handleCreateCustomer = () => {
  if (!isActive) {
    openPaywall("Add a Customer");
    return;
  }
  setSelectedCustomer(null);
  setFormDialogMode("create");
  setFormDialogOpen(true);
};
```

### Pattern 6: Polling with useVerifySession
**What:** Custom hook for the success page that polls verify-session endpoint.
**Example:**
```typescript
// src/features/subscription/hooks/useVerifySession.ts
export function useVerifySession(sessionId: string | null) {
  const [attemptCount, setAttemptCount] = useState(0);
  const maxAttempts = 10;
  const intervalMs = 3000;

  const { data, isLoading } = useVerifySessionQuery(sessionId!, {
    skip: !sessionId || attemptCount >= maxAttempts,
    pollingInterval: intervalMs,
  });

  // RTK Query pollingInterval handles the repeat; track attempts via useEffect
  // When data.verified === true, stop polling
  // When attemptCount >= 10, show soft success

  return {
    isVerified: data?.verified ?? false,
    isPolling: attemptCount < maxAttempts && !data?.verified,
    isTimedOut: attemptCount >= maxAttempts && !data?.verified,
  };
}
```

**Important RTK Query pollingInterval caveat:** `pollingInterval` re-fetches automatically. To count attempts, use a `useEffect` that increments on each data change. Alternatively, use `setTimeout` + `refetch()` manually for more control over the attempt counter.

### Anti-Patterns to Avoid
- **Do NOT use a route-level redirect for the soft gate** -- the decision is explicitly "no hard redirect" (D-01). Components render normally, write actions are gated.
- **Do NOT import Stripe.js or any Stripe React packages** -- hosted checkout uses plain HTTP redirect.
- **Do NOT duplicate subscription fetch logic** -- all subscription state flows through SubscriptionContext/useSubscription.
- **Do NOT show paywall or CTA during loading** -- D-05 and D-24 require hiding CTA until subscription status is confirmed.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Payment form | Custom card input | Stripe hosted checkout (window.location.href) | PCI compliance, SCA/3DS handled automatically |
| Polling mechanism | setInterval + fetch | RTK Query `pollingInterval` or manual refetch | Automatic cache management, cleanup on unmount |
| Modal dialog | Custom overlay | shadcn/ui `Dialog` component (already installed) | Accessible, keyboard-navigable, portal-rendered |
| Toast notifications | Custom notification | Sonner (already installed) | Consistent with rest of app |

## Common Pitfalls

### Pitfall 1: RTK Query 404 handling for getSubscription
**What goes wrong:** RTK Query treats 404 as an error state (`isError: true`), not as "data is null".
**Why it happens:** Default RTK Query behavior maps non-2xx responses to error state.
**How to avoid:** Use `queryFn` (not `query`) to intercept 404 and return `{ data: null }` instead of `{ error }`. This makes `isSuccess: true` and `data: null` for the no-subscription case.
**Warning signs:** `isError` flashing true on pages for users without subscriptions; error toasts appearing.

### Pitfall 2: Stripe success_url session_id parameter
**What goes wrong:** Session ID not available on the success page.
**Why it happens:** Stripe appends `?session_id={CHECKOUT_SESSION_ID}` using its template syntax in `success_url`. Need to configure `success_url` correctly in the backend (already done in Phase 30) and parse it from `useSearchParams` on the frontend.
**How to avoid:** On the success page, use `useSearchParams()` to extract `session_id` from the URL query string.
**Warning signs:** `sessionId` is undefined when polling begins.

### Pitfall 3: Paywall modal state race with navigation
**What goes wrong:** PaywallModal stays open when user navigates to a different page.
**Why it happens:** Redux state persists across route changes.
**How to avoid:** Close the paywall on route change using a `useEffect` with `useLocation()` dependency, or close it in the `SubscriptionGatedLayout` on `Outlet` change.
**Warning signs:** Modal visible on wrong page after back/forward navigation.

### Pitfall 4: Flash of CTA for active subscribers
**What goes wrong:** Persistent CTA button/banner briefly visible before subscription fetch completes.
**Why it happens:** Component renders before RTK Query has loaded subscription data.
**How to avoid:** Check `isLoading` from `useSubscription()`. Only render CTA when `isLoading === false && !isActive`. Never show CTA while loading (D-24).
**Warning signs:** Brief flash of "Unlock Full Access" button for paying users on page load.

### Pitfall 5: Subscribe pages needing SubscriptionContext
**What goes wrong:** `/subscribe`, `/subscribe/success`, `/subscribe/cancel` pages can't access `useSubscription()` because they're outside `SubscriptionGatedLayout`.
**Why it happens:** These routes are intentionally placed outside the gated layout (GATE-03).
**How to avoid:** Either (a) wrap the subscription provider at a higher level than the gate (e.g., in `AuthenticatedLayout`), or (b) subscribe pages use RTK Query hooks directly without the context. Option (a) is cleaner -- SubscriptionProvider wraps AuthenticatedLayout, SubscriptionGatedLayout only handles the gate logic.
**Warning signs:** Runtime error "useSubscription must be used within SubscriptionProvider" on /subscribe pages.

### Pitfall 6: Polling continues after component unmount
**What goes wrong:** Verify-session polling continues after user navigates away from success page.
**Why it happens:** RTK Query `pollingInterval` doesn't automatically stop on unmount if the query is still subscribed.
**How to avoid:** RTK Query handles cleanup when the component unmounts (subscription count drops to 0). The `skip` parameter ensures polling stops when verified. Verify this behavior in testing.
**Warning signs:** Network tab shows continued polling requests after leaving the success page.

## Code Examples

### Frontend Types (Subscription)
```typescript
// src/features/subscription/types/subscription.ts
export interface Subscription {
  id: string;
  userId: string;
  stripeCustomerId: string;
  stripeSubscriptionId?: string;
  status: SubscriptionStatus;
  currentPeriodEnd?: string;
  trialEnd?: string;
  cancelAtPeriodEnd: boolean;
  canceledAt?: string;
  createdAt: string;
  updatedAt: string;
}

export type SubscriptionStatus =
  | "trialing"
  | "active"
  | "past_due"
  | "canceled"
  | "incomplete";

export interface VerifySessionResponse {
  status: string;
  verified: boolean;
}
```

### API Response Shapes (from backend)
```
GET /v1/subscription
  Success: { data: [{ id, userId, stripeCustomerId, status, ... }] }
  Not found: HTTP 404 { errors: [{ code: "CORE_0", message: "..." }] }

POST /v1/subscription/checkout
  Success: { data: [{ url: "https://checkout.stripe.com/..." }] }
  Already active: HTTP 422 { errors: [{ code: "...", message: "Subscription already active" }] }

GET /v1/subscription/verify-session?sessionId=cs_xxx
  Success: { data: [{ status: "trialing", verified: true }] }
  Not yet: { data: [{ status: "pending", verified: false }] }
```

### Adding "Subscription" to apiSlice tagTypes
```typescript
// In src/services/api.ts -- add "Subscription" to tagTypes array
tagTypes: [
  "User", "Business", "Customer", "Item", "Quote",
  "Migration", "TaxRate", "JobType", "Job", "VisitType", "Schedule",
  "Subscription",  // <-- add this
],
```

### Feature Label Enumeration for Paywall Triggers
Write actions across existing pages that need paywall gating:

| Page | Action | Feature Label |
|------|--------|---------------|
| CustomersPage | Add Customer | "Add a Customer" |
| CustomersPage | Edit Customer | "Edit Customer Details" |
| CustomersPage | Toggle Status | "Update Customer Status" |
| JobsPage | Create Job | "Create Jobs" |
| JobsPage | Edit Job | "Edit Job Details" |
| JobDetailPage | Edit Job Details | "Edit Job Details" |
| QuotesPage | Create Quote | "Create Quotes" |
| QuoteDetailPage | Send Quote | "Send This Quote" |
| QuoteDetailPage | Transition Quote | "Update Quote Status" |
| ItemsPage | Add Item | "Add an Item" |
| ItemsPage | Edit Item | "Edit Item Details" |
| BusinessPage | Edit Business | "Edit Business Details" |
| SettingsPage | (exempt -- always accessible) | N/A |

### useCheckout Hook
```typescript
// src/features/subscription/hooks/useCheckout.ts
import { useCreateCheckoutSessionMutation } from "../api/subscriptionApi";
import { toast } from "sonner";

export function useCheckout() {
  const [createCheckout, { isLoading }] = useCreateCheckoutSessionMutation();

  const startCheckout = async () => {
    try {
      const result = await createCheckout().unwrap();
      // Redirect to Stripe hosted checkout
      window.location.href = result.url;
    } catch (error) {
      toast.error("Unable to start checkout. Please try again.");
    }
  };

  return { startCheckout, isLoading };
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Stripe Elements (custom form) | Stripe hosted Checkout | Current best practice | Zero PCI scope, automatic SCA/3DS |
| Hard redirect to paywall page | Soft gate with modal on write | Decision D-01 | Better UX -- users can browse freely |
| Separate subscription API slice | injectEndpoints on shared apiSlice | RTK Query pattern | Single cache, unified auth headers |

## Open Questions

1. **SubscriptionProvider placement**
   - What we know: Subscribe pages are outside SubscriptionGatedLayout but may need subscription state (e.g., to show "already subscribed" message).
   - What's unclear: Whether to put SubscriptionProvider at AuthenticatedLayout level or only inside SubscriptionGatedLayout.
   - Recommendation: Place SubscriptionProvider inside AuthenticatedLayout (wrapping Outlet). This gives all authenticated routes access to subscription state while keeping the gate logic in SubscriptionGatedLayout. Subscribe pages can then check `isActive` and redirect to dashboard if already subscribed.

2. **RTK Query pollingInterval vs manual polling for verify-session**
   - What we know: RTK Query supports `pollingInterval` natively. Manual polling gives more control over attempt counting.
   - What's unclear: Whether pollingInterval auto-stops on `skip: true` immediately or after one more tick.
   - Recommendation: Use manual `setTimeout` + `refetch()` pattern for precise attempt counting. Cleaner than fighting pollingInterval semantics.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest (not detected) / ESLint + TypeScript |
| Config file | No test runner configured for trade-flow-ui |
| Quick run command | `npm run lint && npm run typecheck` |
| Full suite command | `npm run lint && npm run typecheck` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| GATE-01 | Business routes gated by SubscriptionGatedLayout | manual | Navigate to /customers without subscription -> paywall on write action | N/A |
| GATE-02 | Settings accessible without subscription | manual | Navigate to /settings without subscription -> page loads | N/A |
| GATE-03 | Subscribe pages accessible without subscription | manual | Navigate to /subscribe without subscription -> page loads | N/A |
| GATE-04 | Support users bypass gate | manual | Login as support user -> full write access | N/A |
| GATE-05 | No flash of subscribe page for active users | manual | Login as subscribed user -> no CTA flash | N/A |
| ACQ-04 | /subscribe/cancel with try again | manual | Abandon Stripe checkout -> /subscribe/cancel loads with button | N/A |

### Sampling Rate
- **Per task commit:** `npm run lint && npm run typecheck`
- **Per wave merge:** `npm run lint && npm run typecheck && npm run build`
- **Phase gate:** Full lint + typecheck + build clean before verification

### Wave 0 Gaps
- No unit test framework configured for trade-flow-ui (only ESLint + TypeScript static analysis)
- All validation for this phase is via lint, typecheck, and manual verification
- Playwright E2E tests exist (Phase 24-25) but subscription E2E is not in scope for Phase 32

*(Frontend has no unit test runner. Validation relies on TypeScript compiler, ESLint, and manual testing.)*

## Sources

### Primary (HIGH confidence)
- **Existing codebase** -- `trade-flow-ui/src/services/api.ts`, `userApi.ts`, `App.tsx`, `ProtectedRoute.tsx`, `DashboardLayout.tsx`, `store/index.ts`, `store/slices/onboardingSlice.ts` -- all patterns verified by reading source
- **Backend API** -- `trade-flow-api/src/subscription/controllers/subscription.controller.ts`, `responses/subscription.response.ts`, `enums/subscription-status.enum.ts` -- exact API contract verified

### Secondary (MEDIUM confidence)
- RTK Query `queryFn` pattern for 404-as-data -- documented in RTK Query official docs, verified against training data
- RTK Query `pollingInterval` behavior -- documented feature, used in production apps

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new packages, all patterns from existing codebase
- Architecture: HIGH -- follows existing layout route, RTK Query inject, Redux slice patterns exactly
- Pitfalls: HIGH -- identified from direct code reading and RTK Query behavior knowledge
- Route structure: HIGH -- verified against current App.tsx and ProtectedRoute pattern

**Research date:** 2026-03-29
**Valid until:** 2026-04-28 (stable -- all patterns from existing codebase)
