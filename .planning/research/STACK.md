# Stack Research

**Domain:** Onboarding overhaul, public landing page, no-card Stripe trial, hard paywall
**Researched:** 2026-03-31
**Confidence:** HIGH

---

## Recommended Stack Additions

### Frontend -- New Libraries

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| motion | 12.x | Landing page scroll animations, section reveals, hero transitions | Successor to Framer Motion (rebranded 2025). Declarative React API (`<motion.div>`), built-in scroll-linked animations on native ScrollTimeline for hardware acceleration. 30M+ monthly npm downloads. Import from `motion/react`. |
| react-intersection-observer | 10.x | Trigger animations when landing page sections scroll into view | Lightweight (2KB) React hook wrapping native IntersectionObserver API. Pairs with Motion's `whileInView` or standalone `useInView` hook for lazy reveal triggers. |

### Frontend -- No New Libraries Needed

| Capability | Existing Solution | Why No Addition |
|------------|-------------------|-----------------|
| Routing (public vs auth vs onboarding) | react-router-dom 7.13.0 | Already supports layout routes, loaders, and route-level guards. Add new route groups -- no new package. |
| Wizard/stepper UI | Radix UI primitives + custom state | Two-step onboarding (profile + business) is trivial with React state. A stepper library adds more weight than value for 2 steps. |
| Form validation | react-hook-form 7.71.1 + valibot 1.2.0 | Existing form stack handles profile and business setup forms. |
| Toast notifications | sonner 2.0.7 | Existing toast system for success/error feedback during onboarding. |
| Icons | lucide-react 0.563.0 | Landing page icons (features section, CTAs) covered by existing icon set. |
| Styling/layout | Tailwind CSS 4.1.18 | Landing page layout, responsive design, dark/light all handled by existing utility classes. |

### Backend -- No New Libraries Needed

| Capability | Existing Solution | Why No Addition |
|------------|-------------------|-----------------|
| Stripe no-card trial | stripe SDK 21.x (already installed) | Direct Subscription API with `trial_period_days` + `trial_settings.end_behavior.missing_payment_method`. No new Stripe packages. |
| Webhook handling | BullMQ + existing StripeWebhookProcessor | Same webhook pipeline from v1.6 handles `customer.subscription.created` event for API-created trials. |
| User profile updates | Existing UserModule services | Profile name field update is a standard PATCH operation. |
| Business creation | Existing BusinessCreator service | Mandatory business setup reuses existing creation flow with simplified inputs. |

---

## Core Technologies (Detail)

### 1. Motion (animation library) -- Frontend

**Package:** `motion` (NOT `framer-motion` -- legacy name, no longer maintained)
**Import:** `from "motion/react"`
**Version:** 12.x (latest 12.38.0 as of 2026-03-31)

**Use cases in this milestone:**
- Hero section fade-in and slide-up on page load
- Feature cards stagger animation as they scroll into view
- Pricing card hover/focus micro-interactions
- Section reveal animations on scroll (paired with IntersectionObserver)
- Smooth page transitions between landing and auth routes (optional, low priority)

**Key APIs needed:**
```typescript
// Basic reveal animation
import { motion } from "motion/react";

<motion.div
  initial={{ opacity: 0, y: 20 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, margin: "-100px" }}
  transition={{ duration: 0.5 }}
>
  <FeatureCard />
</motion.div>

// Stagger children
<motion.div variants={containerVariants} initial="hidden" whileInView="visible">
  {features.map((f) => (
    <motion.div key={f.id} variants={itemVariants}>...</motion.div>
  ))}
</motion.div>
```

**Why Motion over alternatives:**
- GSAP: Overkill for section reveals; imperative API clashes with React declarative patterns; commercial license for SaaS
- CSS animations only: No scroll-linked timing, no stagger orchestration, verbose for coordinated sequences
- react-spring: Less ecosystem momentum, weaker scroll animation support
- No animation library: Landing pages without scroll animations feel static and unpolished; this is marketing surface

**Bundle impact:** ~15KB gzipped for core React module. Tree-shakeable -- only imported APIs ship.

### 2. react-intersection-observer -- Frontend

**Package:** `react-intersection-observer`
**Version:** 10.x (latest 10.0.3 as of 2026-03-31)

**Why include when Motion has `whileInView`:**
Motion's `whileInView` is sufficient for most cases. This package is a lightweight fallback for non-animated visibility triggers (lazy loading images, analytics section tracking, "currently visible" nav highlighting). Include only if landing page needs visibility-driven logic beyond animation triggers. **Optional -- evaluate during implementation.**

### 3. Stripe Subscription API (no-card trial) -- Backend

**Approach: Direct Subscription API, NOT Stripe Checkout**

The v1.6 implementation uses Stripe Checkout (hosted page) with card collection for trial start. For no-card trials, there are two options:

**Option A -- Stripe Checkout with `payment_method_collection: "if_required"` (REJECTED):**
- Still redirects user to Stripe-hosted page
- Page shows "Start trial" button with no card fields -- confusing empty page
- Extra redirect away from the app and back
- Poor UX for a trial that collects nothing

