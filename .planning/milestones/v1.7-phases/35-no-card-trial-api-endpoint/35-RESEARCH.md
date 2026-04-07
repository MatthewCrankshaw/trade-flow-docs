# Phase 35: No-Card Trial API Endpoint - Research

**Researched:** 2026-04-02
**Domain:** Stripe Subscriptions API -- trial creation without payment method, NestJS subscription module extension
**Confidence:** HIGH

## Summary

This phase adds a `POST /v1/subscription/trial` endpoint that creates a Stripe subscription with a 30-day free trial requiring no payment method. The Stripe Subscriptions API natively supports this via `trial_period_days` combined with `trial_settings.end_behavior.missing_payment_method: 'cancel'`. The endpoint writes a local MongoDB subscription record synchronously (belt-and-suspenders with webhook), and the webhook processor gains a new `customer.subscription.created` handler for compatibility with both Checkout-originated and API-originated subscriptions.

All building blocks exist from Phases 29-30: the `SubscriptionModule`, `STRIPE_CLIENT` injection token, `SubscriptionRepository` with upsert patterns, `StripeWebhookProcessor`, entity schema with `trialEnd`/`status`/`currentPeriodEnd` fields, and the `InvalidRequestError` class. This phase adds a new service, a new controller route, one new webhook event handler, and modifies the existing `checkout.session.completed` handler to use upsert.

**Primary recommendation:** Use `stripe.subscriptions.create()` with `trial_period_days: 30` and `trial_settings.end_behavior.missing_payment_method: 'cancel'`. Reuse the existing Stripe Customer if one exists. Write the local record synchronously in the response, then let webhooks reconcile.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Reuse existing Stripe Customer if one exists for the user. Look up `stripeCustomerId` on the local subscription record by `userId`. Only create a new Stripe Customer if no record exists. One Customer per user in Stripe.
- **D-02:** Write the local subscription record synchronously in the endpoint response (status=trialing, stripeSubscriptionId, trialEnd, currentPeriodEnd). User sees trial active immediately without waiting for webhook.
- **D-03:** Use the same `STRIPE_PRICE_ID` env var that the Checkout flow reads. One price for both flows.
- **D-04:** Return the full local subscription record in the response body (status, trialEnd, currentPeriodEnd, etc.). Matches existing `GET /v1/subscription` response shape.
- **D-05:** Set `trial_settings.end_behavior.missing_payment_method: 'cancel'` on the `stripe.subscriptions.create()` call. Explicit in code, version-controlled, no manual Stripe Dashboard configuration per environment.
- **D-06:** Add `customer.subscription.created` as a new event type in `StripeWebhookProcessor`. It upserts by `stripeCustomerId` -- if a record already exists (from sync write), it updates; if not (Checkout flow or race condition), it creates.
- **D-07:** Update `checkout.session.completed` handler to upsert instead of insert-only. Handles edge cases where a user starts trial, cancels, then goes through Checkout. Both webhook paths become resilient to pre-existing records.
- **D-08:** Only add `customer.subscription.created` -- do NOT add `customer.subscription.trial_will_end`. Trial nudge emails (NUDGE-01/02) are explicitly deferred to a future milestone.
- **D-09:** Trial endpoint rejects if ANY local subscription record exists for the userId (any status: trialing, active, past_due, canceled). Trial is a one-time benefit for brand-new users only.
- **D-10:** No re-trial after cancellation. Users with canceled subscriptions must resubscribe via Checkout/Billing Portal with a card.
- **D-11:** Return HTTP 422 (`InvalidRequestError`) with message like "Trial already used. Manage your subscription in Billing." Consistent with Phase 30's duplicate checkout guard pattern.
- **D-12:** When trial ends without a payment method, Stripe fires `customer.subscription.deleted`. The existing Phase 30 handler already sets `status=canceled` and `canceledAt`. No additional handling needed.
- **D-13:** Phase 38's differentiated paywall messaging can infer "trial expired" vs "user canceled" by comparing `trialEnd` and `canceledAt` timestamps on the existing entity fields. No new schema fields required.

