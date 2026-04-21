---
phase: 57
slug: impersonation-frontend
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-20
---

# Phase 57 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| Redux store -> prepareHeaders | Token switching based on impersonation state | Impersonation JWT token |
| API response -> baseQuery wrapper | 401 status triggers impersonation cleanup | HTTP status codes |
| Redux store -> Route Guards | Impersonation state bypasses onboarding/paywall | Boolean flags |
| DashboardLayout -> navigation config | Navigation items change based on impersonation state | UI config |
| Permission check -> button visibility | Frontend hides button for unauthorized users | Role/permission data |
| API response -> Frontend | Adding user PII (name, email) to response payload | User PII |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-57-01 | Information Disclosure | impersonationSlice | mitigate | Token stored in Redux only — no localStorage/sessionStorage persistence | closed |
| T-57-02 | Elevation of Privilege | prepareHeaders | mitigate | Token switching only when `state.impersonation.active` AND `state.impersonation.token` are both truthy | closed |
| T-57-03 | Elevation of Privilege | baseQuery wrapper | mitigate | 401 during active impersonation triggers `endImpersonation()` + `resetApiState()` | closed |
| T-57-04 | Tampering | apiSlice | mitigate | `apiSlice.util.resetApiState()` called on both startSession AND endSession | closed |
| T-57-05 | Elevation of Privilege | OnboardingGuard | mitigate | Bypass gated by `state.impersonation.active` (set only via authenticated backend call) | closed |
| T-57-06 | Elevation of Privilege | PaywallGuard | mitigate | Bypass gated by `state.impersonation.active` (same as T-57-05) | closed |
| T-57-07 | Spoofing | ImpersonationBanner | accept | Banner z-[60] ensures visibility above all Radix overlays; no dismiss mechanism | closed |
| T-57-08 | Elevation of Privilege | ImpersonateUserButton | mitigate | Button hidden entirely when viewer lacks support roles; backend enforces 403 | closed |
| T-57-09 | Elevation of Privilege | ImpersonateUserButton | mitigate | Button hidden when target has any support role IDs; backend enforces 403 | closed |
| T-57-10 | Repudiation | ImpersonateUserDialog | mitigate | Required reason field ensures audit trail; submitted to POST /v1/impersonation/start | closed |
| T-57-11 | Spoofing | ImpersonateUserDialog | accept | Reason field is user-supplied free text; backend is responsible for sanitization | closed |
| T-57-04-01 | Information Disclosure | IImpersonationResponse.targetUser | accept | Only support users with impersonate_user permission can call endpoint; data already visible on support user detail page | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-57-01 | T-57-07 | Banner visibility is defense-in-depth; no dismiss ensures user awareness. Failure mode is cosmetic only — backend still enforces impersonation constraints. | plan author | 2026-04-19 |
| AR-57-02 | T-57-11 | Free text input sanitization is a backend responsibility (XSS/injection). Frontend validates non-empty only. | plan author | 2026-04-19 |
| AR-57-03 | T-57-04-01 | Target user PII (name, email) only exposed to authenticated support users who already have access to this data via user detail pages. No new data exposure. | plan author | 2026-04-19 |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-20 | 12 | 12 | 0 | gsd-secure-phase |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-20
