---
phase: 56-impersonation-backend-audit
reviewed: 2026-04-19T12:00:00Z
depth: standard
files_reviewed: 22
files_reviewed_list:
  - trade-flow-api/src/impersonation/enums/impersonation-status.enum.ts
  - trade-flow-api/src/impersonation/entities/impersonation-audit.entity.ts
  - trade-flow-api/src/impersonation/data-transfer-objects/impersonation-audit.dto.ts
  - trade-flow-api/src/impersonation/requests/start-impersonation.request.ts
  - trade-flow-api/src/impersonation/requests/end-impersonation.request.ts
  - trade-flow-api/src/impersonation/responses/impersonation.response.ts
  - trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts
  - trade-flow-api/src/impersonation/impersonation.module.ts
  - trade-flow-api/src/impersonation/services/impersonation-creator.service.ts
  - trade-flow-api/src/impersonation/services/impersonation-terminator.service.ts
  - trade-flow-api/src/impersonation/controllers/impersonation.controller.ts
  - trade-flow-api/src/auth/interfaces/impersonation-payload.interface.ts
  - trade-flow-api/src/auth/types/express.d.ts
  - trade-flow-api/src/auth/auth.guard.ts
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/tsconfig.json
  - trade-flow-api/package.json
  - trade-flow-api/src/impersonation/test/mocks/impersonation-mock-generator.ts
  - trade-flow-api/src/impersonation/test/repositories/impersonation-audit.repository.spec.ts
  - trade-flow-api/src/impersonation/test/services/impersonation-creator.service.spec.ts
  - trade-flow-api/src/impersonation/test/services/impersonation-terminator.service.spec.ts
  - trade-flow-api/src/impersonation/test/controllers/impersonation.controller.spec.ts
findings:
  critical: 1
  warning: 2
  info: 2
  total: 5
status: issues_found
---

# Phase 56: Code Review Report -- Impersonation Backend Audit

**Reviewed:** 2026-04-19
**Depth:** standard
**Files Reviewed:** 22
**Status:** issues_found

## Summary

This is a re-review of the impersonation backend feature after the previous review's critical and warning findings were addressed. The four critical issues from the prior review (unverified `jwt.decode()` routing, missing secret validation, missing ownership/role checks on `/end`, impersonation token bearer bypass) have all been resolved correctly. The code now routes on `iss` claim, validates `IMPERSONATION_JWT_SECRET` at construction time, enforces support-role gates on both endpoints, and checks session ownership in the terminator.

The overall security posture is solid: lateral impersonation prevention is in place (support users cannot impersonate other support users), the HS256 JWT path is correctly isolated from the Firebase RS256 path, request validation uses `@Matches` to prevent ObjectId injection, and the audit trail is append-only at the service layer.

One critical issue remains: ended impersonation sessions are not revoked at the auth guard level, meaning an impersonation JWT remains usable for up to 30 minutes after the support user explicitly ends the session. Two warnings and two info items are also noted.

## Critical Issues

### CR-01: Ended impersonation sessions remain usable until JWT expiry

**File:** `trade-flow-api/src/auth/auth.guard.ts:109-128`
**Related:** `trade-flow-api/src/impersonation/services/impersonation-terminator.service.ts:31`

**Issue:** When a support user calls `/impersonation/end`, the `ImpersonationTerminator` marks the audit record as `ENDED` in MongoDB. However, `JwtAuthGuard.handleImpersonationToken()` only verifies the JWT signature, expiry, and that the target/support users exist -- it never queries the audit trail to confirm the session is still `ACTIVE`. This means the impersonation JWT remains fully functional for up to 30 minutes after explicit termination.

This is a meaningful security gap: if a support user ends a session because they noticed suspicious activity or realized they were impersonating the wrong user, the token they already issued continues to grant access to the target user's data. Any client that cached the impersonation token can keep using it.

**Fix:** Inject `ImpersonationAuditRepository` into `JwtAuthGuard` and validate session status during token verification:

