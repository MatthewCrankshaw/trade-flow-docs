---
phase: 56-impersonation-backend-audit
reviewed: 2026-04-19T14:30:00Z
depth: standard
files_reviewed: 17
files_reviewed_list:
  - trade-flow-api/src/impersonation/enums/impersonation-status.enum.ts
  - trade-flow-api/src/impersonation/entities/impersonation-audit.entity.ts
  - trade-flow-api/src/impersonation/data-transfer-objects/impersonation-audit.dto.ts
  - trade-flow-api/src/impersonation/requests/start-impersonation.request.ts
  - trade-flow-api/src/impersonation/requests/end-impersonation.request.ts
  - trade-flow-api/src/impersonation/responses/impersonation.response.ts
  - trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts
  - trade-flow-api/src/impersonation/impersonation.module.ts
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/impersonation/services/impersonation-creator.service.ts
  - trade-flow-api/src/impersonation/services/impersonation-terminator.service.ts
  - trade-flow-api/src/impersonation/controllers/impersonation.controller.ts
  - trade-flow-api/src/auth/interfaces/impersonation-payload.interface.ts
  - trade-flow-api/src/auth/types/express.d.ts
  - trade-flow-api/src/auth/auth.guard.ts
  - trade-flow-api/src/support/support.module.ts
  - trade-flow-api/nodemon-worker.json
findings:
  critical: 1
  warning: 1
  info: 3
  total: 5
status: issues_found
---

# Phase 56: Code Review Report — Impersonation Backend Audit

**Reviewed:** 2026-04-19T14:30:00Z
**Depth:** standard
**Files Reviewed:** 17
**Status:** issues_found

## Summary

This review covers the complete impersonation backend implementation across 17 files. The previous review's critical and warning findings have all been addressed: the audit repository check in `handleImpersonationToken` is present (lines 119-121 of `auth.guard.ts`), self-impersonation is blocked (creator service line 42), concurrent session prevention is in place (creator service lines 50-57), and the terminator enforces ownership before ending a session.

The overall security architecture is sound: the issuer-based token routing correctly separates HS256 impersonation tokens from Firebase RS256 tokens without trusting an unverified claim for routing decisions (the guard decodes first to peek at `iss`, then verifies with the correct key); lateral impersonation (support impersonating support) is blocked; request validation uses `@Matches` with a strict ObjectId regex to prevent injection; and the JWT secret is validated at construction time in both the creator service and the guard.

One critical issue remains: the impersonation management endpoints (`/impersonation/start` and `/impersonation/end`) do not explicitly reject requests authenticated with an impersonation token. An impersonating support user could call these endpoints while using their impersonation JWT. The current code path happens to be safe due to defense-in-depth checks in the creator service (target user cannot have support roles), but the protection is implicit rather than enforced at the right layer. One warning and three info items are also noted.

## Critical Issues

### CR-01: Impersonation endpoints accept impersonation tokens as authentication

**File:** `trade-flow-api/src/impersonation/controllers/impersonation.controller.ts:22-52`
**Related:** `trade-flow-api/src/auth/auth.guard.ts:112-146`

**Issue:** Both `/impersonation/start` and `/impersonation/end` are protected only by `JwtAuthGuard`. When `JwtAuthGuard` processes an impersonation JWT, it sets `request.user = targetUser` and `request.impersonator = supportUser`. The controller then checks `request.user.supportRoles.length === 0` — if the impersonated (target) user has no support roles, the request is rejected. This is the common case and it works, but the security invariant relies on the target user having no support roles rather than on the endpoint explicitly rejecting impersonation tokens.

The specific risk: if a future code path allows a target user to gain support roles after a session has started (e.g., a role assignment race), a support user using their impersonation JWT could call `/impersonation/start` with `request.user` being the target user (now with support roles), and start an impersonation chain. The `authUser.id === targetUser.id` self-impersonation check in the creator would not catch this case.

More broadly, allowing impersonation tokens to authenticate management endpoints violates the principle that session management actions should require proof of the support user's own identity, not their impersonated identity.

**Fix:** In `handleImpersonationToken`, detect when the request URL targets the impersonation management routes and reject:

```typescript
private async handleImpersonationToken(token: string, request: Request): Promise<boolean> {
  const path = request.path;
  if (path.includes("/impersonation/start") || path.includes("/impersonation/end")) {
    throw new UnauthorizedException("Impersonation tokens cannot be used to manage impersonation sessions");
  }

  // ... existing verification logic
}
```

