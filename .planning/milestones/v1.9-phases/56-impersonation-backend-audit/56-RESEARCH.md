# Phase 56: Impersonation Backend & Audit - Research

**Researched:** 2026-04-18
**Domain:** NestJS JWT impersonation tokens, append-only audit logging, guard extension
**Confidence:** HIGH

## Summary

This phase builds an impersonation system on top of the existing NestJS authentication infrastructure. A support user with the `impersonate_user` permission (seeded in Phase 51, enforced via `@RequiresPermission` from Phase 52) can initiate a time-limited "login as" session for any customer user. The system produces an API-signed JWT (using `jsonwebtoken` already installed at v9.0.2) with a dedicated `IMPERSONATION_JWT_SECRET`, and modifies the existing `JwtAuthGuard` to detect and handle this second token type. All sessions are logged in an append-only `impersonation_audit` MongoDB collection.

The architecture is straightforward: a new `ImpersonationModule` following the one-feature-per-directory convention, two services (creator and terminator), an audit-only repository (create + find, no update/delete), and modifications to the existing `JwtAuthGuard` to branch on token type. The JWT signing uses HMAC HS256 with a 30-minute expiry -- simple symmetric signing is appropriate since the same API both signs and verifies.

**Primary recommendation:** Use `jsonwebtoken` directly (not `@nestjs/jwt`) for impersonation token signing/verification to keep the impersonation secret fully isolated from any existing JWT module configuration. Extend `JwtAuthGuard` with a try-Firebase-first, try-impersonation-second strategy using an `issuer` claim for detection.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** API-signed JWT using `jsonwebtoken` (already installed) with a dedicated `IMPERSONATION_JWT_SECRET` env var. Firebase is not involved in impersonation tokens.
- **D-02:** Impersonation token payload contains: `type: "impersonation"`, `targetUserId`, `supportUserId`, `sessionId`, `exp` (30 min from creation).
- **D-03:** JwtAuthGuard detects token type by checking for impersonation type. Firebase-issued tokens follow existing flow. Impersonation tokens are verified against `IMPERSONATION_JWT_SECRET`, then hydrate `request.user` as the target user and `request.impersonator` as the support user.
- **D-04:** Full read/write access during impersonation. The impersonated `request.user` behaves identically to the real customer -- all endpoints work normally. The audit trail (via `request.impersonator`) tracks who actually performed the action.
- **D-05:** Maximum session duration is 30 minutes, enforced server-side via JWT expiry. Support users can start a new session if more time is needed.
- **D-06:** Two termination paths: (1) Explicit -- `POST /v1/impersonation/end` records `endedAt` timestamp in audit log, status "ended". (2) Automatic -- JWT expires after 30 min, next API call returns 401 with message directing support user to start a new session. Audit entry has no `endedAt`, status "expired".
- **D-07:** Append-only enforced at repository level -- `ImpersonationAuditRepository` exposes only `create()` and `find*()` methods. No `update()` or `delete()` methods exist. No DB-level enforcement.
- **D-08:** Session-level logging only -- one document per impersonation session. Fields: `supportUserId`, `targetUserId`, `reason`, `startedAt`, `endedAt` (nullable), `status` ("active" | "ended" | "expired"). Individual API actions during the session are not logged.
- **D-09:** Dedicated `ImpersonationModule` following one-feature-per-directory convention. Contains: controller, creator/terminator services, audit repository, entity, DTO, response, and status enum.
- **D-10:** `POST /v1/impersonation/start` -- accepts `targetUserId` (string) and `reason` (string, min 10 chars). Protected by `@RequiresPermission('impersonate_user')`. Returns impersonation JWT token in response.
- **D-11:** `POST /v1/impersonation/end` -- accepts session ID. Records `endedAt` timestamp and sets status to "ended" in audit log.
- **D-12:** Reason field validated with `@IsString()`, `@IsNotEmpty()`, `@MinLength(10)`. Free text, no predefined categories. Returns 422 if too short.

