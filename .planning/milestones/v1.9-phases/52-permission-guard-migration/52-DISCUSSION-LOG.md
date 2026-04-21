# Phase 52: Permission Guard & Migration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 52-permission-guard-migration
**Areas discussed:** Guard & decorator design, Migration scope, Error responses, Endpoint protection

---

## Guard & Decorator Design

### Relationship to existing Policy/AccessControllerFactory

| Option | Description | Selected |
|--------|-------------|----------|
| Complement | @RequiresPermission is guard-level capability check. Policies stay as service-level resource ownership. Two distinct layers, no changes to policies. | ✓ |
| Extend policies | Add permission checks INTO existing policy classes. Merges capability + ownership into one layer. | |
| Replace policies | Migrate policies to use permissions entirely. Bigger refactor, unified model. | |

**User's choice:** Complement
**Notes:** User prompted this discussion by asking about the existing policy and access controller pattern. The two-layer model was clarified before proceeding.

### Multiple permission support

| Option | Description | Selected |
|--------|-------------|----------|
| Single permission only | One decorator per permission. Stack multiple decorators if needed. | |
| Multi with OR logic | @RequiresPermission('a', 'b') — user needs ANY listed permission. | ✓ |
| Multi with AND + OR | Both @RequiresPermission (OR) and @RequiresAllPermissions (AND). | |

**User's choice:** Multi with OR logic
**Notes:** None

### Guard execution order

| Option | Description | Selected |
|--------|-------------|----------|
| After JwtAuth, before Subscription | JwtAuthGuard → PermissionGuard → SubscriptionGuard. Permission needs request.user. | ✓ |
| After JwtAuth, after Subscription | Subscription check first, then permission. | |
| You decide | Claude determines optimal ordering. | |

**User's choice:** After JwtAuth, before Subscription
**Notes:** None

---

## Migration Scope

### Fate of isSupportUser()/isSuperUser() utilities

| Option | Description | Selected |
|--------|-------------|----------|
| Replace with permission checks | Delete utilities. All callers switch to specific permissions. No role-name-based checks. | ✓ |
| Keep as convenience wrappers | Rewrite internals to check permissions. Callers don't change. | |
| Deprecate gradually | Mark @deprecated, add hasPermission(). Migrate over multiple phases. | |

**User's choice:** Replace with permission checks
**Notes:** None

### Permission check helper style

| Option | Description | Selected |
|--------|-------------|----------|
| Utility function | hasPermission(user, permission) standalone function. Follows existing pattern. | ✓ |
| Method on user DTO | user.hasPermission('manage_users'). DTOs are currently plain interfaces. | |
| You decide | Claude determines best approach. | |

**User's choice:** Utility function
**Notes:** None

### Migration scope breadth

| Option | Description | Selected |
|--------|-------------|----------|
| Only success criteria checks | Migrate SubscriptionGuard/PaywallGuard only. Defer other checks. | |
| All hardcoded checks | Find and migrate every isSupportUser()/isSuperUser() call site across entire codebase. | ✓ |
| You decide | Claude determines scope based on codebase research. | |

**User's choice:** All hardcoded checks
**Notes:** None

---

## Error Responses

### Error detail level

| Option | Description | Selected |
|--------|-------------|----------|
| Reveal permission name | 403 with "Missing required permission: manage_users" and details object. | ✓ |
| Generic forbidden | 403 with generic "Forbidden" message. No indication of what's missing. | |
| You decide | Claude determines appropriate detail level. | |

**User's choice:** Reveal permission name
**Notes:** User clarified that the error handler already obfuscates error messages and details in production, so revealing permission names is safe and helpful for development/debugging.

### Error class

| Option | Description | Selected |
|--------|-------------|----------|
| Existing ForbiddenError | Reuse ForbiddenError with permission_denied type in details. No new class. | ✓ |
| New PermissionDeniedError | Separate error class for permission failures. Distinct error codes. | |

**User's choice:** Existing ForbiddenError
**Notes:** None

---

## Endpoint Protection

### Scope of @RequiresPermission application

| Option | Description | Selected |
|--------|-------------|----------|
| Support endpoints only | Only add to support-specific endpoints (Phases 53-57). Business endpoints keep using policies. | ✓ |
| All endpoints now | Proactively add to every controller. Business Administrator has all permissions so behaviour identical. | |
| You decide | Claude determines scope during research. | |

**User's choice:** Support endpoints only
**Notes:** None

### Build vs apply in this phase

| Option | Description | Selected |
|--------|-------------|----------|
| Build + apply to migrated checks | Build guard/decorator, apply where hardcoded checks are migrated. Guard proven working. | ✓ |
| Build only, apply in Phase 53 | Build infrastructure with tests only. Phase 53 is first consumer. | |

**User's choice:** Build + apply to migrated checks
**Notes:** None

---

## Claude's Discretion

- hasPermission() utility file location
- hasAnyPermission() variant
- SetMetadata key name
- Test structure and mock strategy

## Deferred Ideas

None