```typescript
private async handleImpersonationToken(token: string, request: Request): Promise<boolean> {
  try {
    const payload = jwt.verify(token, this.jwtSecret, {
      algorithms: ["HS256"],
      issuer: "trade-flow-impersonation",
    }) as IImpersonationPayload;

    // Verify the session has not been ended or expired
    const session = await this.auditRepository.findBySessionId(payload.sessionId);
    if (!session || session.status !== ImpersonationStatus.ACTIVE) {
      throw new UnauthorizedException("Impersonation session is no longer active");
    }

    const targetUser = await this.userRetriever.getById(payload.targetUserId);
    if (!targetUser) {
      throw new UnauthorizedException("Invalid impersonation token");
    }

    const supportUser = await this.userRetriever.getById(payload.supportUserId);
    if (!supportUser) {
      throw new UnauthorizedException("Invalid impersonation token");
    }

    request.user = targetUser;
    request.impersonator = supportUser;
    return true;
  } catch (error) {
    // ... existing error handling
  }
}
```

This adds one database query per impersonation-authenticated request, which is acceptable for a security-sensitive feature with short-lived sessions.

## Warnings

### WR-01: No self-impersonation prevention

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:36-84`

**Issue:** The `ImpersonationCreator.create()` method checks that the target user is not a support user (line 42), but does not check whether `authUser.id === targetUser.id`. A support user can impersonate themselves, creating an audit trail entry and a token that sets them as both `request.user` and `request.impersonator`. While this is not a direct security vulnerability (they already have access to their own data), it pollutes the audit log with meaningless entries and could mask actual abuse in audit review.

**Fix:** Add a self-impersonation check before the support-role check:

```typescript
if (authUser.id === targetUser.id) {
  throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Cannot impersonate yourself");
}
```

### WR-02: Concurrent impersonation sessions not prevented

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:36-84`
**Related:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:57-63`

**Issue:** A support user can start multiple concurrent impersonation sessions targeting the same or different users. There is no check for an existing `ACTIVE` session before creating a new one. While the `findBySupportUserId` repository method exists, it is never called during session creation. Multiple active sessions for the same support user complicate audit trails and mean that ending one session does not revoke the other tokens (especially relevant given CR-01 above).

**Fix:** Before creating a new session, query for existing active sessions and either reject the request or auto-terminate the prior session:

```typescript
const existingSessions = await this.auditRepository.findBySupportUserId(authUser.id);
const activeSessions = existingSessions.filter(s => s.status === ImpersonationStatus.ACTIVE);
if (activeSessions.length > 0) {
  throw new ForbiddenError(
    ErrorCodes.ACTION_FORBIDDEN,
    "An active impersonation session already exists. End it before starting a new one.",
  );
}
```

Alternatively, add a dedicated `findActiveBySupportUserId` method to avoid fetching all historical sessions.

## Info

### IN-01: `EXPIRY_MINUTES` constant and `expiresIn` string literal are duplicated

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:20,72`

**Issue:** The expiry is expressed in two places: `private static readonly EXPIRY_MINUTES = 30` (line 20, used in the response at line 82) and the string literal `"30m"` passed to `jwt.sign()` (line 72). If one is updated without the other, the response will report a different expiry than the token actually contains.

**Fix:** Derive the `expiresIn` string from the constant:

```typescript
expiresIn: `${ImpersonationCreator.EXPIRY_MINUTES}m`,
```

### IN-02: Audit trail `markEnded` has no database-level idempotency guard

**File:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:65-71`

**Issue:** `markEnded()` issues an unconditional `$set` with no filter on `status`. The service layer prevents double-ending via a status check in `ImpersonationTerminator.end()`, but the repository itself will silently overwrite `endedAt` and `status` if called directly by a future caller. For an audit trail, idempotency at the persistence level is preferable.

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

---

_Reviewed: 2026-04-19_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
