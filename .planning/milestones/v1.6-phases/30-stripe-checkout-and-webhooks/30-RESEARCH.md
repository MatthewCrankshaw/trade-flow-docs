# Phase 30: Stripe Checkout and Webhooks - Research

**Researched:** 2026-03-29
**Domain:** Stripe webhook event processing, BullMQ async job handling, MongoDB upsert patterns
**Confidence:** HIGH

## Summary

Phase 30 builds the webhook event processing pipeline on top of the Phase 29 foundation. The webhook controller (already created in Phase 29 with signature verification) enqueues validated Stripe events onto a BullMQ `STRIPE_WEBHOOKS` queue. A new `StripeWebhookProcessor` in the worker consumes events and upserts subscription records in MongoDB. Five event types are handled: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, and `invoice.payment_failed`. A `GET /v1/subscription/verify-session` endpoint provides a race-condition safety net for the success page.

A critical finding from this research: **Stripe SDK v21 pins to API version `2025-03-31.basil`**, which introduces breaking changes to subscription and invoice object structures. Specifically: (1) `current_period_end` has moved from the subscription level to `items.data[0].current_period_end`, (2) `invoice.subscription` has moved to `invoice.parent.subscription_details.subscription`, and (3) subscription creation in Checkout is deferred until after payment completion (but `session.subscription` is guaranteed non-null at `checkout.session.completed` time). The implementation must account for these basil-version field paths.

**Primary recommendation:** Follow the existing BullMQ processor pattern (EchoProcessor/WorkerHost) exactly, add `STRIPE_WEBHOOKS` to QUEUE_NAMES, enqueue full Stripe event objects, and use one handler method per event type in the processor. All repository writes must be upserts keyed by `stripeCustomerId` or `stripeSubscriptionId` for idempotency. Access `current_period_end` via `subscription.items.data[0].current_period_end` (basil API path).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Webhook events processed asynchronously via BullMQ. `POST /v1/webhooks/stripe` validates signature, enqueues raw event on `STRIPE_WEBHOOKS` queue, returns 200 immediately. No synchronous processing.
- **D-02:** Add `STRIPE_WEBHOOKS` to `QUEUE_NAMES` constant. New dedicated queue.
- **D-03:** New `StripeWebhookProcessor` in WorkerModule, following EchoProcessor pattern. Handles 5 event types.
- **D-04:** Look up local subscription by `stripeCustomerId` (all 5 events carry `customer` field).
- **D-05:** `checkout.session.completed` writes: `status = trialing`, `stripeSubscriptionId` (from `session.subscription`), `stripeLatestCheckoutSessionId` (from `session.id`).
- **D-06:** `customer.subscription.updated` syncs `status`, `currentPeriodEnd`, `trialEnd`, `cancelAtPeriodEnd`.
- **D-07:** `customer.subscription.deleted` sets `status = canceled`, writes `canceledAt = new Date()`.
- **D-08:** `invoice.payment_succeeded` sets `status = active`.
- **D-09:** `invoice.payment_failed` sets `status = past_due`.
- **D-10:** All writes use upsert by `stripeCustomerId` or `stripeSubscriptionId`. Same event twice produces one record.
- **D-11:** Out-of-order events: if no local record exists, log warning and throw error so BullMQ moves job to failed queue. 200 already returned to Stripe.
- **D-12:** `GET /v1/subscription/verify-session?sessionId=` protected by JwtAuthGuard. Fast path: lookup by `stripeLatestCheckoutSessionId`. Fallback: Stripe API `checkout.sessions.retrieve`.
- **D-13:** `stripeLatestCheckoutSessionId` stored on subscription entity. Overwritten on new checkout (upsert by userId).
- **D-14:** Duplicate checkout guard: check for existing subscription with `status` in `[trialing, active]` before creating Checkout Session. Throw `InvalidRequestError` (422) if found.
- **D-15:** Add `stripeLatestCheckoutSessionId` (string, optional) to subscription entity. Only schema change Phase 30 requires.