Alternatively, create a separate `SupportJwtAuthGuard` that explicitly requires a Firebase token (not an impersonation token) and apply it to the impersonation management endpoints. This makes the constraint explicit and enforced at the guard layer rather than relying on downstream role checks.

## Warnings

### WR-01: Issuer-based token routing uses unverified `jwt.decode()` before verification

**File:** `trade-flow-api/src/auth/auth.guard.ts:44-49`

**Issue:** The guard decodes the token without verification to peek at the `iss` claim, then routes to either `handleImpersonationToken` (which verifies with `jwtSecret`) or the standard Passport Firebase path. This pattern is intentional and documented, but it introduces a subtle risk: any token — including a malformed or attacker-crafted one — with `iss: "trade-flow-impersonation"` will bypass the Firebase verification path and enter `handleImpersonationToken`, which then calls `jwt.verify()`.

The current implementation is safe because `jwt.verify()` in `handleImpersonationToken` will reject tokens signed with the wrong secret. However, the error handling in `handleImpersonationToken` only catches `jwt.TokenExpiredError` and `jwt.JsonWebTokenError` — it re-throws everything else. If `jwt.verify()` throws an unexpected error (e.g., a malformed algorithm header), the re-throw propagates as a 500 rather than a 401.

**Fix:** Tighten the catch block to treat any verification failure as a 401, not just known JWT error types:

```typescript
} catch (error) {
  if (error instanceof UnauthorizedException) {
    throw error;
  }
  if (error instanceof jwt.TokenExpiredError) {
    throw new UnauthorizedException("Impersonation session expired. Please start a new session.");
  }
  // Treat all JWT errors (including JsonWebTokenError, NotBeforeError, and unexpected errors)
  // as authentication failures to avoid leaking internal error details.
  this.logger.error("Impersonation token verification failed", { error });
  throw new UnauthorizedException("Invalid impersonation token");
}
```

## Info

### IN-01: `EXPIRY_MINUTES` constant and `expiresIn` string literal are not linked

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:20,86`

**Issue:** The expiry is expressed in two independent places: `private static readonly EXPIRY_MINUTES = 30` (line 20, used in the response at line 95) and the string literal `"30m"` passed to `jwt.sign()` (line 86). If one is updated without the other, the response will advertise a different expiry than the token actually contains — a confusing but silent inconsistency.

**Fix:** Derive the `expiresIn` string from the constant:

```typescript
expiresIn: `${ImpersonationCreator.EXPIRY_MINUTES}m`,
```

### IN-02: `markEnded` has no database-level idempotency guard

**File:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:65-71`

**Issue:** `markEnded()` issues an unconditional `updateOne` with no filter on `status`. The service layer prevents double-ending via a status check in `ImpersonationTerminator`, but the repository itself will silently overwrite `endedAt` and `status` if called again by a future caller. For an audit trail that should be append-only, idempotency at the persistence layer is preferable.

**Fix:** Add a status filter so the update only applies when the session is still `ACTIVE`:

```typescript
public async markEnded(sessionId: string, endedAt: DateTime): Promise<void> {
  await this.writer.updateOne<IImpersonationAuditEntity>(
    ImpersonationAuditRepository.COLLECTION,
    { _id: new ObjectId(sessionId), status: ImpersonationStatus.ACTIVE },
    { $set: { endedAt: endedAt.toJSDate(), status: ImpersonationStatus.ENDED, updatedAt: new Date() } },
  );
}
```

### IN-03: `ImpersonationModule` is registered redundantly in `AppModule`

**File:** `trade-flow-api/src/app.module.ts:66`
**Related:** `trade-flow-api/src/support/support.module.ts:11`

**Issue:** `AppModule` imports `ImpersonationModule` directly (line 66) and also imports `SupportModule` (line 65), which itself imports `ImpersonationModule` (support.module.ts line 11). NestJS deduplicates module registrations, so this is not a runtime error, but the direct `AppModule` import is redundant — `ImpersonationModule` is already available transitively through `SupportModule`. The redundant import creates the impression that there are consumers of `ImpersonationModule` outside of `SupportModule` at the app root level, which is misleading.

**Fix:** Remove the direct `ImpersonationModule` import from `AppModule`. It is fully covered by `SupportModule`. If `ImpersonationModule` needs to be available to modules outside `SupportModule` in the future, add it back then with a comment explaining why.

```typescript
// In app.module.ts — remove this import:
import { ImpersonationModule } from "@impersonation/impersonation.module";

// And remove from the imports array:
// ImpersonationModule,   <-- remove
```

---

_Reviewed: 2026-04-19T14:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
