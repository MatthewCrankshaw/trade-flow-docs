---
phase: 51
slug: 51-rbac-data-model-seed
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-19
---

# Phase 51 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| None (Plan 01) | Data model types only — no endpoints, no user input, no runtime behavior changes | None |
| Environment variable → seeder (Plan 02) | SUPER_USER_EMAIL env var determines who gets Super User role | Server-side config, never exposed to clients |
| MongoDB → role repositories (Plan 02) | Permission data from DB hydrated onto request.user via role repos | Non-secret permission definitions |

---

## Threat Register

| Threat ID | Category | Component | Disposition | Mitigation | Status |
|-----------|----------|-----------|-------------|------------|--------|
| T-51-01 | Elevation of Privilege | PermissionRepository cache | accept | Cache holds non-secret permission definitions populated from DB on boot. No user input reaches cache. | closed |
| T-51-02 | Elevation of Privilege | RbacSeeder.ensureSuperUserRole | mitigate | SUPER_USER_EMAIL checked server-side via ConfigService; unset or unmatched email returns early with no assignment. User lookup required before role is granted. | closed |
| T-51-03 | Elevation of Privilege | Permission hydration failure | mitigate | `(entity.permissionIds ?? [])` null-guard in both role repositories. Empty/undefined permissionIds yields `DtoCollection.empty()` — not a crash, not a silent grant. Phase 52 guards treat empty collection as "no access". | closed |
| T-51-04 | Denial of Service | Missing backfill on existing roles | mitigate | `seedBusinessAdministratorRoles` calls `collection.updateMany` with filter `{ roleName: BUSINESS_ADMINISTRATOR, permissionIds: { $exists: false } }` to backfill pre-existing roles without permissionIds. | closed |
| T-51-05 | Information Disclosure | Permission seed data | accept | Permission names/descriptions are system capability labels, not secrets. No sensitive data in seed payload. | closed |

*Status: open · closed*
*Disposition: mitigate (implementation required) · accept (documented risk) · transfer (third-party)*

---

## Accepted Risks Log

| Risk ID | Threat Ref | Rationale | Accepted By | Date |
|---------|------------|-----------|-------------|------|
| AR-51-01 | T-51-01 | PermissionRepository cache is populated from DB on boot with non-secret data (permission names/descriptions). No user input or runtime mutation. Cache serves read-only lookups for guard enforcement — no privilege escalation possible. | gsd-security-auditor | 2026-04-19 |
| AR-51-02 | T-51-05 | Permission seed data (names, descriptions, categories) describes system capabilities only. Contains no credentials, PII, or sensitive business data. Public disclosure would not enable attack. | gsd-security-auditor | 2026-04-19 |

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-19 | 5 | 5 | 0 | gsd-security-auditor |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-19
