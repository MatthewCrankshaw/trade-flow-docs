# Architecture Research -- v1.9 Support & Admin Tools

**Domain:** Support user experience, user management dashboard, customer impersonation, role administration
**Researched:** 2026-04-18
**Confidence:** HIGH (grounded in existing Trade Flow codebase patterns and NestJS authorization documentation)

---

## Executive Summary

v1.9 is an **authorization-layer milestone, not a data-model milestone**. The existing codebase already has the support role on user entities (shipped in v1.6), SubscriptionGuard with `@SkipSubscriptionCheck` (v1.6/v1.7), and three-tier route guards in the UI (auth > onboarding > paywall). The work is about exposing admin surfaces that leverage existing data, adding a new impersonation mechanism, and building a role management API -- not creating new business entities.

The key architectural decisions are:

1. **A new `SupportGuard` (backend) and `SupportRoute` (frontend)** that gate admin-only surfaces, layered on top of the existing `JwtAuthGuard`. The guard reads the user's role from the existing `IUserDto` on `request.user` -- no new auth infrastructure needed.

2. **Impersonation via API-issued short-lived tokens, not Firebase custom claims.** The support user's own Firebase JWT authenticates the impersonation request; the API validates the support role, then returns a time-boxed impersonation context containing the target user's data. The frontend stores the impersonation context in Redux (not localStorage), swaps the business context, and renders the app as the target user -- but all API calls still carry the support user's real Firebase JWT plus an `X-Impersonate-User` header. The API resolves the effective user via middleware that checks the header, validates the caller is support, and injects the impersonated user's `IUserDto` into `request.user` while preserving `request.supportUser` for audit. No Firebase Admin SDK needed.

3. **A new `src/admin/` backend module** (not `src/support/`) following the Controller > Service > Repository pattern. This module owns: user listing with aggregated subscription/business data, role grant/revoke, and impersonation session management. It does NOT duplicate business/customer/job modules -- those are accessed through existing endpoints with the impersonated user context.

4. **Frontend support feature module** at `src/features/support/` with its own RTK Query API slice (`supportApi`), components, and hooks. Support pages already exist in `src/pages/support/` (SupportDashboardPage, SupportBusinessesPage, SupportUsersPage) -- these get wired to real data.

---

## Recommended Architecture

### Component Map

```
Frontend (trade-flow-ui)
========================
src/pages/support/
  SupportDashboardPage.tsx      [EXISTS - wire to real data]
  SupportBusinessesPage.tsx     [EXISTS - wire to real data]
  SupportUsersPage.tsx          [EXISTS - wire to real data]

src/features/support/
  api/supportApi.ts             [NEW - RTK Query endpoints for admin]
  components/
    UserManagementTable.tsx      [NEW - all users with membership summaries]
    UserDetailPanel.tsx          [NEW - single user detail with actions]
    ImpersonationBanner.tsx      [NEW - persistent banner during impersonation]
    RoleManagementDialog.tsx     [NEW - grant/revoke support role]
  hooks/
    useImpersonation.ts          [NEW - start/stop impersonation, context]
    useSupportUsers.ts           [NEW - fetch/filter user list]
  index.ts                       [NEW - barrel export]

src/features/auth/components/
  SupportRoute.tsx               [NEW - route guard for support-only pages]

src/store/slices/
  impersonationSlice.ts          [NEW - impersonation state in Redux]

src/components/layouts/
  DashboardLayout.tsx            [MODIFY - show impersonation banner]


Backend (trade-flow-api)
========================
src/admin/                       [NEW MODULE]
  admin.module.ts
  controllers/
    admin-user.controller.ts     [NEW - user listing, role mgmt]
    admin-impersonation.controller.ts [NEW - start/stop impersonation]
  services/
    admin-user-retriever.service.ts   [NEW - cross-business user queries]
    admin-role-manager.service.ts     [NEW - grant/revoke support role]
    impersonation-session.service.ts  [NEW - session creation/validation]
  guards/
    support-role.guard.ts        [NEW - @SupportOnly decorator + guard]
  decorators/
    support-only.decorator.ts    [NEW - SetMetadata decorator]
    effective-user.decorator.ts  [NEW - extracts impersonated or real user]
  middleware/
    impersonation.middleware.ts  [NEW - resolves X-Impersonate-User header]
  data-transfer-objects/
    admin-user.dto.ts            [NEW - user + subscription + business summary]
  responses/
    admin-user.response.ts       [NEW - response shape for user listing]
  test/
    controllers/
    services/
    mocks/

src/auth/
  auth.guard.ts                  [MODIFY - expose support role check helper]

src/user/
  repositories/
    user.repository.ts           [MODIFY - add findAll with pagination]
  services/
    user-retriever.service.ts    [MODIFY - add cross-business retrieval for admin]

src/app.module.ts                [MODIFY - import AdminModule]
```

