---
phase: 56-impersonation-backend-audit
fixed_at: 2026-04-19T00:00:00Z
review_path: .planning/phases/56-impersonation-backend-audit/56-REVIEW.md
iteration: 1
findings_in_scope: 8
fixed: 8
skipped: 0
status: all_fixed
---

# Phase 56: Code Review Fix Report

**Fixed at:** 2026-04-19
**Source review:** .planning/phases/56-impersonation-backend-audit/56-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 8
- Fixed: 8
- Skipped: 0

## Fixed Issues

### CR-01: Auth guard routes on unverified jwt.decode() claim

**Files modified:** `trade-flow-api/src/auth/auth.guard.ts`
**Commit:** 1961c51
**Applied fix:** Replaced the unverified `decoded["type"] === "impersonation"` routing check with an `iss` (issuer) claim inspection: `jwt.decode(token, { complete: true })` is used solely to read the `iss` field. If `iss === "trade-flow-impersonation"`, the token is routed to `handleImpersonationToken`. Signature verification still happens inside that method via `jwt.verify`. This removes the attacker-controlled `type` payload field as the routing discriminator.

---

### CR-02: Missing guard for undefined IMPERSONATION_JWT_SECRET

**Files modified:** `trade-flow-api/src/auth/auth.guard.ts`, `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts`
**Commit:** 055eca3
**Applied fix:** Both `JwtAuthGuard` and `ImpersonationCreator` now validate `IMPERSONATION_JWT_SECRET` at construction time. If the env var is absent, an `Error` is thrown immediately on startup rather than allowing `jwt.sign`/`jwt.verify` to operate with `undefined` (which jsonwebtoken silently treats as an empty-string secret). A `private readonly jwtSecret: string` field is used throughout in place of repeated `configService.get()` calls with `as string` casts.

---

### CR-03: /impersonation/end has no ownership or role check

**Files modified:** `trade-flow-api/src/impersonation/controllers/impersonation.controller.ts`, `trade-flow-api/src/impersonation/services/impersonation-terminator.service.ts`, `trade-flow-api/src/impersonation/test/controllers/impersonation.controller.spec.ts`, `trade-flow-api/src/impersonation/test/services/impersonation-terminator.service.spec.ts`
**Commit:** 30906d2
**Applied fix:** The `end` controller endpoint now requires `@Req()` and checks `request.user.supportRoles.length === 0`, throwing `ForbiddenError` for non-support callers — consistent with the `start` endpoint. `ImpersonationTerminator.end()` now accepts a `callerUserId` parameter and validates that `auditEntry.supportUserId === callerUserId`; a mismatch throws `ResourceNotFoundError` with the same "not found" message as a missing session to avoid confirming session existence. Tests updated accordingly with a new ownership-mismatch test case.

---

### CR-04: Impersonation token bearer can call /impersonation/end

**Files modified:** `trade-flow-api/src/impersonation/controllers/impersonation.controller.ts`, `trade-flow-api/src/impersonation/services/impersonation-terminator.service.ts`, `trade-flow-api/src/impersonation/test/controllers/impersonation.controller.spec.ts`, `trade-flow-api/src/impersonation/test/services/impersonation-terminator.service.spec.ts`
**Commit:** 30906d2
**Applied fix:** Resolved by the CR-03 fix. When an impersonation token is used, `request.user` is set to the target user (who has no support roles), so the `supportRoles.length === 0` gate correctly blocks the impersonated identity from ending any session. The ownership check in the service layer further prevents cross-session termination. The `@Req()` inline type now exposes `user: IUserDto` for the `end` endpoint.

---

### WR-01: status field typed as string in entity, cast unsafely in repository

**Files modified:** `trade-flow-api/src/impersonation/entities/impersonation-audit.entity.ts`, `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts`
**Commit:** 001594c
**Applied fix:** `IImpersonationAuditEntity.status` narrowed from `string` to `ImpersonationStatus`. The `ImpersonationStatus` enum import was added to the entity file. The unsafe `entity.status as ImpersonationStatus` cast in `toDto()` was removed — the type now flows correctly without assertion.

---

### WR-02: sessionId not validated as ObjectId format — throws 500 on invalid input

**Files modified:** `trade-flow-api/src/impersonation/requests/end-impersonation.request.ts`, `trade-flow-api/src/impersonation/requests/start-impersonation.request.ts`
**Commit:** 13da930
**Applied fix:** Added `@Matches(/^[a-f\d]{24}$/i, { message: "sessionId must be a valid identifier" })` to `EndImpersonationRequest.sessionId` and `@Matches(/^[a-f\d]{24}$/i, { message: "targetUserId must be a valid identifier" })` to `StartImpersonationRequest.targetUserId`. Invalid ObjectId strings are now rejected at the request boundary with a 400 validation error instead of propagating as a 500 from the `ObjectId` constructor.

---

### WR-03: as never type suppression in repository queries hides type errors

**Files modified:** `trade-flow-api/src/impersonation/repositories/impersonation-audit.repository.ts`
**Commit:** 269065c
**Applied fix:** Removed `as never` casts from `findBySupportUserId` (the `findMany` filter) and `markEnded` (both the filter and update arguments). With `IImpersonationAuditEntity` extending `IBaseEntity extends Record<string, unknown>`, the MongoDB driver's `Filter<T>` and `UpdateFilter<T>` types are satisfied directly. TypeScript typecheck confirmed no errors introduced.

---

### WR-04: Information leakage in impersonation token error messages

**Files modified:** `trade-flow-api/src/auth/auth.guard.ts`
**Commit:** 055eca3
**Applied fix:** Both `UnauthorizedException("Target user not found")` and `UnauthorizedException("Support user not found")` in `handleImpersonationToken` replaced with the single generic message `"Invalid impersonation token"`. A token bearer can no longer enumerate internal user IDs by observing which distinct error is returned.

---

_Fixed: 2026-04-19_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