### Claude's Discretion
- BullMQ job options for `STRIPE_WEBHOOKS` queue (attempts, backoff strategy)
- Exact payload shape enqueued (full Stripe event object vs subset)
- StripeWebhookProcessor method structure (one method per event type vs switch statement)
- Repository method naming for new lookup patterns

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ACQ-01 | User can initiate a 30-day free trial by completing Stripe Checkout | D-14 duplicate guard + D-05 checkout.session.completed handler creates record with status=trialing |
| ACQ-03 | User redirected to `/subscribe/success` after completing Stripe Checkout | Phase 29 configured success_url; Phase 30 handles the webhook that fires after redirect |
| ACQ-04 | User redirected to `/subscribe/cancel` if they abandon Checkout | Phase 29 configured cancel_url; no Phase 30 handler needed (no webhook fires on abandon) |
| ACQ-05 | `/subscribe/success` page polls for subscription status and verifies session server-side | D-12 verify-session endpoint: fast path via stripeLatestCheckoutSessionId, fallback via Stripe API |
| WBHK-02 | `checkout.session.completed` creates local subscription with status `trialing` | D-05: extract stripeSubscriptionId from session.subscription, stripeLatestCheckoutSessionId from session.id |
| WBHK-03 | `customer.subscription.updated` syncs status, currentPeriodEnd, trialEnd, cancelAtPeriodEnd | D-06: map Stripe subscription fields to local record; currentPeriodEnd from items.data[0].current_period_end (basil API) |
| WBHK-04 | `customer.subscription.deleted` sets status=canceled and records canceledAt | D-07: simple status + timestamp update |
| WBHK-05 | `invoice.payment_succeeded` sets status=active | D-08: extract customer from invoice.customer; update status |
| WBHK-06 | `invoice.payment_failed` sets status=past_due | D-09: extract customer from invoice.customer; update status |
| WBHK-07 | Webhook handler is idempotent -- same event twice produces no duplicates | D-10: upsert by stripeCustomerId/stripeSubscriptionId; MongoDB findOneAndUpdate with upsert:true |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| stripe | 21.x | Stripe SDK -- event type definitions, checkout.sessions.retrieve for verify-session fallback | Installed in Phase 29; pins to API version 2025-03-31.basil |
| @nestjs/bullmq | 11.0.4 | @Processor decorator, WorkerHost class, BullModule.registerQueue | Installed in Phase 20 |
| bullmq | 5.71.0 | Queue/Worker/Job types, job options configuration | Installed in Phase 20 |
| mongoose | 9.x | MongoDB ODM for subscription repository upserts | Already installed |

### Supporting (already installed)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @nestjs/common | 11.1.x | @Query() decorator for verify-session querystring | Controller parameter decoration |
| @nestjs/config | 4.0.x | ConfigService for STRIPE_WEBHOOK_SECRET | Webhook processor reads secret for Stripe API calls |

### No New Dependencies
Phase 30 requires zero new npm installations. All dependencies installed in Phases 20 and 29.

## Architecture Patterns

### Recommended File Structure
```
src/subscription/
  controllers/
    subscription.controller.ts    # (Phase 29 -- add verify-session endpoint)
    webhook.controller.ts         # (Phase 29 -- add queue enqueue logic)
  services/
    subscription-creator.service.ts   # (Phase 29 -- add duplicate checkout guard)
    subscription-retriever.service.ts # NEW: verify-session logic
  repositories/
    subscription.repository.ts    # (Phase 29 -- add new lookup/upsert methods)
  entities/
    subscription.entity.ts        # (Phase 29 -- add stripeLatestCheckoutSessionId)

src/worker/
  processors/
    stripe-webhook.processor.ts   # NEW: StripeWebhookProcessor

src/queue/
  queue.constant.ts               # Add STRIPE_WEBHOOKS to QUEUE_NAMES
  queue.module.ts                 # Register STRIPE_WEBHOOKS queue
  services/
    queue-producer.service.ts     # Add enqueueStripeWebhook method
```