### Claude's Discretion
- Exact JWT signing options (algorithm, issuer claim for detection)
- ImpersonationAuditEntity schema details (indexes on supportUserId, targetUserId, startedAt)
- Whether to add a `GET /v1/impersonation/active` endpoint to check if user has an active session
- Error message wording for expired sessions and lateral impersonation attempts
- Whether `ImpersonationModule` imports `UserModule` directly or uses a shared service for user lookup
- Test structure and mock strategy

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| IMP-01 | Support user with impersonation permission can initiate a "login as" session for any customer user | ImpersonationCreator service + support-role check on POST /v1/impersonation/start |
| IMP-02 | Support user cannot impersonate other support users (prevents lateral privilege movement) | Service-level check: verify target user has no support roles; throw ForbiddenError if they do |
| IMP-06 | Impersonation sessions are time-limited (maximum duration enforced) | JWT `expiresIn: '30m'` enforced server-side; expired tokens rejected with 401 + descriptive message |
| IAUD-01 | Every impersonation session is logged with who, whom, when, and reason | ImpersonationAuditRepository.create() called at session start; endedAt updated at session end |
| IAUD-02 | Impersonation audit log is stored in a dedicated `impersonation_audit` collection | Dedicated entity/repository with `static readonly COLLECTION = "impersonation_audit"` |
| IAUD-03 | Impersonation audit entries are append-only (cannot be modified or deleted) | Repository exposes only `create()` and `find*()` methods; the one exception is `updateEndedAt()` for session termination (D-11) -- this is a controlled, single-field update, not a general update |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Impersonation token signing | API / Backend | -- | Secret key must never leave the server; signing is a backend-only operation |
| Impersonation token verification | API / Backend | -- | JwtAuthGuard runs server-side on every request |
| Target user hydration | API / Backend | -- | UserRetriever fetches full IUserDto from database |
| Lateral impersonation prevention | API / Backend | -- | Must be enforced server-side; frontend checks are bypassable |
| Session lifecycle management | API / Backend | -- | Start/end endpoints, JWT expiry enforcement |
| Audit logging | Database / Storage | API / Backend | Append-only collection; API writes, DB stores |
| `request.impersonator` propagation | API / Backend | -- | Guard sets property on request object for downstream use |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| jsonwebtoken | 9.0.2 (installed) | Sign/verify impersonation JWTs | Already in project; direct usage keeps impersonation secret isolated from NestJS JWT module [VERIFIED: CLAUDE.md lists jsonwebtoken 9.0.2] |
| class-validator | 0.14.1 (installed) | DTO validation for start/end requests | Project standard for request validation [VERIFIED: CLAUDE.md] |
| class-transformer | 0.5.1 (installed) | Object transformation | Project standard [VERIFIED: CLAUDE.md] |
| luxon | 3.5.1 (installed) | Timestamp generation for audit entries | Project standard for date/time [VERIFIED: CLAUDE.md] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| mongodb | 7.0.0 (installed) | Native MongoDB driver for audit collection | Repository operations via MongoDbWriter/MongoDbFetcher [VERIFIED: CLAUDE.md] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| jsonwebtoken direct | @nestjs/jwt JwtService | NestJS JWT module would require a second module registration with different secret; direct jsonwebtoken is simpler and more explicit for an isolated concern |

**Installation:**
No new packages required. All dependencies are already installed.

**Version verification:**
- jsonwebtoken: 9.0.2 installed, 9.0.3 latest [VERIFIED: npm registry]
- @nestjs/jwt: 11.0.0 installed, 11.0.2 latest [VERIFIED: npm registry]
- Neither version delta affects this phase's functionality.

## Architecture Patterns

### System Architecture Diagram

