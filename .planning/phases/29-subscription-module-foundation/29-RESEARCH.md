# Phase 29: Subscription Module Foundation - Research

**Researched:** 2026-03-29
**Domain:** Stripe SDK integration, NestJS raw body handling, MongoDB subscription schema
**Confidence:** HIGH

## Summary

Phase 29 wires up the Stripe SDK as a NestJS injectable provider, enables raw body preservation globally for webhook signature verification, defines the subscription MongoDB entity with all fields needed through Phase 33, and exposes two proof-of-wiring endpoints: `POST /v1/subscription/checkout` (returns a Stripe Checkout Session URL) and `POST /v1/webhooks/stripe` (validates Stripe signatures, rejects invalid ones with 400).

The implementation follows established NestJS patterns already in the codebase -- factory providers (like EmailModule/SendGrid), `@Public()` decorator for unauthenticated routes, and standard Controller-Service-Repository layering. The only cross-cutting change is adding `rawBody: true` to `NestFactory.create()` in `main.ts`, which must happen before anything else in this phase.

**Primary recommendation:** Install `stripe` v21.x, enable `rawBody: true` globally in main.ts, create SubscriptionModule with STRIPE_CLIENT factory provider, implement the two endpoints following existing module conventions exactly.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Create a Stripe Customer explicitly before creating the Checkout Session. Call `stripe.customers.create({ email, metadata: { userId } })` at the start of the checkout handler, store `stripeCustomerId` on the local subscription record.
- **D-02:** User email comes from the Firebase JWT (`request.user.email` via JwtAuthGuard). No extra MongoDB lookup needed.
- **D-03:** Pass `customer: stripeCustomerId` to the Checkout Session (not `customer_email`). Creates explicit userId-to-Stripe Customer link.
- **D-04:** Read frontend base URL from a `FRONTEND_URL` env var (added to `.env.example`).
- **D-05:** `success_url` format: `${FRONTEND_URL}/subscribe/success?session_id={CHECKOUT_SESSION_ID}` (Stripe-side substitution).
- **D-06:** `cancel_url` format: `${FRONTEND_URL}/subscribe/cancel`
- **D-07:** Define the complete subscription entity now with all fields Phase 30-33 will need. Phase 29 populates only what it touches.
- **D-08:** Full entity fields: `userId`, `stripeCustomerId`, `stripeSubscriptionId` (optional), `status` (SubscriptionStatus enum), `currentPeriodEnd` (optional), `trialEnd` (optional), `cancelAtPeriodEnd` (boolean, default false), `canceledAt` (optional), `createdAt`, `updatedAt`.
- **D-09:** `SubscriptionStatus` enum mirrors Stripe exactly: `trialing`, `active`, `past_due`, `canceled`, `incomplete`.
- **D-10:** `POST /v1/webhooks/stripe` must be public -- no JwtAuthGuard. Use `@Public()` decorator.
- **D-11:** `rawBody: true` in `NestFactory.create(AppModule, { rawBody: true })` -- global, applied to all routes. Blocking first task.
- **D-12:** SubscriptionModule is a standard NestJS feature module. Stripe SDK injected as factory provider with `STRIPE_CLIENT` injection token inside SubscriptionModule.
- **D-13:** Subscription keyed by `userId` (one subscription per user).

