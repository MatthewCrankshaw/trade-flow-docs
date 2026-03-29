# Phase 31: Subscription API Endpoints and Tests - Research

**Researched:** 2026-03-29
**Domain:** Stripe subscription management API, NestJS guards, Jest unit testing
**Confidence:** HIGH

## Summary

Phase 31 adds three subscription management endpoints (GET subscription, DELETE subscription for cancellation, POST portal session) and comprehensive unit tests for all subscription services and the repository. It also introduces a global `SubscriptionGuard` that enforces subscription status on write operations at the API level. This phase builds entirely on the Phase 29/30 foundation -- the SubscriptionModule, entity, repository, Stripe provider, and webhook infrastructure already exist.

The endpoints use standard Stripe Node.js SDK calls: `stripe.subscriptions.update({ cancel_at_period_end: true })` for cancellation and `stripe.billingPortal.sessions.create({ customer, return_url })` for the portal. The guard follows the established NestJS `APP_GUARD` + `SetMetadata` bypass pattern already used by `JwtAuthGuard` + `@Public()`. Unit tests follow the project's AAA pattern with `jest.Mocked<T>`, mock generators, and NestJS `Test.createTestingModule`.

**Primary recommendation:** Implement the three endpoints as new route handlers on the existing `SubscriptionController`, create `SubscriptionGuard` as a global `APP_GUARD` with `@SkipSubscriptionCheck()` decorator, and write unit tests following the existing `BusinessCreator`/`BusinessRepository` test patterns with a new `SubscriptionMockGenerator`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** When no subscription record exists for the authenticated user, return 404 using `ResourceNotFoundError`.
- **D-02:** On successful cancel, return 200 with the full updated subscription response (cancelAtPeriodEnd: true, currentPeriodEnd set). Frontend updates store without a follow-up GET.
- **D-03:** If the user has no active subscription to cancel, throw `InvalidRequestError` (HTTP 422) with message "No active subscription to cancel".
- **D-04:** DELETE handler calls `stripe.subscriptions.update({ cancel_at_period_end: true })` and immediately updates the local record (cancelAtPeriodEnd: true) -- no waiting for webhook. Optimistic local sync.
- **D-05:** Stripe Billing Portal `return_url`: `${FRONTEND_URL}/settings?tab=billing`.
- **D-06:** Invalid Stripe webhook signature rejection is tested at the controller/guard level in a `SubscriptionController` (or webhook guard) spec, not the updater service.
- **D-07:** `SubscriptionUpdaterService` tests cover only subscription update business logic -- the five event types and their effects on the local record. No authorization logic in the updater service.
- **D-08:** Test scope matches REQUIREMENTS.md exactly: TEST-01 (SubscriptionCreatorService), TEST-02 (SubscriptionUpdaterService -- 5 event types + controller spec for signature), TEST-03 (SubscriptionRetrieverService), TEST-04 (SubscriptionRepository).
- **D-09:** New global `SubscriptionGuard` enforces subscription status on write operations (POST, PUT, PATCH, DELETE). GET and HEAD always bypass. Returns 403 (`ForbiddenError`) when blocked. Status check: only `trialing` and `active` pass.
- **D-10:** `SubscriptionGuard` registered as `APP_GUARD` in `AppModule`, running after `JwtAuthGuard`. Routes opt out via `@SkipSubscriptionCheck()` custom decorator. Bypassed routes: all `@Public()` routes, all subscription management routes, webhook routes.
- **D-11:** Support users bypass `SubscriptionGuard` by checking `request.user.supportRoles`. If truthy/non-empty, guard allows through without DB lookup.
- **D-12:** Statuses that pass the guard: `trialing` and `active` only. `past_due`, `canceled`, `incomplete`, or no record = blocked.
- **D-13:** Guard performs DB lookup via `SubscriptionRepository.findByUserId` to retrieve current status.