### Pattern 1: Webhook Controller Enqueue (Controller Side)
**What:** Validate signature (already done in Phase 29), then enqueue the full Stripe event onto BullMQ queue.
**When to use:** All webhook events follow this path.
**Example:**
```typescript
// In webhook.controller.ts (extending Phase 29 code)
@Post("stripe")
@Public()
async handleStripeWebhook(
  @Req() req: RawBodyRequest<Request>,
  @Headers("stripe-signature") signature: string,
): Promise<void> {
  const event = this.stripe.webhooks.constructEvent(
    req.rawBody,
    signature,
    this.webhookSecret,
  );
  await this.queueProducer.enqueueStripeWebhook(event);
  // Returns 200 implicitly (void return with no exception)
}
```

### Pattern 2: BullMQ Processor (Worker Side)
**What:** A `@Processor` class extending `WorkerHost` that handles each event type.
**When to use:** Worker-side processing of Stripe webhook events.
**Example:**
```typescript
// Source: Existing EchoProcessor pattern in trade-flow-api
import { Processor, WorkerHost } from "@nestjs/bullmq";
import { Job } from "bullmq";
import Stripe from "stripe";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

@Processor(QUEUE_NAMES.STRIPE_WEBHOOKS)
export class StripeWebhookProcessor extends WorkerHost {
  private readonly logger = new AppLogger(StripeWebhookProcessor.name);

  constructor(
    private readonly subscriptionRepository: SubscriptionRepository,
  ) {
    super();
  }

  async process(job: Job<Stripe.Event>): Promise<void> {
    const event = job.data;
    this.logger.log(`Processing Stripe event: ${event.type}`, { eventId: event.id });

    switch (event.type) {
      case "checkout.session.completed":
        await this.handleCheckoutSessionCompleted(event.data.object as Stripe.Checkout.Session);
        break;
      case "customer.subscription.updated":
        await this.handleSubscriptionUpdated(event.data.object as Stripe.Subscription);
        break;
      case "customer.subscription.deleted":
        await this.handleSubscriptionDeleted(event.data.object as Stripe.Subscription);
        break;
      case "invoice.payment_succeeded":
        await this.handleInvoicePaymentSucceeded(event.data.object as Stripe.Invoice);
        break;
      case "invoice.payment_failed":
        await this.handleInvoicePaymentFailed(event.data.object as Stripe.Invoice);
        break;
      default:
        this.logger.warn(`Unhandled Stripe event type: ${event.type}`);
    }
  }
}
```

### Pattern 3: QueueProducer Extension
**What:** Add a typed method to the existing QueueProducer for Stripe webhook events.
**When to use:** Webhook controller calls this to enqueue events.
**Example:**
```typescript
// In queue-producer.service.ts (extending existing QueueProducer)
@InjectQueue(QUEUE_NAMES.STRIPE_WEBHOOKS)
private readonly stripeWebhooksQueue: Queue,

public async enqueueStripeWebhook(event: Stripe.Event): Promise<void> {
  await this.stripeWebhooksQueue.add(event.type, event, {
    jobId: event.id, // Stripe event ID as job ID for deduplication
  });
}
```

### Pattern 4: MongoDB Upsert for Idempotency
**What:** Use `findOneAndUpdate` with `upsert: true` to create-or-update subscription records.
**When to use:** All webhook event handlers that write to the subscription collection.
**Example:**
```typescript
// In subscription.repository.ts
async upsertByStripeCustomerId(
  stripeCustomerId: string,
  updates: Partial<ISubscriptionDto>,
): Promise<ISubscriptionDto> {
  const entity = await this.collection.findOneAndUpdate(
    { stripeCustomerId },
    { $set: { ...updates, updatedAt: new Date() }, $setOnInsert: { createdAt: new Date() } },
    { upsert: true, returnDocument: "after" },
  );
  return this.mapToDto(entity);
}
```

