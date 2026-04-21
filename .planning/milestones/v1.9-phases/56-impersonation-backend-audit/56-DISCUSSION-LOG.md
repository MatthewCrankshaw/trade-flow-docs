# Phase 56: Impersonation Backend & Audit - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 56-impersonation-backend-audit
**Areas discussed:** Token strategy, Session lifecycle, Audit collection design, API surface

---

## Token Strategy

### How should impersonation tokens work?

| Option | Description | Selected |
|--------|-------------|----------|
| API-signed JWT | API signs a separate JWT with its own secret (IMPERSONATION_JWT_SECRET). Contains targetUserId, supportUserId, sessionId, expiry. JwtAuthGuard detects token type and hydrates request.user as target, request.impersonator as support user. Stateless. | ✓ |
| Session header approach | Support user keeps Firebase token. Sends X-Impersonate-Session header. Middleware looks up session in DB on every request. More DB load but simpler token management. | |

**User's choice:** API-signed JWT
**Notes:** User asked whether signing a separate JWT was possible alongside Firebase auth. Confirmed that Firebase and API-signed JWTs can coexist — JwtAuthGuard distinguishes by issuer/type.

### Should impersonated requests be restricted to read-only?

| Option | Description | Selected |
|--------|-------------|----------|
| Full access | Support user can do everything the customer can. Audit trail tracks who actually did it via request.impersonator. | ✓ |
| Read-only | Support user can only view customer data during impersonation. | |

**User's choice:** Full access

---

## Session Lifecycle

### Maximum session duration?

| Option | Description | Selected |
|--------|-------------|----------|
| 30 minutes | Short enough to limit risk, long enough for support. Can start new session if needed. | ✓ |
| 1 hour | More room for complex tasks. Longer exposure window. | |
| Configurable via env var | Default 30 min, override via IMPERSONATION_MAX_DURATION_MINUTES. | |

**User's choice:** 30 minutes

### How should session termination work?

| Option | Description | Selected |
|--------|-------------|----------|
| Explicit end endpoint | POST /v1/impersonation/end records end timestamp. Token also expires naturally. Both paths: clean exit and timeout. | ✓ |
| Token expiry only | No explicit end endpoint. Token expires, frontend discards. Simpler but no clean end timestamp. | |

**User's choice:** Explicit end endpoint

---

## Audit Collection Design

### How should append-only be enforced?

| Option | Description | Selected |
|--------|-------------|----------|
| Repository-level only | Repository exposes only create() and find() methods. No update/delete. Code-level enforcement. | ✓ |
| Repository + DB-level | Same plus MongoDB validator or role restriction. Defence in depth but more operational complexity. | |

**User's choice:** Repository-level only

### Should audit log capture individual API actions?

| Option | Description | Selected |
|--------|-------------|----------|
| Session-level only | One document per session: who, whom, when, reason, end status. Matches IAUD-01 exactly. | ✓ |
| Session + actions | Also log each API request during session. More forensic detail but more write volume. | |

**User's choice:** Session-level only

---

## API Surface

### Own module or inside auth module?

| Option | Description | Selected |
|--------|-------------|----------|
| Own module | New ImpersonationModule with controller, services, repository, entities. Follows one-feature-per-directory. | ✓ |
| Inside auth module | Add to existing auth module. Fewer directories but mixes concerns. | |

**User's choice:** Own module

### Reason field validation?

| Option | Description | Selected |
|--------|-------------|----------|
| Free text, min length | Required string, min 10 characters. No predefined categories. | ✓ |
| Predefined categories + notes | Dropdown of reasons plus optional notes. More structured but requires maintaining category list. | |

**User's choice:** Free text, min length (10 characters)

---

## Claude's Discretion

- JWT signing options (algorithm, issuer claim)
- Audit entity indexes
- Whether to add GET /v1/impersonation/active endpoint
- Error message wording
- Module dependency strategy (UserModule import)
- Test structure and mocks

## Deferred Ideas

None — discussion stayed within phase scope.
