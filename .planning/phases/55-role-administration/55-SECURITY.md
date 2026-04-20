---
phase: 55
slug: role-administration
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-20
---

# Phase 55 — Security Audit

## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Browser -> API | Untrusted HTTP requests from authenticated users to POST/DELETE /v1/users/:id/support-role |
| Super user -> target user | Privilege escalation vector: granting or revoking elevated access to another user |
| Guard -> Controller | Guards execute before controller try/catch; errors must be NestJS HttpExceptions, not domain errors |
| Viewer role check (frontend) | Frontend visibility logic hides buttons; backend PermissionGuard is the real enforcement |

## Threat Register

| Threat ID | Category | Component | Disposition | Status | Evidence |
|-----------|----------|-----------|-------------|--------|----------|
| T-55-01 | Elevation of Privilege | POST /v1/users/:id/support-role | mitigate | CLOSED | `@RequiresPermission("manage_roles")` + `@UseGuards(JwtAuthGuard, PermissionGuard)` at lines 14, 19 of `support-role-admin.controller.ts` |
| T-55-02 | Elevation of Privilege | DELETE /v1/users/:id/support-role | mitigate | CLOSED | `@RequiresPermission("manage_roles")` present. Self-revocation guard present (`"self_revocation"`). Super-user protection present (`"system_managed_role"`). Last-admin count check implemented: `countBySupportRoleId(adminRole.id)` called before removal, throws `ForbiddenError(..., "last_admin_protection")` when count <= 1. Test added confirming block. |
| T-55-03 | Tampering | Param :id | mitigate | CLOSED | `userRetriever.getById(targetUserId)` with explicit null check throwing `ResourceNotFoundError` (lines 24-27, 53-56). MongoDB ObjectId constructor in repository throws on malformed strings. |
| T-55-04 | Denial of Service | Self-revocation of last admin | mitigate | CLOSED | Self-revocation blocked by ForbiddenError. Super-user role cannot be revoked. Last-admin count check now blocks revocation when only one support administrator remains (`countBySupportRoleId` <= 1 throws `"last_admin_protection"`). Same fix as T-55-02. |
| T-55-05 | Information Disclosure | Error messages | accept | CLOSED | Accepted: ForbiddenError details strings are only reachable by authenticated super users who pass PermissionGuard. Logged in accepted risks below. |
| T-55-06 | Repudiation | Role changes | accept | CLOSED | Accepted: AppLogger calls present at lines 39 and 69 of `support-role-assigner.service.ts`. Full audit log deferred. Logged in accepted risks below. |
| T-55-07 | Tampering | RTK Query mutations | mitigate | CLOSED | Mutations in `supportApi.ts` send only `userId` in URL path — no request body. Backend validates userId and permissions independently. |
| T-55-08 | Spoofing | Viewer role check in RoleActions | mitigate | CLOSED | `viewerIsSuperUser` check at line 23 of `RoleActions.tsx` provides defense in depth. Backend PermissionGuard is primary enforcement. |
| T-55-09 | Information Disclosure | Error toast messages | accept | CLOSED | Accepted: error toasts visible only to the authenticated user performing the action. Logged in accepted risks below. |
| T-55-10 | Elevation of Privilege | SubscriptionGuard bypass | mitigate | CLOSED | `@SkipSubscriptionCheck()` present at lines 20 and 35 of `support-role-admin.controller.ts`, on endpoints already gated by `@RequiresPermission("manage_roles")`. No open bypass. |
| T-55-11 | Information Disclosure | PermissionGuard error message | mitigate | CLOSED | `permission.guard.ts` throws `ForbiddenException` with `details: "Missing required permission: ${requiredPermissions.join(", ")}"` — only the permission name is exposed, no internal IDs or user data. |

## Resolved Threats Detail

### T-55-02 / T-55-04 — Last-Admin Protection (RESOLVED 2026-04-20)

**Fix applied:** Added `countBySupportRoleId(adminRole.id)` check in `SupportRoleAssigner.revoke()` before calling `removeSupportRoleId`. Throws `ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "last_admin_protection")` when count <= 1. Test case added confirming revocation is blocked for the last support administrator.

## Accepted Risks Log

| Risk ID | Threat ID | Description | Accepted By | Date |
|---------|-----------|-------------|-------------|------|
| AR-55-01 | T-55-05 | ForbiddenError detail strings ("self_revocation", "system_managed_role", "last_admin_protection") reveal reason for rejection. Acceptable because only authenticated super users with manage_roles permission reach these endpoints. | Phase 55 threat model | 2026-04-19 |
| AR-55-02 | T-55-06 | Role changes logged via AppLogger (info level, structured JSON via Pino). Full immutable audit trail deferred to a future milestone per REQUIREMENTS.md. | Phase 55 threat model | 2026-04-19 |
| AR-55-03 | T-55-09 | Error toasts in RevokeRoleDialog show specific failure reasons ("You cannot revoke your own role.", "Cannot revoke the last super user."). Visible only to the authenticated user performing the action. | Phase 55 threat model | 2026-04-19 |

## Security Audit Trail

| Date | Action | By |
|------|--------|----|
| 2026-04-19 | Phase 55-01 implemented: backend endpoints, SupportRoleAssigner, PermissionGuard infrastructure | Executor |
| 2026-04-19 | Phase 55-02 implemented: frontend RTK Query mutations, RoleActions, dialogs, SupportUserDetailPage | Executor |
| 2026-04-19 | Phase 55-03 implemented: SkipSubscriptionCheck on endpoints, PermissionGuard ForbiddenException fix | Executor |
| 2026-04-20 | Security audit performed against threat register | GSD Security Auditor |
| 2026-04-20 | T-55-02/T-55-04 gap closed: last_admin_protection check added to revoke() | GSD Secure Phase |

## Sign-Off Checklist

- [x] All threats in register verified by disposition
- [x] Accepted risks documented with rationale
- [x] T-55-02 last-admin count check implemented in `SupportRoleAssigner.revoke()`
- [x] T-55-04 last-admin DoS path blocked by count check
- [x] Re-audit after gap closure to confirm CLOSED status
- [x] SECURITY.md status updated to `verified` and `threats_open: 0` after closure