```
Support User Request (Bearer: impersonation-jwt)
        |
        v
  +-------------------+
  |   JwtAuthGuard    |  <-- Extended: detect token type
  |                   |
  |  1. Extract token |
  |  2. Try decode    |
  |     header/payload|
  |  3. If type=      |
  |    "impersonation"|-----> Verify with IMPERSONATION_JWT_SECRET
  |     else          |-----> Verify with Firebase public keys (existing)
  |  4. Hydrate user  |
  +-------------------+
        |
        | request.user = target user (IUserDto)
        | request.impersonator = support user (IUserDto)  [only on impersonation]
        v
  +---------------------+
  | PermissionGuard     |  <-- Phase 52: checks permissions on request.user
  +---------------------+
        |
        v
  +---------------------+
  | SubscriptionGuard   |  <-- Existing: checks subscription on request.user
  +---------------------+
        |
        v
  +---------------------+
  | Feature Controller  |  <-- Any endpoint; request.user = target user
  +---------------------+


  === Impersonation Session Lifecycle ===

  POST /v1/impersonation/start
        |
        v
  +-------------------------+
  | ImpersonationController |
  | supportRoles.length     |
  | check (interim D-10)    |
  +-------------------------+
        |
        v
  +-------------------------+
  | ImpersonationCreator    |
  | 1. Validate target is   |
  |    customer (not support)|
  | 2. Create audit entry   |
  | 3. Sign JWT with        |
  |    jsonwebtoken          |
  | 4. Return token          |
  +-------------------------+
        |
        v
  +-----------------------------+
  | ImpersonationAuditRepository|
  | create() -> audit entry     |
  | (append-only)               |
  +-----------------------------+
        |
        v
  [impersonation_audit collection]


  POST /v1/impersonation/end
        |
        v
  +---------------------------+
  | ImpersonationTerminator   |
  | 1. Find audit entry       |
  | 2. Set endedAt + "ended"  |
  +---------------------------+
```

### Recommended Project Structure
```
src/impersonation/
  impersonation.module.ts
  controllers/
    impersonation.controller.ts
  services/
    impersonation-creator.service.ts
    impersonation-terminator.service.ts
  repositories/
    impersonation-audit.repository.ts
  entities/
    impersonation-audit.entity.ts
  data-transfer-objects/
    impersonation-audit.dto.ts
  requests/
    start-impersonation.request.ts
    end-impersonation.request.ts
  responses/
    impersonation.response.ts
  enums/
    impersonation-status.enum.ts
  test/
    services/
      impersonation-creator.service.spec.ts
      impersonation-terminator.service.spec.ts
    controllers/
      impersonation.controller.spec.ts
    mocks/
      impersonation-mock-generator.ts
```

### Pattern 1: Dual-Token JwtAuthGuard

**What:** Extend the existing JwtAuthGuard to handle two token types -- Firebase JWTs and impersonation JWTs -- using the token payload to determine verification strategy.

**When to use:** When the same API must accept tokens from different issuers on all endpoints.

**Example:**
```typescript
// Source: Derived from project's auth.guard.ts pattern + jsonwebtoken docs
// In auth.guard.ts (modified)

import * as jwt from "jsonwebtoken";

// Inside canActivate method:
const token = extractTokenFromHeader(request);

// Peek at token payload without verification to determine type
const decoded = jwt.decode(token, { complete: true });
if (decoded?.payload?.type === "impersonation") {
  // Verify against impersonation secret
  const payload = jwt.verify(token, process.env.IMPERSONATION_JWT_SECRET, {
    algorithms: ["HS256"],
    issuer: "trade-flow-impersonation",
  });

  // Hydrate target user (CONFIRMED: getById exists)
  const targetUser = await this.userRetriever.getById(payload.targetUserId);
  const supportUser = await this.userRetriever.getById(payload.supportUserId);

  request.user = targetUser;
  request.impersonator = supportUser;
} else {
  // Existing Firebase JWT verification flow
  // ...existing code...
}
```

[VERIFIED: jsonwebtoken `decode()` and `verify()` API from Context7 /auth0/node-jsonwebtoken]
[VERIFIED: `UserRetriever.getById(userId: string): Promise<IUserDto | null>` exists at src/user/services/user-retriever.service.ts:17]

### Pattern 2: Append-Only Repository

**What:** A repository that intentionally omits update/delete methods, enforcing write-once semantics at the application layer.

**When to use:** Audit logs, compliance records, event histories.

**Example:**
```typescript
// Source: Project convention from CONVENTIONS.md + MongoDbWriter pattern

@Injectable()
export class ImpersonationAuditRepository {
  static readonly COLLECTION = "impersonation_audit";

  constructor(
    private readonly writer: MongoDbWriter,
    private readonly fetcher: MongoDbFetcher,
  ) {}

  async create(dto: IImpersonationAuditDto): Promise<IImpersonationAuditDto> {
    // Insert audit entry -- no update/delete methods exposed
    return this.writer.insertOne<IImpersonationAuditEntity, IImpersonationAuditDto>(
      ImpersonationAuditRepository.COLLECTION,
      this.toEntity(dto),
      this.toDto,
    );
  }

  async findBySessionId(sessionId: string): Promise<IImpersonationAuditDto | null> {
    return this.fetcher.findOne<IImpersonationAuditEntity, IImpersonationAuditDto>(
      ImpersonationAuditRepository.COLLECTION,
      { _id: new ObjectId(sessionId) },
      this.toDto,
    );
  }

  // Controlled single-field update for session termination only
  async markEnded(sessionId: string, endedAt: string): Promise<void> {
    await this.writer.updateOne(
      ImpersonationAuditRepository.COLLECTION,
      { _id: new ObjectId(sessionId) },
      { $set: { endedAt, status: ImpersonationStatus.ENDED } },
    );
  }
}
```

