---
phase: 56-impersonation-backend-audit
reviewed: 2026-04-19T00:00:00Z
depth: standard
files_reviewed: 19
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
  - trade-flow-api/src/impersonation/test/mocks/impersonation-mock-generator.ts
  - trade-flow-api/src/impersonation/test/repositories/impersonation-audit.repository.spec.ts
  - trade-flow-api/src/impersonation/test/services/impersonation-creator.service.spec.ts
  - trade-flow-api/src/impersonation/test/services/impersonation-terminator.service.spec.ts
  - trade-flow-api/src/impersonation/test/controllers/impersonation.controller.spec.ts
findings:
  critical: 4
  warning: 4
  info: 2
  total: 10
status: issues_found
---

# Phase 56: Code Review Report — Impersonation Backend Audit

**Reviewed:** 2026-04-19
**Depth:** standard
**Files Reviewed:** 19
**Status:** issues_found

## Summary

This is an admin impersonation feature for a multi-tenant SaaS API. The overall structure is sound: the lateral-impersonation prevention check (support-user-cannot-impersonate-support-user) is present and tested, the audit trail is append-only at the service level, and the HS256 JWT signing path is correctly isolated from the Firebase RS256 path in the auth guard.

However, four critical security issues require remediation before this feature can ship:

1. The auth guard routes tokens to the impersonation handler based on an **unverified** `jwt.decode()` claim, allowing any attacker who can present a crafted token header to trigger the impersonation code path before any signature is checked.
2. The `IMPERSONATION_JWT_SECRET` env var is never validated for presence; a missing secret causes `jwt.sign()` to silently use `undefined`, which jsonwebtoken accepts as an empty-string secret.
3. The `/impersonation/end` endpoint has **no ownership check** — any authenticated user, including a regular customer with a valid Firebase JWT, can terminate any impersonation session by guessing or obtaining a `sessionId`.
4. Related to (3): the `end` endpoint also has no support-role guard, so a user holding an impersonation token (where `request.user` is the target/customer) could call `/end` against any session.

---

## Critical Issues

### CR-01: Auth guard routes on unverified `jwt.decode()` claim

**File:** `trade-flow-api/src/auth/auth.guard.ts:35-37`

**Issue:** The guard reads the `type` field from `jwt.decode(token)` — which performs no signature verification — and uses it to bypass the Firebase Passport strategy entirely, routing instead to `handleImpersonationToken`. An attacker can construct a JWT with any header/payload (e.g., `{"type":"impersonation","targetUserId":"...","supportUserId":"...","sessionId":"..."}`) signed with a known-weak or random key. The call to `jwt.decode()` will succeed and the token will be routed to `handleImpersonationToken`, which then calls `jwt.verify()` correctly — but only after the routing decision has already been made on untrusted data. The window of risk is the gap between routing and verification: if `jwt.verify()` throws and the catch block re-throws, NestJS will return an error, so the immediate authentication bypass is blocked. The deeper risk is that this pattern is fragile — any future change inside the try/catch (e.g., a caught `JsonWebTokenError` being silently swallowed) would open a full authentication bypass. Additionally, a crafted impersonation-shaped token prevents a legitimate Firebase user with a coincidental `type` claim in their token from authenticating normally.

**Fix:** Check a verifiable property to distinguish token types, not a claim. The cleanest approach is to peek at the `iss` (issuer) claim, which cannot be spoofed without the corresponding secret:

```typescript
// In canActivate(), replace the unverified decode-based branch with issuer detection.
// jwt.decode is still acceptable here only to inspect the issuer — verification
// happens immediately inside handleImpersonationToken before any trust is placed.
const decoded = jwt.decode(token, { complete: true });
const issuer = decoded && typeof decoded !== "string" ? (decoded.payload as { iss?: string }).iss : undefined;

if (issuer === "trade-flow-impersonation") {
  return this.handleImpersonationToken(token, request);
}
// Fall through to Firebase Passport strategy for all other tokens.
```