### Data Flow

#### Support User Login (Bypass Onboarding)

```
1. Support user authenticates via Firebase (existing flow)
2. Backend JwtAuthGuard validates JWT, extracts user from DB
3. User entity has role = "support" (set via migration or role mgmt)
4. Frontend checks user.role after login:
   - If "support" AND no business associated:
     - Skip onboarding wizard entirely
     - Redirect to /support (not /dashboard)
   - If "support" AND has a business:
     - Normal app access PLUS /support routes available
5. OnboardingGuard (frontend) checks: if user.role === "support", bypass
6. PaywallGuard (frontend) checks: already bypassed (v1.6 SubscriptionGuard
   has support role check)
```

The frontend three-tier guard chain (auth > onboarding > paywall) needs a single modification: the onboarding guard must check for support role and skip if present. The paywall guard already handles this (v1.6).

#### User Management Dashboard

```
Frontend:
1. SupportUsersPage mounts, calls useGetAdminUsersQuery(filters)
2. RTK Query: GET /v1/admin/users?page=1&limit=50&search=...

Backend:
1. AdminUserController.getUsers() guarded by @SupportOnly + JwtAuthGuard
2. SupportRoleGuard validates request.user.role === "support"
3. AdminUserRetriever.findAll(filters, pagination):
   - Queries users collection (all users, not business-scoped)
   - For each user, aggregates:
     a. Business association from business_users collection
     b. Subscription status from subscriptions collection
     c. Last login timestamp
   - Returns paginated AdminUserDto[]
4. Controller maps to AdminUserResponse[]
5. Returns createResponse(responses, pagination)

Frontend:
1. UserManagementTable renders with search, filter, sort
2. Each row shows: name, email, role, subscription status, business name,
   last login, created date
3. Row actions: View Details, Impersonate, Manage Role
```

#### Customer Impersonation Flow

```
Start Impersonation:
1. Support user clicks "Impersonate" on user row
2. UI dispatches startImpersonation(targetUserId)
3. RTK Query: POST /v1/admin/impersonation/start
   Body: { targetUserId }
   Headers: Authorization: Bearer <support-user-firebase-jwt>

Backend:
1. ImpersonationController.start() guarded by @SupportOnly
2. ImpersonationSessionService.create(supportUser, targetUserId):
   a. Validates target user exists
   b. Validates target is NOT a support user (cannot impersonate support)
   c. Creates impersonation_sessions document:
      { supportUserId, targetUserId, startedAt, expiresAt (1h), active: true }
   d. Logs audit event: "IMPERSONATION_START" with both user IDs
   e. Returns target user's IUserDto + their business data + session ID
3. Returns { sessionId, targetUser, targetBusiness }

Frontend:
4. impersonationSlice stores: { active: true, sessionId, targetUser,
   targetBusiness, supportUser: <original user> }
5. RTK Query prepareHeaders detects impersonation active:
   - Keeps Authorization: Bearer <support-user-firebase-jwt>
   - Adds X-Impersonate-User: <targetUserId>
   - Adds X-Impersonation-Session: <sessionId>
6. useCurrentBusiness hook returns targetBusiness instead of support user's
7. ImpersonationBanner renders at top of page:
   "Viewing as [User Name] (user@email.com) — [End Session]"
8. All subsequent API calls carry impersonation headers

Backend (on impersonated requests):
1. ImpersonationMiddleware (runs before guards):
   a. Checks X-Impersonate-User header exists
   b. If present, validates:
      - Caller (from JWT) has support role
      - Session ID is valid and not expired
      - Target user matches session
   c. If valid:
      - request.user = target user's IUserDto
      - request.supportUser = caller's IUserDto (for audit)
      - request.impersonationSession = session data
   d. If invalid: 403 Forbidden
2. All downstream services (CustomerRetriever, JobRetriever, etc.) see
   request.user as the target user -- existing policy checks work unchanged
3. Policies see the impersonated user, so business ownership checks pass

Stop Impersonation:
1. Support user clicks "End Session" on banner
2. POST /v1/admin/impersonation/:sessionId/end
3. Session marked inactive, endedAt timestamp set
4. Frontend clears impersonationSlice, restores supportUser context
5. Redirects to /support
```