### Claude's Discretion
- Stripe SDK mock structure for tests (exact shape of `jest.fn()` mocks for `stripe.subscriptions.*`, `stripe.billingPortal.sessions.create`, `stripe.webhooks.constructEvent`)
- `SubscriptionMockGenerator` design (follow existing mock generator conventions)
- Repository method naming for `findByUserId`, `findByStripeSubscriptionId`
- Response DTO shape for subscription endpoints (extend existing ISubscriptionDto or create ISubscriptionResponse)
- Controller method names and route handler structure
- BullMQ job caching: guard may cache the subscription lookup per-request to avoid double DB call

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| BILL-01 | `GET /v1/subscription` returns status, `currentPeriodEnd`, `trialEnd`, and `cancelAtPeriodEnd` for authenticated user | `SubscriptionRetriever.findByUser()` calls `SubscriptionRepository.findByUserId()`, maps to response DTO. Returns 404 if no record (D-01). |
| BILL-02 | `DELETE /v1/subscription` cancels at period end and updates local record | Calls `stripe.subscriptions.update(stripeSubId, { cancel_at_period_end: true })`, updates local record immediately (D-04), returns full subscription response (D-02). Throws 422 if no active sub (D-03). |
| BILL-03 | `POST /v1/subscription/portal` creates Stripe Billing Portal session and returns URL | Calls `stripe.billingPortal.sessions.create({ customer: stripeCustomerId, return_url })`, returns `{ url: session.url }`. Requires existing subscription with `stripeCustomerId`. |
| TEST-01 | `SubscriptionCreatorService` unit tests with Stripe SDK mocked | Mock STRIPE_CLIENT with `jest.fn()` for `customers.create`, `checkout.sessions.create`. Test checkout session creation, Stripe Customer creation, error paths. |
| TEST-02 | `SubscriptionUpdaterService` unit tests for each webhook event type + signature rejection | Mock STRIPE_CLIENT. Test 5 event types (checkout.session.completed, customer.subscription.updated/deleted, invoice.payment_succeeded/failed). Signature rejection in separate controller spec. |
| TEST-03 | `SubscriptionRetrieverService` unit tests for found/not-found | Mock repository. Test found returns DTO, not-found throws ResourceNotFoundError. |
| TEST-04 | `SubscriptionRepository` unit tests for upsert, findByUserId, findByStripeSubscriptionId | Mock MongoConnectionService. Test each repository method with collection mock. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| stripe | 21.0.1 | Stripe API SDK (subscriptions.update, billingPortal.sessions.create) | Already installed in Phase 29; official Stripe Node.js SDK |
| @nestjs/core | 11.1.12 | NestJS framework core (guards, decorators, DI) | Project framework |
| jest | 30.2.0 | Test runner and assertion library | Project test framework |
| @nestjs/testing | 11.1.12 | Test.createTestingModule for unit tests | NestJS testing utilities |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| mongodb | 7.0.0 | ObjectId generation in mock data | Test fixtures |

No new packages need to be installed for Phase 31.

## Architecture Patterns

### Recommended File Structure
```
src/subscription/
  controllers/
    subscription.controller.ts          # Add GET, DELETE, POST portal routes
  services/
    subscription-creator.service.ts     # Exists from Phase 29
    subscription-retriever.service.ts   # NEW: findByUser
    subscription-updater.service.ts     # Exists from Phase 30 (webhook events)
  repositories/
    subscription.repository.ts          # Exists; may need findByStripeSubscriptionId
  guards/
    subscription.guard.ts               # NEW: global SubscriptionGuard
  decorators/
    skip-subscription-check.decorator.ts # NEW: @SkipSubscriptionCheck()
  responses/
    subscription.response.ts            # Extend with ISubscriptionResponse
  test/
    services/
      subscription-creator.service.spec.ts   # NEW (TEST-01)
      subscription-retriever.service.spec.ts # NEW (TEST-03)
      subscription-updater.service.spec.ts   # NEW (TEST-02)
    repositories/
      subscription.repository.spec.ts        # NEW (TEST-04)
    controllers/
      subscription.controller.spec.ts        # NEW (webhook signature test)
    guards/
      subscription.guard.spec.ts             # NEW (guard unit tests)
    mocks/
      subscription-mock-generator.ts         # NEW
```

