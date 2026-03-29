# Phase 30: Stripe Checkout and Webhooks - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Add webhook event handlers (5 types) and the verify-session endpoint on top of the Phase 29 foundation. The webhook route ingests Stripe events, validates signatures, and enqueues them on a BullMQ queue for asynchronous processing. A dedicated `StripeWebhookProcessor` in the worker handles each event type and upserts the local subscription record. A `GET /v1/subscription/verify-session?sessionId=` endpoint lets the success page confirm checkout completion without waiting for the webhook. This phase is backend-only (`trade-flow-api`).

</domain>

<decisions>
## Implementation Decisions

### Webhook Ingestion Architecture
- **D-01:** Webhook events are processed asynchronously via BullMQ. `POST /v1/webhooks/stripe` validates the Stripe signature and payload structure, enqueues the raw Stripe event on the `STRIPE_WEBHOOKS` queue, and immediately returns 200. No synchronous event processing in the controller.
- **D-02:** Add `STRIPE_WEBHOOKS` to the `QUEUE_NAMES` constant (alongside the existing echo queue). New dedicated queue — do not reuse the general-purpose queue.
- **D-03:** A new `StripeWebhookProcessor` is added to the worker (`WorkerModule`), following the existing `EchoProcessor` pattern. It handles the five event types: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`.

### Webhook Reconciliation — How Events Map to Local Records
- **D-04:** Look up the local subscription record by `stripeCustomerId`. All five Stripe event types carry the `customer` field. Phase 29 stores `stripeCustomerId` at checkout creation time, making this lookup deterministic.
- **D-05:** `checkout.session.completed` handler writes: `status = trialing`, `stripeSubscriptionId` (from `session.subscription`), and `stripeLatestCheckoutSessionId` (from `session.id`). This is the initial population of the subscription record's Stripe IDs.
- **D-06:** `customer.subscription.updated` syncs `status`, `currentPeriodEnd`, `trialEnd`, and `cancelAtPeriodEnd` from the subscription object.
- **D-07:** `customer.subscription.deleted` sets `status = canceled` and writes `canceledAt = new Date()`.
- **D-08:** `invoice.payment_succeeded` sets `status = active`.
- **D-09:** `invoice.payment_failed` sets `status = past_due`.

### Upsert Idempotency
- **D-10:** All repository writes use upsert by `stripeCustomerId` (for `checkout.session.completed`) or `stripeSubscriptionId` (for subscription/invoice events). Processing the same event twice produces one record, not two.

### Out-of-Order Event Handling
- **D-11:** If `StripeWebhookProcessor` processes an event and no local subscription record exists (e.g., a subscription event fires before checkout.session.completed), it logs a warning (event type + stripeCustomerId) and throws an error so BullMQ moves the job to the failed queue. The 200 was already returned to Stripe — no retry risk.

### verify-session Endpoint
- **D-12:** `GET /v1/subscription/verify-session?sessionId=` is protected by JwtAuthGuard. Fast path: look up local subscription by `stripeLatestCheckoutSessionId`. If found and status is `trialing` or `active`, return the subscription status. If not found (webhook hasn't fired yet), fall back to retrieving the Checkout Session from the Stripe API and return its `payment_status`.
- **D-13:** `stripeLatestCheckoutSessionId` is stored on the subscription entity. When a new checkout is initiated for the same user, this field is overwritten with the latest session ID (upsert by `userId` in the checkout handler).

### Duplicate Checkout Guard
- **D-14:** Before creating a Stripe Checkout Session, the checkout service checks for an existing subscription record with `status` in `[trialing, active]`. If found, throw `InvalidRequestError` (HTTP 422) with a clear message. Prevents duplicate Stripe customers and conflicting subscription states.

### Entity Schema Addition
- **D-15:** Add `stripeLatestCheckoutSessionId` (string, optional) to the subscription entity defined in Phase 29. This is the only schema change Phase 30 requires.

### Claude's Discretion
- BullMQ job options for `STRIPE_WEBHOOKS` queue (attempts, backoff strategy)
- Exact payload shape enqueued (full Stripe event object vs subset)
- StripeWebhookProcessor method structure (one method per event type vs switch statement)
- Repository method naming for the new lookup patterns (`findByStripeCustomerId`, `findByStripeSubscriptionId`)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.6 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.6 requirements; Phase 30 covers ACQ-01, ACQ-03, ACQ-04, ACQ-05, WBHK-02 through WBHK-07

### Roadmap
- `.planning/ROADMAP.md` §Phase 30 — Phase goal, success criteria, dependency on Phase 29

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/phases/29-subscription-module-foundation/29-CONTEXT.md` — Stripe customer creation, entity schema, module structure, env vars, all decisions this phase builds on

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — NestJS naming patterns, error handling, logging
- `.planning/codebase/ARCHITECTURE.md` — Module structure, controller→service→repo layering

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `QueueProducer` + `QUEUE_NAMES`: existing BullMQ producer from Phase 1.4 — add `STRIPE_WEBHOOKS` entry and inject into the webhook controller
- `EchoProcessor`: existing BullMQ processor pattern — `StripeWebhookProcessor` follows the same class structure
- `WorkerModule`: already exists — `StripeWebhookProcessor` registers here
- `@Public()` decorator: Phase 29 applied it to `POST /v1/webhooks/stripe`; Phase 30 inherits this
- `AppLogger`, `ConfigService`, `InvalidRequestError`: existing cross-cutting utilities

### Established Patterns
- Processor pattern: `@Processor(QUEUE_NAMES.X)` class with `@Process()` decorated handler method(s)
- Upsert pattern: follow existing repository upsert conventions (`findOneAndUpdate` with `upsert: true`)
- Factory provider: Stripe SDK injected via `STRIPE_CLIENT` token (established in Phase 29)

### Integration Points
- `src/subscription/` — Phase 30 adds handlers inside existing SubscriptionModule
- `src/worker.ts` / `WorkerModule` — register `StripeWebhookProcessor`
- `QUEUE_NAMES` constant — add `STRIPE_WEBHOOKS`
- Entity defined in Phase 29 — add `stripeLatestCheckoutSessionId` field only

</code_context>

<specifics>
## Specific Ideas

- Field naming: `stripeLatestCheckoutSessionId` (user-specified — not `stripeCheckoutSessionId`)
- Queue pattern: validate + enqueue in controller, all processing in worker processor — never process synchronously in the webhook controller
- Out-of-order handling: failed queue is the correct destination (not silent discard) so events are inspectable

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 30-stripe-checkout-and-webhooks*
*Context gathered: 2026-03-29*