### Claude's Discretion
- Exact `stripe.subscriptions.create()` parameter structure beyond what's specified above
- Repository method naming for the new lookup patterns (e.g., `findByUserId`, `findByStripeCustomerId`)
- How the trial creator service is structured (new service class vs extending existing)
- Unit test structure and scope
- Stripe webhook registration configuration (which events to register in Stripe Dashboard -- documented in README or deploy notes)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TRIAL-01 | User's 30-day free trial starts automatically after business setup without entering payment details | Stripe `subscriptions.create()` with `trial_period_days: 30` supports no-card trials natively. Sync write ensures user sees trial active immediately. |
| TRIAL-04 | Subscription auto-cancels when trial ends without a payment method on file | `trial_settings.end_behavior.missing_payment_method: 'cancel'` configures Stripe to auto-cancel. Fires `customer.subscription.deleted` which Phase 30's handler already processes. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| stripe (Node.js SDK) | Already installed in trade-flow-api (Phase 29) | Stripe API client | Injected via `STRIPE_CLIENT` token; provides typed `subscriptions.create()` |
| @nestjs/core | 11.1.12 | NestJS framework | Existing project framework |
| mongoose | 9.1.5 | MongoDB ODM | Existing project ODM |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| class-validator | 0.14.1 | DTO validation | Request body validation if trial endpoint accepts a body (likely no body needed) |
| luxon | 3.5.1 | Date/time handling | Converting Stripe Unix timestamps to Date objects for entity fields |

No new packages required. All dependencies are already installed from Phases 29-30.

## Architecture Patterns

### Recommended Project Structure
```
src/subscription/
  controllers/
    subscription.controller.ts     # Add POST /trial route
  services/
    subscription-trial-creator.service.ts  # NEW -- trial creation logic
    subscription-checkout-creator.service.ts  # Existing
  repositories/
    subscription.repository.ts     # Add findByUserId method
  processors/
    stripe-webhook.processor.ts    # Add customer.subscription.created handler
```

### Pattern 1: Trial Creator Service (New Service Class)
**What:** A dedicated `SubscriptionTrialCreator` service following the existing single-responsibility naming (e.g., `BusinessCreator`, `SubscriptionCheckoutCreator`).
**When to use:** Always -- this is the established project pattern.
**Example:**
```typescript
// Source: Project conventions from CLAUDE.md + Phase 29 D-12 patterns
@Injectable()
export class SubscriptionTrialCreator {
  private readonly logger = new AppLogger(SubscriptionTrialCreator.name);

  constructor(
    @Inject(STRIPE_CLIENT) private readonly stripe: Stripe,
    private readonly subscriptionRepository: SubscriptionRepository,
    private readonly configService: ConfigService,
  ) {}

  async create(authUser: IUserDto): Promise<ISubscriptionDto> {
    // 1. Check for existing subscription (any status) -- D-09
    const existing = await this.subscriptionRepository.findByUserId(authUser.userId);
    if (existing) {
      throw new InvalidRequestError(
        ErrorCodes.DUPLICATE_RESOURCE,
        "Trial already used. Manage your subscription in Billing.",
      );
    }

    // 2. Create Stripe Customer -- D-01
    const customer = await this.stripe.customers.create({
      email: authUser.email,
      metadata: { userId: authUser.userId },
    });

    // 3. Create Stripe Subscription with trial -- D-05
    const priceId = this.configService.get<string>("STRIPE_PRICE_ID");
    const subscription = await this.stripe.subscriptions.create({
      customer: customer.id,
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

    // 4. Write local record synchronously -- D-02
    const dto: ISubscriptionDto = {
      userId: authUser.userId,
      stripeCustomerId: customer.id,
      stripeSubscriptionId: subscription.id,
      status: SubscriptionStatus.TRIALING,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      trialEnd: new Date(subscription.trial_end * 1000),
      cancelAtPeriodEnd: false,
    };

    return this.subscriptionRepository.upsertByStripeCustomerId(dto);
  }
}
```

### Pattern 2: Webhook Upsert for customer.subscription.created
**What:** New event handler in `StripeWebhookProcessor` that upserts by `stripeCustomerId`.
**When to use:** When Stripe fires `customer.subscription.created` for any subscription origin (Checkout or API).
**Example:**
```typescript
// Source: Phase 30 D-04 reconciliation pattern
private async handleSubscriptionCreated(event: Stripe.Event): Promise<void> {
  const subscription = event.data.object as Stripe.Subscription;
  const stripeCustomerId = typeof subscription.customer === "string"
    ? subscription.customer
    : subscription.customer.id;

  await this.subscriptionRepository.upsertByStripeCustomerId({
    stripeCustomerId,
    stripeSubscriptionId: subscription.id,
    status: subscription.status as SubscriptionStatus,
    currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    trialEnd: subscription.trial_end ? new Date(subscription.trial_end * 1000) : undefined,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
  });
}
```

### Pattern 3: Checkout Handler Upsert Migration (D-07)
**What:** Change `checkout.session.completed` handler from insert to upsert by `stripeCustomerId`.
**When to use:** Handles edge case where user starts trial, cancels, then subscribes via Checkout.
**Why:** The existing handler may use insert-only semantics. Changing to upsert ensures idempotency regardless of prior subscription state.

