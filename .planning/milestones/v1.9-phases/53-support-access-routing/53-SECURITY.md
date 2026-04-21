---
phase: 53
slug: support-access-routing
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-19
---

# Phase 53 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| Client route guard (SupportGuard) | UX convenience only; backend API (Phase 52 PermissionGuard) is authoritative enforcement | User role data from RTK Query cache (server-authoritative) |
| Login redirect logic | Client-side routing decision based on server-provided role data | supportRoles array from useGetCurrentUserQuery |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-53-01 | Tampering | SupportGuard client-side bypass via URL manipulation | accept | Frontend guards are UX convenience only; backend API enforces permissions via Phase 52 PermissionGuard. Direct navigation to /support without support role results in redirect to /dashboard, not data exposure. | closed |
| T-53-02 | Elevation of Privilege | Support role spoofing in client state | accept | Role data from useGetCurrentUserQuery is server-authoritative; RTK Query cache reflects server state. API endpoints enforce permissions independently of client routing. | closed |
| T-53-03 | Information Disclosure | Navigation items reveal support routes exist | accept | Route paths (/support, /support/users) are not sensitive; no data is exposed without backend authorization. | closed |
| T-53-04 | Spoofing | Manipulated location.state.from to redirect to arbitrary URL | mitigate | React Router Navigate with replace:true uses client-side routing only (same-origin SPA navigation). location.state.from cannot redirect to external URLs — it is a path string consumed by useNavigate, which only handles internal routes. | closed |
| T-53-05 | Information Disclosure | Stale role data after role revocation shows support nav | accept | RTK Query refetches user on page load/token refresh. Phase 55 (RADM-04) addresses immediate effect of role changes. No sensitive data in support nav items themselves. | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-53-01 | T-53-01 | Frontend guards are defense-in-depth only; backend PermissionGuard (Phase 52) is authoritative. No data exposed via client-side bypass. | phase-plan | 2026-04-19 |
| AR-53-02 | T-53-02 | Client cannot fabricate supportRoles — data is server-authoritative via API response. | phase-plan | 2026-04-19 |
| AR-53-03 | T-53-03 | Route paths are not sensitive information; all data access requires backend authorization. | phase-plan | 2026-04-19 |
| AR-53-04 | T-53-05 | RTK Query refetches on page load. Immediate revocation effect deferred to Phase 55 (RADM-04). Acceptable because support nav items expose no data. | phase-plan | 2026-04-19 |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-19 | 5 | 5 | 0 | gsd-secure-phase |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-19
