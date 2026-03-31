# Phase 29: Subscription Module Foundation - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire up everything the API needs before Stripe can be used: raw body preservation for webhook signature verification, Stripe SDK as an injectable provider, the full subscription MongoDB entity/repo/enum, and the two endpoints that prove the wiring works — `POST /v1/subscription/checkout` (returns a Checkout Session URL) and `POST /v1/webhooks/stripe` (rejects invalid signatures with 400). Phase 30 adds webhook event handlers on top of this foundation.

</domain>

<decisions>
## Implementation Decisions

### Stripe Customer Creation
- **D-01:** Create a Stripe Customer explicitly before creating the Checkout Session. Call `stripe.customers.create({ email, metadata: { userId } })` at the start of the checkout handler, store `stripeCustomerId` on the local subscription record.
- **D-02:** User email comes from the Firebase JWT (`request.user.email` via JwtAuthGuard). No extra MongoDB lookup needed — Firebase auth always has an email on the decoded token.
- **D-03:** Pass `customer: stripeCustomerId` to the Checkout Session (not `customer_email`). This creates an explicit userId ↔ Stripe Customer link from the start and makes Phase 30 webhook reconciliation deterministic.

### Checkout Redirect URLs
- **D-04:** Read frontend base URL from a `FRONTEND_URL` env var (added to `.env.example`). Known values:
  - Dev: `http://localhost:5173`
  - Staging: `https://stage.tradeflowhub.uk`
  - Production: `https://tradeflowhub.uk`
- **D-05:** `success_url` format: `${FRONTEND_URL}/subscribe/success?session_id={CHECKOUT_SESSION_ID}` — the `{CHECKOUT_SESSION_ID}` template literal is a Stripe-side substitution that embeds the session ID, required by ACQ-05's server-side verification in Phase 30.
- **D-06:** `cancel_url` format: `${FRONTEND_URL}/subscribe/cancel`

### Subscription Entity — Full Schema Upfront
- **D-07:** Define the complete subscription entity now with all fields Phase 30–33 will need. Phase 29 populates only what it touches (userId, stripeCustomerId, status); Phase 30 fills the rest via webhooks. No schema migration needed later.
- **D-08:** Full entity fields: `userId` (string), `stripeCustomerId` (string), `stripeSubscriptionId` (string, optional — populated in Phase 30 after webhook), `status` (SubscriptionStatus enum), `currentPeriodEnd` (Date, optional), `trialEnd` (Date, optional), `cancelAtPeriodEnd` (boolean, default false), `canceledAt` (Date, optional), `createdAt` (Date), `updatedAt` (Date).
- **D-09:** `SubscriptionStatus` enum mirrors Stripe's subscription status values exactly: `trialing`, `active`, `past_due`, `canceled`, `incomplete`. Same strings as Stripe's `subscription.status` field — webhook handlers map directly with no translation layer.

### Webhook Route Access Control
- **D-10:** `POST /v1/webhooks/stripe` must be public — no JwtAuthGuard. Use the same `@Public()` decorator pattern as the public quote endpoint.
- **D-11:** `rawBody: true` in `NestFactory.create(AppModule, { rawBody: true })` — global, applied to all routes. Required before signature verification can work. This is a blocking first task.

### Module Structure
- **D-12:** SubscriptionModule is a standard NestJS feature module registered in AppModule. Stripe SDK injected as a factory provider with a `STRIPE_CLIENT` injection token inside SubscriptionModule (not a separate StripeModule — keep scope minimal for Phase 29).
- **D-13:** Subscription keyed by `userId` (one subscription per user, consistent with Firebase auth identity).

### Claude's Discretion
- Stripe provider factory implementation details (how to read STRIPE_SECRET_KEY via ConfigService)
- Repository method naming and upsert patterns (follow existing repository conventions)
- DTO/response shape for the checkout endpoint response (just `{ url: string }` is sufficient)
- Unit test scope for Phase 29 (success criteria don't require tests; tests are Phase 31)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.6 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.6 requirements; Phase 29 covers ACQ-02 and WBHK-01

### Roadmap
- `.planning/ROADMAP.md` §Phase 29 — Phase goal, success criteria, and dependency chain

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — NestJS module/service/repo naming, file patterns, error handling
- `.planning/codebase/ARCHITECTURE.md` — Module structure, controller→service→repo layering, CoreModule globals

### Prior Phase Context
- `.planning/phases/25-api-seeding-onboarding-tests/25-CONTEXT.md` — Most recent context; establishes patterns used in v1.5

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `JwtAuthGuard` + `@Public()` decorator: existing pattern for marking routes public (used on public quote endpoint) — webhook route uses the same decorator
- `AppLogger`: inject as `new AppLogger(SubscriptionCreator.name)` per existing service convention
- `ConfigService` from `@nestjs/config`: existing pattern for reading env vars in providers and services
- `ResourceNotFoundError`, `InvalidRequestError`: existing error classes for consistent HTTP error mapping

### Established Patterns
- Factory provider pattern: existing modules use `{ provide: TOKEN, useFactory: (config) => new Client(config.get('KEY')), inject: [ConfigService] }` for external SDKs (see EmailModule / SendGrid)
- Entity shape: `I{Name}Entity extends { _id: ObjectId, createdAt: Date, updatedAt: Date }` — subscription entity follows the same interface
- Module registration: feature modules declared in `AppModule` imports array — SubscriptionModule registers the same way

### Integration Points
- `src/app.module.ts` — add SubscriptionModule to imports
- `src/main.ts` — add `rawBody: true` to NestFactory.create options (blocking first change)
- `.env.example` — add `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID`, `FRONTEND_URL`

</code_context>

<specifics>
## Specific Ideas

- Known frontend URLs: dev=`http://localhost:5173`, staging=`https://stage.tradeflowhub.uk`, prod=`https://tradeflowhub.uk`
- Stripe Dashboard prerequisite (from STATE.md): Product/Price (£6/month), webhook endpoint, Customer Portal must be configured before Phase 29 can be executed against real Stripe

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 29-subscription-module-foundation*
*Context gathered: 2026-03-29*
