# Phase 29: Subscription Module Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-29
**Phase:** 29-subscription-module-foundation
**Areas discussed:** Stripe Customer creation, Checkout redirect URLs, Subscription entity completeness

---

## Stripe Customer creation

| Option | Description | Selected |
|--------|-------------|----------|
| Create Stripe Customer first | Before checkout session, call stripe.customers.create and store stripeCustomerId. Explicit userId↔Stripe link from the start. | ✓ |
| Pass customer_email, let Stripe auto-create | Simpler Phase 29; stripeCustomerId recovered from webhook in Phase 30. | |
| You decide | Claude picks the approach. | |

**User's choice:** Create Stripe Customer first

---

### Follow-up: Email source

| Option | Description | Selected |
|--------|-------------|----------|
| Firebase JWT | Extract email from request.user.email (decoded JWT via JwtAuthGuard). No extra DB call. | ✓ |
| MongoDB user record | Look up user in MongoDB. More consistent with app data but extra DB call. | |

**User's choice:** Firebase JWT

---

## Checkout redirect URLs

| Option | Description | Selected |
|--------|-------------|----------|
| FRONTEND_URL env var | Add FRONTEND_URL to .env; API builds success_url and cancel_url from it. | ✓ |
| Hard-code per environment | Hard-code URLs directly in the service. | |
| You decide | Claude picks. | |

**User's choice:** FRONTEND_URL env var

**Notes:** User provided actual URL values during discussion:
- Staging: `https://stage.tradeflowhub.uk`
- Production: `https://tradeflowhub.uk`
- (Dev implied: `http://localhost:5173`)

---

## Subscription entity completeness

| Option | Description | Selected |
|--------|-------------|----------|
| Full schema now | Define all fields Phase 30-33 need upfront. Phase 29 populates a subset; Phase 30 fills rest via webhooks. | ✓ |
| Minimal schema, expand in Phase 30 | Phase 29 only defines userId, stripeCustomerId, status. | |

**User's choice:** Full schema now

---

### Follow-up: SubscriptionStatus enum values

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror Stripe exactly | Values: trialing, active, past_due, canceled, incomplete. Direct mapping from Stripe subscription.status. | ✓ |
| Custom enum values | Define own strings, map from Stripe in webhook handler. | |

**User's choice:** Mirror Stripe exactly

---

## Claude's Discretion

- Stripe provider factory details (ConfigService injection pattern)
- Repository method naming and upsert patterns
- DTO/response shape for checkout endpoint (`{ url: string }` implied)
- Unit test scope for Phase 29 (tests are Phase 31 concern)

## Deferred Ideas

None.
