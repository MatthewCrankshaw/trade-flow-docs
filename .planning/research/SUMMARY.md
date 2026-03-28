# Project Research Summary

**Project:** Trade Flow v1.6 -- Stripe Subscription Billing
**Synthesized:** 2026-03-28

---

## Stack Additions

**Backend (trade-flow-api):**
- `stripe` SDK (`^17.x`) — only new dependency needed
- `rawBody: true` added to `NestFactory.create()` in `main.ts` — critical for webhook signature verification
- No new packages on the frontend — hosted Checkout eliminates Stripe.js/Elements requirement

**Stripe Dashboard (pre-code):**
- Product: "Trade Flow Pro", Price: £6/month GBP recurring → store `STRIPE_PRICE_ID`
- Webhook endpoint configured pointing to `POST /v1/webhooks/stripe` with 5 subscribed events
- Customer Portal enabled with cancel + update payment method permissions

**New env vars:** `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID`

---

## Feature Table Stakes

**Subscription acquisition:** Stripe Checkout flow (hosted), card-required, 30-day trial. `/subscribe` page with PricingCard → Checkout Session → Stripe hosted page → `/subscribe/success` callback.

**Access enforcement:** `SubscriptionGate` React component wrapping all business routes. Read-only redirect to `/subscribe` when `status` is `past_due`, `canceled`, or `incomplete`. Settings always accessible.

**Trial experience:** Persistent trial banner showing days remaining. Banner disappears on `active`.

**Billing management:** Settings > Billing tab with `SubscriptionStatusCard` — shows status, dates, cancel CTA (→ `cancel_at_period_end`), "Manage Billing" button (→ Stripe Billing Portal for card updates and invoice history).

**Webhook-driven sync:** `customer.subscription.created/updated/deleted`, `invoice.payment_succeeded/failed`, `checkout.session.completed` → local MongoDB subscription record kept in sync.

---

## Architecture Integration Points

| Layer | New | Modified |
|-------|-----|---------|
| API | `SubscriptionModule` (controller, webhook controller, 3 services, repository, entity, DTO, response, enum) | `main.ts` (`rawBody: true`), `AppModule` (import SubscriptionModule) |
| Frontend | `src/features/subscription/` (components, hooks, RTK Query api), 3 new pages | `App.tsx` (routes + SubscriptionGate), `SettingsPage.tsx` (Billing tab), `AppLayout` (TrialBanner) |
| MongoDB | New `subscriptions` collection with unique indexes on `userId` and `stripeSubscriptionId` | — |

**Suggested build order:** API foundation → Stripe Checkout + webhooks → API endpoints + tests → UI gate + pages → UI billing management → OpenAPI spec

**Data flow:** `Stripe Checkout → webhook → MongoDB subscription record → GET /v1/subscription → SubscriptionGate → access granted/revoked`

---

## Watch Out For

### Critical
1. **Raw body destroyed by JSON parser** — add `rawBody: true` to `NestFactory.create` as the first task; without it, all webhook signature verifications fail silently
2. **Webhook idempotency** — use `upsert` (not `insert`) keyed on `stripeSubscriptionId`; add unique MongoDB indexes to prevent duplicate records from retry storms
3. **Checkout race condition** — webhook arrives after success page redirect; implement session verification endpoint (`GET /v1/subscription/verify-session?sessionId=`) so success page can confirm without waiting for webhook
4. **Trial enforcement** — use `status` field for access control, never `trialEnd < Date.now()`; Stripe manages trial-to-active transitions
5. **Cancel with `cancel_at_period_end`**, not `stripe.subscriptions.cancel()` — immediate cancel removes paid access

### Moderate
6. **SubscriptionGate flicker** — show loading skeleton during initial status fetch; never redirect during `isLoading === true`
7. **Webhook secret confusion** — Stripe CLI secret (local dev) ≠ Dashboard webhook secret (deployed); document in `.env.example`
8. **Support role bypass** — implement in both frontend gate AND backend guard; test both paths
9. **Stripe API version pinning** — pin version in SDK initialization; match Dashboard webhook endpoint version

---

## Key Decisions Implied by Research

| Decision | Rationale |
|----------|-----------|
| Stripe Checkout (hosted) over Stripe Elements | Zero PCI scope, SCA handled, 3DS handled automatically; no additional frontend packages |
| Local MongoDB subscription record (not Stripe as source of truth) | Avoids Stripe API call on every gated request; webhook sync keeps it current |
| `rawBody: true` globally (not route-specific middleware) | NestJS built-in; simpler than custom middleware; doesn't affect JSON parsing elsewhere |
| `cancel_at_period_end` on cancel | User retains access until period end; correct SaaS behavior |
| Session verification endpoint as race condition safety net | Webhook may arrive 1-30s after redirect; polling + server-side Stripe query eliminates the gap |
| Subscription keyed by `userId` | Per-user billing; aligns with Firebase auth identity; simpler for solo-operator model |
