# Domain Pitfalls

**Domain:** Adding Stripe subscription billing (Checkout + webhooks) to an existing NestJS 11 / MongoDB SaaS app
**Researched:** 2026-03-28
**Confidence:** HIGH (well-documented Stripe integration patterns, NestJS Express body parsing is a known challenge, Stripe docs are authoritative)

## Critical Pitfalls

Mistakes that cause broken billing, lost revenue, or require architectural rework.

### Pitfall 1: Webhook Raw Body vs Global JSON Body Parser -- Signature Verification Fails Silently

**What goes wrong:**
Stripe webhook signature verification (`stripe.webhooks.constructEvent(rawBody, sig, secret)`) requires the **raw, unparsed** request body as a Buffer. NestJS with Express applies `express.json()` globally (either explicitly in `main.ts` or implicitly via `@nestjs/platform-express`). By the time the webhook controller receives the request, `req.body` is already a parsed JavaScript object. Passing the parsed object to `constructEvent()` produces a signature mismatch, and every webhook is rejected with `Stripe signature verification failed`. The developer sees webhooks arriving in Stripe's dashboard as 400/500 errors and has no idea why.

**Why it happens:**
NestJS uses Express under the hood. Express's JSON body parser reads the raw stream, parses it to JSON, and discards the raw bytes. The raw body is gone by the time middleware/guards/controllers execute. This is the single most common Stripe + Express integration failure, and NestJS makes it slightly worse because the body parser setup is implicit and buried in the platform adapter.

**Consequences:**
- Every webhook fails signature verification -- subscriptions never sync to local DB
- Stripe retries failed webhooks for up to 3 days, but all retries fail too
- Users complete Checkout, Stripe creates the subscription, but the app never knows
- Subscription status in MongoDB stays stale forever -- users get locked out or get free access

**Prevention:**
NestJS + Express supports the `rawBody` option on the application factory. Enable it in `main.ts`:

```typescript
// main.ts
const app = await NestFactory.create(AppModule, {
  rawBody: true,  // Preserves raw body on req.rawBody for ALL requests
});
```

This tells Express to buffer the raw body alongside the parsed body. Then in the webhook controller, access `req.rawBody`:

```typescript
// stripe-webhook.controller.ts
@Post('webhook')
async handleWebhook(
  @Req() req: RawBodyRequest<Request>,
  @Headers('stripe-signature') signature: string,
) {
  const event = this.stripe.webhooks.constructEvent(
    req.rawBody,      // Buffer, not parsed object
    signature,
    this.webhookSecret,
  );
  // ... handle event
}
```

**Do NOT use these alternatives (they cause other problems):**
- Excluding the webhook route from the global JSON parser via middleware ordering -- fragile and breaks when routes change.
- Using a separate Express instance for webhooks -- unnecessary complexity.
- Manually re-serializing `JSON.stringify(req.body)` -- this produces a different byte sequence than the original payload (key ordering, whitespace) and signature verification still fails.