### Pattern 5: Verify Session Endpoint
**What:** Protected endpoint that checks subscription status by checkout session ID.
**When to use:** Success page calls this after redirect from Stripe Checkout.
**Example:**
```typescript
// In subscription.controller.ts
@Get("verify-session")
@UseGuards(JwtAuthGuard)
async verifySession(
  @Query("sessionId") sessionId: string,
  @Req() req: IAuthenticatedRequest,
): Promise<IVerifySessionResponse> {
  return this.subscriptionRetriever.verifySession(sessionId, req.user);
}
```

### Anti-Patterns to Avoid
- **Processing webhook events synchronously in the controller:** Always enqueue and return 200 immediately. Synchronous processing risks Stripe timeouts (20s) and retries.
- **Using `insertOne` instead of upsert:** Creates duplicates when the same event is replayed. Always use `findOneAndUpdate` with `upsert: true`.
- **Reading `subscription.current_period_end` at the top level:** In Stripe API v2025-03-31.basil, this field no longer exists on the subscription object. Must use `subscription.items.data[0].current_period_end`.
- **Reading `invoice.subscription` directly:** In basil API, use `invoice.parent.subscription_details.subscription` for the subscription ID, or use `invoice.customer` directly (which remains a top-level field).
- **Forgetting to register STRIPE_WEBHOOKS queue in QueueModule:** The queue must be registered via `BullModule.registerQueue({ name: QUEUE_NAMES.STRIPE_WEBHOOKS })` in QueueModule, otherwise BullMQ throws "Missing queue" errors.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Event deduplication | Custom dedup logic with Redis sets | BullMQ `jobId: event.id` + upsert pattern | BullMQ prevents duplicate job IDs; upsert prevents duplicate DB records |
| Webhook retry handling | Custom retry loop with exponential backoff | BullMQ built-in `attempts` + `backoff` options | BullMQ handles retry scheduling, dead-letter queue, and job lifecycle |
| Event type routing | Custom event registry / plugin system | Simple switch statement in processor | Only 5 event types; a switch is clear and maintainable |
| Stripe API type casting | Manual interface definitions for Stripe objects | `Stripe.Checkout.Session`, `Stripe.Subscription`, `Stripe.Invoice` from stripe SDK | Full TypeScript types maintained by Stripe |

**Key insight:** BullMQ's `jobId` parameter provides first-layer deduplication (same event ID cannot be enqueued twice), and MongoDB upsert provides second-layer idempotency (same data written to same record). Together they make the pipeline fully idempotent.

## Common Pitfalls

### Pitfall 1: Accessing current_period_end at Subscription Level (Basil API Breaking Change)
**What goes wrong:** `subscription.current_period_end` is `undefined`, causing null/undefined to be written to the local record.
**Why it happens:** Stripe API v2025-03-31.basil (pinned by stripe SDK v21) moved `current_period_end` from the subscription level to the subscription item level.
**How to avoid:** Access via `subscription.items.data[0].current_period_end`. Since Trade Flow uses a single-item subscription (one price), `items.data[0]` is always the correct item.
**Warning signs:** `currentPeriodEnd` is null/undefined in the local subscription record after a `customer.subscription.updated` event.

### Pitfall 2: Accessing invoice.subscription Directly (Basil API Breaking Change)
**What goes wrong:** `invoice.subscription` is `undefined` when trying to look up the local record.
**Why it happens:** Stripe API v2025-03-31.basil moved `invoice.subscription` to `invoice.parent.subscription_details.subscription`.
**How to avoid:** For invoice events (payment_succeeded, payment_failed), use `invoice.customer` (still top-level) to look up the local record by `stripeCustomerId` rather than by subscription ID. This is simpler and aligns with D-04.
**Warning signs:** "Subscription not found" errors when processing invoice events.

