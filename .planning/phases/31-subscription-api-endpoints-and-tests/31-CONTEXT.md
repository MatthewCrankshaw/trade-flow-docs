# Phase 31: Subscription API Endpoints and Tests - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Add three subscription management endpoints on top of the Phase 29/30 foundation: `GET /v1/subscription` (retrieve current subscription state), `DELETE /v1/subscription` (cancel at period end), and `POST /v1/subscription/portal` (create Stripe Billing Portal session). Add unit test coverage for all services and the repository. Phase is backend-only (`trade-flow-api`).

</domain>

<decisions>
## Implementation Decisions

### GET /v1/subscription — Missing Subscription State
- **D-01:** When no subscription record exists for the authenticated user, return 404 using `ResourceNotFoundError`. Phase 32's RTK Query hook treats a 404 as "no subscription" and `SubscriptionGate` redirects to `/subscribe`. Clean HTTP semantics, consistent with other resource-not-found patterns.

### DELETE /v1/subscription — Response Shape and Error Handling
- **D-02:** On success, return 200 with the full updated subscription response (cancelAtPeriodEnd: true, currentPeriodEnd set). Frontend can update the store immediately without a follow-up GET — Billing tab reflects cancellation state right away.
- **D-03:** If the user has no active subscription to cancel, throw `InvalidRequestError` (HTTP 422) with a clear message ("No active subscription to cancel"). Consistent with how other invalid operations are handled across the API — 422 fits better semantically than 404 for a failed action.
- **D-04:** The handler calls `stripe.subscriptions.update({ cancel_at_period_end: true })` and immediately updates the local record (cancelAtPeriodEnd: true) — no waiting for webhook. Optimistic local sync is appropriate here since the Stripe call is synchronous and the requirement explicitly states "updates the local record".

### POST /v1/subscription/portal — Return URL
- **D-05:** `return_url` for the Stripe Billing Portal session: `${FRONTEND_URL}/settings?tab=billing`. The `?tab=billing` query param pre-selects the Billing tab (Phase 33 wires up the tab routing). Using settings root with a tab identifier keeps the URL stable regardless of future tab path changes.

### Unit Test Scope — Responsibility Boundaries
- **D-06:** Invalid Stripe webhook signature rejection is tested at the controller/guard level in a `SubscriptionController` (or webhook guard) spec. The controller spec verifies: valid signature → 200 and enqueued; invalid signature → 400.
- **D-07:** `SubscriptionUpdaterService` tests cover only subscription update business logic — the five event types (`checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`) and their effects on the local record. No authorization logic in the updater service.
- **D-08:** Test scope matches REQUIREMENTS.md exactly:
  - TEST-01: `SubscriptionCreatorService` — Checkout Session creation with Stripe SDK mocked
  - TEST-02: `SubscriptionUpdaterService` — each of the 5 event types (Stripe SDK mocked); signature rejection tested separately in controller spec
  - TEST-03: `SubscriptionRetrieverService` — found and not-found subscription scenarios
  - TEST-04: `SubscriptionRepository` — upsert, findByUserId, findByStripeSubscriptionId operations
  - Additional: `SubscriptionController` (webhook route) spec — valid signature → 200; invalid signature → 400

### Claude's Discretion
- Stripe SDK mock structure for tests (exact shape of `jest.fn()` mocks for `stripe.subscriptions.*`, `stripe.billingPortal.sessions.create`, `stripe.webhooks.constructEvent`)
- `SubscriptionMockGenerator` design (follow existing mock generator conventions)
- Repository method naming for `findByUserId`, `findByStripeSubscriptionId`
- Response DTO shape for subscription endpoints (extend existing ISubscriptionDto or create ISubscriptionResponse)
- Controller method names and route handler structure

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.6 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.6 requirements; Phase 31 covers BILL-01, BILL-02, BILL-03, TEST-01, TEST-02, TEST-03, TEST-04

### Roadmap
- `.planning/ROADMAP.md` §Phase 31 — Phase goal, success criteria, dependency on Phase 30

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/phases/29-subscription-module-foundation/29-CONTEXT.md` — Entity schema (D-07–D-09), STRIPE_CLIENT injection token (D-12), module structure (D-13), env vars, Stripe customer creation pattern
- `.planning/phases/30-stripe-checkout-and-webhooks/30-CONTEXT.md` — Webhook processor architecture (D-01–D-03), updater service called by StripeWebhookProcessor, upsert patterns (D-10), out-of-order handling (D-11), verify-session endpoint (D-12–D-13)

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — NestJS naming, file patterns, error handling, logging
- `.planning/codebase/ARCHITECTURE.md` — Module structure, controller→service→repo layering, CoreModule globals
- `.planning/codebase/TESTING.md` — Test file organization, mock generator pattern, AAA structure, jest.Mocked<T> usage

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ResourceNotFoundError`: existing error class → used for GET /v1/subscription 404 case (D-01)
- `InvalidRequestError`: existing error class → used for DELETE /v1/subscription 422 case (D-03)
- `JwtAuthGuard` + `@Public()`: all three endpoints are protected by JwtAuthGuard (standard, not public)
- `AppLogger`: inject as `new AppLogger(ClassName.name)` per existing service convention
- `ConfigService`: read `FRONTEND_URL` env var for portal return URL (D-05)
- Mock generators: `BusinessMockGenerator`, `UserMockGenerator` in `src/*/test/mocks/` — `SubscriptionMockGenerator` follows same pattern

### Established Patterns
- Service splitting: Creator/Retriever/Updater per resource — `SubscriptionCreatorService`, `SubscriptionRetrieverService`, `SubscriptionUpdaterService` already named in REQUIREMENTS.md
- Test structure: `src/subscription/test/services/*.spec.ts`, `src/subscription/test/repositories/*.spec.ts`, `src/subscription/test/controllers/*.spec.ts`
- Response pattern: `ISubscriptionResponse` interface with camelCase fields matching the entity DTO
- Provider injection: `{ provide: STRIPE_CLIENT, useValue: { ... jest.fn()s ... } }` in test module setup (same as factory provider pattern in production)

### Integration Points
- `src/subscription/` — Phase 31 adds new methods to existing services and new endpoint handlers to existing controller
- GET and DELETE routes on `SubscriptionController` alongside the existing checkout and verify-session routes from Phases 29–30
- `FRONTEND_URL` env var already in `.env.example` from Phase 29

</code_context>

<specifics>
## Specific Ideas

- Portal return URL: `${FRONTEND_URL}/settings?tab=billing` — the `?tab=billing` query param is intentional; Phase 33 wires up tab routing in Settings and will read this param to pre-select Billing
- Invalid signature test: test the controller/guard, not the updater service — "SubscriptionUpdaterService should not test authorization, only subscription update business logic" (user's words)
- DELETE response: return full updated subscription (200), not 204 — lets frontend update immediately without a refetch

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 31-subscription-api-endpoints-and-tests*
*Context gathered: 2026-03-29*