**Option B -- Direct Subscription API (RECOMMENDED):**
- Create subscription server-side via `stripe.subscriptions.create()`
- No redirect, no Checkout page, instant trial activation
- User stays in the app throughout onboarding
- Stripe Customer created during onboarding, subscription created immediately after

**API call pattern:**
```typescript
// Backend: SubscriptionCreator service
const subscription = await this.stripe.subscriptions.create({
  customer: stripeCustomerId,
  items: [{ price: priceId }],
  trial_period_days: 30,
  payment_settings: {
    save_default_payment_method: "on_subscription",
  },
  trial_settings: {
    end_behavior: {
      missing_payment_method: "cancel",
    },
  },
});
```

**Key parameters:**
| Parameter | Value | Purpose |
|-----------|-------|---------|
| `trial_period_days` | `30` | 30-day free trial |
| `payment_settings.save_default_payment_method` | `"on_subscription"` | When user adds card later via Billing Portal, auto-attach to subscription |
| `trial_settings.end_behavior.missing_payment_method` | `"cancel"` | Auto-cancel if no card added by trial end (user must re-subscribe) |

**Webhook events to handle (same pipeline as v1.6):**
- `customer.subscription.created` -- new event for API-created subscriptions (Checkout fires `checkout.session.completed` instead)
- `customer.subscription.updated` -- trial ending, card added, status changes
- `customer.subscription.deleted` -- trial expired with no card (auto-cancel)

**Basil API note:** Same v2025-03-31.basil field paths apply. `current_period_end` at `items.data[0].current_period_end`.

**Migration from v1.6 Checkout approach:**
- Keep Checkout flow for re-subscription (user who canceled and wants to restart)
- Add direct API flow for initial no-card trial during onboarding
- Both flows feed into the same webhook processor and local subscription record
- Existing `SubscriptionGuard` and paywall logic unchanged

### 4. Routing Architecture -- Frontend

**No new packages. React Router 7.x layout routes handle all three zones.**

```
Route Structure:
/                          -- Public (no auth required)
/login                     -- Public (no auth required)
/signup                    -- Public (no auth required)

/onboarding/profile        -- Auth required, no business required
/onboarding/business       -- Auth required, no business required
/onboarding/trial          -- Auth required, business required, no subscription

/dashboard                 -- Auth + business + valid subscription required
/customers                 -- Auth + business + valid subscription required
/jobs                      -- Auth + business + valid subscription required
/quotes                    -- Auth + business + valid subscription required
/settings                  -- Auth + business + valid subscription required
```

**Guard layers (nested layout routes):**

```typescript
// Route tree structure
<Route element={<PublicLayout />}>        {/* No auth check */}
  <Route path="/" element={<LandingPage />} />
  <Route path="/login" element={<LoginPage />} />
  <Route path="/signup" element={<SignupPage />} />
</Route>

<Route element={<AuthGuard />}>           {/* Requires Firebase auth */}
  <Route element={<OnboardingGuard />}>   {/* Redirects if profile/business incomplete */}
    <Route element={<SubscriptionGuard />}> {/* Hard paywall if no valid subscription */}
      <Route element={<AppLayout />}>
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/customers" element={<CustomersPage />} />
        {/* ... */}
      </Route>
    </Route>
  </Route>
  <Route path="/onboarding/*" element={<OnboardingWizard />} />
</Route>
```

**Guard logic:**
| Guard | Check | Redirect To |
|-------|-------|-------------|
| `AuthGuard` | Firebase user exists | `/login` |
| `OnboardingGuard` | User has profile name AND business | `/onboarding/profile` or `/onboarding/business` |
| `SubscriptionGuard` | Subscription status in `[trialing, active]` | Full-screen paywall (not a redirect -- renders blocking overlay) |

**Hard paywall pattern:**
The v1.6 soft paywall intercepts write actions with a modal. The v1.7 hard paywall replaces the entire app shell with a blocking screen when subscription is invalid. This is a layout-level guard, not a per-action check.

```typescript
// SubscriptionGuard component (replaces soft modal approach)
function SubscriptionGuard({ children }) {
  const subscription = useAppSelector(selectSubscription);
  const isValid = subscription?.status === "trialing" || subscription?.status === "active";

  if (!isValid) {
    return <PaywallScreen />;  // Full-screen block, not a modal
  }

  return <Outlet />;
}
```

---

## Installation

```bash
# Frontend (trade-flow-ui)
npm install motion

# Optional -- only if non-animation visibility triggers needed
npm install react-intersection-observer
```

```bash
# Backend (trade-flow-api)
# No new packages. Stripe SDK 21.x already installed.
```

---

## Alternatives Considered