### Pattern 1: Global Guard with Metadata Bypass
**What:** `SubscriptionGuard` implements `CanActivate`, registered as `APP_GUARD` in `AppModule`. Uses `Reflector` to check for `skipSubscriptionCheck` metadata. Also checks for `isPublic` metadata (the existing `@Public()` decorator key).
**When to use:** Any endpoint that requires subscription enforcement.
**Example:**
```typescript
// Source: NestJS official docs + project @Public() pattern
@Injectable()
export class SubscriptionGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly subscriptionRepository: SubscriptionRepository,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. Skip if route is public (no JWT = no user = no subscription check)
    const isPublic = this.reflector.getAllAndOverride<boolean>("isPublic", [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // 2. Skip if @SkipSubscriptionCheck() is present
    const skip = this.reflector.getAllAndOverride<boolean>("skipSubscriptionCheck", [
      context.getHandler(),
      context.getClass(),
    ]);
    if (skip) return true;

    // 3. Skip read methods (GET, HEAD)
    const request = context.switchToHttp().getRequest();
    const method = request.method.toUpperCase();
    if (method === "GET" || method === "HEAD") return true;

    // 4. Skip support users
    const user = request.user;
    if (user?.supportRoles && user.supportRoles.length > 0) return true;

    // 5. DB lookup
    const subscription = await this.subscriptionRepository.findByUserId(user.userId);
    if (!subscription) throw new ForbiddenError(...);
    if (subscription.status !== "trialing" && subscription.status !== "active") {
      throw new ForbiddenError(...);
    }
    return true;
  }
}
```

### Pattern 2: @SkipSubscriptionCheck() Decorator
**What:** `SetMetadata("skipSubscriptionCheck", true)` -- identical pattern to `@Public()`.
**Example:**
```typescript
// Source: project @Public() decorator pattern
import { SetMetadata } from "@nestjs/common";

export const SkipSubscriptionCheck = () => SetMetadata("skipSubscriptionCheck", true);
```

### Pattern 3: Stripe SDK Mocking in Tests
**What:** Mock the Stripe SDK at the provider level using the STRIPE_CLIENT injection token.
**Example:**
```typescript
// Source: project factory provider + jest mock patterns
const mockStripe = {
  subscriptions: {
    update: jest.fn(),
  },
  billingPortal: {
    sessions: {
      create: jest.fn(),
    },
  },
  customers: {
    create: jest.fn(),
  },
  checkout: {
    sessions: {
      create: jest.fn(),
      retrieve: jest.fn(),
    },
  },
  webhooks: {
    constructEvent: jest.fn(),
  },
};

// In Test.createTestingModule:
{
  provide: STRIPE_CLIENT,
  useValue: mockStripe,
}
```

### Pattern 4: SubscriptionMockGenerator
**What:** Static factory class following `BusinessMockGenerator` pattern.
**Example:**
```typescript
// Source: project BusinessMockGenerator pattern
export class SubscriptionMockGenerator {
  static createSubscriptionDto(overrides?: Partial<ISubscriptionDto>): ISubscriptionDto {
    return {
      id: new ObjectId().toString(),
      userId: "firebase-uid-123",
      stripeCustomerId: "cus_test123",
      stripeSubscriptionId: "sub_test123",
      status: SubscriptionStatus.ACTIVE,
      currentPeriodEnd: new Date("2026-05-01"),
      trialEnd: new Date("2026-04-29"),
      cancelAtPeriodEnd: false,
      canceledAt: undefined,
      createdAt: new Date(),
      updatedAt: new Date(),
      ...overrides,
    };
  }

  static createUserDto(overrides?: Partial<IUserDto>): IUserDto {
    // Delegate to or replicate UserMockGenerator pattern
    return {
      // ...standard user fields
      ...overrides,
    };
  }
}
```

### Anti-Patterns to Avoid
- **Testing Stripe SDK internals:** Mock at the injection boundary (STRIPE_CLIENT), not with `jest.mock("stripe")`. The factory provider pattern means the test module just provides a mock object.
- **Synchronous webhook processing in controller:** The webhook controller enqueues to BullMQ (Phase 30 architecture). Do not add synchronous processing.
- **Duplicating @Public() logic:** SubscriptionGuard should check the `isPublic` metadata key, not re-implement public route detection.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Stripe subscription cancellation | Custom cancel logic | `stripe.subscriptions.update({ cancel_at_period_end: true })` | Stripe handles prorations, billing cycle, and customer communication |
| Billing portal UI | Custom payment method management | `stripe.billingPortal.sessions.create()` | Stripe Customer Portal handles card updates, invoice history, PCI compliance |
| Guard metadata inspection | Manual route metadata parsing | `Reflector.getAllAndOverride()` | NestJS built-in utility, handles handler + class level metadata correctly |
| Test module DI | Manual class instantiation in tests | `Test.createTestingModule()` | NestJS testing module handles DI wiring, lifecycle hooks, and provider resolution |

**Key insight:** Phase 31 endpoints are thin wrappers around Stripe SDK calls. The guard is the most complex new code; the endpoints themselves are straightforward CRUD-style operations.

## Common Pitfalls