**For Trade Flow specifically:** The existing `main.ts` creates the app with `NestFactory.create(AppModule)`. Adding `{ rawBody: true }` is a one-line change. The `rawBody` option has been available since NestJS 10. The global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })` does not interfere with raw body access -- it only validates decorated DTOs, and the webhook endpoint should use `@Req()` directly, not a DTO class.

**Detection:**
- Stripe dashboard shows webhook deliveries with 400/500 responses
- Logs show "Stripe signature verification failed" or "No signatures found matching the expected signature"
- Subscription records never appear in MongoDB despite successful Checkout sessions

**Phase to address:** Phase 1 (Subscription Module scaffold) -- raw body must work before any webhook handler is written.

---

### Pitfall 2: Webhook Idempotency Failures -- Duplicate Subscription Records from Retry Storms

**What goes wrong:**
Stripe guarantees **at-least-once delivery** -- the same event can be delivered multiple times. A webhook handler that naively creates a subscription record on `customer.subscription.created` will create duplicates when Stripe retries (due to network timeout, 5xx from your server, or Stripe's own retry logic). The MongoDB collection ends up with 2-3 subscription documents for the same Stripe subscription, and queries like "find subscription by userId" return unpredictable results.

**Why it happens:**
Stripe retries webhook deliveries if it doesn't receive a 2xx response within 20 seconds. Common causes of retries: (a) the handler takes too long (does multiple DB writes, calls external services), (b) the server returns 500 due to an unrelated bug, (c) network blip causes Stripe's request to timeout even though the handler succeeded. Developers assume webhooks are delivered exactly once.

**Consequences:**
- Duplicate subscription records in MongoDB
- `findOne({ userId })` returns the wrong record (oldest duplicate, not latest)
- Subscription cancellation only deletes/updates one duplicate, leaving a "zombie" active subscription
- Revenue reporting is wrong (counts subscriptions twice)

**Prevention:**
1. **Store and check the Stripe event ID:** Before processing any event, check if `event.id` has already been processed. Use a `processed_events` collection (or a field on the subscription document):

```typescript
// In webhook handler, BEFORE any business logic:
const alreadyProcessed = await this.processedEventsRepository.exists(event.id);
if (alreadyProcessed) {
  return; // Return 200 to Stripe, but do nothing
}
// Process the event...
// After successful processing:
await this.processedEventsRepository.create({ eventId: event.id, processedAt: new Date() });
```

2. **Use upsert instead of insert for subscription records:** When handling `customer.subscription.created`, use `findOneAndUpdate` with `upsert: true` keyed on `stripeSubscriptionId` (not `userId` -- a user could theoretically have multiple subscription attempts):

```typescript
await this.subscriptionModel.findOneAndUpdate(
  { stripeSubscriptionId: subscription.id },
  { $set: { status, currentPeriodEnd, /* ... */ } },
  { upsert: true },
);
```

3. **Respond 200 quickly:** Return a 200 response as soon as the event is validated and queued/recorded. Do not perform slow operations (sending emails, updating multiple collections) synchronously in the webhook handler. Stripe's 20-second timeout is generous but not infinite.

**For Trade Flow specifically:** The subscription is userId-keyed per the milestone requirements. Create a unique index on `{ userId: 1 }` in the subscriptions collection to prevent duplicate records at the database level. Use `findOneAndUpdate` with `upsert: true` keyed on `userId` for subscription creation/updates. This makes the handler naturally idempotent -- re-processing the same event just overwrites with the same data. Also add a unique index on `{ stripeSubscriptionId: 1 }` as a secondary safeguard.

**Detection:**
- `db.subscriptions.aggregate([{ $group: { _id: "$userId", count: { $sum: 1 } } }, { $match: { count: { $gt: 1 } } }])` returns results
- Users report being asked to subscribe even though they already did
- Stripe dashboard shows webhook retries (check "Pending webhooks" count)

**Phase to address:** Phase 1 (Subscription Module) -- upsert pattern and unique indexes must be in place from the first webhook handler.

---

### Pitfall 3: Checkout Session Completion Race Condition -- User Redirected Before Webhook Arrives

**What goes wrong:**
After completing Stripe Checkout, the user is redirected to `success_url` (e.g., `/subscribe/success`). The frontend immediately checks subscription status via `GET /v1/subscription`. But the webhook that creates the local subscription record hasn't arrived yet (webhooks are asynchronous and can be delayed 1-30 seconds). The API returns "no subscription found," and the user sees an error or is redirected back to the subscribe page, even though payment succeeded.

**Why it happens:**
Stripe Checkout completion triggers two things simultaneously: (a) the browser redirect to `success_url`, and (b) a webhook event to your server. The redirect is instant (client-side). The webhook goes through Stripe's event pipeline, which involves queuing, serialization, and an HTTP request to your server. The webhook almost always arrives after the redirect, sometimes significantly so under load.

**Consequences:**
- User pays, sees "Subscribe" page again, panics
- User clicks "Subscribe" again, potentially creating a second Checkout session
- Support tickets: "I paid but it says I'm not subscribed"
- Even worse: if the frontend shows "subscription failed" and the user contacts their bank

**Prevention:**
1. **Use the `checkout.session.completed` event as the primary trigger:** In addition to `customer.subscription.created`, handle `checkout.session.completed` to create the initial subscription record. This event fires reliably and contains the subscription ID.

2. **Optimistic polling on the success page:** The `/subscribe/success` page should poll `GET /v1/subscription` every 2-3 seconds for up to 30 seconds, showing a "Setting up your subscription..." loading state:

```typescript
// SubscribeSuccessPage.tsx
const { data, isLoading } = useGetSubscriptionQuery(undefined, {
  pollingInterval: 3000,  // RTK Query polling
});

