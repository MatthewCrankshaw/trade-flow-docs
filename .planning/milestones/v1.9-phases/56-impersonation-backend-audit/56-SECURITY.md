---
phase: 56-impersonation-backend-audit
slug: impersonation-backend-audit
status: SECURED
threats_open: 0
asvs_level: 1
created: 2026-04-20
---

# Security Audit — Phase 56: Impersonation Backend Audit

## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| API request → ImpersonationController | Untrusted Bearer tokens and request bodies from support clients |
| ImpersonationController → ImpersonationCreator | Authenticated support user initiating a session; targetUserId must not reference another support user |
| JwtAuthGuard → downstream endpoints | After impersonation token verification, request.user is set to the target user — all downstream endpoints trust this identity |
| SubscriptionGuard impersonation bypass | Impersonated requests skip subscription checks; correctness relies on JwtAuthGuard having already cryptographically verified the token and validated session status |
| API request → impersonation-audit.repository.ts | Audit entries are permanent; the repository interface controls what mutations are possible |

## Threat Register

| Threat ID | Category | Component | Disposition | Status | Evidence |
|-----------|----------|-----------|-------------|--------|----------|
| T-56-01 | Tampering | impersonation-audit.repository.ts | mitigate | CLOSED | Repository exposes `create()`, `findBySessionId()`, `findBySupportUserId()`, and `markEnded()` only. No public `update()` or `delete()` method exists. The only `updateOne()` call is inside `markEnded()` — the single controlled mutation. File: `src/impersonation/repositories/impersonation-audit.repository.ts` lines 65–71. |
| T-56-02 | Spoofing | start-impersonation.request.ts | mitigate | CLOSED | `@IsString()`, `@IsNotEmpty()`, and `@MinLength(10)` decorators present on `reason`; `@IsString()`, `@IsNotEmpty()`, and `@Matches(/^[a-f\d]{24}$/i)` on `targetUserId`. NestJS `ValidationPipe` rejects invalid payloads with 400. File: `src/impersonation/requests/start-impersonation.request.ts` lines 1–13. |
| T-56-03 | Spoofing | auth.guard.ts | mitigate | CLOSED | `jwt.decode()` used to peek at the `iss` claim (routing only). `jwt.verify()` called with `algorithms: ["HS256"]` and `issuer: "trade-flow-impersonation"` before any payload data is trusted. Note: implementation routes by `iss` claim rather than `type` claim as described in the plan — the security property is identical: unverified peek, then full cryptographic verification before trusting payload. File: `src/auth/auth.guard.ts` lines 44–48 (peek) and 114–117 (verify). |
| T-56-04 | Elevation of Privilege | impersonation-creator.service.ts | mitigate | CLOSED | `targetUser.supportRoles.length > 0` throws `ForbiddenError` at line 46, which is BEFORE `this.auditRepository.create()` at line 75. Lateral impersonation of support users is blocked with no audit entry created. File: `src/impersonation/services/impersonation-creator.service.ts` lines 46–48. |
| T-56-06 | Spoofing | auth.guard.ts | mitigate | CLOSED | `jwt.TokenExpiredError` caught specifically in `handleImpersonationToken()`. Returns 401 with message "Impersonation session expired. Please start a new session." — prevents use of expired tokens beyond the 30-minute hard limit. File: `src/auth/auth.guard.ts` lines 138–140. |
| T-56-07 | Repudiation | impersonation-creator.service.ts | mitigate | CLOSED | `this.auditRepository.create(auditDto)` at line 75 executes BEFORE `jwt.sign()` at line 77. If signing fails, the audit entry already exists. `reason` field enforced with `@MinLength(10)` on the request class. File: `src/impersonation/services/impersonation-creator.service.ts` lines 75–91. |
| T-56-08 | Information Disclosure | auth.guard.ts | mitigate | CLOSED | `jwt.JsonWebTokenError` (covers all other JWT errors) returns the generic message "Invalid impersonation token" with no payload details. Only `TokenExpiredError` receives a distinct, non-leaking message. File: `src/auth/auth.guard.ts` lines 141–143. |
| T-56-09 | Elevation of Privilege | impersonation.controller.ts | mitigate | CLOSED | Controller checks `!request.user.supportRoles || request.user.supportRoles.length === 0` and throws `ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "Impersonation requires support role")` before delegating to the creator service. Present on both `start` and `end` endpoints. File: `src/impersonation/controllers/impersonation.controller.ts` lines 31–33 and 46–48. |
| T-56-10 | Elevation of Privilege | subscription.guard.ts | mitigate | CLOSED | `request.impersonator` check at line 53 returns `true` only when the field is present. The field is set exclusively by `JwtAuthGuard.handleImpersonationToken()` after cryptographic `jwt.verify()` + audit session status validation (`session.status !== ImpersonationStatus.ACTIVE`). File: `src/subscription/guards/subscription.guard.ts` lines 53–55. |

## Accepted Risks Log

None. All threats in this phase are mitigated.

## Unregistered Threat Flags

The following items were noted in SUMMARY files but do not map to new unregistered threats:

| Flag | Source | Assessment |
|------|--------|------------|
| T-56-05 (DoS — extra decode per request) | 56-02-PLAN.md threat model | Declared accepted in plan's own threat model. Not in the cross-plan register provided. Disposition: accept. Low overhead (base64 parse, no crypto). |
| T-56-11 (Tampering — auto-termination modifies audit entries) | 56-04-PLAN.md threat model | Declared accepted in plan's own threat model. Uses the same controlled `markEnded()` path as normal session termination. Audit trail preserved with `endedAt` timestamp. |
| T-56-12 (DoS — rapid session creation) | 56-04-PLAN.md threat model | Declared accepted in plan's own threat model. Limited to users with support roles; each session is logged. |
| TypeScript errors from Phase 57 commit e9e6682 | 56-04-SUMMARY.md | Out-of-scope for this phase; noted as Phase 57 responsibility. No security impact — compilation errors in response type narrowing, not in auth or audit logic. |

## Security Audit Trail

| Date | Auditor | Action |
|------|---------|--------|
| 2026-04-20 | gsd-security-auditor (Claude Sonnet 4.6) | Initial audit of Plans 01–04. Verified all 9 registered threats against implementation source. All mitigations confirmed present in code. |

## Notes

**T-56-03 routing mechanism deviation:** The plan described routing impersonation tokens by checking `decoded.type === "impersonation"`. The implementation routes by checking `decoded.payload.iss === "trade-flow-impersonation"`. Both achieve the same security property: a non-cryptographic peek is used only for routing decisions, and `jwt.verify()` with full algorithm and issuer constraints is always called before any payload data is acted upon. The implementation's approach is equally sound — the `iss` claim is part of the signed JWT and is validated by `jwt.verify()` with the `issuer` option.

**T-56-09 end endpoint:** The plan specified the support-role check on the `start` endpoint. The implementation also applies the same check on the `end` endpoint (line 46–48), which is a defence-in-depth improvement not listed as a threat.

## Sign-Off Checklist

- [x] All 9 registered threats verified against implementation source code
- [x] No registered threat has an open status
- [x] Unregistered flags from SUMMARY.md incorporated and assessed
- [x] Implementation files not modified during audit
- [x] SECURITY.md written to `.planning/phases/56-impersonation-backend-audit/56-SECURITY.md`
- [x] ASVS Level 1 requirements met (authentication, input validation, error handling reviewed)
