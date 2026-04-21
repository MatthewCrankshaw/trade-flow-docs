# Phase 56: Impersonation Backend & Audit - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

The API supports creating time-limited impersonation sessions with full audit logging in a dedicated append-only collection. A support user with the `impersonate_user` permission can start a "login as" session for any customer user, receiving a time-limited token that resolves to the target user's identity on subsequent API calls. Impersonation of other support users is blocked. Every session is logged with who, whom, when, and reason.

</domain>

<decisions>
## Implementation Decisions

### Token Strategy
- **D-01:** API-signed JWT using `jsonwebtoken` (already installed) with a dedicated `IMPERSONATION_JWT_SECRET` env var. Firebase is not involved in impersonation tokens.
- **D-02:** Impersonation token payload contains: `type: "impersonation"`, `targetUserId`, `supportUserId`, `sessionId`, `exp` (30 min from creation).
- **D-03:** JwtAuthGuard detects token type by checking for impersonation type. Firebase-issued tokens follow existing flow. Impersonation tokens are verified against `IMPERSONATION_JWT_SECRET`, then hydrate `request.user` as the target user and `request.impersonator` as the support user.
- **D-04:** Full read/write access during impersonation. The impersonated `request.user` behaves identically to the real customer — all endpoints work normally. The audit trail (via `request.impersonator`) tracks who actually performed the action.

### Session Lifecycle
- **D-05:** Maximum session duration is 30 minutes, enforced server-side via JWT expiry. Support users can start a new session if more time is needed.
- **D-06:** Two termination paths: (1) Explicit — `POST /v1/impersonation/end` records `endedAt` timestamp in audit log, status "ended". (2) Automatic — JWT expires after 30 min, next API call returns 401 with message directing support user to start a new session. Audit entry has no `endedAt`, status "expired".

### Audit Collection Design
- **D-07:** Append-only enforced at repository level — `ImpersonationAuditRepository` exposes only `create()` and `find*()` methods. No `update()` or `delete()` methods exist. No DB-level enforcement.
- **D-08:** Session-level logging only — one document per impersonation session. Fields: `supportUserId`, `targetUserId`, `reason`, `startedAt`, `endedAt` (nullable), `status` ("active" | "ended" | "expired"). Individual API actions during the session are not logged.

### API Surface
- **D-09:** Dedicated `ImpersonationModule` following one-feature-per-directory convention. Contains: controller, creator/terminator services, audit repository, entity, DTO, response, and status enum.
- **D-10:** `POST /v1/impersonation/start` — accepts `targetUserId` (string) and `reason` (string, min 10 chars). Protected by `@RequiresPermission('impersonate_user')`. Returns impersonation JWT token in response.
- **D-11:** `POST /v1/impersonation/end` — accepts session ID. Records `endedAt` timestamp and sets status to "ended" in audit log.
- **D-12:** Reason field validated with `@IsString()`, `@IsNotEmpty()`, `@MinLength(10)`. Free text, no predefined categories. Returns 422 if too short.

### Claude's Discretion
- Exact JWT signing options (algorithm, issuer claim for detection)
- ImpersonationAuditEntity schema details (indexes on supportUserId, targetUserId, startedAt)
- Whether to add a `GET /v1/impersonation/active` endpoint to check if user has an active session
- Error message wording for expired sessions and lateral impersonation attempts
- Whether `ImpersonationModule` imports `UserModule` directly or uses a shared service for user lookup
- Test structure and mock strategy

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` — Permission/role data model, `impersonate_user` permission seeded, hydration strategy
- `.planning/phases/52-permission-guard-migration/52-CONTEXT.md` — `@RequiresPermission` decorator, `PermissionGuard`, `hasPermission()` utility, guard execution order

### Auth & JWT Infrastructure
- `trade-flow-api/src/auth/auth.guard.ts` — JwtAuthGuard to extend with impersonation token detection
- `trade-flow-api/src/auth/strategies/` — JWT strategy validating against Firebase public keys (impersonation uses separate verification)
- `trade-flow-api/src/user/services/user-retriever.service.ts` — UserRetriever hydration flow (needed to hydrate target user from impersonation token)

### Module & Repository Patterns
- `trade-flow-api/src/core/services/mongo/mongo-db-writer.service.ts` — MongoDbWriter for insert operations
- `trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts` — MongoDbFetcher for query operations
- `trade-flow-api/src/core/collections/dto.collection.ts` — DtoCollection pattern
- `.planning/codebase/CONVENTIONS.md` — File naming, class naming, module structure conventions

### Error Handling
- `trade-flow-api/src/core/errors/` — ForbiddenError (403 for lateral impersonation), InvalidRequestError (422 for validation), error utilities

### Downstream Consumer
- `.planning/phases/57-impersonation-frontend/` — Phase 57 will consume the impersonation token and session endpoints built here

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `jsonwebtoken` package — Already installed, supports signing/verifying JWTs with custom secrets
- `@nestjs/jwt` — JWT module available for NestJS integration
- `JwtAuthGuard` — Existing guard to extend with impersonation token type detection
- `UserRetriever.getByExternalAuthUserId()` — Hydration flow for building full IUserDto from user ID
- `ForbiddenError` / `InvalidRequestError` — Existing error classes for 403/422 responses
- `@RequiresPermission` decorator — Built in Phase 52, ready to protect impersonation endpoints

### Established Patterns
- One feature per NestJS module with controller/services/repository/entities/DTOs
- Service naming: `[Name]Creator`, `[Name]Retriever`, `[Name]Updater` (here: Creator + Terminator)
- Repository wraps `MongoDbWriter`/`MongoDbFetcher` with typed entity-DTO conversion
- Guards use `@Injectable()` + `CanActivate` interface
- DTOs validated with `class-validator` decorators

### Integration Points
- `JwtAuthGuard` — Must be updated to detect and handle impersonation tokens alongside Firebase tokens
- `UserModule` — ImpersonationModule needs access to `UserRetriever` for hydrating target user
- `AppModule` — Register ImpersonationModule
- `.env` — New `IMPERSONATION_JWT_SECRET` env var required

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 56-impersonation-backend-audit*
*Context gathered: 2026-04-18*
