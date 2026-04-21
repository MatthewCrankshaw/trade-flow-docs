---
phase: 52
slug: permission-guard-migration
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-19
---

# Phase 52 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| HTTP client → PermissionGuard | Untrusted request; user identity comes from JwtAuthGuard (upstream) | Bearer JWT token |
| PermissionGuard → request.user | User DTO with hydrated permissions; trusts JwtAuthGuard ran first | IUserDto with role/permission collections |
| Policy canRead/canWrite | Authorization decision points where support bypass logic changed from role-name to permission checks | authUser IUserDto |
| SubscriptionGuard → support bypass | Global guard where support bypass changed from supportRoles.length > 0 to permission check | bypass_subscription permission presence |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-52-01 | Elevation of Privilege | PermissionGuard | mitigate | Guard returns `true` only when no `@RequiresPermission` metadata is set; when metadata exists, user MUST have at least one required permission — default-deny. Verified: PermissionGuard 6 tests pass including "missing permission rejects" and "no user rejects" cases. | closed |
| T-52-02 | Elevation of Privilege | hasPermission | mitigate | Empty `DtoCollection.some()` returns `false`; `!!role.permissions &&` guard handles missing permissions field; null/undefined user throws ForbiddenError — no fallback to "allow". Verified: 9 tests covering empty collections, missing permissions field, no-roles user. | closed |
| T-52-03 | Spoofing | PermissionGuard | accept | Guard trusts `request.user` from JwtAuthGuard. JwtAuthGuard validates Firebase JWT with RS256 public keys — spoofing requires compromising Firebase infrastructure. See Accepted Risks Log. | closed |
| T-52-04 | Information Disclosure | ForbiddenError message | accept | Error message reveals required permission name (e.g. "Missing required permission: manage_users"). Acceptable: error handler obfuscates details in production; leaking permission names poses low risk. See Accepted Risks Log. | closed |
| T-52-05 | Elevation of Privilege | Policy migration (15 files) | mitigate | Every `isSupportUser()` call replaced with `hasPermission(authUser, "view_support_dashboard")` — fail-closed when permissions not hydrated. Phase 51 seeds Super User and Admin roles with this permission. Verified: policy test mocks updated with permission-carrying role objects; 855 tests pass. | closed |
| T-52-06 | Elevation of Privilege | MigrationPolicy | mitigate | `isSuperUser()` replaced with `hasPermission(authUser, "manage_migrations")` — only Super User role has this permission. Admin role does NOT have manage_migrations, preserving restricted access. Verified: MigrationPolicy migration confirmed in commit 6dbd7c2. | closed |
| T-52-07 | Denial of Service | Subscription bypass | mitigate | `supportRoles.length > 0` replaced with `hasPermission(user, "bypass_subscription")` — fail-closed behavior: if permissions fail to hydrate, hasPermission returns false and bypass is denied. Negative test added: support user without bypass_subscription permission receives ForbiddenError. Verified: subscription.guard.spec.ts updated, CI passes. | closed |
| T-52-08 | Elevation of Privilege | Deleted utilities | mitigate | Old utilities (`is-support-user.utility.ts`, `is-super-user.utility.ts`) deleted only after ALL call sites migrated. TypeScript compilation would immediately fail if any import remains, preventing deployment of broken code. Verified: `grep -r "is-support-user.utility\|is-super-user.utility" src/` returns zero results; `npm run ci` exits 0. | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| R-52-01 | T-52-03 | PermissionGuard trusts request.user from JwtAuthGuard. Mitigating spoofing at this layer would require duplicating Firebase JWT validation. JwtAuthGuard is the authoritative trust boundary for identity; PermissionGuard operates downstream. Compromising this requires attacking Firebase RS256 infrastructure. | gsd-security-auditor | 2026-04-19 |
| R-52-02 | T-52-04 | ForbiddenError includes required permission name in details field (e.g. "Missing required permission: manage_users"). In production, the global error handler maps ForbiddenError to a 403 response with only the ErrorCodes enum value — permission names are not leaked to HTTP clients. Low risk accepted. | gsd-security-auditor | 2026-04-19 |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-19 | 8 | 8 | 0 | gsd-security-auditor |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-19