#### Role Management Flow

```
Grant Support Role:
1. Support user opens RoleManagementDialog for target user
2. POST /v1/admin/users/:userId/roles
   Body: { role: "support" }
3. AdminRoleManager.grantRole(supportUser, targetUserId, role):
   a. Validates caller is support (guard already checked)
   b. Validates target user exists
   c. Validates target doesn't already have role
   d. Updates user entity: role = "support"
   e. Logs audit: "ROLE_GRANTED" with granter + grantee + role
   f. Returns updated user
4. UI invalidates user list cache

Revoke Support Role:
1. DELETE /v1/admin/users/:userId/roles/support
2. Same validation + cannot revoke own role
3. Sets role back to default (e.g., "user")
4. Logs audit: "ROLE_REVOKED"
```

---

## New Components (Detailed)

### Backend: AdminModule (`src/admin/`)

**Why a new module, not extending UserModule:**
- UserModule owns self-service user operations (get own profile, update own profile)
- AdminModule owns cross-tenant operations (list all users, manage roles, impersonation)
- Separation of concerns: different guards, different policies, different query patterns
- AdminModule depends on UserRepository (read) but does NOT share controllers

**Module structure follows existing conventions:**

```
src/admin/
  admin.module.ts               # Imports: CoreModule, UserModule (for UserRepository)
  controllers/
    admin-user.controller.ts    # @Controller('v1/admin')
    admin-impersonation.controller.ts  # @Controller('v1/admin/impersonation')
  services/
    admin-user-retriever.service.ts    # Cross-business user queries with aggregation
    admin-role-manager.service.ts      # Role grant/revoke with audit
    impersonation-session.service.ts   # Session CRUD + validation
  repositories/
    impersonation-session.repository.ts # MongoDB impersonation_sessions collection
  guards/
    support-role.guard.ts       # CanActivate: checks request.user.role === "support"
  decorators/
    support-only.decorator.ts   # @SupportOnly() = @UseGuards(JwtAuthGuard, SupportRoleGuard)
    effective-user.decorator.ts # @EffectiveUser() param decorator
  middleware/
    impersonation.middleware.ts # NestMiddleware: resolves X-Impersonate-User
  entities/
    impersonation-session.entity.ts
  data-transfer-objects/
    admin-user.dto.ts           # IAdminUserDto: user + subscription + business summary
    impersonation-session.dto.ts
  responses/
    admin-user.response.ts
    impersonation-session.response.ts
  enums/
    audit-action.enum.ts        # IMPERSONATION_START, IMPERSONATION_END, ROLE_GRANTED, ROLE_REVOKED
  test/
    controllers/
    services/
    mocks/
```

### Backend: SupportRoleGuard

```typescript
// src/admin/guards/support-role.guard.ts
@Injectable()
export class SupportRoleGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user; // Set by JwtAuthGuard (runs first)
    if (!user || user.role !== UserRole.SUPPORT) {
      throw new ForbiddenError("Support role required");
    }
    return true;
  }
}
```

**Guard ordering:** `@UseGuards(JwtAuthGuard, SupportRoleGuard)` -- JwtAuthGuard populates `request.user`, then SupportRoleGuard checks the role. This matches the existing pattern where JwtAuthGuard runs first on all protected routes.

### Backend: @SupportOnly Composite Decorator

```typescript
// src/admin/decorators/support-only.decorator.ts
export const SupportOnly = () =>
  applyDecorators(
    UseGuards(JwtAuthGuard, SupportRoleGuard),
    SkipSubscriptionCheck(),  // Support users bypass subscription (v1.6 decision)
  );
```

**Why composite:** Every admin endpoint needs both auth + support role + subscription bypass. Repeating three decorators per method is error-prone. The composite decorator makes it a single annotation.

### Backend: ImpersonationMiddleware