if (!data && isLoading) {
  return <LoadingSpinner message="Setting up your subscription..." />;
}
```

3. **Pass `session_id` in the success URL and verify server-side:** Configure Checkout with `success_url: '/subscribe/success?session_id={CHECKOUT_SESSION_ID}'`. The success page sends this to the API, which calls `stripe.checkout.sessions.retrieve(sessionId)` to verify payment and create the subscription record synchronously if the webhook hasn't arrived yet:

```typescript
// GET /v1/subscription/verify-session?sessionId=cs_xxx
// Server calls stripe.checkout.sessions.retrieve(sessionId)
// If session.payment_status === 'paid' and no local record exists, create it
```

This is the belt-and-suspenders approach: the webhook creates the record eventually, but the session verification creates it immediately on user demand.

**For Trade Flow specifically:** Use approach #3. The `success_url` should include `{CHECKOUT_SESSION_ID}` (Stripe's template variable). The `/subscribe/success` page extracts the session ID from the URL, calls a verification endpoint, and either shows the subscription confirmation or a loading/retry state. This eliminates the race condition entirely.

**Detection:**
- Users land on success page and see "no subscription" error
- Stripe dashboard shows successful Checkout sessions with no corresponding webhook delivery yet
- Subscription record appears in MongoDB seconds after the user reports the problem

**Phase to address:** Phase 2 (Checkout + Webhook integration) -- success page must handle the race from day one.

---

### Pitfall 4: Subscription Status Sync Lag -- Local DB Diverges from Stripe

**What goes wrong:**
The local MongoDB subscription record becomes stale because webhook events are missed, delayed, or processed out of order. A user cancels their subscription in Stripe's billing portal, but the local record still shows `active`. Or worse: an `invoice.payment_failed` event arrives but the handler only updates `status` to `past_due` without updating `currentPeriodEnd`, creating an inconsistent record. Over time, the local DB diverges from Stripe's source of truth, and users get incorrect access levels.

**Why it happens:**
Multiple causes: (a) webhook endpoint was down during a deployment, (b) a specific event type wasn't handled (e.g., `customer.subscription.updated` after a plan change), (c) events arrive out of order (`subscription.updated` before `subscription.created`), (d) the handler partially fails (updates status but crashes before updating period dates). Stripe is the source of truth, but the app treats the local record as the source of truth for access control.

**Consequences:**
- Users get free access after canceling (local still says `active`)
- Users get locked out despite having an active subscription (local says `canceled`)
- `currentPeriodEnd` is wrong, causing premature or delayed trial expiry enforcement
- No easy way to detect the drift until a user reports it

**Prevention:**
1. **Handle ALL relevant subscription lifecycle events:** At minimum:
   - `checkout.session.completed` -- initial subscription creation
   - `customer.subscription.created` -- subscription record created (backup for checkout)
   - `customer.subscription.updated` -- status changes, period changes, plan changes
   - `customer.subscription.deleted` -- subscription canceled and period ended
   - `invoice.payment_succeeded` -- payment confirmed, update `currentPeriodEnd`
   - `invoice.payment_failed` -- payment failed, update status to `past_due`

2. **Always overwrite from the event data, never merge partially:**
```typescript
// GOOD: Full overwrite from Stripe event
await this.subscriptionModel.findOneAndUpdate(
  { stripeSubscriptionId: subscription.id },
  {
    $set: {
      status: subscription.status,
      currentPeriodStart: new Date(subscription.current_period_start * 1000),
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      cancelAtPeriodEnd: subscription.cancel_at_period_end,
      trialEnd: subscription.trial_end ? new Date(subscription.trial_end * 1000) : null,
      updatedAt: new Date(),
    },
  },
  { upsert: true },
);