This does not change the security surface materially — `handleImpersonationToken` still verifies the signature before trusting anything — but it removes the `type` claim (an attacker-controlled payload field) as the routing discriminator.

---

### CR-02: Missing guard for undefined `IMPERSONATION_JWT_SECRET`

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:56-71`

**Issue:** `configService.get<string>("IMPERSONATION_JWT_SECRET")` returns `undefined` when the environment variable is not set. The value is immediately cast with `as string` and passed to `jwt.sign()`. The `jsonwebtoken` library accepts `undefined` as the secret parameter and signs the token using an empty string, producing a token that any caller can forge by signing with the same empty string. This is a silent misconfiguration failure with no startup-time guard.

```typescript
// Current — no null check:
const secret = this.configService.get<string>("IMPERSONATION_JWT_SECRET");
const token = jwt.sign({ ... }, secret as string, { ... });
```

The same `secret as string` pattern is repeated in `auth.guard.ts:104` in `handleImpersonationToken`.

**Fix:** Validate the secret at service construction time (not at call time) so the application fails to start rather than issuing forgeable tokens:

```typescript
// In ImpersonationCreator constructor:
constructor(
  private readonly auditRepository: ImpersonationAuditRepository,
  private readonly userRetriever: UserRetriever,
  private readonly configService: ConfigService,
) {
  const secret = this.configService.get<string>("IMPERSONATION_JWT_SECRET");
  if (!secret) {
    throw new Error("IMPERSONATION_JWT_SECRET environment variable is required");
  }
  this.jwtSecret = secret;
}

