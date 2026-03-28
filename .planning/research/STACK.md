# Stack Research

**Domain:** Stripe subscription billing for NestJS 11 + React 19 SaaS app
**Researched:** 2026-03-28
**Confidence:** HIGH

---

## New Dependencies

### Backend (trade-flow-api)

| Package | Version | Purpose | Justification |
|---------|---------|---------|---------------|
| `stripe` | `^17.x` | Stripe Node.js SDK | Official SDK — creates Checkout Sessions, Billing Portal sessions, verifies webhook signatures via `stripe.webhooks.constructEvent()` |

No other backend packages required. NestJS raw body handling uses a built-in option (see below), not a separate package.

### Frontend (trade-flow-ui)

No new packages required. Using Stripe Checkout (hosted page) means no Stripe.js or React Stripe.js needed. The UI redirects to `checkout.stripe.com` via `window.location.href = session.url`.

**Do NOT add:** `@stripe/stripe-js`, `@stripe/react-stripe-js` — these are only needed for embedded payment forms (Stripe Elements). Hosted Checkout eliminates that complexity entirely.

---

## NestJS Raw Body Configuration

The webhook endpoint (`POST /v1/webhooks/stripe`) requires the **raw unparsed request body** to verify Stripe's HMAC signature. NestJS applies a global JSON body parser by default, which destroys the raw buffer before the controller sees it.

### Solution: `rawBody: true` on NestFactory

```typescript
// main.ts
const app = await NestFactory.create(AppModule, { rawBody: true });
```

This enables NestJS's built-in `RawBodyRequest` support (available since NestJS 10). The raw buffer is accessible via `req.rawBody` on any route, while parsed JSON still works everywhere else.

In the webhook controller:

```typescript
import { RawBodyRequest } from '@nestjs/common';
import { Request } from 'express';

@Post()
async handleWebhook(@Req() req: RawBodyRequest<Request>) {
  const sig = req.headers['stripe-signature'] as string;
  const event = this.stripeService.constructEvent(req.rawBody, sig);
}
```

**Important:** Do NOT use `@Body()` on the webhook route — it receives the parsed object, not the raw buffer.

---

## Stripe Dashboard Configuration (one-time, pre-code)

Before writing any code:

1. Create **Product**: "Trade Flow Pro"
2. Create **Price**: £6.00/month, GBP, recurring monthly → note the `price_xxx` ID
3. Configure **Webhook endpoint** pointing to `POST /v1/webhooks/stripe`
   - Events: `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`
4. Note the **Webhook Signing Secret** (`whsec_xxx`)
5. Enable **Customer Portal** in Stripe Dashboard (Billing → Customer portal) and configure allowed actions (cancel, update payment method)

### New Environment Variables

| Variable | Description |
|----------|-------------|
| `STRIPE_SECRET_KEY` | `sk_live_xxx` or `sk_test_xxx` |
| `STRIPE_WEBHOOK_SECRET` | `whsec_xxx` from dashboard webhook config |
| `STRIPE_PRICE_ID` | Price ID of the £6/month recurring price |

---

## What NOT to Add

- **No Stripe.js / React Stripe.js** — hosted Checkout handles card input; no embedded forms
- **No webhook queue/retry system** — Stripe retries failed webhooks for 3 days; idempotent handlers are sufficient
- **No third-party NestJS Stripe wrapper** (`nestjs-stripe`, `@golevelup/nestjs-stripe`) — thin wrappers add a version dependency for no benefit
- **No BullMQ processor for webhooks** — webhooks are synchronous HTTP events; inline processing in the controller is correct
- **No Stripe CLI dependency in production** — `stripe listen` is dev-only for local webhook testing