```typescript
// src/admin/middleware/impersonation.middleware.ts
@Injectable()
export class ImpersonationMiddleware implements NestMiddleware {
  constructor(
    private readonly userRetriever: UserRetriever,
    private readonly sessionService: ImpersonationSessionService,
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const impersonateUserId = req.headers["x-impersonate-user"];
    const sessionId = req.headers["x-impersonation-session"];

    if (!impersonateUserId || !sessionId) {
      return next(); // Not an impersonation request, proceed normally
    }

    // req.user is NOT yet populated (middleware runs before guards)
    // Store headers for later validation by the guard pipeline
    req["impersonationHeaders"] = { impersonateUserId, sessionId };
    return next();
  }
}
```

**Critical design decision: middleware sets headers, guard validates.**

Middleware runs before guards in NestJS. At middleware time, `request.user` is not yet populated by JwtAuthGuard. Two approaches were considered:

1. **Middleware does full validation** -- requires duplicating JWT validation logic or injecting JwtStrategy. Rejected: violates single-responsibility.

2. **Middleware stores headers, post-auth interceptor resolves** -- cleaner. An interceptor (or a second guard in the chain) runs after JwtAuthGuard, validates the impersonation headers against the now-populated `request.user`, and swaps the user context. This is the recommended approach.

Revised flow:

```
Request arrives
  -> ImpersonationMiddleware stores raw headers on request
  -> JwtAuthGuard validates Firebase JWT, sets request.user (the support user)
  -> ImpersonationGuard (runs after JwtAuthGuard):
      - Reads impersonation headers from request
      - Validates request.user.role === "support"
      - Validates session is active and not expired
      - Loads target user from UserRepository
      - Sets request.supportUser = request.user (original support user)
      - Sets request.user = target user (impersonated user)
  -> Controller/Service sees request.user as the impersonated user
  -> Audit logging uses request.supportUser
```

**Why this is safe:** The impersonation swap happens INSIDE the NestJS guard pipeline, after authentication is confirmed. The support user's real Firebase JWT is always validated first. The target user is loaded from the database, not from a client-supplied token.

### Backend: ImpersonationGuard

```typescript
// src/admin/guards/impersonation.guard.ts
@Injectable()
export class ImpersonationGuard implements CanActivate {
  constructor(
    private readonly userRetriever: UserRetriever,
    private readonly sessionService: ImpersonationSessionService,
  ) {}

  async canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const headers = request["impersonationHeaders"];

    if (!headers) return true; // Not impersonating, pass through

    const callerUser = request.user;
    if (!callerUser || callerUser.role !== UserRole.SUPPORT) {
      throw new ForbiddenError("Impersonation requires support role");
    }

    const session = await this.sessionService.validate(
      headers.sessionId,
      callerUser.id,
      headers.impersonateUserId,
    );

    const targetUser = await this.userRetriever.findByIdOrFail(
      headers.impersonateUserId,
    );

    request.supportUser = callerUser;
    request.user = targetUser;
    request.impersonationSession = session;

    return true;
  }
}
```

**Global vs per-route:** Register ImpersonationGuard globally (after JwtAuthGuard) so impersonation works on ALL existing endpoints without modifying them. This is the key architectural advantage -- existing CustomerController, JobController, QuoteController etc. all work unchanged because `request.user` is the impersonated user.

### Backend: Impersonation Session Entity

```typescript
// New MongoDB collection: impersonation_sessions
interface IImpersonationSessionEntity {
  _id: ObjectId;
  supportUserId: string;      // The support user who initiated
  targetUserId: string;       // The user being impersonated
  startedAt: DateTime;        // Luxon DateTime
  expiresAt: DateTime;        // startedAt + 1 hour
  endedAt?: DateTime;         // When manually ended
  active: boolean;            // false when ended or expired
  createdAt: DateTime;
  updatedAt: DateTime;
}
```

**TTL:** Sessions auto-expire after 1 hour. The `expiresAt` field is checked on every impersonated request. A MongoDB TTL index cleans up old session documents after 30 days.

**Indexes:**
- `{ supportUserId: 1, active: 1 }` -- find active sessions for a support user
- `{ targetUserId: 1, active: 1 }` -- check if a user is currently being impersonated
- `{ expiresAt: 1 }` -- TTL index for cleanup

### Backend: Admin User Aggregation

The user management dashboard needs data from three collections in a single response:

