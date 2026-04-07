# Phase 40: SubscriptionGuard Onboarding Bypass - Research

**Researched:** 2026-04-07
**Domain:** NestJS global guard bypass, decorator-based route exemption
**Confidence:** HIGH

## Summary

Phase 40 closes the critical integration gap INT-01 identified in the v1.7 milestone audit: the global `SubscriptionGuard` (registered as `APP_GUARD` in `app.module.ts`) blocks all non-GET requests for users without a subscription, which prevents new users from completing onboarding. Specifically, two onboarding endpoints are blocked: `PATCH /v1/user/me` (save display name, Step 1) and `POST /v1/user/me/business` (create business, Step 2). The third onboarding call, `POST /v1/subscription/trial`, already has `@SkipSubscriptionCheck()` applied (from Phase 35).

The fix is surgical: add the existing `@SkipSubscriptionCheck()` decorator to exactly two controller methods. The decorator and guard infrastructure were built in Phase 31 and are already in use on all subscription controller routes and the WebhookController. No new patterns, libraries, or architectural changes are needed. This is a two-line code change with corresponding unit test updates to verify the bypass works correctly.

**Primary recommendation:** Add `@SkipSubscriptionCheck()` to `UserController.patch()` and `BusinessController.create()` methods. Write unit tests verifying the guard allows these requests without a subscription. Verify all other write endpoints remain guarded.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TRIAL-01 | User's 30-day free trial starts automatically after business setup without entering payment details | Trial endpoint already has @SkipSubscriptionCheck; this phase unblocks the two preceding onboarding steps so the flow reaches trial creation |
| ONBD-01 | New user required to enter display name before accessing app | PATCH /v1/user/me must be accessible without subscription -- add @SkipSubscriptionCheck to UserController.patch() |
| ONBD-02 | New user required to enter business name and select trade before accessing app | POST /v1/user/me/business must be accessible without subscription -- add @SkipSubscriptionCheck to BusinessController.create() |
| ONBD-03 | Country defaults to UK and currency defaults to GBP during business setup | Business creation endpoint must be reachable -- blocked by guard without this fix |
| ONBD-04 | Tax rates, job types, visit types, items, quote email template auto-created on business setup | Business creation triggers default resource creation -- blocked by guard without this fix |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/core | 11.1.12 | NestJS framework (guards, decorators, DI) | Already installed, project framework [VERIFIED: project CLAUDE.md] |
| @nestjs/common | 11.1.12 | SetMetadata, CanActivate, ExecutionContext | Already installed [VERIFIED: project CLAUDE.md] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jest | 30.2.0 | Unit testing guard behavior | Already installed [VERIFIED: project CLAUDE.md] |
| @nestjs/testing | 11.1.12 | NestJS test module for guard specs | Already installed [VERIFIED: project CLAUDE.md] |

**Installation:** No new packages needed. All dependencies already installed.

## Architecture Patterns

### Existing Guard Infrastructure (Phase 31)

The SubscriptionGuard was built in Phase 31 (v1.6) with the following architecture [VERIFIED: Phase 31 plan and summary]:

**Guard file:** `trade-flow-api/src/subscription/guards/subscription.guard.ts`
**Decorator file:** `trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts`
**Registration:** `trade-flow-api/src/app.module.ts` line 59 as `{ provide: APP_GUARD, useClass: SubscriptionGuard }`

**Guard logic (canActivate):**
1. Check `isPublic` metadata -- if true, return true (public routes)
2. Check `skipSubscriptionCheck` metadata -- if true, return true
3. If HTTP method is GET or HEAD -- return true (reads always allowed)
4. If user has `supportRoles` -- return true (support bypass)
5. DB lookup: `subscriptionRepository.findByUserId(user.externalAuthUserId)`
6. If no subscription OR status not TRIALING/ACTIVE -- throw `ForbiddenError`
7. Return true

**Decorator:**
```typescript
// Source: Phase 31 plan, file: skip-subscription-check.decorator.ts
import { SetMetadata } from "@nestjs/common";
export const SkipSubscriptionCheck = () => SetMetadata("skipSubscriptionCheck", true);
```

### Pattern: Adding @SkipSubscriptionCheck to Existing Routes