### Anti-Patterns to Avoid
- **Creating a new Stripe Customer every time:** Always check for existing `stripeCustomerId` on the local record first (D-01). A user who had a trial and canceled already has a Stripe Customer.
- **Processing trial creation in webhook only:** The sync write (D-02) ensures the user sees their trial active immediately. The webhook is belt-and-suspenders, not the primary path.
- **Using the new Trial Offer API (preview):** Stripe's new Trial Offer API requires `billing_mode[type]=flexible` which is a preview feature. The project uses classic billing mode. Use the stable `trial_period_days` parameter instead.
- **Adding `customer.subscription.trial_will_end` handler:** Explicitly deferred (D-08). Do not add this event type.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Trial expiry auto-cancel | Custom cron/scheduler to check trial end dates | `trial_settings.end_behavior.missing_payment_method: 'cancel'` | Stripe handles this server-side, fires `customer.subscription.deleted` event |
| Unix timestamp conversion | Manual math (`* 1000`) scattered everywhere | Centralized helper or inline `new Date(ts * 1000)` | Stripe returns seconds, JS expects milliseconds -- easy off-by-1000x bug |
| Subscription status mapping | Custom status strings | `SubscriptionStatus` enum mirroring Stripe values exactly (Phase 29 D-09) | Already established -- `trialing`, `active`, `past_due`, `canceled` |
| Idempotent writes | Check-then-insert with race conditions | MongoDB `findOneAndUpdate` with `upsert: true` | Already established in Phase 30 -- atomic upsert prevents duplicates |

**Key insight:** Stripe handles the entire trial lifecycle (creation, period tracking, auto-cancel on expiry). The API endpoint's job is to create the subscription with the right parameters and write the local record. No custom trial management logic is needed.

## Common Pitfalls

### Pitfall 1: Stripe Customer Lookup vs Creation Race Condition
**What goes wrong:** Two concurrent trial requests for the same user could both find no existing record and both create Stripe Customers.
**Why it happens:** No locking between the findByUserId check and the Stripe customer.create call.
**How to avoid:** This is a low-risk scenario (single user, single button click). The D-09 guard (reject if ANY record exists) means the second request fails at the upsert stage. MongoDB unique index on `userId` provides the final safety net.
**Warning signs:** Duplicate Stripe Customers in the Stripe Dashboard for the same email.

### Pitfall 2: Stripe Timestamps Are Unix Seconds, Not Milliseconds
**What goes wrong:** Storing `subscription.trial_end` directly as a Date produces a date in January 1970.
**Why it happens:** Stripe returns Unix timestamps in seconds; JavaScript `Date` constructor expects milliseconds.
**How to avoid:** Always multiply by 1000: `new Date(subscription.trial_end * 1000)`.
**Warning signs:** Dates showing as 1970 in the database or API responses.

### Pitfall 3: Webhook Registration in Stripe Dashboard
**What goes wrong:** The `customer.subscription.created` event is handled in code but never fires because it's not registered in the Stripe Dashboard webhook endpoint configuration.
**Why it happens:** Stripe only sends events that are explicitly enabled on the webhook endpoint.
**How to avoid:** Add `customer.subscription.created` to the webhook endpoint's event list in Stripe Dashboard (dev, staging, prod). Document this as a deploy step.
**Warning signs:** Trial creates successfully but webhook processor never logs the event.

### Pitfall 4: checkout.session.completed Upsert Breaking Existing Flow
**What goes wrong:** Changing checkout.session.completed from insert to upsert could inadvertently overwrite fields that should be preserved.
**Why it happens:** A naive upsert replaces all fields. If the local record has data the webhook event doesn't carry, that data is lost.
**How to avoid:** Use `$set` with only the specific fields the event provides. Do not overwrite `userId` or other fields not present in the event payload.
**Warning signs:** Missing `userId` or `stripeLatestCheckoutSessionId` after webhook processing.

### Pitfall 5: Trial Guard Checking Wrong Scope
**What goes wrong:** The "trial already used" guard checks by `businessId` instead of `userId`, allowing the same user to create multiple trials across businesses.
**Why it happens:** The subscription entity is keyed by `userId` (Phase 29 D-13), but a developer might mistakenly scope the guard to the business.
**How to avoid:** Always look up by `userId` from the JWT auth user (D-09). One trial per Firebase identity, regardless of business.
**Warning signs:** Same user has multiple subscription records.

## Code Examples

### Stripe subscriptions.create() for No-Card Trial
```typescript
// Source: https://docs.stripe.com/billing/subscriptions/trials/free-trials
// Verified against Stripe API reference: https://docs.stripe.com/api/subscriptions/create
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

// subscription.status === "trialing"
// subscription.trial_end === Unix timestamp (seconds)
// subscription.current_period_end === Unix timestamp (seconds)
```