```typescript
// IAdminUserDto combines:
interface IAdminUserDto {
  // From users collection
  id: string;
  externalId: string;         // Firebase UID
  email: string;
  displayName: string;
  role: UserRole;
  status: UserStatus;
  onboardingStep: OnboardingStep;
  createdAt: DateTime;

  // From business_users + businesses collections (aggregated)
  business?: {
    id: string;
    name: string;
    trade: PrimaryTrade;
    status: BusinessStatus;
  };

  // From subscriptions collection (aggregated)
  subscription?: {
    status: SubscriptionStatus;   // active, trialing, past_due, canceled, etc.
    currentPeriodEnd: DateTime;
    cancelAtPeriodEnd: boolean;
  };
}
```

**Aggregation approach:** MongoDB aggregation pipeline in `AdminUserRetriever` with `$lookup` to join business_users, businesses, and subscriptions in a single query. This avoids N+1 queries on the user list.

```
users collection
  $lookup -> business_users (by userId)
  $lookup -> businesses (by businessId from business_users)
  $lookup -> subscriptions (by businessId)
  $project -> flatten into IAdminUserDto shape
```

**Pagination:** Mandatory. Default limit 50, max 200. Search across email and displayName via `$regex` (case-insensitive). Filter by role, subscription status, business trade.

### Frontend: SupportRoute Guard

```typescript
// src/features/auth/components/SupportRoute.tsx
// Route guard that checks user.role === "support"
// Renders children if support, redirects to /dashboard if not
// Sits alongside ProtectedRoute in the auth feature
```

**Route structure in App.tsx:**

```
/login              -> LoginPage (public)
/                   -> LandingPage (public)

/dashboard          -> ProtectedRoute > OnboardingGuard > PaywallGuard > DashboardPage
/customers          -> ProtectedRoute > OnboardingGuard > PaywallGuard > CustomersPage
...

/support            -> ProtectedRoute > SupportRoute > SupportDashboardPage
/support/users      -> ProtectedRoute > SupportRoute > SupportUsersPage
/support/businesses -> ProtectedRoute > SupportRoute > SupportBusinessesPage
```

Support routes skip OnboardingGuard and PaywallGuard entirely -- support users may not have a business and are exempt from billing.

### Frontend: Impersonation State (Redux)

```typescript
// src/store/slices/impersonationSlice.ts
interface ImpersonationState {
  active: boolean;
  sessionId: string | null;
  targetUser: UserResponse | null;
  targetBusiness: BusinessResponse | null;
  supportUser: UserResponse | null;  // The real logged-in support user
}
```

**Key integration points:**

1. **`useCurrentBusiness` hook** -- when impersonation is active, returns `targetBusiness` instead of the support user's business. This makes all existing feature hooks (useCustomersList, useJobsList, etc.) automatically fetch the impersonated user's data.

2. **`prepareHeaders` in api.ts** -- when impersonation is active, adds `X-Impersonate-User` and `X-Impersonation-Session` headers to every API call. The Firebase JWT remains the support user's own token.

3. **`DashboardLayout`** -- renders `ImpersonationBanner` at the top when active. Banner shows target user's name/email, a prominent "End Session" button, and is styled distinctively (e.g., amber background) so the support user never forgets they are impersonating.

4. **Navigation** -- when impersonating, hide /support nav items. Show the standard user navigation. Add "End Impersonation" to the header dropdown.

---

## Patterns to Follow

### Pattern 1: Composite Guard Decorator

**What:** Combine multiple guards and decorators into a single annotation.
**When:** All endpoints in a controller share the same guard requirements.

```typescript
// Instead of repeating on every method:
@UseGuards(JwtAuthGuard, SupportRoleGuard)
@SkipSubscriptionCheck()

// Use composite:
@SupportOnly()
```

This follows the existing `@SkipSubscriptionCheck` pattern from v1.6 but combines it with guards.

### Pattern 2: Global Guard with Conditional Logic

**What:** Register ImpersonationGuard globally so it runs on every authenticated request, but no-ops when impersonation headers are absent.
**When:** A cross-cutting concern must apply to all existing endpoints without modifying them.

```typescript
// In admin.module.ts or app.module.ts:
{
  provide: APP_GUARD,
  useClass: ImpersonationGuard,
}
```

**Why this works:** ImpersonationGuard checks for `X-Impersonate-User` header. If absent, returns `true` immediately (no-op). If present, validates and swaps `request.user`. Every existing controller automatically supports impersonation without code changes.