**What:** Apply the existing decorator to controller methods that must be accessible during onboarding
**When to use:** When a specific write endpoint must work for users without a subscription
**Example:**
```typescript
// Source: Established pattern from Phase 31 (subscription.controller.ts)
// Already used on: @Post("checkout"), @Get("verify-session"), @Get(), @Delete(), @Post("portal"), @Post("trial")
import { SkipSubscriptionCheck } from "@subscription/decorators/skip-subscription-check.decorator";

@SkipSubscriptionCheck()
@Patch("me")
async patch(@Req() req: Request, @Body() body: UpdateUserRequest) {
  // ... existing implementation unchanged
}
```

### Files to Modify

```
trade-flow-api/
  src/
    user/
      controllers/
        user.controller.ts          # ADD @SkipSubscriptionCheck() to patch() method
    business/
      controllers/
        business.controller.ts      # ADD @SkipSubscriptionCheck() to create() method
    subscription/
      decorators/
        skip-subscription-check.decorator.ts  # READ ONLY -- import from here
      guards/
        subscription.guard.ts       # READ ONLY -- verify guard logic
      test/
        guards/
          subscription.guard.spec.ts  # MODIFY -- add test cases for new bypass routes
```

### Anti-Patterns to Avoid
- **Do NOT modify the guard logic itself:** The fix is adding decorators to controller methods, not changing the guard's canActivate logic. The guard already respects @SkipSubscriptionCheck. [VERIFIED: Phase 31 plan]
- **Do NOT add @SkipSubscriptionCheck at the class level on UserController or BusinessController:** This would exempt ALL routes in those controllers (including update, delete, etc.). Only the specific onboarding methods should be exempted. [ASSUMED -- class-level decorator would bypass guard for all methods]
- **Do NOT create a new decorator or guard:** The existing `@SkipSubscriptionCheck()` pattern is the established mechanism. Reuse it.
- **Do NOT remove the guard from app.module.ts:** The global guard must remain active for all other write endpoints.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Guard bypass mechanism | Custom middleware or conditional logic in guard | @SkipSubscriptionCheck decorator | Already established in Phase 31, used on 6+ routes |
| Route-specific exemption | Whitelist array in guard config | SetMetadata decorator pattern | NestJS standard pattern; decouples guard from route knowledge |

**Key insight:** This phase requires zero new code patterns. The decorator, guard, and bypass mechanism all exist and are tested. The only work is applying the decorator to two additional controller methods and verifying correctness.

## Common Pitfalls

### Pitfall 1: Applying Decorator at Class Level Instead of Method Level
**What goes wrong:** Adding `@SkipSubscriptionCheck()` to the `UserController` class exempts ALL user routes (including future write endpoints) from subscription enforcement.
**Why it happens:** Class-level decorators are less typing and feel cleaner. But NestJS guard metadata resolution checks both handler and class level.
**How to avoid:** Apply `@SkipSubscriptionCheck()` ONLY to the specific method (`patch()` in UserController, `create()` in BusinessController). Other methods in these controllers must remain guarded.
**Warning signs:** Users without subscriptions can update their profile fields beyond onboarding, or create multiple businesses.

### Pitfall 2: Wrong Import Path for Decorator
**What goes wrong:** Import fails or resolves to wrong module because of path alias misconfiguration.
**Why it happens:** The decorator lives at `@subscription/decorators/skip-subscription-check.decorator` using the jest path alias convention. In actual source imports, use the tsconfig path alias.
**How to avoid:** Check the existing import pattern in `subscription.controller.ts` (where the decorator is already used) and replicate it exactly.
**Warning signs:** TypeScript compilation error on the import statement.

### Pitfall 3: Missing Test Updates for Guard Spec
**What goes wrong:** The guard tests don't cover the new bypass routes, so a future refactor could accidentally remove the decorator without test failure.
**Why it happens:** The decorator is applied on the controller, not the guard. Existing guard tests may not exercise routes from other controllers.
**How to avoid:** Add test cases to the guard spec (or to the user/business controller specs) that verify: (a) PATCH /v1/user/me returns success without a subscription, (b) POST /v1/user/me/business returns success without a subscription.
**Warning signs:** No test fails if the decorator is removed.