// BAD: Only updating status
await this.subscriptionModel.findOneAndUpdate(
  { stripeSubscriptionId: subscription.id },
  { $set: { status: subscription.status } },  // Missing period dates!
);
```

3. **Add a reconciliation endpoint:** Create an admin/cron endpoint that fetches the subscription from Stripe's API and overwrites the local record. Run this periodically (daily) or on-demand when a user reports a discrepancy:

```typescript
// Called by cron or admin action
const stripeSubscription = await this.stripe.subscriptions.retrieve(stripeSubscriptionId);
await this.updateLocalFromStripe(stripeSubscription);
```

4. **Handle out-of-order events gracefully:** Include `event.created` timestamp or `subscription.updated` timestamp. Only apply an update if the event is newer than the last processed event for that subscription.

**For Trade Flow specifically:** Since this is a single-plan, single-price product (GBP 6/month), the subscription lifecycle is simpler than multi-plan SaaS. But the same events fire. The `customer.subscription.updated` event is particularly important -- it fires when: trial ends and billing starts, payment method changes, subscription is set to cancel at period end, and subscription renews each month. Map every `subscription.status` value from Stripe to the local status enum.

**Detection:**
- Compare `db.subscriptions.find({})` status values against Stripe dashboard
- Users report access level doesn't match their subscription status
- `currentPeriodEnd` in local DB is in the past but status is still `active`

**Phase to address:** Phase 2 (Webhook event handlers) -- all event types handled from initial implementation, not added incrementally.

---

### Pitfall 5: Trial End Edge Cases -- Stripe-Managed Trial vs App-Enforced Trial Mismatch

**What goes wrong:**
The app implements its own trial expiry logic (checking `trialEnd` date against `Date.now()`) that conflicts with Stripe's trial management. Stripe's trial ends when Stripe says it ends (fires `customer.subscription.updated` with `status: 'active'` and `trial_end` in the past). The app's frontend checks `trialEnd < now` and shows "trial expired" 30 seconds before Stripe converts to active billing, or 30 seconds after. Users see flickering between "trial" and "active" states. Worse: if the app enforces trial expiry locally and blocks access, but Stripe hasn't charged the card yet, the user is locked out during the transition.

**Why it happens:**
Clock skew between the app server and Stripe's servers. Timezone handling errors (Stripe uses Unix timestamps in UTC; the app might compare against local time). The app checking `trialEnd` independently rather than trusting the `status` field from Stripe.

**Consequences:**
- User locked out during trial-to-active transition (minutes to hours)
- "Trial expired" message shown to users who just started paying
- Double enforcement: app blocks access AND Stripe charges the card, but the subscription status webhook hasn't arrived yet

**Prevention:**
1. **Never check `trialEnd` date for access control.** Use the `status` field exclusively:
   - `trialing` = full access (trial active)
   - `active` = full access (paying)
   - `past_due` = read-only (payment failed, grace period)
   - `canceled` = read-only (subscription ended)
   - `incomplete` = no access (initial payment failed)
   - `incomplete_expired` = no access (initial payment abandoned)

2. **The `trialEnd` field is for display only** (showing "X days remaining" in the trial banner). Never use it for authorization decisions.

3. **Trust Stripe's status transitions:** When Stripe converts trial to active, it fires `customer.subscription.updated` with `status: 'active'`. The local record should reflect this. Between the trial ending and the webhook arriving, the local record still says `trialing` -- this is fine because it means the user retains access during the transition, which is the correct UX.

4. **Handle the `trialing` -> `past_due` transition:** If the card fails on first charge after trial, Stripe fires `invoice.payment_failed` and sets subscription status to `past_due`. The app should transition to read-only mode here, not when the trial end date passes.

**For Trade Flow specifically:** The milestone specifies "read-only access when trial expires." This should be implemented as "read-only when status is `past_due`, `canceled`, or `incomplete`" -- NOT "read-only when `trialEnd < Date.now()`". The trial banner ("X days remaining") can safely use `trialEnd` for display because imprecision of a few seconds doesn't matter for a countdown display.

**Detection:**
- Users report being locked out despite having an active subscription
- Logs show access denied for users whose Stripe subscription is `active`
- Trial banner shows "0 days remaining" but status is still `trialing`

**Phase to address:** Phase 2 (Status enforcement logic) -- define the status-to-access mapping before building the SubscriptionGate.

---

### Pitfall 6: Frontend Gate Flickering -- SubscriptionGate Flashes Block State During Load

**What goes wrong:**
The `SubscriptionGate` component wraps all business routes and checks subscription status. On page load (or hard refresh), the subscription query is in a loading state. During this loading period, the gate doesn't know the subscription status and either: (a) flashes the "Subscribe" page for 200-500ms before the API responds and shows the real content, or (b) flashes the app content before redirecting to the subscribe page. Both create a jarring, unprofessional UX. This is especially noticeable on slower connections and on every page refresh.

**Why it happens:**
RTK Query's `useGetSubscriptionQuery()` starts in `isLoading: true` state. The gate component renders immediately with no data. Developers write:
```typescript
// BAD: Flashes subscribe page during loading
if (!subscription) return <Navigate to="/subscribe" />;
```
This treats "no data yet" the same as "no subscription exists."

**Consequences:**
- Every page refresh shows a flash of the subscribe/paywall page
- Users think their subscription disappeared (momentary panic)
- On mobile/slow connections, the flash lasts 1-2 seconds
- Support tickets: "I keep getting asked to subscribe"

**Prevention:**
1. **Show a loading state, not a redirect, while status is unknown:**

```typescript
// SubscriptionGate.tsx
const { data: subscription, isLoading, isError } = useGetSubscriptionQuery();
const { user } = useAuth();

// Support role bypasses gate entirely
if (user?.role === 'support') return <Outlet />;