### Pattern 3: Aggregation Pipeline for Cross-Collection Queries

**What:** Use MongoDB `$lookup` to join users with their business and subscription data in a single query.
**When:** Admin dashboard needs to display denormalized data from multiple collections.

This is new for Trade Flow (existing queries are single-collection), but appropriate for the admin context where cross-tenant visibility is required.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Firebase Custom Claims for Impersonation

**What:** Using Firebase Admin SDK to create custom tokens or set custom claims to impersonate users.
**Why bad:** (a) Trade Flow backend does not use Firebase Admin SDK -- adding it is a new dependency for a single feature. (b) Custom claims propagation has up to 1-hour delay unless the client forces a token refresh. (c) Creates a Firebase-side state that must be cleaned up on session end. (d) Mixing Firebase auth state with application-level impersonation makes debugging harder.
**Instead:** Keep Firebase JWT as the support user's identity. Impersonation is an application-layer concept resolved by the backend from HTTP headers.

### Anti-Pattern 2: Separate Admin API Service/Deployment

**What:** Deploying admin endpoints as a separate NestJS application.
**Why bad:** The admin module needs access to the same MongoDB instance, the same user/business/subscription repositories, and the same guard infrastructure. A separate deployment doubles ops burden for no security gain (the guard check is equivalent).
**Instead:** AdminModule lives in the same API process, gated by SupportRoleGuard.

### Anti-Pattern 3: Duplicating Business Logic for Admin Views

**What:** Creating AdminCustomerRetriever, AdminJobRetriever, etc. that bypass policies.
**Why bad:** Policy bypass is exactly what impersonation solves. When impersonating, `request.user` IS the target user, so existing policy checks pass naturally. Duplicating retrievers creates a parallel code path that diverges from the real user experience.
**Instead:** Impersonation swaps the user context; existing services work unchanged.

### Anti-Pattern 4: Storing Impersonation State in localStorage

**What:** Persisting impersonation session in browser localStorage so it survives page refresh.
**Why bad:** Security risk -- if a support user walks away from their machine, anyone with browser access continues as the impersonated user. Impersonation should require re-initiation after page refresh or timeout.
**Instead:** Store in Redux (in-memory only). Page refresh clears the impersonation state. Support user must re-initiate from the /support page.

---

## Integration Points with Existing Code

### Files Modified (Not New)

| File | Change | Why |
|------|--------|-----|
| `src/app.module.ts` | Import AdminModule | Register the new module |
| `src/auth/auth.guard.ts` | No change needed | JwtAuthGuard already populates request.user; support role check is downstream |
| `src/user/repositories/user.repository.ts` | Add `findAll(filters, pagination)` method | User listing for admin dashboard; currently only has findById/findByExternalId |
| `trade-flow-ui/src/services/api.ts` | Add impersonation headers in prepareHeaders | Conditional header injection when impersonation is active |
| `trade-flow-ui/src/hooks/useCurrentBusiness.ts` | Check impersonation state first | Return impersonated business when active |
| `trade-flow-ui/src/App.tsx` | Add SupportRoute guard, new /support routes | Route configuration for admin pages |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` | Render ImpersonationBanner | Visual indicator during impersonation |
| `trade-flow-ui/src/store/index.ts` | Add impersonationSlice reducer | Redux store configuration |
| Frontend onboarding guard | Skip for support role users | Allow support users without business to access app |

### Files NOT Modified

| File | Why Unchanged |
|------|---------------|
| `src/customer/controllers/customer.controller.ts` | Impersonation handled by global guard; controller sees impersonated user naturally |
| `src/job/controllers/job.controller.ts` | Same reason |
| `src/quote/controllers/quote.controller.ts` | Same reason |
| `src/business/policies/business.policy.ts` | Policy checks impersonated user's business ownership; works correctly |
| `src/subscription/guards/subscription.guard.ts` | Support role bypass already exists (v1.6) |
| All existing service files | No changes needed; they operate on request.user which is swapped by ImpersonationGuard |

### New MongoDB Collections

| Collection | Purpose | Indexes |
|------------|---------|---------|
| `impersonation_sessions` | Audit trail + active session validation | `(supportUserId, active)`, `(targetUserId, active)`, TTL on `expiresAt` |

### New API Endpoints

| Method | Path | Guard | Purpose |
|--------|------|-------|---------|
| GET | `/v1/admin/users` | @SupportOnly | Paginated user list with aggregated data |
| GET | `/v1/admin/users/:id` | @SupportOnly | Single user detail |
| POST | `/v1/admin/users/:id/roles` | @SupportOnly | Grant role |
| DELETE | `/v1/admin/users/:id/roles/:role` | @SupportOnly | Revoke role |
| POST | `/v1/admin/impersonation/start` | @SupportOnly | Start impersonation session |
| POST | `/v1/admin/impersonation/:sessionId/end` | @SupportOnly | End impersonation session |
| GET | `/v1/admin/impersonation/active` | @SupportOnly | Get current active session (if any) |

---

## Scalability Considerations

| Concern | At 10 users | At 1K users | At 10K users |
|---------|-------------|-------------|--------------|
| User listing query | Instant; no index needed | Fine with `$lookup`; ensure indexes on business_users.userId | Aggregation pipeline may slow; add pre-computed materialized view or cache user summaries |
| Impersonation sessions | In-memory validation fast | Same | TTL index keeps collection small; no concern |
| Concurrent impersonation | N/A with <5 support staff | Support staff < 10; no concern | Still fine; sessions are lightweight |

---

## Build Order (Dependency-Driven)

The build order reflects strict dependencies:

```
Phase 1: SupportGuard + Admin Module Shell + Support Login Bypass
  (No dependencies on other phases; enables all subsequent work)
  Delivers: @SupportOnly guard, AdminModule registered, support users
            bypass onboarding in frontend