private readonly jwtSecret: string;
```

Apply the same pattern in `JwtAuthGuard` for the guard's usage, or move secret resolution to a shared provider that validates on startup.

---

### CR-03: `/impersonation/end` has no ownership or role check

**File:** `trade-flow-api/src/impersonation/controllers/impersonation.controller.ts:40-49`

**Issue:** The `end` endpoint applies `@UseGuards(JwtAuthGuard)` but performs no additional authorization. Any authenticated user — including a regular customer with a valid Firebase JWT — can POST to `/v1/impersonation/end` with any `sessionId` and terminate an active impersonation session. There is no check that the caller is the support user who started the session, or even that the caller has a support role. The `ImpersonationTerminator.end()` service also has no ownership validation. Compare with the `start` endpoint which checks `request.user.supportRoles.length === 0` at line 29 — the `end` endpoint has no equivalent gate.

**Fix:** Add a support-role check consistent with the `start` endpoint, and additionally verify that the calling user is the same support user who created the session:

```typescript
@UseGuards(JwtAuthGuard)
@Post("impersonation/end")
public async end(
  @Req() request: { user: IUserDto },
  @Body() body: EndImpersonationRequest,
): Promise<IResponse<void>> {
  try {
    if (!request.user.supportRoles || request.user.supportRoles.length === 0) {
      throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Impersonation requires support role");
    }
    await this.impersonationTerminator.end(body.sessionId, request.user.id);
    return createResponse([]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

Update `ImpersonationTerminator.end()` to accept and validate `callerUserId`:

```typescript
public async end(sessionId: string, callerUserId: string): Promise<void> {
  const auditEntry = await this.auditRepository.findBySessionId(sessionId);
  if (!auditEntry) {
    throw new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "Impersonation session not found");
  }

  if (auditEntry.supportUserId !== callerUserId) {
    throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Session not found");
  }

  if (auditEntry.status !== ImpersonationStatus.ACTIVE) {
    throw new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Session is not active");
  }

  this.logger.log("Ending impersonation session", { sessionId });
  await this.auditRepository.markEnded(sessionId, DateTime.utc());
}
```

Note: The ownership-mismatch error message should be indistinguishable from "not found" to avoid confirming that a session exists.

---

### CR-04: Impersonation token bearer can call `/impersonation/end`

**File:** `trade-flow-api/src/impersonation/controllers/impersonation.controller.ts:40-49`
**Related:** `trade-flow-api/src/auth/auth.guard.ts:119`

**Issue:** When a request carries an impersonation token, `JwtAuthGuard.handleImpersonationToken()` sets `request.user = targetUser` (the impersonated customer) and `request.impersonator = supportUser`. The `end` controller endpoint reads only `@Body()` — it does not use `@Req()` at all — so it does not check whether the caller is the original support user or the impersonated target user. A session holder (possessing a valid impersonation token) can call `/end` to terminate their own session, which may be intentional, but because there is no ownership check (CR-03), they could also terminate *other* sessions by guessing session IDs. Additionally, if a future endpoint accidentally allows impersonation tokens in a context where only support users should act, the `request.impersonator` field on the Express request is not surfaced in the inline typed `@Req()` parameter, making it easy for developers to miss.

**Fix:** The ownership check in CR-03 resolves the cross-session termination risk. Additionally, when the caller holds an impersonation token, `request.user` is the target user (who has no support roles), so the support-role gate in CR-03 would correctly block the impersonated identity from ending arbitrary sessions. To allow a session holder to end their own session via an impersonation token, a separate self-termination check could use `request.impersonator` if that pattern is needed. Ensure the `@Req()` inline type in the controller exposes `impersonator` if that path is intentional:

```typescript
public async end(
  @Req() request: { user: IUserDto; impersonator?: IUserDto },
  @Body() body: EndImpersonationRequest,
): Promise<IResponse<void>>
```

---

## Warnings

### WR-01: `status` field typed as `string` in entity, cast unsafely in repository

**File:** `trade-flow-api/src/impersonation/entities/impersonation-audit.entity.ts:9`
**Related:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:81`

**Issue:** `IImpersonationAuditEntity` declares `status: string`. In `toDto()`, the repository casts this with `entity.status as ImpersonationStatus`. This is an unsafe type assertion: if a document in MongoDB has a `status` value that is not a member of the enum (e.g., from a future migration or a manual database edit), the cast will succeed at compile time but produce an invalid enum value at runtime, breaking any `switch` or strict equality check downstream.

**Fix:** Narrow the entity type and validate at the repository boundary:

```typescript
// In entity:
status: ImpersonationStatus;

// Or if entity must stay as string, validate in toDto():
private toImpersonationStatus(value: string): ImpersonationStatus {
  const valid = Object.values(ImpersonationStatus) as string[];
  if (!valid.includes(value)) {
    throw new Error(`Unexpected impersonation status from database: ${value}`);
  }
  return value as ImpersonationStatus;
}
```

---

### WR-02: `sessionId` not validated as ObjectId format — throws 500 on invalid input

**File:** `trade-flow-api/src/impersonation/requests/end-impersonation.request.ts:1-7`
**Related:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:46`

**Issue:** `EndImpersonationRequest.sessionId` is validated only as a non-empty string (`@IsString()`, `@IsNotEmpty()`). In `ImpersonationAuditRepository.findBySessionId()`, the value is passed directly to `new ObjectId(sessionId)`. If the caller provides a string that is not a valid 24-character hex ObjectId (e.g., `"not-an-objectid"`), the `ObjectId` constructor throws a `BSONError` synchronously. This exception is not caught by the repository or service layer and will propagate as an unhandled 500 Internal Server Error instead of a 422 validation error. The same risk exists for `StartImpersonationRequest.targetUserId`.

**Fix:** Add a custom class-validator decorator or use `@Matches(/^[a-f\d]{24}$/i)` on both fields to reject invalid ObjectId strings at the request boundary:

```typescript
import { IsNotEmpty, IsString, Matches } from "class-validator";

export class EndImpersonationRequest {
  @IsString()
  @IsNotEmpty()
  @Matches(/^[a-f\d]{24}$/i, { message: "sessionId must be a valid identifier" })
  declare sessionId: string;
}
```

Apply the same `@Matches` decorator to `StartImpersonationRequest.targetUserId`.

---

### WR-03: `as never` type suppression in repository queries hides type errors

**File:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:61, 68`

**Issue:** Two queries use `as never` to coerce the filter/update arguments passed to `MongoDbFetcher.findMany` and `MongoDbWriter.updateOne`. This suppresses TypeScript's type checking for those call sites entirely — any structural mismatch between the query shape and the MongoDB driver's expected types will go undetected at compile time.

```typescript
// Line 61 — filter cast to never:
const collection = await this.fetcher.findMany<IImpersonationAuditEntity>(ImpersonationAuditRepository.COLLECTION, {
  supportUserId,
} as never);

// Line 68 — filter and update cast to never:
{ _id: new ObjectId(sessionId) } as never,
{ $set: { endedAt: endedAt.toJSDate(), status: ImpersonationStatus.ENDED, updatedAt: new Date() } } as never,
```

**Fix:** Use the same pattern as existing repositories in the codebase — if `MongoDbFetcher`/`MongoDbWriter` accept `Filter<T>` or `UpdateFilter<T>` from the MongoDB driver, import and use those types explicitly:

```typescript
import type { Filter, UpdateFilter } from "mongodb";

// findMany:
const collection = await this.fetcher.findMany<IImpersonationAuditEntity>(
  ImpersonationAuditRepository.COLLECTION,
  { supportUserId } satisfies Filter<IImpersonationAuditEntity>,
);
```

---

### WR-04: Information leakage in impersonation token error messages

**File:** `trade-flow-api/src/auth/auth.guard.ts:110-116`

**Issue:** `handleImpersonationToken()` throws `UnauthorizedException("Target user not found")` and `UnauthorizedException("Support user not found")` as distinct messages. These messages confirm to the token bearer whether the target or support user ID embedded in the token refers to an existing internal user. An attacker who has obtained a valid impersonation token (or can forge one, see CR-02) can enumerate internal user IDs by varying the `targetUserId` and observing which error message is returned.

**Fix:** Return a single generic message for all failure conditions inside `handleImpersonationToken`:

```typescript
const targetUser = await this.userRetriever.getById(payload.targetUserId);
if (!targetUser) {
  throw new UnauthorizedException("Invalid impersonation token");
}

const supportUser = await this.userRetriever.getById(payload.supportUserId);
if (!supportUser) {
  throw new UnauthorizedException("Invalid impersonation token");
}
```

---

## Info

### IN-01: Audit trail has no database-level guard against duplicate `markEnded` calls

**File:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts:65-71`

**Issue:** `markEnded()` issues an unconditional `$set` with no filter on `status`. The service layer prevents double-ending via a status check, but the repository itself will silently overwrite `endedAt` and `status` if called directly (e.g., in tests or future callers). For an audit trail, idempotency at the persistence level is preferable.

**Fix:** Add a conditional filter to `markEnded` so the update only applies when status is still `ACTIVE`:

```typescript
public async markEnded(sessionId: string, endedAt: DateTime): Promise<void> {
  await this.writer.updateOne<IImpersonationAuditEntity>(
    ImpersonationAuditRepository.COLLECTION,
    { _id: new ObjectId(sessionId), status: ImpersonationStatus.ACTIVE } as never,
    { $set: { endedAt: endedAt.toJSDate(), status: ImpersonationStatus.ENDED, updatedAt: new Date() } } as never,
  );
}
```

---

### IN-02: `EXPIRY_MINUTES` constant and `expiresIn: "30m"` string are duplicated

**File:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts:20, 65`

**Issue:** The expiry is expressed in two places: `private static readonly EXPIRY_MINUTES = 30` (used in the response) and the string literal `"30m"` passed to `jwt.sign()`. If one is updated without the other, the response will report a different expiry than the token actually contains.

**Fix:** Derive the `expiresIn` string from the constant to keep them in sync:

```typescript
private static readonly EXPIRY_MINUTES = 30;

// In create():
const token = jwt.sign(
  { ... },
  this.jwtSecret,
  {
    algorithm: "HS256",
    expiresIn: `${ImpersonationCreator.EXPIRY_MINUTES}m`,
    issuer: "trade-flow-impersonation",
    subject: targetUser.id,
  },
);
```

---

_Reviewed: 2026-04-19_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
