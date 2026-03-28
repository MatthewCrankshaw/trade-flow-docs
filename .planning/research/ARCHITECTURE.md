# Architecture Patterns

**Domain:** Stripe subscription billing integration into existing NestJS 11 strict-layered architecture
**Researched:** 2026-03-28
**Confidence:** HIGH

---

## New Components

### Backend: SubscriptionModule

```
src/subscription/
├── subscription.module.ts
├── controllers/
│   ├── subscription.controller.ts        # Authenticated endpoints (JwtAuthGuard)
│   └── stripe-webhook.controller.ts      # Unauthenticated; raw body; Stripe signature guard
├── services/
│   ├── subscription-creator.service.ts   # Create Stripe Checkout Session
│   ├── subscription-retriever.service.ts # Fetch local subscription status
│   └── subscription-updater.service.ts   # Webhook event processing → DB sync
├── repositories/
│   └── subscription.repository.ts        # MongoDB CRUD (collection: subscriptions)
├── entities/
│   └── subscription.entity.ts
├── data-transfer-objects/
│   └── subscription.dto.ts               # ISubscriptionDto
├── responses/
│   └── subscription.response.ts          # ISubscriptionResponse
├── enums/
│   └── subscription-status.enum.ts       # TRIALING | ACTIVE | PAST_DUE | CANCELED | INCOMPLETE
└── test/
    ├── services/
    ├── repositories/
    └── mocks/
        └── subscription.mock.ts
```

### Backend: StripeWebhookController (separate controller in SubscriptionModule)

The webhook controller must NOT use `@Body()`. It uses `@Req() req: RawBodyRequest<Request>` to access `req.rawBody` (Buffer) for `stripe.webhooks.constructEvent()`. No `JwtAuthGuard` — Stripe signature verification is the auth mechanism.

Route: `POST /v1/webhooks/stripe` — must be excluded from the global `ValidationPipe` (raw body bypass via controller-level guard instead of global pipe).

### Frontend: New Feature Module

```
src/features/subscription/
├── components/
│   ├── PricingCard.tsx           # Plan details + "Start Free Trial" CTA
│   ├── SubscriptionStatusCard.tsx # Current status, dates, Manage Billing CTA
│   ├── SubscriptionGate.tsx      # Route wrapper — redirects if not active/trialing
│   └── TrialBanner.tsx           # Persistent banner with days remaining
├── hooks/
│   └── useSubscription.ts        # Wraps RTK Query, returns { subscription, isActive, isTrialing }
└── api/
    └── subscriptionApi.ts        # RTK Query endpoints for subscription CRUD
```

### Frontend: New Pages

```
src/pages/
├── SubscribePage.tsx             # /subscribe — shows PricingCard
├── SubscriptionSuccessPage.tsx   # /subscribe/success — polls status, redirects to dashboard
└── SubscriptionCancelPage.tsx    # /subscribe/cancel — abandoned checkout, try again CTA
```

---

## Modified Existing Components

### Backend

| File | Change |
|------|--------|
| `src/main.ts` | Add `rawBody: true` to `NestFactory.create()` options |
| `src/app.module.ts` | Import `SubscriptionModule` |
| `src/user/` (optional) | Optionally expose `subscriptionStatus` on user DTO for frontend efficiency |

### Frontend

| File | Change |
|------|--------|
| `src/App.tsx` | Add `/subscribe`, `/subscribe/success`, `/subscribe/cancel` routes outside SubscriptionGate; wrap business routes in `<SubscriptionGate>` |
| `src/store/api.ts` | Add `"Subscription"` to `tagTypes` |
| `src/components/layout/AppLayout.tsx` (or equivalent) | Mount `<TrialBanner />` above main content |
| `src/pages/SettingsPage.tsx` | Add Billing tab with `SubscriptionStatusCard` |

---

## Data Flow

### Subscribe Flow
```
UI: /subscribe → PricingCard CTA clicked
→ POST /v1/subscription/checkout (authenticated)
→ SubscriptionCreator creates Stripe Checkout Session (trial_period_days: 30)
→ returns { url: "https://checkout.stripe.com/..." }
→ UI: window.location.href = url
→ User completes Checkout on Stripe
→ Stripe redirects to /subscribe/success
→ Stripe fires webhook: customer.subscription.created
→ POST /v1/webhooks/stripe → StripeWebhookController
→ SubscriptionUpdater creates subscription record in MongoDB (status=TRIALING)
→ UI polls GET /v1/subscription → status = trialing → trial banner shown
```

### Webhook Status Sync
```
Stripe event fires
→ POST /v1/webhooks/stripe (raw body)
→ stripe.webhooks.constructEvent(rawBody, sig, webhookSecret) → verified
→ SubscriptionUpdater routes by event.type
→ subscription.repository.upsertByStripeSubscriptionId(update)
→ Next GET /v1/subscription returns updated status
→ SubscriptionGate re-evaluates → access granted or revoked
```

### Access Enforcement (Frontend)
```
App.tsx route tree:
<ProtectedRoute>                        ← requires Firebase auth
  /subscribe, /subscribe/success, /subscribe/cancel   ← no gate
  /settings                             ← no gate (always accessible)
  <SubscriptionGate>                    ← requires active/trialing subscription
    /dashboard, /customers, /jobs, /quotes, /items, /business
  </SubscriptionGate>
</ProtectedRoute>
```

SubscriptionGate logic:
1. Call `useSubscription()` → `GET /v1/subscription`
2. If `isLoading` → show skeleton (avoid flash)
3. If `isActive || isTrialing` → render children
4. Else → redirect to `/subscribe`

---

## Build Order (suggested phases)

1. **API foundation** — SubscriptionModule skeleton, MongoDB entity/repo, SubscriptionStatus enum, `main.ts` rawBody config
2. **Stripe Checkout** — Checkout Session creation, webhook handler with event routing, subscription record CRUD
3. **API endpoints** — GET status, POST portal session, DELETE (cancel at period end), unit tests
4. **UI gate + pages** — SubscriptionGate, /subscribe page, success/cancel pages, RTK Query integration
5. **UI billing management** — TrialBanner, Settings > Billing tab, SubscriptionStatusCard
6. **OpenAPI spec** — document all new endpoints

---

## Key Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| Subscription keyed by `userId` (not `businessId`) | Per our milestone decision; aligns with Firebase auth identity; simpler for solo-operator model |
| `rawBody: true` in NestFactory | NestJS built-in; avoids middleware complexity; applies globally without disrupting JSON parsing elsewhere |
| StripeWebhookController separate from SubscriptionController | Different auth mechanism (signature vs JWT); different body parsing; clean separation |
| Local MongoDB subscription record (not Stripe as source of truth) | Avoids Stripe API call on every auth-gated request; webhook-driven sync keeps it current |
| Frontend polls on success page | Webhook may arrive seconds after redirect; polling `GET /v1/subscription` 2-3 times ensures gate opens |
| Settings always accessible | User must be able to resubscribe even when gated |
