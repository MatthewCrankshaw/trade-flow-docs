---
phase: 56-impersonation-backend-audit
fixed_at: 2026-04-19T13:00:00Z
review_path: .planning/phases/56-impersonation-backend-audit/56-REVIEW.md
iteration: 2
findings_in_scope: 3
fixed: 3
skipped: 0
status: all_fixed
---

# Phase 56: Code Review Fix Report

**Fixed at:** 2026-04-19T13:00:00Z
**Source review:** .planning/phases/56-impersonation-backend-audit/56-REVIEW.md
**Iteration:** 2

**Summary:**
- Findings in scope: 3
- Fixed: 3
- Skipped: 0

## Fixed Issues

### CR-01: Ended impersonation sessions remain usable until JWT expiry

**Files modified:** `trade-flow-api/src/auth/auth.guard.ts`, `trade-flow-api/src/auth/auth.module.ts`
**Commit:** 44417ea
**Applied fix:** Injected `ImpersonationAuditRepository` into `JwtAuthGuard` and added a session status check in `handleImpersonationToken()`. After verifying the JWT signature, the guard now queries the audit trail to confirm the session is still `ACTIVE` before granting access. Updated `AuthModule` to import `ImpersonationModule` (via `forwardRef`) so the repository is available for dependency injection.

### WR-01: No self-impersonation prevention

**Files modified:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts`
**Commit:** 7b8d331
**Applied fix:** Added an `authUser.id === targetUser.id` check in `ImpersonationCreator.create()` before the support-role check, throwing `ForbiddenError` with message "Cannot impersonate yourself".

### WR-02: Concurrent impersonation sessions not prevented

**Files modified:** `trade-flow-api/src/impersonation/services/impersonation-creator.service.ts`
**Commit:** 0ad769f
**Applied fix:** Added a check for existing active sessions using the existing `findBySupportUserId` repository method, filtering for `ACTIVE` status. If an active session exists, the request is rejected with a `ForbiddenError` instructing the support user to end the existing session first.

---

_Fixed: 2026-04-19T13:00:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 2_