// Loading: show skeleton, NOT the subscribe page
if (isLoading) return <FullPageSpinner />;

// Error fetching: show a retry state, not the subscribe page
if (isError) return <SubscriptionErrorState />;

// No subscription or inactive: redirect to subscribe
if (!subscription || !isActiveStatus(subscription.status)) {
  return <Navigate to="/subscribe" replace />;
}

// Active/trialing: render the app
return <Outlet />;
```

2. **Cache the subscription status in Redux store:** After the first successful fetch, store the status in a Redux slice. On subsequent navigations within the same session, the gate reads from the cached store (instant) while RTK Query re-validates in the background. This eliminates the flash on in-app navigation.

3. **Use `skipToken` or conditional fetching:** Don't fetch subscription status until the user is authenticated. Fetching before auth is ready causes an unnecessary loading state:

```typescript
const { data: user } = useAuth();
const { data: subscription, isLoading } = useGetSubscriptionQuery(
  user ? undefined : skipToken  // Don't fetch until user is known
);
```

4. **Persist subscription status to localStorage:** On app load, read the last-known status from localStorage for an instant initial render, then update when the API responds. This eliminates the flash even on hard refresh:

```typescript
const cachedStatus = localStorage.getItem('subscriptionStatus');
if (isLoading && cachedStatus === 'active') return <Outlet />;
```

**For Trade Flow specifically:** The app already uses RTK Query with Redux store. The `SubscriptionGate` should wrap routes inside the authenticated layout (after `ProtectedRoute`). The loading state should match the existing app loading patterns (likely the same spinner used during initial auth check). The gate must also handle the support role exemption -- check user role before checking subscription.

**Detection:**
- Record a Playwright trace of a page refresh while subscribed -- look for the subscribe page frame
- Users report brief flash of "Subscribe" page on every refresh
- The subscribe page gets pageview analytics even from subscribed users

**Phase to address:** Phase 3 (Frontend SubscriptionGate) -- loading state handling must be designed with the gate, not retrofitted.

---

## Moderate Pitfalls

### Pitfall 7: Webhook Endpoint Not Excluded from CORS/Auth Guards

**What goes wrong:**
The Stripe webhook endpoint fails because: (a) the global `JwtAuthGuard` rejects the request (no Firebase JWT in Stripe's webhook delivery), (b) CORS middleware blocks the request (Stripe's servers aren't in the allowed origins), or (c) the global `ValidationPipe` rejects the raw body because it doesn't match any DTO.

**Prevention:**
- Exclude the webhook route from `JwtAuthGuard` -- use a custom `@Public()` decorator or apply the guard at the controller/route level instead of globally.
- CORS is only enforced on browser requests (preflight + `Origin` header). Stripe's server-to-server webhook requests don't include an `Origin` header, so CORS middleware typically doesn't block them. However, verify this in your specific middleware configuration.
- The global `ValidationPipe` with `whitelist: true` only applies to decorated DTO parameters. Since the webhook controller uses `@Req()` directly (not a DTO), the ValidationPipe is not triggered. But verify this assumption with a test.
- Stripe webhook verification (`constructEvent`) IS the authentication for this endpoint. Do not add additional auth on top.

**For Trade Flow specifically:** The existing API uses `@UseGuards(JwtAuthGuard)` at the controller level (not globally). This means new controllers are unguarded by default -- the webhook controller simply doesn't add the guard. Verify this pattern holds by checking existing controllers. CORS in `main.ts` checks the `Origin` header against `CORS_ORIGIN` env var -- Stripe webhooks don't send `Origin`, so they pass through.

**Phase to address:** Phase 1 (Subscription Module) -- webhook endpoint accessibility verified in the first integration test.

---

### Pitfall 8: Stripe API Version Mismatch Between Code and Dashboard

**What goes wrong:**
The Stripe Node SDK defaults to a specific API version, but the Stripe dashboard webhook endpoint configuration may use a different version. Event payloads have different field names or structures across API versions. The webhook handler expects `subscription.status` but receives `subscription.plan.active` (hypothetical field change). Or `trial_end` is a timestamp in one version but an ISO string in another.

**Prevention:**
- Pin the Stripe API version explicitly when initializing the Stripe client:
```typescript
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: '2024-12-18.acacia',  // Pin to specific version
});
```
- In the Stripe Dashboard, set the webhook endpoint to use the same API version (Settings > Webhooks > API version).
- When upgrading the `stripe` npm package, check the changelog for API version changes and update both the SDK initialization and dashboard webhook version together.
- Write unit tests that parse real webhook payloads (captured from Stripe CLI during development) to catch field mapping errors.

**Phase to address:** Phase 1 (Stripe SDK setup) -- version pinning from first initialization.

---

### Pitfall 9: Stripe Checkout Session Created but User Never Completes -- Orphaned Sessions

**What goes wrong:**
The API creates a Stripe Checkout session (`POST /v1/subscription/checkout`), but the user abandons the Checkout page (closes browser, navigates away, card declined). The session expires after 24 hours (default). Meanwhile, the app may have created a "pending" subscription record locally, or the frontend shows a stale "completing payment" state. If the user tries to subscribe again, the app might try to use the expired session or refuse because a "pending" record exists.

**Prevention:**
- Do NOT create a local subscription record when creating the Checkout session. The local record should only be created by the webhook (`checkout.session.completed` or `customer.subscription.created`).
- Set `expires_at` on the Checkout session to a shorter window (e.g., 30 minutes) to reduce the window of confusion:
```typescript
const session = await this.stripe.checkout.sessions.create({
  // ...
  expires_at: Math.floor(Date.now() / 1000) + 30 * 60, // 30 minutes
});
```
- The Checkout endpoint should be idempotent: if the user clicks "Subscribe" again, create a new Checkout session. Don't block them because a previous session exists.
- Handle `checkout.session.expired` webhook event to clean up any optimistic local state (if you created any).

**For Trade Flow specifically:** The `POST /v1/subscription/checkout` endpoint should create the Stripe Checkout session and return the URL. No local subscription record until webhook confirmation. If a user with an existing expired/canceled subscription clicks Subscribe, create a new Checkout session -- Stripe handles the customer record reuse.

**Phase to address:** Phase 2 (Checkout flow) -- no premature local record creation.

---

### Pitfall 10: `cancel_at_period_end` Mishandled -- User Loses Access Immediately on Cancel

**What goes wrong:**
When a user cancels their subscription, the handler calls `stripe.subscriptions.cancel(subscriptionId)` which cancels **immediately** -- the subscription ends right now, not at the end of the billing period. The user paid for the full month but loses access mid-period. Alternatively, the handler correctly uses `cancel_at_period_end` but the frontend shows "Canceled" and hides features, even though the user should have access until `currentPeriodEnd`.

**Prevention:**
1. **Use `cancel_at_period_end`, not immediate cancellation:**
```typescript
// CORRECT: Cancel at end of billing period
await this.stripe.subscriptions.update(subscriptionId, {
  cancel_at_period_end: true,
});

