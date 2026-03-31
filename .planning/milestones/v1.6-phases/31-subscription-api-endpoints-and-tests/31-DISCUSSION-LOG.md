# Phase 31: Subscription API Endpoints and Tests - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.

**Date:** 2026-03-29
**Areas discussed:** Missing subscription state, Cancel endpoint response, Billing Portal return URL, TEST-02 scope

---

## Area: Missing Subscription State

**Q:** GET /v1/subscription: what should it return when no subscription record exists for the user?

**Options presented:**
- 404 Not Found — ResourceNotFoundError; Phase 32 SubscriptionGate treats 404 as "no subscription"
- 200 with null fields — nullable response type
- 200 with 'incomplete' status — default subscription-shaped object

**Selected:** 404 Not Found

---

## Area: Cancel Endpoint Response

**Q1:** DELETE /v1/subscription: what should the endpoint return on success?

**Options presented:**
- 200 with updated subscription — full response with cancelAtPeriodEnd: true
- 204 No Content — simpler, frontend must refetch

**Selected:** 200 with updated subscription

**Q2:** DELETE /v1/subscription: what if the user has no active subscription to cancel?

**Options presented:**
- 422 InvalidRequestError — consistent with invalid operation handling
- 404 ResourceNotFoundError — accurate but less semantically appropriate for DELETE

**Selected:** 422 InvalidRequestError

---

## Area: Billing Portal Return URL

**Q:** POST /v1/subscription/portal: where should the return_url send users when they close the Stripe Billing Portal?

**Options presented:**
- `${FRONTEND_URL}/settings` — return to Settings root
- `${FRONTEND_URL}/settings/billing` — return to Billing tab directly (doesn't exist until Phase 33)
- `${FRONTEND_URL}/dashboard` — return home

**Selected:** `${FRONTEND_URL}/settings` with note: "Add some identifier to the URL to make sure that the currently selected tab is indicated so that it will always open on the correct tab."

**Resolved to:** `${FRONTEND_URL}/settings?tab=billing`

---

## Area: TEST-02 Scope

**Q:** TEST-02 says SubscriptionUpdaterService should cover "invalid signature rejection" — but signature verification is in the webhook controller. How should this be handled?

**Options presented:**
- Controller test for signatures + updater for events
- Only updater service tests (treat TEST-02 wording as an error)

**User response:** "We will need to have specific tests to cover invalid signature rejection on the Stripe webhook auth guard. The subscription updater service should not test authorization only subscription update business logic."

**Resolved to:** Controller/guard spec covers signature rejection; SubscriptionUpdaterService spec covers only the 5 event types and their effects on the local record.

---

## Area: SubscriptionGuard — API-Level Access Control (Update Session 2026-03-29)

**Context:** User requested an update to add API-level subscription enforcement — not just UI-level gating.

**Q1:** Which API operations should be blocked for unsubscribed users?

| Option | Selected |
|--------|----------|
| All write endpoints (POST/PUT/PATCH/DELETE) | ✓ |
| Write + some reads | |
| Scoped to job/quote/customer only | |

**Q2:** Implementation approach?

| Option | Selected |
|--------|----------|
| New SubscriptionGuard (NestJS guard, 403 Forbidden) | ✓ |
| Extend existing policies | |
| Middleware | |

**Q3:** Guard application scope?

| Option | Selected |
|--------|----------|
| Global APP_GUARD with @SkipSubscriptionCheck() decorator | ✓ |
| Per-controller @UseGuards() | |
| Route-group level | |

**Q4:** Which subscription statuses pass the guard?

| Option | Selected |
|--------|----------|
| trialing + active only | ✓ |
| trialing + active + past_due | |
| Any non-canceled | |

**Q5:** Support user bypass?

| Option | Selected |
|--------|----------|
| Check request.user.supportRoles on IUserDto | ✓ |
| Separate SupportGuard | |
| Hardcoded email list | |

**Notes:** `supportRoles` already exists on the user DTO — no schema changes needed.

**Q6:** Route exemptions?

| Option | Selected |
|--------|----------|
| Only @Public() routes and subscription routes | ✓ |
| Also exempt business settings | |
| Also exempt user/profile routes | |

**Resolved:** SubscriptionGuard blocks write methods globally. Subscription routes (checkout, verify-session, GET/DELETE subscription, POST portal, POST webhooks/stripe) decorated with @SkipSubscriptionCheck(). Support users bypass via supportRoles check. Returns 403 ForbiddenError.

---
