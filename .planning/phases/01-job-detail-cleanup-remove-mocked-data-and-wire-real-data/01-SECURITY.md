---
phase: 01-job-detail-cleanup-remove-mocked-data-and-wire-real-data
audited_at: 2026-04-23
asvs_level: 1
threats_total: 10
threats_closed: 10
threats_open: 0
result: SECURED
---

# Security Audit — Phase 01: Job Detail Cleanup

## Summary

All 10 registered threats verified. 5 mitigate-disposition threats have confirmed implementation evidence. 5 accept-disposition threats are logged in the accepted risks register below. No unregistered threat flags were raised in any SUMMARY.md.

## Threat Verification

| Threat ID | Category | Disposition | Status | Evidence |
|-----------|----------|-------------|--------|----------|
| T-01-01 | Information Disclosure | mitigate | CLOSED | `job-event.policy.ts:11,17` — `authUser.businessIds.includes(resource.businessId)` in `canRead()`; `job-event-retriever.service.ts:23-25` — `ForbiddenError` thrown when policy returns false |
| T-01-02 | Tampering | mitigate | CLOSED | `job-event.controller.ts` — controller exposes only `@Get("job-event")`; no `@Post`, `@Put`, `@Patch`, or `@Body` decorators present; `jobEventCreator.create` found only in 11 server-side service files, never in controller |
| T-01-03 | Spoofing | mitigate | CLOSED | `job-event.controller.ts:1,15` — `JwtAuthGuard` imported and applied via `@UseGuards(JwtAuthGuard)` on the GET route |
| T-01-04 | Denial of Service | accept | CLOSED | Logged in accepted risks register below |
| T-01-05 | Input Validation | mitigate | CLOSED | `job-event.repository.ts:30` — `new ObjectId(jobId)` throws on invalid input; `job-event.controller.ts:21,25-26` — try/catch with `createHttpError` propagates the exception as an HTTP error |
| T-01-06 | Information Disclosure | accept | CLOSED | Logged in accepted risks register below |
| T-01-07 | Tampering | accept | CLOSED | Logged in accepted risks register below |
| T-01-08 | Tampering | mitigate | CLOSED | `job-event.controller.ts` — no write routes; event writes confirmed only in server-side services: quote-creator, quote-transition, schedule-creator, schedule-updater, schedule-transition, job-updater, estimate-creator, estimate-transition, estimate-to-quote-converter, estimate-lost-marker (11 service files, metadata derived from server state post-operation) |
| T-01-09 | Repudiation | accept | CLOSED | Logged in accepted risks register below |
| T-01-10 | Information Disclosure | accept | CLOSED | Logged in accepted risks register below |

## Accepted Risks Register

| Threat ID | Category | Rationale | Owner | Review Trigger |
|-----------|----------|-----------|-------|----------------|
| T-01-04 | Denial of Service | `jobId` is a single ObjectId string; query is indexed by jobId; sole-trader workload produces negligible query volume; no rate limiting required at v1 scale | Phase 01 | Team growth or public exposure of the API |
| T-01-06 | Information Disclosure | Customer and JobType data is fetched via authenticated RTK Query hooks backed by `JwtAuthGuard`; no new data exposure introduced by this phase — existing API authorization applies | Phase 01 | Addition of new data fields to the job detail page |
| T-01-07 | Tampering | Commercial summary is a client-side aggregation of server-provided quote data for display only; no write path; server remains the authoritative source of truth | Phase 01 | If totalQuotedAmount is ever used as input to a server write operation |
| T-01-09 | Repudiation | Events record action type and resource IDs but not the authenticated user ID who triggered the action; acceptable for v1 activity feed (display-only); user attribution can be added when audit requirements increase | Phase 01 | Compliance or audit requirement introduced |
| T-01-10 | Information Disclosure | Event metadata contains only IDs (quoteId, scheduleId, estimateId) and status strings; no PII; displayed exclusively to the authenticated business owner who already has access to the referenced entities | Phase 01 | If metadata schema expands to include PII (names, contact details, etc.) |

## Unregistered Flags

None — no `## Threat Flags` sections were present in SUMMARY.md files for plans 01-01, 01-02, 01-03, or 01-04.
