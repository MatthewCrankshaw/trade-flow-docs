---
phase: 54
slug: user-management
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-20
---

# Phase 54 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| Client → API | SupportGuard + JwtAuthGuard + PermissionGuard(manage_users) | User PII, subscription data, business data |
| API → MongoDB | Server-side aggregation pipelines with parameterized queries | User documents, subscription documents |
| API → Firebase Admin | Server-side SDK with service account credentials | Auth metadata (created date, last sign-in) |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-54-02 | Tampering | query-filter-parser.utility.ts | mitigate | Filter field names validated against allowlist at consuming endpoint level (Plan 02) | closed |
| T-54-05 | Information Disclosure | support-user.controller.ts | mitigate | Endpoint restricted to `manage_users` permission holders; response excludes sensitive fields (no passwords, no tokens) | closed |
| T-54-08 | Tampering | support-user.controller.ts (users/:id) | mitigate | :id parameter validated as ObjectId by repository; invalid IDs throw ResourceNotFoundError → 404 | closed |
| T-54-10 | Information Disclosure | UserListTable.tsx | accept | Page protected by SupportGuard (Phase 53); data visibility matches user's permission level | closed |
| T-54-12 | Tampering | MembershipMetrics.tsx filter links | accept | Filter params hardcoded client-side; server validates all query params independently | closed |
| T-54-gc-02 | Tampering | firebase-auth-metadata.service.ts | accept | Debug logging does not expose sensitive data — only logs error.message, not credentials | closed |
| T-54-gc-04 | Spoofing | LoginPage.tsx | accept | Redirect target hardcoded /support, not user-controlled; actual access controlled by SupportGuard | closed |
| T-54-08-01 | Information Disclosure | UserListTable | accept | Data already delivered to client via API; only renders fields present in SupportUser response object | closed |
| T-54-09-01 | Information Disclosure | createdAt field in user list response | accept | Non-sensitive metadata; endpoint already restricted to support users with manage_users permission | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-54-01 | T-54-10 | UI page access controlled by SupportGuard; server enforces permission independently | plan author | 2026-04-20 |
| AR-54-02 | T-54-12 | Client-side filter params are display-only; server validates all inputs | plan author | 2026-04-20 |
| AR-54-03 | T-54-gc-02 | Only error.message logged; no credentials or tokens in debug output | plan author | 2026-04-20 |
| AR-54-04 | T-54-gc-04 | Redirect is hardcoded string; open redirect not possible | plan author | 2026-04-20 |
| AR-54-05 | T-54-08-01 | No new data exposure; renders existing response fields | plan author | 2026-04-20 |
| AR-54-06 | T-54-09-01 | createdAt is non-sensitive; endpoint already gated by permission | plan author | 2026-04-20 |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-20 | 9 | 9 | 0 | gsd-secure-phase |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-20