[ASSUMED: MongoDbWriter/MongoDbFetcher method signatures inferred from project conventions]

### Pattern 3: JWT Signing with jsonwebtoken

**What:** Direct usage of jsonwebtoken for signing impersonation tokens with HS256 and 30-minute expiry.

**Example:**
```typescript
// Source: Context7 /auth0/node-jsonwebtoken
import * as jwt from "jsonwebtoken";

const token = jwt.sign(
  {
    type: "impersonation",
    targetUserId: targetUser.id,
    supportUserId: supportUser.id,
    sessionId: auditEntry.id,
  },
  this.impersonationSecret,
  {
    algorithm: "HS256",
    expiresIn: "30m",
    issuer: "trade-flow-impersonation",
    subject: targetUser.id,
  },
);
```

[VERIFIED: jsonwebtoken sign options from Context7 /auth0/node-jsonwebtoken]

### Anti-Patterns to Avoid

- **Registering a second JwtModule:** Do not use `@nestjs/jwt` `JwtModule.register()` with the impersonation secret. The existing JwtModule is configured for Firebase. Using `jsonwebtoken` directly avoids configuration conflicts and keeps concerns isolated. [CITED: Context7 /nestjs/jwt -- module registration is global or per-module, complicates multi-secret setups]
- **Updating audit entries with a general `update()` method:** The repository must not expose a generic update. Only the specific `markEnded()` method should exist for session termination.
- **Checking token type via try-catch on Firebase verify:** Do not attempt Firebase verification first and catch errors to try impersonation. Use `jwt.decode()` (no verification) to peek at the token type, then route to the correct verification path. This avoids unnecessary Firebase key fetches and misleading error logs.
- **Storing impersonation state in memory/Redis:** The JWT itself carries all session state. No server-side session store is needed. The audit collection records history, not active state.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JWT signing/verification | Custom token format or encryption | `jsonwebtoken` (installed) | Battle-tested, handles expiry, algorithms, claims validation |
| Request validation | Manual field checking | `class-validator` decorators on request DTOs | Project standard; integrates with NestJS ValidationPipe |
| Timestamp generation | `new Date().toISOString()` | `DateTime.utc().toISO()` from Luxon | Project standard for consistent date handling |
| Permission checking | Manual role array inspection | `@RequiresPermission('impersonate_user')` from Phase 52 | Not yet built; interim hardcoded supportRoles check in controller delivers D-10 |

**Key insight:** This phase composes existing infrastructure (guards, repository patterns, error classes) rather than inventing new patterns. The only novel element is the dual-token detection in JwtAuthGuard.

## Common Pitfalls

### Pitfall 1: Token Type Detection Race Condition
**What goes wrong:** If `jwt.decode()` returns null (malformed token), the guard crashes or falls through without proper error handling.
**Why it happens:** `jwt.decode()` does not throw on invalid tokens -- it returns null.
**How to avoid:** Always check `decoded !== null` before accessing `decoded.payload.type`. If null, proceed to Firebase flow (it will reject and return 401 normally).
**Warning signs:** Unhandled null reference errors in auth guard logs.

### Pitfall 2: Expired Token Error Message
**What goes wrong:** `jsonwebtoken.verify()` throws `TokenExpiredError` when a 30-min-old token is used, but the guard catches it generically and returns a bare 401 without the "start a new session" guidance.
**Why it happens:** The existing guard error handling may not distinguish `TokenExpiredError` from other JWT errors.
**How to avoid:** Catch `jwt.TokenExpiredError` specifically in the impersonation verification branch and return a 401 with a descriptive message: "Impersonation session expired. Please start a new session."
**Warning signs:** Support users seeing generic "Unauthorized" instead of actionable guidance.