### Pitfall 4: Confusing Endpoint Paths
**What goes wrong:** Developer applies decorator to the wrong method because the audit mentions `/v1/user/me` but the controller structure uses relative paths.
**Why it happens:** The user controller is mounted at `/v1/user` and the `patch()` method handles the `me` sub-route. Similarly, business controller is at `/v1/business` with a `create()` method.
**How to avoid:** Read the actual controller files before modifying. Verify the method that handles the specific route used during onboarding.
**Warning signs:** Decorator is on the right class but wrong method.

## Code Examples

### UserController -- Add Decorator to patch() Method
```typescript
// Source: Established pattern from subscription.controller.ts (Phase 31)
// File: trade-flow-api/src/user/controllers/user.controller.ts
import { SkipSubscriptionCheck } from "@subscription/decorators/skip-subscription-check.decorator";

// Add decorator above existing @Patch("me") route
@SkipSubscriptionCheck()
@Patch("me")
@UseGuards(JwtAuthGuard)
async patch(@AuthUser() authUser: IUserDto, @Body() body: UpdateUserRequest): Promise<IResponse<IUserResponse>> {
  // ... existing implementation -- NO changes to method body
}
```

### BusinessController -- Add Decorator to create() Method
```typescript
// Source: Established pattern from subscription.controller.ts (Phase 31)
// File: trade-flow-api/src/business/controllers/business.controller.ts
import { SkipSubscriptionCheck } from "@subscription/decorators/skip-subscription-check.decorator";

// Add decorator above existing @Post() route
@SkipSubscriptionCheck()
@Post()
@UseGuards(JwtAuthGuard)
async create(@AuthUser() authUser: IUserDto, @Body() body: CreateBusinessRequest): Promise<IResponse<IBusinessResponse>> {
  // ... existing implementation -- NO changes to method body
}
```