Phase 2: User Management Dashboard (Backend)
  (Depends on Phase 1 for guard)
  Delivers: GET /v1/admin/users with aggregation pipeline,
            GET /v1/admin/users/:id

Phase 3: User Management Dashboard (Frontend)
  (Depends on Phase 2 for API)
  Delivers: SupportUsersPage wired to real data, UserManagementTable,
            search/filter/sort, UserDetailPanel

Phase 4: Role Management
  (Depends on Phase 2 for user retrieval)
  Delivers: POST/DELETE role endpoints, RoleManagementDialog,
            cannot-revoke-own-role validation

Phase 5: Impersonation (Backend)
  (Depends on Phase 1 for guard infrastructure)
  Delivers: ImpersonationMiddleware, ImpersonationGuard (global),
            session CRUD, start/end endpoints, audit logging

Phase 6: Impersonation (Frontend)
  (Depends on Phase 5 for API, Phase 3 for user list as entry point)
  Delivers: impersonationSlice, useImpersonation hook,
            ImpersonationBanner, prepareHeaders integration,
            useCurrentBusiness override, "End Session" flow

Phase 7: Polish + Testing
  Delivers: E2E tests, edge cases (expired session handling,
            concurrent impersonation prevention, role change during
            active session), audit log review
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 6 (user management must exist before impersonation UI).
**Parallel track:** Phase 4 (role mgmt) can run alongside Phase 3.
**Phase 5 (impersonation backend) can run alongside Phase 2/3** since it only depends on Phase 1.

---

## Sources

- [NestJS Guards Documentation](https://docs.nestjs.com/guards) -- guard execution order, CanActivate interface, SetMetadata
- [NestJS Authorization Documentation](https://docs.nestjs.com/security/authorization) -- role-based guard patterns, Reflector usage
- [Firebase Custom Claims](https://firebase.google.com/docs/auth/admin/custom-claims) -- evaluated and rejected for impersonation (see anti-patterns)
- [Pigment Engineering -- Safe User Impersonation](https://engineering.pigment.com/2026/04/08/safe-user-impersonation/) -- dual-identity token pattern, audit trail design
- [NestJS Middleware Documentation](https://docs.nestjs.com/middleware) -- middleware vs guard execution order
- Trade Flow source code: `src/auth/auth.guard.ts`, `src/subscription/guards/subscription.guard.ts`, `src/user/entities/user.entity.ts`, `src/core/policies/policy.ts` -- existing patterns
- Trade Flow `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/STRUCTURE.md` -- module organization patterns

---

*Architecture research for: Trade Flow v1.9 Support & Admin Tools*
*Researched: 2026-04-18*
*Confidence: HIGH -- all patterns grounded in existing Trade Flow conventions and NestJS documentation*