### Pitfall 3: Lateral Impersonation Check Timing
**What goes wrong:** The service checks if the target user is a support user, but the check runs AFTER creating the audit entry, leaving a dangling "active" audit record for a rejected impersonation.
**Why it happens:** Validation ordering mistake -- check business rules before writing to database.
**How to avoid:** Validate target user is a customer user BEFORE creating the audit entry. Sequence: (1) fetch target user, (2) check not support user, (3) create audit entry, (4) sign token.
**Warning signs:** Orphaned "active" audit entries with no corresponding impersonation session.

### Pitfall 4: `request.impersonator` Type Safety
**What goes wrong:** Downstream controllers/services access `request.impersonator` without type guards and crash on normal (non-impersonation) requests where it is undefined.
**Why it happens:** TypeScript does not know about the custom `impersonator` property on the Express Request object.
**How to avoid:** Extend the Express Request type declaration to include `impersonator?: IUserDto`. The property is optional -- only set during impersonation. Any code that reads it must handle the undefined case.
**Warning signs:** TypeScript `any` casts or runtime errors on `request.impersonator`.

### Pitfall 5: D-11 End Session Requires Controlled Update
**What goes wrong:** The "append-only" constraint (D-07) seems to conflict with D-11's requirement to record `endedAt` on session end.
**Why it happens:** D-07 says "no update or delete methods" but D-11 requires updating the `endedAt` field.
**How to avoid:** The repository exposes a specific `markEnded(sessionId, endedAt)` method (not a general `update()`). This is a controlled, single-purpose write that preserves the spirit of append-only logging while supporting the session lifecycle. The key constraint is: no arbitrary field updates, no deletes.
**Warning signs:** Confusion during implementation about whether end-session should create a new document vs. update existing.

## Code Examples

### ImpersonationAuditEntity Schema

```typescript
// Source: Project entity convention from CONVENTIONS.md
import type { ObjectId } from "mongodb";

export interface IImpersonationAuditEntity {
  _id: ObjectId;
  supportUserId: string;
  targetUserId: string;
  reason: string;
  startedAt: string;      // ISO 8601 via Luxon
  endedAt: string | null;  // null until session ends
  status: string;          // "active" | "ended" | "expired"
  createdAt: string;
  updatedAt: string;
}
```

[ASSUMED: Entity structure follows project convention with `_id: ObjectId` and ISO string timestamps]

### ImpersonationStatus Enum

```typescript
// Source: Project enum convention from CONVENTIONS.md
export enum ImpersonationStatus {
  ACTIVE = "active",
  ENDED = "ended",
  EXPIRED = "expired",
}
```

### StartImpersonation Request

```typescript
// Source: Project request DTO convention
import { IsNotEmpty, IsString, MinLength } from "class-validator";

export class StartImpersonationRequest {
  @IsString()
  @IsNotEmpty()
  targetUserId: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(10)
  reason: string;
}
```

[VERIFIED: class-validator decorators are project standard per CLAUDE.md]

### ImpersonationCreator Service

```typescript
// Source: Project service convention (Creator pattern)
import { Injectable } from "@nestjs/common";
import * as jwt from "jsonwebtoken";
import { ConfigService } from "@nestjs/config";
import { DateTime } from "luxon";

@Injectable()
export class ImpersonationCreator {
  constructor(
    private readonly auditRepository: ImpersonationAuditRepository,
    private readonly userRetriever: UserRetriever,
    private readonly configService: ConfigService,
  ) {}

  async create(authUser: IUserDto, request: StartImpersonationRequest): Promise<IImpersonationResponse> {
    const targetUser = await this.userRetriever.getById(request.targetUserId);
    if (!targetUser) {
      throw new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "Target user not found");
    }

    // IMP-02: Prevent lateral impersonation
    if (targetUser.supportRoles && targetUser.supportRoles.length > 0) {
      throw new ForbiddenError(ErrorCodes.ACTION_NOT_ALLOWED, "Cannot impersonate support users");
    }

    // IAUD-01: Create audit entry AFTER validation passes
    const auditEntry = await this.auditRepository.create({
      supportUserId: authUser.id,
      targetUserId: targetUser.id,
      reason: request.reason,
      startedAt: DateTime.utc().toISO(),
      endedAt: null,
      status: ImpersonationStatus.ACTIVE,
    });

    // Sign impersonation JWT
    const token = jwt.sign(
      {
        type: "impersonation",
        targetUserId: targetUser.id,
        supportUserId: authUser.id,
        sessionId: auditEntry.id,
      },
      this.configService.get<string>("IMPERSONATION_JWT_SECRET"),
      {
        algorithm: "HS256",
        expiresIn: "30m",
        issuer: "trade-flow-impersonation",
        subject: targetUser.id,
      },
    );

    return { token, sessionId: auditEntry.id, expiresInMinutes: 30 };
  }
}
```