| Recommended | Alternative | Why Not Alternative |
|-------------|-------------|---------------------|
| motion 12.x | GSAP + ScrollTrigger | Commercial license required for SaaS; imperative API; heavier bundle; overkill for section reveals |
| motion 12.x | CSS @keyframes + IntersectionObserver | No orchestration (stagger, spring physics); verbose for coordinated sequences; no scroll-linked timing |
| motion 12.x | react-spring | Smaller ecosystem; weaker scroll animation primitives; less community adoption |
| motion 12.x | No animations | Landing page is marketing surface -- animations are expected table stakes for credibility |
| Direct Stripe API | Stripe Checkout (if_required) | Unnecessary redirect to empty Checkout page; poor UX; user leaves app flow |
| Direct Stripe API | Stripe Elements | Custom card form not needed -- no card collected at trial start |
| Layout route guards | Per-page HOC wrappers | Layout routes are idiomatic React Router 7; centralized guard logic; less boilerplate |
| Hard paywall (blocking screen) | Soft modal (v1.6 pattern) | Modal is dismissible; users can still navigate app without paying; hard gate enforces conversion |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| framer-motion (package) | Legacy name; no longer maintained; redirects to motion | `motion` package, import from `motion/react` |
| @stripe/stripe-js (frontend) | No frontend Stripe needed for no-card trial; Billing Portal handles card collection later | Server-side Stripe SDK only |
| @stripe/react-stripe-js | Same reason -- no card elements in the UI for this milestone | Billing Portal link for adding payment method |
| react-step-wizard / similar | Over-engineering for a 2-step onboarding flow | Local React state + conditional rendering |
| Lottie / rive-react | Heavy animation runtimes for what are simple CSS-style transitions | Motion declarative animations |
| AOS (animate-on-scroll) | jQuery-era library; no React integration; global side effects | Motion `whileInView` |

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| motion@12.x | React 19.x | Full React 19 support confirmed; uses React concurrent features |
| motion@12.x | Vite 7.x | ESM-native; no special Vite config needed |
| motion@12.x | Tailwind CSS 4.x | Motion handles transforms/opacity; Tailwind handles layout/colors; no conflicts |
| react-intersection-observer@10.x | React 19.x | Hook-based; no class component dependencies |
| stripe@21.x | API 2025-03-31.basil | Already pinned in v1.6; same SDK handles direct subscription creation |

---

## Integration Points

### Stripe -- Existing Infrastructure Reuse

| v1.6 Component | v1.7 Usage | Changes Needed |
|----------------|------------|----------------|
| StripeWebhookProcessor | Handles `customer.subscription.created` event | Add handler for new event type alongside existing 5 |
| SubscriptionRepository | Stores trial record from API-created subscription | Add `upsertByStripeSubscriptionId` if not already present |
| SubscriptionGuard (API) | Enforces subscription requirement on protected routes | No changes -- already checks `trialing`/`active` status |
| Billing Portal link | User adds card after trial starts | No changes -- existing Settings > Billing flow |
| verify-session endpoint | Not needed for direct API trial | Skip for API-created trials (subscription record created synchronously) |

### Onboarding -- Replaces Existing Flow

| Current (v1.6) | New (v1.7) | Impact |
|-----------------|------------|--------|
| Dismissible onboarding checklist | Mandatory wizard with redirect guards | Remove existing onboarding widget; add route guards |
| Checkout page at `/subscribe` | Inline trial activation during onboarding | `/subscribe` kept for re-subscription only |
| Soft paywall modal | Hard paywall full-screen block | Remove `useSubscriptionGate` hook (soft modal); replace with layout guard |

### Landing Page -- New Public Surface

| Component | Integration | Notes |
|-----------|-------------|-------|
| Root path `/` | Public route (no auth) | Currently redirects to `/dashboard`; change to render LandingPage |
| Navigation | Conditional: public nav (Login/Sign Up) vs app nav | Based on auth state from AuthProvider |
| Pricing section | Links to `/signup` (not direct Stripe Checkout) | User signs up first, trial created during onboarding |

---

## Sources

- [Stripe: Configure free trials (Checkout)](https://docs.stripe.com/payments/checkout/free-trials) -- `payment_method_collection: "if_required"` for Checkout approach (MEDIUM confidence)
- [Stripe: Use free trial periods on subscriptions](https://docs.stripe.com/billing/subscriptions/trials/free-trials) -- Direct API with `trial_period_days`, `trial_settings.end_behavior.missing_payment_method` (HIGH confidence)
- [Stripe: Configure trial offers](https://docs.stripe.com/billing/subscriptions/trials) -- Flexible billing mode trial offers (MEDIUM confidence -- newer API, may not apply to classic mode)
- [Motion official site](https://motion.dev/) -- Rebrand from Framer Motion, React integration docs (HIGH confidence)
- [Motion npm](https://www.npmjs.com/package/motion) -- v12.38.0 current (HIGH confidence)
- [react-intersection-observer npm](https://www.npmjs.com/package/react-intersection-observer) -- v10.0.3 current (HIGH confidence)
- Phase 30 Research (trade-flow-api) -- Stripe basil API field paths, webhook processor pattern (HIGH confidence)
- [Motion React scroll animations](https://motion.dev/docs/react-scroll-animations) -- Native ScrollTimeline support (HIGH confidence)

---
*Stack research for: v1.7 Onboarding & Landing Page*
*Researched: 2026-03-31*