// WRONG: Immediate cancellation, user loses remaining paid time
await this.stripe.subscriptions.cancel(subscriptionId);
```

2. **Frontend must distinguish "canceling" from "canceled":**
   - `cancel_at_period_end: true` + `status: 'active'` = "Your subscription will end on [date]. You have full access until then."
   - `status: 'canceled'` = "Your subscription has ended. Read-only access."

3. **Store `cancelAtPeriodEnd` in the local subscription record** and use it in the UI to show appropriate messaging. The status will still be `active` until the period actually ends.

**For Trade Flow specifically:** The milestone specifies "cancels at period end, access continues until then." The `DELETE /v1/subscription` endpoint must use `stripe.subscriptions.update` with `cancel_at_period_end: true`, not `stripe.subscriptions.cancel`. The SubscriptionStatusCard should show "Canceling -- access until [date]" when `cancelAtPeriodEnd` is true.

**Phase to address:** Phase 2 (Cancel flow) -- use the correct Stripe API method from the start.

---

### Pitfall 11: Webhook Secret Confusion -- Test vs Live, CLI vs Dashboard

**What goes wrong:**
Stripe has different webhook signing secrets for: (a) the Stripe Dashboard webhook endpoint (starts with `whsec_`), (b) the Stripe CLI (`stripe listen --forward-to`) which generates a temporary local secret (also starts with `whsec_` but is different). Developers use the CLI secret during development, deploy to production with the same secret, and all webhooks fail in production because the production webhook endpoint uses a different secret.

**Prevention:**
- Use environment variables: `STRIPE_WEBHOOK_SECRET` in `.env`.
- Document clearly: the CLI secret (from `stripe listen`) is for local development only. The dashboard secret (from Webhooks > Endpoint > Signing secret) is for deployed environments.
- Add both secrets to `.env.example` with clear comments:
```bash
# Stripe webhook signing secret
# Local dev: use the secret from `stripe listen --forward-to localhost:3000/v1/subscription/webhook`
# Production: use the signing secret from Stripe Dashboard > Webhooks > your endpoint
STRIPE_WEBHOOK_SECRET=whsec_...
```
- In CI/staging, use the Stripe test mode dashboard webhook secret (not the CLI secret).

**Phase to address:** Phase 1 (Environment setup) -- secret management documented before first webhook test.

---

### Pitfall 12: Support Role Bypass Not Applied Consistently

**What goes wrong:**
The support role is supposed to bypass subscription gating entirely. The frontend `SubscriptionGate` checks the role and skips the gate. But the backend doesn't enforce the same bypass. An API request from a support user to create a customer returns 403 because the backend middleware checks subscription status and blocks it. Or vice versa: the backend bypasses but the frontend doesn't, and the support user sees the subscribe page.

**Prevention:**
- Implement the support bypass in **both** frontend gate and backend middleware/guard.
- Backend: Create a `SubscriptionGuard` (NestJS guard) that checks subscription status. The guard should check user role first and skip subscription validation for support users:
```typescript
@Injectable()
export class SubscriptionGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const user = context.switchToHttp().getRequest().user;
    if (user.role === 'support') return true;
    // ... check subscription status
  }
}
```
- Frontend: The `SubscriptionGate` checks `user.role === 'support'` before checking subscription.
- Write a test that verifies a support user can access gated routes without a subscription -- test both API and UI.

**Phase to address:** Phase 3 (Support role integration) -- tested in both frontend gate and backend guard.

---

## Minor Pitfalls

### Pitfall 13: Stripe Customer Not Linked to User -- Lost Association

**What goes wrong:**
Each Stripe Checkout session creates (or reuses) a Stripe Customer object. If the app doesn't store the `stripeCustomerId` on the user/subscription record, there's no way to associate Stripe billing data with the app user later. The Billing Portal link, invoice lookups, and subscription management all require the `stripeCustomerId`.

**Prevention:**
- When creating the Checkout session, set `customer_email` to the user's email (for new customers) or `customer` to the existing `stripeCustomerId` (for returning customers).
- Store `stripeCustomerId` from the `checkout.session.completed` event on the subscription record.
- Before creating a new Checkout session, check if the user already has a `stripeCustomerId` and reuse it. This prevents duplicate Stripe Customer objects for the same user.

**Phase to address:** Phase 2 (Checkout session creation).

---

### Pitfall 14: Stripe Test Mode vs Live Mode Keys Mixed Up

**What goes wrong:**
Test mode keys (`sk_test_`, `pk_test_`) are used in production, or live mode keys are used in development. Test mode keys in production mean no real charges are processed -- users "subscribe" but never pay. Live mode keys in development mean real charges happen during testing.

**Prevention:**
- Validate key prefix on app startup:
```typescript
if (process.env.NODE_ENV === 'production' && process.env.STRIPE_SECRET_KEY.startsWith('sk_test_')) {
  throw new Error('FATAL: Stripe test key used in production');
}
```
- Use separate `.env` files for each environment with clear naming.
- Never commit Stripe keys to the repository (already covered by `.gitignore` for `.env`).

**Phase to address:** Phase 1 (Environment setup) -- key validation on startup.

---

### Pitfall 15: Subscription Amounts in Wrong Currency or Wrong Amount

**What goes wrong:**
Stripe amounts are in the **smallest currency unit** (pence for GBP, cents for USD). A price of GBP 6.00 must be sent as `600`, not `6`. Sending `6` creates a GBP 0.06 subscription. The mistake isn't caught because Stripe happily creates the subscription at any amount, and the error only surfaces when real charges appear on customer bank statements.

**Prevention:**
- Define the price in Stripe Dashboard as a Product + Price object (not dynamically in code). Reference it by `price_id` in the Checkout session:
```typescript
line_items: [{ price: process.env.STRIPE_PRICE_ID, quantity: 1 }],
```
- This is safer than constructing prices dynamically because the price is configured once in the dashboard.
- For Trade Flow at GBP 6/month: create the Product and Price in Stripe Dashboard, store the Price ID in env vars, and reference it in Checkout session creation. Never hardcode amounts.

**Phase to address:** Phase 1 (Stripe product/price setup in dashboard).

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1: Subscription Module scaffold | Raw body not available for webhook signature (#1) | Add `rawBody: true` to `NestFactory.create` options as first task |
| Phase 1: Subscription Module scaffold | Webhook secret confusion (#11) | Document CLI vs dashboard secrets in .env.example |
| Phase 1: Subscription Module scaffold | API version mismatch (#8) | Pin Stripe API version in SDK initialization |
| Phase 2: Checkout + Webhooks | Checkout race condition on success page (#3) | Implement session verification endpoint + polling |
| Phase 2: Checkout + Webhooks | Duplicate subscription records (#2) | Upsert pattern + unique indexes from first handler |
| Phase 2: Checkout + Webhooks | Orphaned Checkout sessions (#9) | No local record until webhook; shorter session expiry |
| Phase 2: Checkout + Webhooks | Status sync lag (#4) | Handle ALL subscription event types; full overwrite pattern |
| Phase 2: Checkout + Webhooks | Cancel kills access immediately (#10) | Use `cancel_at_period_end`, not `cancel()` |
| Phase 2: Checkout + Webhooks | Trial end edge cases (#5) | Status-based access control only; never check trialEnd for auth |
| Phase 3: Frontend gate | Gate flickering (#6) | Loading state shown during fetch; cached status for instant render |
| Phase 3: Frontend gate | Support role inconsistency (#12) | Bypass in both frontend gate AND backend guard; test both |

## "Looks Done But Isn't" Checklist

- [ ] **Webhook receives raw body:** `constructEvent()` succeeds with a real Stripe CLI test event
- [ ] **Idempotent handling:** Send the same webhook event twice; only one subscription record exists
- [ ] **Success page handles delay:** Disconnect webhook endpoint, complete Checkout; success page shows loading and eventually resolves via session verification
- [ ] **All event types handled:** `subscription.created`, `subscription.updated`, `subscription.deleted`, `invoice.payment_failed`, `invoice.payment_succeeded`, `checkout.session.completed`
- [ ] **Trial expiry uses status, not date:** Block access based on `status` field, not `trialEnd < now`
- [ ] **Cancel preserves access:** Cancel subscription; user still has full access until `currentPeriodEnd`
- [ ] **No gate flicker:** Hard refresh while subscribed -- no flash of subscribe page
- [ ] **Support role works without subscription:** Support user accesses all routes with no subscription record
- [ ] **Unique indexes on subscriptions collection:** `{ userId: 1 }` and `{ stripeSubscriptionId: 1 }` both unique
- [ ] **Stripe keys validated on startup:** Test key in production throws fatal error

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Raw body not working (#1) | LOW | Add `rawBody: true` to NestFactory options; one-line fix but requires redeployment and webhook retry |
| Duplicate subscriptions (#2) | MEDIUM | Add unique index (may fail if duplicates exist); write migration to deduplicate; add upsert pattern to handlers |
| Checkout race condition (#3) | LOW | Add session verification endpoint; update success page to poll; no data migration needed |
| Status sync drift (#4) | HIGH | Write reconciliation script that fetches all subscriptions from Stripe API and overwrites local records; run on all users; add missing event handlers |
| Trial enforcement wrong (#5) | MEDIUM | Change access control from date-check to status-check; audit all status-checking code paths; no data migration |
| Gate flickering (#6) | LOW | Add loading state to SubscriptionGate; purely frontend change |
| Immediate cancel (#10) | HIGH | Users already lost access they paid for; need to reactivate affected subscriptions in Stripe; change cancel to `cancel_at_period_end`; communicate with affected users |

## Sources

- Stripe official documentation: Webhook signature verification, Checkout Sessions, Subscription lifecycle, `cancel_at_period_end`, API versioning, raw body requirements -- HIGH confidence (well-documented, stable API)
- NestJS official documentation: `rawBody` option on `NestFactory.create`, `ValidationPipe` behavior with `@Req()`, guard execution order -- HIGH confidence (documented feature since NestJS 10)
- Trade Flow codebase analysis: `main.ts` bootstrap pattern (`NestFactory.create`, `ValidationPipe`, CORS), `JwtAuthGuard` applied at controller level (not global), Express platform adapter, MongoDB/Mongoose ODM -- HIGH confidence (direct codebase reading via ARCHITECTURE.md and STRUCTURE.md)
- Trade Flow PROJECT.md: v1.6 milestone scope, subscription requirements, support role bypass, pricing -- HIGH confidence (direct reading)
- Stripe community patterns: Webhook idempotency, Checkout race conditions, trial management -- MEDIUM confidence (training data, broad community consensus; specific patterns verified against Stripe docs structure)
- NestJS + Stripe integration patterns: Raw body handling, guard exclusions -- MEDIUM confidence (training data, commonly discussed but no specific URL verification possible in this session)

---
*Pitfalls research for: Stripe subscription billing integration with NestJS 11 / MongoDB*
*Researched: 2026-03-28*