[VERIFIED: `UserRetriever.getById(userId: string): Promise<IUserDto | null>` confirmed at src/user/services/user-retriever.service.ts:17. Returns fully hydrated IUserDto with supportRoles.]

### Express Request Type Extension

```typescript
// Source: Standard Express type augmentation pattern
import type { IUserDto } from "@user/data-transfer-objects/user.dto";

declare global {
  namespace Express {
    interface Request {
      impersonator?: IUserDto;
    }
  }
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Cookie-based impersonation sessions | JWT-based stateless impersonation | Current best practice | No server-side session store needed; expiry enforced in token |
| Single auth strategy per guard | Multi-strategy guards with type detection | NestJS pattern | Allows coexistence of Firebase + custom JWT on same endpoints |

**Deprecated/outdated:**
- None relevant. `jsonwebtoken` 9.x is current and stable. The project's version (9.0.2) is one patch behind (9.0.3) which contains no breaking changes. [VERIFIED: npm registry]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | MongoDbWriter/MongoDbFetcher method signatures match the patterns shown in code examples | Architecture Patterns, Code Examples | LOW -- patterns derived from CONVENTIONS.md and STRUCTURE.md; actual signatures may differ slightly in parameter order or generics |
| A2 | ~~UserRetriever has a `getByIdOrFail()` method for internal MongoDB ID lookup~~ **RESOLVED:** `UserRetriever.getById(userId: string): Promise<IUserDto | null>` confirmed at `src/user/services/user-retriever.service.ts:17`. Returns fully hydrated IUserDto with supportRoles. | Code Examples | RESOLVED -- no risk |
| A3 | Express Request type augmentation is the project's pattern for adding custom properties to request | Code Examples | LOW -- standard Express pattern; project may use a different approach |
| A4 | The lateral impersonation check uses `supportRoles.length > 0` on the hydrated IUserDto | Code Examples | LOW -- Phase 52 may change how support role detection works (migrating to permissions); check with Phase 52 implementation |

## Open Questions (RESOLVED)

1. **UserRetriever internal ID lookup** -- RESOLVED
   - **Answer:** `UserRetriever.getById(userId: string): Promise<IUserDto | null>` exists at `src/user/services/user-retriever.service.ts:17`. It returns a fully hydrated `IUserDto` including `supportRoles`. Use `getById()` (not `getByIdOrFail()`) and handle the null return with a ResourceNotFoundError.

2. **Support user detection after Phase 52 migration** -- RESOLVED
   - **Answer:** Check `targetUser.supportRoles.length > 0` on the hydrated IUserDto. Roles still exist on the DTO even after Phase 52 deletes the `isSupportUser()` utility. The controller also uses `request.user.supportRoles.length === 0` as an interim permission guard (D-10) until Phase 52's `@RequiresPermission` decorator is available.

3. **Whether to add GET /v1/impersonation/active endpoint** -- RESOLVED
   - **Answer:** Skip this endpoint. The frontend can detect impersonation from the JWT payload (decode without verification on client side to check `type: "impersonation"`). An endpoint adds unnecessary round-trips.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | package.json jest field |
| Quick run command | `npm run test -- --testPathPattern=impersonation` |
| Full suite command | `npm run test` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| IMP-01 | Creator service signs JWT and returns token for customer target | unit | `npm run test -- --testPathPattern=impersonation-creator` | Wave 0 |
| IMP-02 | Creator service throws ForbiddenError for support user target | unit | `npm run test -- --testPathPattern=impersonation-creator` | Wave 0 |
| IMP-06 | JWT has 30m expiry; guard rejects expired tokens with descriptive error | unit | `npm run test -- --testPathPattern=auth.guard` | Wave 0 |
| IAUD-01 | Audit entry created with all required fields on session start | unit | `npm run test -- --testPathPattern=impersonation-creator` | Wave 0 |
| IAUD-02 | Repository writes to `impersonation_audit` collection | unit | `npm run test -- --testPathPattern=impersonation-audit.repository` | Wave 0 |
| IAUD-03 | Repository has no general update/delete methods | unit (structural) | `npm run test -- --testPathPattern=impersonation-audit.repository` | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=impersonation`
- **Per wave merge:** `npm run ci`
- **Phase gate:** Full `npm run ci` green before verification