### Claude's Discretion
- Stripe provider factory implementation details (how to read STRIPE_SECRET_KEY via ConfigService)
- Repository method naming and upsert patterns (follow existing repository conventions)
- DTO/response shape for the checkout endpoint response (just `{ url: string }` is sufficient)
- Unit test scope for Phase 29 (success criteria don't require tests; tests are Phase 31)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ACQ-02 | API creates a Stripe Checkout Session with `trial_period_days: 30` and returns the hosted checkout URL | Stripe SDK `checkout.sessions.create()` with `subscription_data.trial_period_days: 30`, `mode: "subscription"`, `customer`, `line_items`, `success_url`, `cancel_url`. Returns `session.url`. |
| WBHK-01 | Stripe webhook endpoint verifies request signature using raw body before processing any event | NestJS `rawBody: true` in NestFactory.create enables `req.rawBody` (Buffer). Controller uses `stripe.webhooks.constructEvent(rawBody, sig, secret)` to verify. Returns 400 on failure. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| stripe | 21.0.1 | Stripe API SDK (Checkout Sessions, Customers, webhook verification) | Official Stripe Node.js SDK with full TypeScript support |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @nestjs/common | 11.1.x | RawBodyRequest type, @Req() decorator | Accessing raw body in webhook controller |
| @nestjs/config | 4.0.x | ConfigService for reading env vars | Stripe factory provider reads STRIPE_SECRET_KEY |
| mongoose | 9.x | MongoDB ODM for subscription entity | Schema definition, indexes, CRUD operations |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Factory provider (STRIPE_CLIENT) | Dedicated StripeModule | Factory provider inside SubscriptionModule is simpler for Phase 29 scope; can extract later if needed |
| Global rawBody: true | Route-specific express.raw() middleware | Global is simpler, already decided (D-11), no need for route-level granularity |

**Installation:**
```bash
npm install stripe@21
```

**Version verification:** `stripe` v21.0.1 confirmed via npm registry on 2026-03-29. All other packages already installed in trade-flow-api.

## Architecture Patterns

### Recommended Module Structure
```
src/subscription/
  subscription.module.ts
  controllers/
    subscription.controller.ts
  services/
    subscription-creator.service.ts
  repositories/
    subscription.repository.ts
  data-transfer-objects/
    subscription.dto.ts
  entities/
    subscription.entity.ts
  enums/
    subscription-status.enum.ts
  responses/
    subscription.response.ts
  providers/
    stripe.provider.ts
```

### Pattern 1: STRIPE_CLIENT Factory Provider
**What:** Register the Stripe SDK as an injectable NestJS provider using a factory pattern.
**When to use:** Whenever an external SDK needs configuration from environment variables.
**Example:**
```typescript
// Source: Existing EmailModule/SendGrid pattern in trade-flow-api
import Stripe from "stripe";
import { ConfigService } from "@nestjs/config";

export const STRIPE_CLIENT = "STRIPE_CLIENT";

export const stripeProvider = {
  provide: STRIPE_CLIENT,
  useFactory: (configService: ConfigService): Stripe => {
    return new Stripe(configService.get<string>("STRIPE_SECRET_KEY"));
  },
  inject: [ConfigService],
};
```

### Pattern 2: Raw Body Access in Webhook Controller
**What:** Access the unparsed request body as a Buffer for Stripe signature verification.
**When to use:** Webhook endpoints that require signature verification against the raw bytes.
**Example:**
```typescript
// Source: NestJS official docs (faq/raw-body) + NestJS GitHub integration tests
import { Controller, Post, Req, Headers } from "@nestjs/common";
import { RawBodyRequest } from "@nestjs/common";
import { Request } from "express";

@Controller("v1/webhooks")
export class WebhookController {
  @Post("stripe")
  @Public()
  handleStripeWebhook(
    @Req() req: RawBodyRequest<Request>,
    @Headers("stripe-signature") signature: string,
  ) {
    const event = this.stripe.webhooks.constructEvent(
      req.rawBody,
      signature,
      this.webhookSecret,
    );
    // Phase 30 adds event handlers here
  }
}
```

### Pattern 3: Checkout Session Creation
**What:** Create a Stripe Checkout Session for subscription with trial.
**When to use:** When user initiates subscription flow.
**Example:**
```typescript
// Source: Stripe official docs (payments/checkout/free-trials)
const customer = await this.stripe.customers.create({
  email: authUser.email,
  metadata: { userId: authUser.userId },
});

const session = await this.stripe.checkout.sessions.create({
  mode: "subscription",
  customer: customer.id,
  line_items: [{ price: this.priceId, quantity: 1 }],
  subscription_data: { trial_period_days: 30 },
  success_url: `${this.frontendUrl}/subscribe/success?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${this.frontendUrl}/subscribe/cancel`,
});

return { url: session.url };
```

### Anti-Patterns to Avoid
- **Parsing body before signature check:** Never use `req.body` (parsed JSON) for `constructEvent()`. Always use `req.rawBody` (Buffer). The signature is computed against raw bytes.
- **Using `customer_email` instead of `customer`:** Per D-03, always create a Stripe Customer first and pass `customer: stripeCustomerId`. Using `customer_email` creates a new Stripe customer per session, breaking reconciliation.
- **Separate StripeModule:** Per D-12, keep the Stripe provider scoped to SubscriptionModule. No need for a separate module at this phase.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Webhook signature verification | Manual HMAC comparison | `stripe.webhooks.constructEvent()` | Handles timestamp tolerance, multiple signatures, replay attack prevention |
| Raw body preservation | Custom middleware with express.raw() | NestJS `rawBody: true` option | Built-in NestJS feature since v8, cleaner than middleware |
| Stripe API client | Raw HTTP calls to Stripe API | `stripe` npm package | Full TypeScript types, automatic retries, idempotency key support |
| Checkout flow | Custom payment form | Stripe Checkout (hosted) | PCI compliance, SCA/3DS handled automatically |

**Key insight:** Stripe's SDK handles signature verification timing attacks, replay protection via timestamp tolerance (default 300s), and multiple signature schemes. Rolling your own HMAC check misses these protections.

## Common Pitfalls

### Pitfall 1: rawBody Not Available
**What goes wrong:** `req.rawBody` is `undefined`, causing `constructEvent()` to throw "No signatures found matching the expected signature for payload."
**Why it happens:** `rawBody: true` was not passed to `NestFactory.create()`, or it was added after a body-parsing middleware already consumed the stream.
**How to avoid:** Add `rawBody: true` as the very first change in the phase. Verify it works with a simple test before building the webhook controller.
**Warning signs:** Any "signature verification failed" error during testing.

### Pitfall 2: Wrong Webhook Secret (Test vs Live)
**What goes wrong:** Signature verification fails in all environments.
**Why it happens:** Stripe signs test-mode and live-mode events with different secrets. Using `whsec_test_...` against live events (or vice versa) always fails.
**How to avoid:** Use `STRIPE_WEBHOOK_SECRET` env var, set per environment. For local dev, use Stripe CLI (`stripe listen --forward-to`) which provides its own signing secret.
**Warning signs:** 400 errors on every webhook, even with correct raw body.

### Pitfall 3: Missing `@Public()` on Webhook Route
**What goes wrong:** Stripe webhook calls get 401 Unauthorized because JwtAuthGuard rejects them (no Bearer token).
**Why it happens:** JwtAuthGuard is applied globally or at the module level, and the webhook route is not excluded.
**How to avoid:** Apply `@Public()` decorator to the webhook handler method, following the existing public quote endpoint pattern.
**Warning signs:** 401 responses in Stripe webhook logs.

### Pitfall 4: Checkout Session `{CHECKOUT_SESSION_ID}` Template Literal
**What goes wrong:** The success URL contains the literal string `{CHECKOUT_SESSION_ID}` instead of the actual session ID.
**Why it happens:** Developer wraps it in JavaScript template literal backticks, causing JS to try to resolve it as a variable. This is a Stripe-side template that must be passed as a plain string.
**How to avoid:** The `{CHECKOUT_SESSION_ID}` part must be a literal string in the URL, not a JS expression. Use string concatenation or ensure it is not inside `${}`.
**Warning signs:** Redirect URL contains literal `{CHECKOUT_SESSION_ID}` or `undefined`.

### Pitfall 5: Stripe SDK Import Style
**What goes wrong:** TypeScript compilation errors or runtime `Stripe is not a constructor`.
**Why it happens:** The `stripe` package uses a default export pattern that varies between ESM and CJS.
**How to avoid:** In CommonJS (NestJS backend): `import Stripe from "stripe";` then `new Stripe(apiKey)`. The SDK auto-detects the API version.
**Warning signs:** `TypeError: Stripe is not a constructor` at runtime.

## Code Examples

### Main.ts rawBody Configuration
```typescript
// Source: NestJS official docs + D-11
// In src/main.ts, modify NestFactory.create call:
const app = await NestFactory.create(AppModule, { rawBody: true });
```

### SubscriptionStatus Enum
```typescript
// Source: D-09, mirrors Stripe subscription.status values
export enum SubscriptionStatus {
  TRIALING = "trialing",
  ACTIVE = "active",
  PAST_DUE = "past_due",
  CANCELED = "canceled",
  INCOMPLETE = "incomplete",
}
```

### Subscription Entity Interface
```typescript
// Source: D-08, follows I{Name}Entity pattern from CONVENTIONS.md
import type { ObjectId } from "mongodb";

export interface ISubscriptionEntity {
  _id: ObjectId;
  userId: string;
  stripeCustomerId: string;
  stripeSubscriptionId?: string;
  status: string; // SubscriptionStatus enum value
  currentPeriodEnd?: Date;
  trialEnd?: Date;
  cancelAtPeriodEnd: boolean;
  canceledAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

### Subscription DTO Interface
```typescript
// Source: D-08, follows I{Name}Dto pattern
export interface ISubscriptionDto {
  id: string;
  userId: string;
  stripeCustomerId: string;
  stripeSubscriptionId?: string;
  status: string;
  currentPeriodEnd?: Date;
  trialEnd?: Date;
  cancelAtPeriodEnd: boolean;
  canceledAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}
```

### MongoDB Indexes
```typescript
// Unique indexes per D-13 and success criteria #3
// userId: one subscription per user
// stripeSubscriptionId: unique lookup for webhook reconciliation (sparse for Phase 29 when field is absent)
db.collection("subscriptions").createIndex({ userId: 1 }, { unique: true });
db.collection("subscriptions").createIndex({ stripeSubscriptionId: 1 }, { unique: true, sparse: true });
```

### Webhook Signature Verification (Full Pattern)
```typescript
// Source: Stripe official docs (webhooks/signature)
handleStripeWebhook(rawBody: Buffer, signature: string): Stripe.Event {
  try {
    return this.stripe.webhooks.constructEvent(
      rawBody,
      signature,
      this.webhookSecret,
    );
  } catch (err) {
    throw new InvalidRequestError(
      ErrorCodes.INVALID_WEBHOOK_SIGNATURE,
      "Webhook signature verification failed",
      err.message,
    );
  }
}
```

### Environment Variables
```bash
# Added to .env.example
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID=price_...
FRONTEND_URL=http://localhost:5173
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Custom body-parser middleware for raw body | `NestFactory.create(AppModule, { rawBody: true })` | NestJS 9.x (2022) | No custom middleware needed; `req.rawBody` is a Buffer |
| `stripe.webhooks.constructEvent()` with string body | Same API, but Buffer is now preferred | stripe-node v10+ | Buffer avoids encoding issues; both string and Buffer work |
| `new Stripe(key, { apiVersion: "..." })` | `new Stripe(key)` (auto-detects version) | stripe-node v14+ | API version pinned in SDK release; explicit version param optional |

**Deprecated/outdated:**
- Manual `express.raw()` middleware for specific routes: replaced by NestJS built-in rawBody option
- `customer_email` parameter on Checkout Sessions: still works but creates new customers each time; `customer` parameter is preferred for reconciliation

## Open Questions

1. **ErrorCodes enum value for webhook signature failure**
   - What we know: The codebase uses `ErrorCodes` enum in `src/core/errors/` for error codes
   - What's unclear: Whether a new code needs to be added or an existing one (like `INVALID_REQUEST`) can be reused
   - Recommendation: Check existing ErrorCodes values; add a new one if webhook-specific codes are needed, or reuse `InvalidRequestError` with a descriptive message

2. **Path alias for subscription module**
   - What we know: Every module has a `@modulename/*` path alias in tsconfig.json
   - What's unclear: Whether `@subscription/*` needs to be added to tsconfig.json and tsconfig.build.json
   - Recommendation: Add `@subscription/*` alias pointing to `src/subscription/` in all relevant tsconfig files, following the pattern of existing aliases

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest 29.3.1 |
| Config file | jest config in package.json (trade-flow-api) |
| Quick run command | `npm test -- --testPathPattern=subscription` |
| Full suite command | `npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ACQ-02 | Checkout Session creation returns URL | unit (Stripe SDK mocked) | `npm test -- --testPathPattern=subscription-creator` | No -- Wave 0 (tests deferred to Phase 31 per TEST-01) |
| WBHK-01 | Invalid webhook signature returns 400 | unit (constructEvent mocked) | `npm test -- --testPathPattern=webhook` | No -- Wave 0 (tests deferred to Phase 31 per TEST-02) |

### Sampling Rate
- **Per task commit:** Manual verification via curl/Stripe CLI (success criteria are runtime checks)
- **Per wave merge:** `npm test` (existing tests should not break)
- **Phase gate:** Success criteria 1-3 verified manually

### Wave 0 Gaps
- Tests are explicitly deferred to Phase 31 (TEST-01 through TEST-04)
- Phase 29 success criteria are verified via runtime behavior (curl against running API), not automated tests
- Existing test suite must pass after changes (`npm test`)

## Sources

### Primary (HIGH confidence)
- [NestJS Raw Body FAQ](https://docs.nestjs.com/faq/raw-body) - rawBody setup and RawBodyRequest type
- [NestJS GitHub integration test](https://github.com/nestjs/nest/blob/master/integration/nest-application/raw-body/src/express.controller.ts) - exact RawBodyRequest usage pattern
- [Stripe Checkout Free Trials docs](https://docs.stripe.com/payments/checkout/free-trials) - subscription_data.trial_period_days parameter
- [Stripe Webhook Signature docs](https://docs.stripe.com/webhooks/signature) - constructEvent() parameters and error handling
- npm registry: `stripe` v21.0.1 (verified 2026-03-29)

### Secondary (MEDIUM confidence)
- [Manuel Heidrich blog - NestJS raw body for Stripe](https://manuel-heidrich.dev/blog/how-to-access-the-raw-body-of-a-stripe-webhook-request-in-nestjs/) - NestJS + Stripe integration walkthrough
- [Wanago.io - NestJS Stripe webhooks](https://wanago.io/2021/07/05/api-nestjs-stripe-events-webhooks/) - Comprehensive NestJS + Stripe pattern reference
- [DEV.to Stripe NestJS guide](https://dev.to/imzihad21/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65) - Express rawBody middleware patterns

### Tertiary (LOW confidence)
- None -- all findings verified against official docs

## Project Constraints (from CLAUDE.md)

- **Tech stack:** NestJS (API) patterns -- Controller-Service-Repository layering
- **Two repos:** Phase 29 is API-only (trade-flow-api)
- **Conventions:** File naming (`kebab-case.type.ts`), class naming (`PascalCase`), enum pattern (UPPER_SNAKE keys, lowercase values), `@Public()` decorator for unauthenticated routes
- **Error handling:** Custom error classes with `getCode()`, `getMessage()`, `getDetails()`; controllers catch via try-catch and pass to `createHttpError()`
- **Logging:** `AppLogger` with class name context
- **Code style:** Double quotes, semicolons, 125 char width, trailing commas, 2-space indent
- **Path aliases:** Mandatory for cross-module imports

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Stripe SDK is the only new dependency; version verified against npm registry
- Architecture: HIGH - Follows existing module patterns exactly (factory provider, controller-service-repository)
- Pitfalls: HIGH - rawBody is well-documented; webhook signature verification is a known challenge with well-established solutions

**Research date:** 2026-03-29
**Valid until:** 2026-04-28 (Stripe SDK stable, NestJS rawBody feature stable)