### Pitfall 1: Missing @subscription Path Alias in Jest
**What goes wrong:** Test files using `@subscription/*` imports fail with "Cannot find module" errors.
**Why it happens:** Phase 29 added the subscription module but the `moduleNameMapper` in `package.json` may not include `@subscription` or `@subscription-test` aliases. The documented aliases in CONVENTIONS.md don't include subscription yet.
**How to avoid:** Check `package.json` jest config for `@subscription` alias. If missing, add `"^@subscription/(.*)$": "<rootDir>/subscription/$1"` and `"^@subscription-test/(.*)$": "<rootDir>/subscription/test/$1"` to `moduleNameMapper`. Also check `tsconfig.json` paths.
**Warning signs:** First test file fails to compile with module resolution error.

### Pitfall 2: Guard Registration Order
**What goes wrong:** `SubscriptionGuard` runs before `JwtAuthGuard`, so `request.user` is undefined.
**Why it happens:** `APP_GUARD` providers execute in registration order within the module's providers array.
**How to avoid:** Register `SubscriptionGuard` after `JwtAuthGuard` in the `AppModule` providers array. Verify by checking that `request.user` is always populated when the guard runs (except for `@Public()` routes, which the guard skips first).
**Warning signs:** Guard throws errors about `request.user` being undefined on authenticated routes.

### Pitfall 3: Stripe subscriptions.update vs subscriptions.cancel
**What goes wrong:** Using `stripe.subscriptions.cancel()` immediately terminates the subscription instead of allowing access until period end.
**Why it happens:** Confusion between two Stripe API methods. `cancel()` deletes immediately; `update({ cancel_at_period_end: true })` schedules cancellation.
**How to avoid:** Always use `stripe.subscriptions.update(subId, { cancel_at_period_end: true })`. The CONTEXT.md decision D-04 explicitly specifies this.
**Warning signs:** Users lose access immediately after clicking "Cancel" instead of at period end.

### Pitfall 4: Testing the Wrong Layer for Webhook Signature
**What goes wrong:** Tests for `SubscriptionUpdaterService` include webhook signature verification assertions, bloating the service test and testing the wrong abstraction.
**Why it happens:** Natural inclination to test everything related to webhooks in one place.
**How to avoid:** Decision D-06 explicitly states: signature rejection is tested in the controller spec, not the updater service. The updater service receives already-validated events.
**Warning signs:** Updater service test importing Stripe webhook types or `constructEvent`.

### Pitfall 5: supportRoles Check in Guard
**What goes wrong:** Guard checks `user.supportRoles` with a truthy check but `supportRoles` might be a `DtoCollection` object that is always truthy even when empty.
**Why it happens:** `DtoCollection.empty()` returns an object, not null/undefined/[].
**How to avoid:** Check the actual shape of `supportRoles` on the `IUserDto` interface. If it's a `DtoCollection`, check `supportRoles.length > 0` or `supportRoles.items.length > 0` depending on the collection interface. If it's an array, check `.length > 0`.
**Warning signs:** All authenticated users bypass the subscription guard.

### Pitfall 6: Missing stripeSubscriptionId on Cancel
**What goes wrong:** DELETE handler tries to call `stripe.subscriptions.update(subscription.stripeSubscriptionId, ...)` but `stripeSubscriptionId` is optional/undefined if the webhook hasn't processed yet.
**Why it happens:** Subscription records created at checkout start with `stripeSubscriptionId` as undefined (populated by `checkout.session.completed` webhook in Phase 30).
**How to avoid:** Before calling Stripe, validate that `stripeSubscriptionId` exists. If not, the subscription hasn't been fully activated yet -- throw `InvalidRequestError`.
**Warning signs:** Stripe SDK throws "No such subscription: undefined" error.

## Code Examples

### GET /v1/subscription Handler
```typescript
// Source: project controller patterns + D-01
@Get()
@SkipSubscriptionCheck()
async getSubscription(@Req() req: Request) {
  try {
    const authUser = req.user as IUserDto;
    const subscription = await this.subscriptionRetriever.findByUser(authUser);
    return createResponse([subscription]); // Standard response wrapper
  } catch (error) {
    throw createHttpError(error);
  }
}
```

### DELETE /v1/subscription Handler
```typescript
// Source: project controller patterns + D-02, D-03, D-04
@Delete()
@SkipSubscriptionCheck()
async cancelSubscription(@Req() req: Request) {
  try {
    const authUser = req.user as IUserDto;
    const subscription = await this.subscriptionUpdater.cancelAtPeriodEnd(authUser);
    return createResponse([subscription]); // Returns full updated subscription
  } catch (error) {
    throw createHttpError(error);
  }
}
```