### Guard Bypass Test Pattern
```typescript
// Source: Established test pattern from subscription.guard.spec.ts (Phase 31)
// File: trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
// (or user/test/controllers/ and business/test/controllers/)

describe("SubscriptionGuard bypass for onboarding routes", () => {
  it("should allow PATCH /v1/user/me without subscription (skipSubscriptionCheck metadata)", async () => {
    // Mock reflector to return true for skipSubscriptionCheck
    reflector.getAllAndOverride.mockReturnValue(true);
    const result = await guard.canActivate(mockExecutionContext);
    expect(result).toBe(true);
  });
});
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No subscription enforcement (pre-v1.6) | Global SubscriptionGuard (v1.6 Phase 31) | 2026-03-29 | All write endpoints require active/trialing subscription |
| All write endpoints blocked equally | Selective bypass via @SkipSubscriptionCheck | 2026-03-29 (Phase 31) | Subscription, webhook, checkout routes exempted |
| Onboarding routes blocked (current bug) | Onboarding routes exempted (Phase 40 fix) | This phase | New users can complete onboarding |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Class-level @SkipSubscriptionCheck would exempt all methods in the controller | Anti-Patterns | LOW -- NestJS reflector.getAllAndOverride checks both handler and class; verified by NestJS documentation pattern. If wrong, class-level might only affect class-level metadata, not method-level. Either way, method-level is the safe choice. |
| A2 | The PATCH method in UserController handles the `/me` sub-route used during onboarding | Code Examples | LOW -- v1.7 audit explicitly states "PATCH /v1/user/me" is blocked. Controller path structure confirmed in STRUCTURE.md as "PATCH /v1/user/{id}". The actual route may be `/me` or `/:id` -- must verify by reading the controller file. |
| A3 | BusinessController.create() handles POST /v1/user/me/business (or POST /v1/business) | Code Examples | LOW -- v1.7 audit says "POST /v1/user/me/business" but STRUCTURE.md shows business.controller.ts at "POST /v1/business". Must verify the exact route path by reading the controller. |

## Open Questions

1. **Exact route path for user profile update during onboarding**
   - What we know: Audit says "PATCH /v1/user/me". STRUCTURE.md shows user.controller.ts handles "PATCH /v1/user/{id}".
   - What's unclear: Whether the actual route is `/v1/user/me` (special self-reference) or `/v1/user/:id` with the user's own ID.
   - Recommendation: Read user.controller.ts during execution to identify the exact method. Apply decorator to whichever method handles the onboarding profile save.

2. **Exact route path for business creation during onboarding**
   - What we know: Audit says "POST /v1/user/me/business". STRUCTURE.md shows business.controller.ts at "POST /v1/business".
   - What's unclear: Whether business creation is nested under `/v1/user/me/business` or at `/v1/business`.
   - Recommendation: Read business.controller.ts during execution. Apply decorator to the create() method regardless of route path.

3. **Security consideration: should bypassed endpoints have additional protection?**
   - What we know: @SkipSubscriptionCheck bypasses the subscription guard but JwtAuthGuard still enforces authentication. The user must be logged in.
   - What's unclear: Whether a user with an expired/canceled subscription could abuse these endpoints to change their profile or create additional businesses.
   - Recommendation: Existing business ownership policies already prevent creating duplicate businesses. Profile PATCH only updates the authenticated user's own record. The JwtAuthGuard + existing authorization policies provide sufficient protection. No additional guards needed.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | `trade-flow-api/package.json` (jest section) |
| Quick run command | `cd trade-flow-api && npx jest --testPathPattern="subscription\|user\|business" --no-coverage` |
| Full suite command | `cd trade-flow-api && npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ONBD-01 | PATCH /v1/user/me allowed without subscription | unit | `cd trade-flow-api && npx jest --testPathPattern=subscription.guard -x` | Wave 0 (add test case) |
| ONBD-02 | POST /v1/business allowed without subscription | unit | `cd trade-flow-api && npx jest --testPathPattern=subscription.guard -x` | Wave 0 (add test case) |
| ONBD-01-04 | All other write endpoints still blocked without subscription | unit | `cd trade-flow-api && npx jest --testPathPattern=subscription.guard -x` | Existing (verify no regression) |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npx jest --testPathPattern="subscription.guard" --no-coverage`
- **Per wave merge:** `cd trade-flow-api && npm test`
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] Add test cases to guard spec verifying PATCH user/me and POST business bypass the guard when @SkipSubscriptionCheck metadata is present
- [ ] Verify existing guard tests still pass (no regression)

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | JwtAuthGuard already enforces authentication on these routes |
| V3 Session Management | no | Firebase JWT handles session |
| V4 Access Control | yes | @SkipSubscriptionCheck bypasses subscription check but JwtAuthGuard + resource policies remain active |
| V5 Input Validation | no | Existing DTO validation on user/business endpoints unchanged |
| V6 Cryptography | no | No cryptographic operations |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Subscription bypass abuse (user creates business without subscription) | Elevation of Privilege | Business creation already has ownership policy (one business per user). Profile update is self-only. JwtAuthGuard enforces identity. [VERIFIED: Phase 31 guard + existing policies] |
| Decorator removal regression | Tampering | Unit tests verify decorator presence. Guard spec tests cover bypass behavior. |

## Sources

### Primary (HIGH confidence)
- Phase 31 plan and summary (`31-01-PLAN.md`, `31-01-SUMMARY.md`) -- SubscriptionGuard implementation details, file paths, canActivate logic, @SkipSubscriptionCheck decorator pattern
- v1.7 milestone audit (`v1.7-MILESTONE-AUDIT.md`) -- INT-01 gap identification, affected endpoints, recommended fix
- Phase 40 roadmap entry (`ROADMAP.md` lines 259-268) -- success criteria, affected requirements
- Phase 35 research and summaries -- trial endpoint already has @SkipSubscriptionCheck, confirming the decorator is the correct pattern

### Secondary (MEDIUM confidence)
- Phase 37 research -- onboarding flow sequence (PATCH user -> POST business -> POST trial), confirms which endpoints are called during onboarding

### Tertiary (LOW confidence)
- Exact method signatures and route paths in UserController and BusinessController -- need verification by reading actual source files during execution (documented in Open Questions)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies, all existing infrastructure
- Architecture: HIGH -- exact same decorator pattern used on 6+ existing routes
- Pitfalls: HIGH -- straightforward change with well-understood risks
- Security: HIGH -- JwtAuthGuard and resource policies provide adequate protection even with subscription check bypassed

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable -- no external dependency changes, purely internal decorator application)
