# Phase 52: Permission Guard & Migration - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the `@RequiresPermission()` decorator and `PermissionGuard` infrastructure that validates user permissions on API endpoints, then migrates ALL existing hardcoded role checks (`isSupportUser()`, `isSuperUser()`, `supportRoles.length > 0` checks) to use the new permission system. The guard complements (does not replace) the existing Policy/AccessControllerFactory resource-ownership pattern.

</domain>

<decisions>
## Implementation Decisions

### Guard & Decorator Design
- **D-01:** `@RequiresPermission` is a guard-level capability check that complements the existing Policy/AccessControllerFactory pattern. Two distinct authorization layers: guard checks "can you do this type of thing?" (capability), policy checks "can you access THIS specific resource?" (ownership). No changes to policies in this phase.
- **D-02:** `@RequiresPermission` supports multiple permissions with OR logic — `@RequiresPermission('manage_users', 'view_support_dashboard')` means user needs ANY of the listed permissions. Uses `SetMetadata` to store required permissions, `PermissionGuard` reads them via `Reflector`.
- **D-03:** Guard execution order: `JwtAuthGuard → PermissionGuard → SubscriptionGuard`. Permission check needs `request.user` (from JwtAuth). Support users with `bypass_subscription` permission skip SubscriptionGuard naturally through the RBAC system.

### Migration Scope
- **D-04:** Delete `isSupportUser()` and `isSuperUser()` utilities entirely. All callers switch to checking specific permissions via a new `hasPermission()` utility function. No more role-name-based checks anywhere in the codebase.
- **D-05:** New `hasPermission(user: IUserDto, permission: string): boolean` utility function — standalone in `src/user/utilities/` or `src/auth/utilities/`. Follows existing utility function pattern. Reads permissions from the hydrated role DTOs on the user object (set up in Phase 51).
- **D-06:** Migrate ALL hardcoded role checks across the entire codebase, not just the ones listed in success criteria (SubscriptionGuard, PaywallGuard). Find every `isSupportUser()`/`isSuperUser()` call site and migrate them all.

### Error Responses
- **D-07:** Permission guard rejection reveals the missing permission name in the 403 error: `"Missing required permission: manage_users"` with `details: { required: ["manage_users"], type: "permission_denied" }`. Safe because the existing error handler already obfuscates error messages and details in production.
- **D-08:** Use the existing `ForbiddenError` class with a `permission_denied` type in details. No new error class needed — `createHttpError()` already maps `ForbiddenError` to 403.

### Endpoint Protection
- **D-09:** Only add `@RequiresPermission` to support-specific endpoints (Phases 53-57). Existing business endpoints (jobs, quotes, customers) keep using policies for resource ownership. Permission decorators on business endpoints deferred to when team roles arrive.
- **D-10:** Build the guard infrastructure AND apply it to the migrated checks in this phase. The guard is proven working by end of phase, not just theoretical infrastructure. SubscriptionGuard bypass becomes a permission check (`bypass_subscription`) instead of role array length check.

### Claude's Discretion
- Whether `hasPermission()` lives in `src/user/utilities/` or `src/auth/utilities/`
- Whether to add a `hasAnyPermission(user, permissions[])` variant alongside `hasPermission()`
- Exact `SetMetadata` key name for the permission decorator
- Test structure and mock strategy for the PermissionGuard

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 51 Context (prerequisite)
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` — Permission/role data model decisions, hydration strategy, seeding approach

### Guard & Auth Infrastructure
- `trade-flow-api/src/auth/auth.guard.ts` — JwtAuthGuard pattern to follow for PermissionGuard
- `trade-flow-api/src/subscription/guards/subscription.guard.ts` — SubscriptionGuard with hardcoded support bypass to migrate
- `trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts` — `@SkipSubscriptionCheck` decorator pattern (SetMetadata + Reflector) to follow

### Hardcoded Checks to Migrate
- `trade-flow-api/src/user/utilities/is-support-user.utility.ts` — isSupportUser() utility to delete
- `trade-flow-api/src/user/utilities/is-super-user.utility.ts` — isSuperUser() utility to delete
- All call sites of these utilities across the codebase (researcher must grep for them)

### Authorization Patterns (preserve, don't modify)
- `trade-flow-api/src/core/factories/` — AccessControllerFactory and AuthorizedCreatorFactory (resource-level auth stays unchanged)
- `trade-flow-api/src/business/policies/` — Example policy class (BusinessPolicy) — stays unchanged

### Error Handling
- `trade-flow-api/src/core/errors/` — ForbiddenError class and createHttpError() utility
- `trade-flow-api/src/core/errors/error-codes.enum.ts` — ErrorCodes enum (if permission_denied code needed)

### User DTO & Role Hydration
- `trade-flow-api/src/user/data-transfer-objects/user.dto.ts` — IUserDto with hydrated role DTOs (permissions accessible via roles)
- `trade-flow-api/src/user/services/user-retriever.service.ts` — UserRetriever hydration flow

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `@SkipSubscriptionCheck` decorator — SetMetadata + Reflector pattern, exact template for `@RequiresPermission`
- `ForbiddenError` class — reuse for permission denials, no new error class needed
- `createHttpError()` — maps ForbiddenError to 403 HTTP exception
- `DtoCollection<IPermissionDto>` — permissions already hydrated on role DTOs (Phase 51)

### Established Patterns
- Guards use `@Injectable()` + `CanActivate` interface with `canActivate(context: ExecutionContext)` method
- Decorators use `SetMetadata()` to store metadata, guards read via `Reflector.getAllAndOverride()`
- Utility functions in `src/user/utilities/` — standalone functions taking `IUserDto` as first argument
- Error classes extend a base with `getCode()`, `getMessage()`, `getDetails()` methods

### Integration Points
- `SubscriptionGuard` — must be updated to check `bypass_subscription` permission instead of role array length
- Every call site of `isSupportUser()`/`isSuperUser()` — must be found and migrated
- `UserModule` or `AuthModule` — PermissionGuard registered here
- Controller decorators — `@UseGuards()` ordering updated where PermissionGuard is needed

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches following existing codebase patterns (especially the `@SkipSubscriptionCheck` decorator pattern).

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 52-permission-guard-migration*
*Context gathered: 2026-04-18*