### Controller Route Pattern
```typescript
// Source: Project conventions (CLAUDE.md) + Phase 29/30 controller patterns
@Post("trial")
@UseGuards(JwtAuthGuard)
@HttpCode(HttpStatus.CREATED)
async createTrial(@AuthUser() authUser: IUserDto): Promise<IResponse<ISubscriptionResponse>> {
  const subscription = await this.subscriptionTrialCreator.create(authUser);
  return { data: [this.mapToResponse(subscription)] };
}
```

### Repository findByUserId
```typescript
// Source: Existing repository patterns in project
async findByUserId(userId: string): Promise<ISubscriptionDto | null> {
  const entity = await this.model.findOne({ userId }).exec();
  return entity ? this.mapToDto(entity) : null;
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `trial_end` (absolute timestamp) | `trial_period_days` (relative days) | Stable for years | Simpler -- no need to calculate future timestamps |
| No trial_settings parameter | `trial_settings.end_behavior.missing_payment_method` | Stripe API 2022+ | Enables server-side auto-cancel without custom cron |
| Trial Offer API (preview 2026-03-25) | Legacy `trial_period_days` (stable) | Preview only | Do NOT use Trial Offer API -- requires flexible billing mode (preview), not compatible with existing classic billing setup |
| `plan` parameter | `items[].price` parameter | Plans deprecated 2020 | Use Price objects, not Plans |

**Deprecated/outdated:**
- Stripe `plan` parameter: replaced by `items[].price`
- Trial Offer API: preview only (2026-03-25.dahlia), requires flexible billing mode -- do not use for this project

## Open Questions

1. **Stripe Customer reuse when user had a canceled trial**
   - What we know: D-01 says reuse existing Stripe Customer. D-09 says reject if ANY record exists.
   - What's unclear: If a user had a trial (record exists with status=canceled), D-09 rejects them. The Stripe Customer already exists but will never be reused by the trial endpoint. This is correct behavior -- they must use Checkout/Portal to resubscribe (D-10).
   - Recommendation: No action needed. The trial endpoint only serves brand-new users. Existing users with any subscription history go through Checkout.

2. **Whether the trial endpoint needs a request body**
   - What we know: The user's identity comes from the JWT. The price comes from env var. Trial days are hardcoded to 30.
   - What's unclear: Whether any future flexibility is needed.
   - Recommendation: No request body. The endpoint derives everything from the auth user and config. This matches the simplicity principle.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | Existing jest config in trade-flow-api |
| Quick run command | `npx jest --testPathPattern=subscription --no-coverage -t "trial"` |
| Full suite command | `npx jest --no-coverage` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| TRIAL-01 | POST /trial creates Stripe subscription with 30-day trial, writes local record | unit | `npx jest src/subscription/test/services/subscription-trial-creator.service.spec.ts -x` | Wave 0 |
| TRIAL-01 | Rejects if subscription already exists (any status) | unit | Same test file as above | Wave 0 |
| TRIAL-04 | trial_settings.end_behavior.missing_payment_method set to 'cancel' | unit | Assert on stripe.subscriptions.create() call args | Wave 0 |
| D-06 | customer.subscription.created webhook upserts local record | unit | `npx jest src/subscription/test/processors/stripe-webhook.processor.spec.ts -x` | Wave 0 |
| D-07 | checkout.session.completed uses upsert | unit | Same processor test file | Wave 0 |

### Sampling Rate
- **Per task commit:** `npx jest --testPathPattern=subscription --no-coverage`
- **Per wave merge:** `npx jest --no-coverage`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/subscription/test/services/subscription-trial-creator.service.spec.ts` -- covers TRIAL-01, TRIAL-04, D-09
- [ ] Updated processor tests for `customer.subscription.created` and upserted `checkout.session.completed`

## Sources

### Primary (HIGH confidence)
- Stripe API Reference: subscriptions/create -- trial_period_days, trial_settings parameters (https://docs.stripe.com/api/subscriptions/create)
- Stripe Docs: Free trials without payment method (https://docs.stripe.com/billing/subscriptions/trials/free-trials) -- confirmed `missing_payment_method: 'cancel'` behavior
- Project skill: stripe-best-practices/references/billing.md -- confirmed Billing API is correct approach for subscriptions

### Secondary (MEDIUM confidence)
- Stripe Docs: Trial Offer API (preview 2026-03-25) -- confirmed this is preview-only and should NOT be used (https://docs.stripe.com/billing/subscriptions/trials)

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed from Phases 29-30, no new dependencies
- Architecture: HIGH -- follows established NestJS patterns from project conventions, extends existing subscription module
- Pitfalls: HIGH -- well-understood Stripe integration patterns, project has prior art from Phases 29-30
- Stripe API: HIGH -- verified trial_period_days + trial_settings against official API docs

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (30 days -- Stripe API is stable, project patterns are established)