### Wave 0 Gaps
- [ ] `src/impersonation/test/services/impersonation-creator.service.spec.ts` -- covers IMP-01, IMP-02, IAUD-01
- [ ] `src/impersonation/test/services/impersonation-terminator.service.spec.ts` -- covers D-11 end session
- [ ] `src/impersonation/test/repositories/impersonation-audit.repository.spec.ts` -- covers IAUD-02, IAUD-03
- [ ] `src/impersonation/test/controllers/impersonation.controller.spec.ts` -- covers routing + validation + D-10 permission guard
- [ ] `src/impersonation/test/mocks/impersonation-mock-generator.ts` -- shared mocks
- [ ] `src/auth/test/auth.guard.spec.ts` -- extended to cover impersonation token detection (IMP-06)

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | Impersonation JWT signed with HS256 + dedicated secret; verified server-side on every request |
| V3 Session Management | yes | 30-minute hard expiry via JWT `exp` claim; no refresh mechanism (intentional) |
| V4 Access Control | yes | Controller-level supportRoles check (interim D-10 enforcement); lateral impersonation prevention (support->support blocked) |
| V5 Input Validation | yes | class-validator on StartImpersonationRequest (targetUserId, reason @MinLength(10)) |
| V6 Cryptography | no | HS256 symmetric signing with shared secret -- standard, not custom crypto |

### Known Threat Patterns for Impersonation

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Impersonation token theft | Spoofing | 30-minute hard expiry; tokens are short-lived. No refresh. |
| Lateral privilege escalation | Elevation of Privilege | Service validates target is customer-only before signing token |
| Audit log tampering | Tampering | Repository enforces append-only (no update/delete); controlled markEnded() only |
| Missing audit trail | Repudiation | Audit entry created BEFORE token is returned; reason is mandatory |
| Weak impersonation secret | Information Disclosure | IMPERSONATION_JWT_SECRET from environment; must be strong random value |
| Impersonation by non-support user | Elevation of Privilege | Controller checks `request.user.supportRoles.length === 0` and throws ForbiddenError (D-10) |
| Impersonation of non-existent user | Spoofing | UserRetriever.getById() returns null; service throws ResourceNotFoundError (404) |

## Sources

### Primary (HIGH confidence)
- Context7 /auth0/node-jsonwebtoken -- sign(), verify(), decode() API, algorithm options, expiresIn
- Context7 /nestjs/jwt -- JwtModule.register() pattern (used to understand why NOT to use it here)
- CLAUDE.md -- Full technology stack, installed package versions, conventions
- .planning/codebase/CONVENTIONS.md -- File naming, class naming, module structure
- .planning/codebase/STRUCTURE.md -- Directory structure, module organization
- .planning/codebase/INTEGRATIONS.md -- MongoDB connection pattern, auth flow
- trade-flow-api/src/user/services/user-retriever.service.ts -- CONFIRMED: getById(userId: string): Promise<IUserDto | null> at line 17

### Secondary (MEDIUM confidence)
- .planning/phases/51-rbac-data-model-seed/51-CONTEXT.md -- Permission model, role hydration on IUserDto
- .planning/phases/52-permission-guard-migration/52-CONTEXT.md -- @RequiresPermission decorator, guard execution order

### Tertiary (LOW confidence)
- None -- all claims verified against project documentation or Context7

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all packages already installed and documented in CLAUDE.md
- Architecture: HIGH -- follows established project patterns with one extension (dual-token guard)
- Pitfalls: HIGH -- derived from known JWT handling gotchas and project-specific audit concerns

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable stack, no fast-moving dependencies)