### POST /v1/subscription/portal Handler
```typescript
// Source: Stripe API docs + D-05
@Post("portal")
@SkipSubscriptionCheck()
async createPortalSession(@Req() req: Request) {
  try {
    const authUser = req.user as IUserDto;
    const result = await this.subscriptionCreator.createPortalSession(authUser);
    return createResponse(result); // { url: string }
  } catch (error) {
    throw createHttpError(error);
  }
}
```

### Cancel Service Method
```typescript
// Source: Stripe API + D-04
async cancelAtPeriodEnd(authUser: IUserDto): Promise<ISubscriptionDto> {
  const subscription = await this.subscriptionRepository.findByUserId(authUser.userId);
  if (!subscription || subscription.status === "canceled") {
    throw new InvalidRequestError(ErrorCodes.INVALID_REQUEST, "No active subscription to cancel");
  }

  // Update on Stripe
  const updated = await this.stripe.subscriptions.update(subscription.stripeSubscriptionId, {
    cancel_at_period_end: true,
  });

  // Optimistic local sync
  return await this.subscriptionRepository.upsertByUserId(authUser.userId, {
    cancelAtPeriodEnd: true,
    currentPeriodEnd: new Date(updated.current_period_end * 1000),
    updatedAt: new Date(),
  });
}
```