### Pitfall 3: Not Registering STRIPE_WEBHOOKS Queue in QueueModule
**What goes wrong:** BullMQ throws "Missing queue STRIPE_WEBHOOKS" at injection time, crashing both API (QueueProducer injection) and worker (processor registration).
**Why it happens:** Adding the queue name to QUEUE_NAMES constant but forgetting to register it via `BullModule.registerQueue()`.
**How to avoid:** When adding to QUEUE_NAMES, immediately add the corresponding `BullModule.registerQueue({ name: QUEUE_NAMES.STRIPE_WEBHOOKS })` in QueueModule imports array.
**Warning signs:** NestJS dependency injection error on startup mentioning "STRIPE_WEBHOOKS".

### Pitfall 4: StripeWebhookProcessor Not Registered in WorkerModule
**What goes wrong:** Events sit in the queue forever, never processed. No error in worker logs.
**Why it happens:** Processor class created but not added to WorkerModule's `providers` array.
**How to avoid:** After creating the processor file, immediately add it to `WorkerModule`'s providers. Also import SubscriptionModule (or at minimum SubscriptionRepository) so the processor can inject repository dependencies.
**Warning signs:** Queue size grows but no processing logs appear in worker.

### Pitfall 5: BullMQ Job Serialization of Stripe Event Object
**What goes wrong:** Stripe Event object properties are lost after JSON serialization/deserialization through BullMQ.
**Why it happens:** BullMQ serializes job data to JSON when storing in Redis. Stripe Event objects are plain objects, so this works correctly. However, any Date objects or class instances would be lost.
**How to avoid:** Enqueue the full Stripe event object (it's already plain JSON). In the processor, cast `job.data` to `Stripe.Event` for type safety. Stripe timestamps are Unix epoch numbers (not Date objects), so they survive serialization.
**Warning signs:** Missing properties or `[object Object]` in logged event data.

### Pitfall 6: Worker Missing SubscriptionModule Dependencies
**What goes wrong:** `StripeWebhookProcessor` cannot inject `SubscriptionRepository` because WorkerModule does not import SubscriptionModule.
**Why it happens:** WorkerModule was designed for Phase 22 with only EchoProcessor. Phase 30 adds a processor that depends on a feature module.
**How to avoid:** Import SubscriptionModule into WorkerModule. Ensure SubscriptionModule exports SubscriptionRepository (or the services the processor needs).
**Warning signs:** NestJS dependency injection error in worker startup: "Nest can't resolve dependencies of StripeWebhookProcessor."

### Pitfall 7: Returning Non-200 from Webhook Controller
**What goes wrong:** Stripe retries the webhook repeatedly (up to 3 days), causing duplicate processing.
**Why it happens:** An unhandled exception in the controller (before or during queue enqueue) returns 500, which Stripe interprets as a failure.
**How to avoid:** The controller only does two things: (1) verify signature (throw on failure = correct 400), (2) enqueue (should rarely fail). Wrap the enqueue call in try-catch and still return 200 even if enqueue fails, logging the error for investigation.
**Warning signs:** Stripe Dashboard shows repeated webhook delivery attempts with non-200 responses.

## Code Examples

### QUEUE_NAMES Update
```typescript
// In src/queue/queue.constant.ts
export const QUEUE_NAMES = {
  ECHO: "echo",
  STRIPE_WEBHOOKS: "stripe-webhooks",
} as const;
```

### QueueModule Queue Registration
```typescript
// In src/queue/queue.module.ts -- add to imports array
BullModule.registerQueue({ name: QUEUE_NAMES.STRIPE_WEBHOOKS }),
```

### QueueProducer Stripe Webhook Method
```typescript
// In src/queue/services/queue-producer.service.ts
import Stripe from "stripe";

@InjectQueue(QUEUE_NAMES.STRIPE_WEBHOOKS)
private readonly stripeWebhooksQueue: Queue,

public async enqueueStripeWebhook(event: Stripe.Event): Promise<void> {
  await this.stripeWebhooksQueue.add(event.type, event, {
    jobId: event.id,
    removeOnComplete: { age: 3600, count: 1000 },
    removeOnFail: { age: 86400, count: 5000 },
  });
}
```

### Basil API Field Access for Subscription Updated Handler
```typescript
// CRITICAL: Basil API (2025-03-31.basil) field paths
private async handleSubscriptionUpdated(subscription: Stripe.Subscription): Promise<void> {
  const customerId = typeof subscription.customer === "string"
    ? subscription.customer
    : subscription.customer.id;

  // current_period_end is now at item level, NOT subscription level
  const currentPeriodEnd = subscription.items.data[0]?.current_period_end
    ? new Date(subscription.items.data[0].current_period_end * 1000)
    : undefined;

  const trialEnd = subscription.trial_end
    ? new Date(subscription.trial_end * 1000)
    : undefined;

  await this.subscriptionRepository.upsertByStripeCustomerId(customerId, {
    status: subscription.status,
    currentPeriodEnd,
    trialEnd,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
  });
}
```

### Basil API Field Access for Invoice Events
```typescript
// Invoice events: use invoice.customer (still top-level in basil)
private async handleInvoicePaymentSucceeded(invoice: Stripe.Invoice): Promise<void> {
  const customerId = typeof invoice.customer === "string"
    ? invoice.customer
    : invoice.customer?.id;

  if (!customerId) {
    this.logger.warn("Invoice payment succeeded but no customer ID found", { invoiceId: invoice.id });
    return;
  }

  const existing = await this.subscriptionRepository.findByStripeCustomerId(customerId);
  if (!existing) {
    throw new Error(`No subscription found for stripeCustomerId: ${customerId}`);
  }

  await this.subscriptionRepository.upsertByStripeSubscriptionId(
    existing.stripeSubscriptionId,
    { status: SubscriptionStatus.ACTIVE },
  );
}
```

### Checkout Session Completed Handler
```typescript
private async handleCheckoutSessionCompleted(session: Stripe.Checkout.Session): Promise<void> {
  const customerId = typeof session.customer === "string"
    ? session.customer
    : session.customer?.id;

  const subscriptionId = typeof session.subscription === "string"
    ? session.subscription
    : (session.subscription as Stripe.Subscription)?.id;

  await this.subscriptionRepository.upsertByStripeCustomerId(customerId, {
    stripeSubscriptionId: subscriptionId,
    stripeLatestCheckoutSessionId: session.id,
    status: SubscriptionStatus.TRIALING,
  });
}
```

### Verify Session Service Logic
```typescript
// In subscription-retriever.service.ts
async verifySession(sessionId: string, authUser: IUserDto): Promise<IVerifySessionResponse> {
  // Fast path: check local record
  const subscription = await this.subscriptionRepository.findByCheckoutSessionId(sessionId);
  if (subscription && (subscription.status === SubscriptionStatus.TRIALING || subscription.status === SubscriptionStatus.ACTIVE)) {
    return { status: subscription.status, verified: true };
  }

  // Fallback: query Stripe API directly
  const session = await this.stripe.checkout.sessions.retrieve(sessionId);
  return { status: session.payment_status, verified: true };
}
```

### BullMQ Job Options Recommendation
```typescript
// Recommended defaults for STRIPE_WEBHOOKS queue jobs
{
  jobId: event.id,                            // Deduplication by Stripe event ID
  removeOnComplete: { age: 3600, count: 1000 }, // Keep completed jobs for 1 hour or last 1000
  removeOnFail: { age: 86400, count: 5000 },    // Keep failed jobs for 24 hours or last 5000
  attempts: 3,                                  // Retry up to 3 times on failure
  backoff: { type: "exponential", delay: 5000 }, // 5s, 10s, 20s between retries
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `subscription.current_period_end` | `subscription.items.data[0].current_period_end` | Stripe API 2025-03-31.basil | Must access billing period at item level |
| `invoice.subscription` (top-level) | `invoice.parent.subscription_details.subscription` | Stripe API 2025-03-31.basil | Invoice events need different field path for subscription ID; using invoice.customer is simpler |
| Subscription created before payment | Subscription created after payment completion | Stripe API 2025-03-31.basil | checkout.session.completed guaranteed to have subscription; no incomplete subscriptions from failed payments |
| `@Process()` decorator on methods | `process()` method override on WorkerHost | @nestjs/bullmq 11.x | Single process() method, use switch/if for different job names |

**Deprecated/outdated:**
- `subscription.current_period_start` / `subscription.current_period_end`: Removed in basil; use `items.data[].current_period_start/end`
- `invoice.subscription`: Removed in basil; use `invoice.parent.subscription_details.subscription` or `invoice.customer`
- `invoice.subscription_details`: Removed in basil; use `invoice.parent.subscription_details`

## Open Questions

1. **SubscriptionModule export scope for WorkerModule**
   - What we know: WorkerModule currently imports only CoreModule, ConfigModule, LoggerModule, QueueModule
   - What's unclear: Whether SubscriptionModule should be imported wholesale into WorkerModule, or whether SubscriptionRepository should be extracted and provided directly
   - Recommendation: Import SubscriptionModule into WorkerModule and ensure SubscriptionModule exports the repository. This follows NestJS conventions and avoids provider duplication.

2. **Stripe Event types available on `items.data` in webhook payloads**
   - What we know: `customer.subscription.updated` includes `items.data` with `current_period_end` on each item
   - What's unclear: Whether `items.data` is always expanded in webhook payloads or needs explicit expansion
   - Recommendation: Stripe webhook payloads include the full subscription object with items expanded by default. Verify during testing with Stripe CLI (`stripe trigger customer.subscription.updated`). HIGH confidence this works without expansion.

3. **checkout.session.completed and session.subscription availability in basil**
   - What we know: Basil defers subscription creation until payment completion; `checkout.session.completed` fires after payment
   - What's unclear: Whether `session.subscription` is guaranteed to be a string (not null) when session mode is "subscription"
   - Recommendation: Stripe documentation confirms "the associated invoice and invoice payment are non-null" at checkout.session.completed time, strongly implying subscription is also non-null. Add a defensive null check and log a warning if null. HIGH confidence it will be populated.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest 29.3.1 |
| Config file | jest config in package.json (trade-flow-api) |
| Quick run command | `npm test -- --testPathPattern=stripe-webhook` |
| Full suite command | `npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| WBHK-02 | checkout.session.completed creates record with status=trialing | unit (processor + repo mocked) | `npm test -- --testPathPattern=stripe-webhook.processor` | No -- Wave 0 |
| WBHK-03 | customer.subscription.updated syncs fields | unit (processor + repo mocked) | `npm test -- --testPathPattern=stripe-webhook.processor` | No -- Wave 0 |
| WBHK-04 | customer.subscription.deleted sets canceled | unit (processor + repo mocked) | `npm test -- --testPathPattern=stripe-webhook.processor` | No -- Wave 0 |
| WBHK-05 | invoice.payment_succeeded sets active | unit (processor + repo mocked) | `npm test -- --testPathPattern=stripe-webhook.processor` | No -- Wave 0 |
| WBHK-06 | invoice.payment_failed sets past_due | unit (processor + repo mocked) | `npm test -- --testPathPattern=stripe-webhook.processor` | No -- Wave 0 |
| WBHK-07 | Idempotent -- same event twice, one record | unit (repo upsert test) | `npm test -- --testPathPattern=subscription.repository` | No -- Wave 0 |
| ACQ-05 | verify-session returns subscription status | unit (retriever service) | `npm test -- --testPathPattern=subscription-retriever` | No -- Wave 0 |
| ACQ-01 | Duplicate checkout guard | unit (creator service) | `npm test -- --testPathPattern=subscription-creator` | No -- Wave 0 |

### Sampling Rate
- **Per task commit:** `npm test -- --testPathPattern=subscription --no-coverage`
- **Per wave merge:** `npm test -- --no-coverage`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/worker/test/processors/stripe-webhook.processor.spec.ts` -- covers WBHK-02 through WBHK-07
- [ ] `src/subscription/test/services/subscription-retriever.service.spec.ts` -- covers ACQ-05
- [ ] `src/subscription/test/services/subscription-creator.service.spec.ts` -- covers ACQ-01 (duplicate guard)

Note: Formal unit tests for these services are deferred to Phase 31 (TEST-01 through TEST-04). Phase 30 may include basic happy-path tests at Claude's discretion.

## Sources

### Primary (HIGH confidence)
- [Stripe Subscription Object (basil API)](https://docs.stripe.com/api/subscriptions/object?api-version=2025-03-31.basil) -- Confirmed current_period_end moved to items level, trial_end/cancel_at_period_end/canceled_at/status remain top-level
- [Stripe Basil Changelog -- Subscription Item Billing Periods](https://docs.stripe.com/changelog/basil/2025-03-31/deprecate-subscription-current-period-start-and-end) -- Breaking change: subscription-level periods removed
- [Stripe Basil Changelog -- Invoice Parent Field](https://docs.stripe.com/changelog/basil/2025-03-31/adds-new-parent-field-to-invoicing-objects) -- invoice.subscription moved to invoice.parent.subscription_details.subscription
- [Stripe Basil Changelog -- Checkout Session Latency](https://docs.stripe.com/changelog/basil/2025-03-31/checkout-legacy-subscription-upgrade) -- Subscription creation deferred until payment completion
- [Stripe Event Types Reference](https://docs.stripe.com/api/events/types) -- Verified all 5 event types and their data.object types
- [Stripe Using Webhooks with Subscriptions](https://docs.stripe.com/billing/subscriptions/webhooks) -- Event flow for subscription lifecycle
- Phase 22 Research + Plan 01 (trade-flow-api) -- EchoProcessor pattern, WorkerHost, @Processor decorator
- Phase 21 Research (trade-flow-api) -- QueueProducer pattern, QUEUE_NAMES constant, BullModule.registerQueue

### Secondary (MEDIUM confidence)
- [BullMQ Retrying Failing Jobs](https://docs.bullmq.io/guide/retrying-failing-jobs) -- attempts, backoff configuration
- [BullMQ Auto-removal of Jobs](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs) -- removeOnComplete, removeOnFail options
- [Stripe Node.js SDK Releases](https://github.com/stripe/stripe-node/releases) -- v21 pins to 2025-03-31.basil

### Tertiary (LOW confidence)
- None -- all findings verified against official docs

## Project Constraints (from CLAUDE.md)

- **Tech stack:** NestJS (API) patterns -- Controller-Service-Repository layering
- **Repo:** Phase 30 is API-only (trade-flow-api) + worker
- **Conventions:** File naming (`kebab-case.type.ts`), class naming (`PascalCase`), service naming (single-responsibility: Creator/Retriever/Updater)
- **Error handling:** Custom error classes with `getCode()`, `getMessage()`, `getDetails()`; `InvalidRequestError` for 422
- **Logging:** `AppLogger` with class name context -- `new AppLogger(StripeWebhookProcessor.name)`
- **Code style:** Double quotes, semicolons, 125 char width, trailing commas, 2-space indent
- **Path aliases:** `@subscription/*`, `@queue/*`, `@worker/*`, `@core/*` for cross-module imports
- **Worker pattern:** Processors in `src/worker/processors/`, extend `WorkerHost`, register in `WorkerModule` providers
- **Queue pattern:** `QUEUE_NAMES` constant, `QueueProducer` typed methods, `BullModule.registerQueue()` in QueueModule

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies; all patterns established in prior phases
- Architecture: HIGH -- follows EchoProcessor/WorkerHost pattern exactly; webhook controller extends Phase 29 code
- Pitfalls: HIGH -- basil API breaking changes verified against official Stripe changelog; BullMQ patterns verified against docs
- Stripe basil field paths: HIGH -- verified against official Stripe API reference for 2025-03-31.basil version

**Research date:** 2026-03-29
**Valid until:** 2026-04-28 (Stripe basil API stable; BullMQ patterns stable)