### Billing Portal Service Method
```typescript
// Source: Stripe API docs
async createPortalSession(authUser: IUserDto): Promise<{ url: string }> {
  const subscription = await this.subscriptionRepository.findByUserId(authUser.userId);
  if (!subscription) {
    throw new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "No subscription found");
  }

  const session = await this.stripe.billingPortal.sessions.create({
    customer: subscription.stripeCustomerId,
    return_url: `${this.configService.get<string>("FRONTEND_URL")}/settings?tab=billing`,
  });

  return { url: session.url };
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `stripe.subscriptions.cancel()` | `stripe.subscriptions.update({ cancel_at_period_end: true })` | Stripe recommends this for graceful cancellation | User retains access until period end |
| Stripe API version pinning | SDK v21.x pins to 2026-01-28 API version | Stripe v21 release | TypeScript types match the pinned API version |

## Open Questions

1. **@subscription Path Alias**
   - What we know: Phase 29 created the subscription module. The documented aliases in CONVENTIONS.md/TESTING.md don't include `@subscription`.
   - What's unclear: Whether Phase 29 execution added `@subscription/*` to `tsconfig.json` paths and jest `moduleNameMapper`.
   - Recommendation: Executor must check at task time and add if missing. Plan should include a conditional task or verification step.

2. **supportRoles Type Shape**
   - What we know: `IUserDto.supportRoles` exists and is documented as `DtoCollection.empty()` in mock generators.
   - What's unclear: Exact runtime shape -- is it an array, a DtoCollection object, or something else?
   - Recommendation: Executor must read `IUserDto` interface to determine the correct emptiness check for the guard.

3. **SubscriptionUpdater -- Cancel Method Placement**
   - What we know: `SubscriptionUpdaterService` handles webhook event processing (Phase 30). The cancel-at-period-end logic is a user-initiated action.
   - What's unclear: Whether `cancelAtPeriodEnd` belongs on the updater service or deserves a separate service class.
   - Recommendation: Place it on the updater service since it updates an existing subscription. This keeps the Creator (checkout + portal) / Retriever (GET) / Updater (webhook events + cancel) split clean.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | `package.json` jest field |
| Quick run command | `npm test -- --testPathPattern=subscription` |
| Full suite command | `npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BILL-01 | GET /v1/subscription returns subscription data | unit | `npm test -- --testPathPattern=subscription-retriever` | Wave 0 |
| BILL-02 | DELETE /v1/subscription cancels at period end | unit | `npm test -- --testPathPattern=subscription-updater` | Wave 0 |
| BILL-03 | POST /v1/subscription/portal returns portal URL | unit | `npm test -- --testPathPattern=subscription-creator` | Wave 0 |
| TEST-01 | SubscriptionCreatorService tests | unit | `npm test -- --testPathPattern=subscription-creator.service` | Wave 0 |
| TEST-02 | SubscriptionUpdaterService tests (5 events) + controller webhook spec | unit | `npm test -- --testPathPattern="subscription-updater.service\|subscription.controller"` | Wave 0 |
| TEST-03 | SubscriptionRetrieverService tests (found/not-found) | unit | `npm test -- --testPathPattern=subscription-retriever.service` | Wave 0 |
| TEST-04 | SubscriptionRepository tests (upsert, findByUserId, findByStripeSubscriptionId) | unit | `npm test -- --testPathPattern=subscription.repository` | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm test -- --testPathPattern=subscription`
- **Per wave merge:** `npm test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/subscription/test/mocks/subscription-mock-generator.ts` -- shared mock data factory
- [ ] `src/subscription/test/services/subscription-creator.service.spec.ts` -- covers TEST-01
- [ ] `src/subscription/test/services/subscription-retriever.service.spec.ts` -- covers TEST-03
- [ ] `src/subscription/test/services/subscription-updater.service.spec.ts` -- covers TEST-02
- [ ] `src/subscription/test/repositories/subscription.repository.spec.ts` -- covers TEST-04
- [ ] `src/subscription/test/controllers/subscription.controller.spec.ts` -- covers webhook signature test (TEST-02 partial)
- [ ] `src/subscription/test/guards/subscription.guard.spec.ts` -- covers D-09 through D-13
- [ ] Path alias `@subscription` and `@subscription-test` in jest moduleNameMapper (if not already present from Phase 29)

## Project Constraints (from CLAUDE.md)

- **Tech stack:** NestJS API with strict Controller-Service-Repository layering
- **Naming:** Services are `{Name}{Action}` (e.g., `SubscriptionCreator`), files are kebab-case with type suffix
- **Error handling:** Custom error classes (`ResourceNotFoundError`, `InvalidRequestError`, `ForbiddenError`) mapped to HTTP status via `createHttpError()`
- **Logging:** `new AppLogger(ClassName.name)` per class
- **Code style:** Double quotes, semicolons, 125 char width, trailing commas, 2-space indent
- **Test pattern:** AAA structure, `jest.Mocked<T>`, `Test.createTestingModule`, `afterEach(jest.clearAllMocks)`
- **Mock pattern:** Static generator classes (`SubscriptionMockGenerator`) with factory methods accepting `Partial<T>` overrides
- **Path aliases:** `@subscription/*` for imports (verify exists from Phase 29)
- **Guards:** Implement `CanActivate`, use `Reflector` for metadata, register as `APP_GUARD`

## Sources

### Primary (HIGH confidence)
- [Stripe API: Create Portal Session (Node.js)](https://docs.stripe.com/api/customer_portal/sessions/create?lang=node) -- Billing portal session creation API, parameters, response shape
- [Stripe API: Update Subscription (Node.js)](https://docs.stripe.com/api/subscriptions/update?lang=node) -- `cancel_at_period_end` parameter, response object with status/dates
- [NestJS Guards Documentation](https://docs.nestjs.com/guards) -- CanActivate interface, ExecutionContext, APP_GUARD, Reflector
- Phase 29 Research (`29-RESEARCH.md`) -- Stripe v21.0.1 version, STRIPE_CLIENT injection pattern
- Phase 29 CONTEXT.md -- Entity schema (D-07/D-08), module structure (D-12), enum values (D-09)
- Phase 30 CONTEXT.md -- Webhook processor architecture, upsert patterns, repository methods
- Phase 31 CONTEXT.md -- All 13 locked decisions governing this phase
- `TESTING.md` -- Jest patterns, mock generators, AAA structure, repository testing
- `CONVENTIONS.md` -- File naming, class naming, error handling, code style

### Secondary (MEDIUM confidence)
- [NestJS SetMetadata bypass pattern](https://dev.to/dannypule/exclude-route-from-nest-js-authgaurd-h0) -- Community-verified pattern for custom decorator metadata bypass
- [Stripe cancel_at_period_end changelog](https://docs.stripe.com/changelog/basil/2025-05-28/cancel-at-enums) -- Recent Stripe API changes around cancellation

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed, versions confirmed from Phase 29 research
- Architecture: HIGH -- patterns established in Phase 29/30, guard pattern well-documented in NestJS
- Pitfalls: HIGH -- derived from locked decisions and known project patterns
- Testing: HIGH -- follows existing test patterns documented in TESTING.md

**Research date:** 2026-03-29
**Valid until:** 2026-04-28 (stable -- no fast-moving dependencies)
