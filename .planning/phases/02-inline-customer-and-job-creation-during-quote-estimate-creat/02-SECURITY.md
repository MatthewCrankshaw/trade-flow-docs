---
phase: 02-inline-customer-and-job-creation-during-quote-estimate-creat
asvs_level: 1
generated: 2026-04-22
threats_total: 6
threats_closed: 6
threats_open: 0
block_on: critical
result: SECURED
---

# Security Audit — Phase 02: Inline Customer and Job Creation

## Summary

**Threats Closed:** 6/6
**Threats Open:** 0/6
**ASVS Level:** 1
**Block on:** critical
**Result:** SECURED

## Threat Verification

| Threat ID | Category | Disposition | Evidence | Status |
|-----------|----------|-------------|----------|--------|
| T-02-01 | Tampering | mitigate | `trade-flow-api/src/job/services/job-creator.service.ts` lines 52-56: conditional `if (!job.title \|\| job.title.trim() === "")`, `countByCustomerId` call, and server-side interpolation `${jobType.name} - ${customer.name} #${sequentialNumber}`. Both `jobType` and `customer` are fetched from DB via `findByIdOrFail`/`findById` before title generation — client cannot supply arbitrary values. | CLOSED |
| T-02-02 | Information Disclosure | accept | Accepted risk — see accepted risks log below. | CLOSED |
| T-02-03 | Tampering | mitigate | `trade-flow-ui/src/lib/forms/schemas/inlineCustomer.schema.ts` lines 3-9: valibot schema enforces `minLength(1)` and `maxLength(100)` on `name`. `InlineCustomerForm.tsx` line 37: `values.name.trim()` passed to mutation. Backend `create-customer.request.ts` enforces 1-64 char validation via class-validator. React renders user-supplied text via JSX which auto-escapes, preventing XSS. | CLOSED |
| T-02-04 | Elevation of Privilege | accept | Accepted risk — see accepted risks log below. | CLOSED |
| T-02-05 | Spoofing | accept | Accepted risk — see accepted risks log below. | CLOSED |
| T-02-06 | Spoofing | accept | Accepted risk — see accepted risks log below. | CLOSED |

## Accepted Risks Log

| Threat ID | Category | Component | Rationale |
|-----------|----------|-----------|-----------|
| T-02-02 | Information Disclosure | `job.repository.ts` — `countByCustomerId` | Method reveals the count of jobs per customer, but this is the authenticated user's own data. `businessId` is always scoped to the auth user's business via the existing `JwtAuthGuard` + policy enforcement chain. No cross-tenant disclosure is possible. Risk accepted: information is non-sensitive and access-controlled. |
| T-02-04 | Elevation of Privilege | `InlineCustomerForm.tsx` | Uses the existing `POST /v1/customer` endpoint unchanged. `JwtAuthGuard` + `CustomerPolicy` already enforce business ownership. No new authentication or authorization surface is introduced. Risk accepted: no new attack surface. |
| T-02-05 | Spoofing | `CreateJobDialog.tsx` | All mutations (customer, job type, job creation) use authenticated RTK Query endpoints. Firebase JWT is injected via the shared `prepareHeaders` callback. No change to the authentication flow. Risk accepted: no new auth surface. |
| T-02-06 | Spoofing | `CreateEstimateForm.tsx` | Uses the same authenticated RTK Query endpoints as `CreateQuoteForm`. Firebase JWT enforced on all API calls via `prepareHeaders`. No new auth surface introduced. Risk accepted: no new auth surface. |

## Unregistered Flags

None. No `## Threat Flags` section was present in any SUMMARY.md (02-01, 02-02, 02-03).

## Verification Notes

- `job-creator.service.ts`: server-side data retrieval (`customerRetriever.findByIdOrFail`, `jobTypeRetriever.findById`) precedes title generation; the client can only influence which existing customer/job-type IDs are looked up, both of which are already access-controlled by the retrievers.
- `job.repository.ts`: `countByCustomerId` filters by both `businessId` and `customerId` as `ObjectId` instances (lines 54-59), preventing cross-tenant count leakage.
- `InlineCustomerForm.tsx`: client-side valibot validation is defence-in-depth; the backend class-validator is the authoritative enforcement boundary.
- `CreateJobDialog.tsx`: `InlineCustomerForm` rendered as a sibling dialog (lines 488-493), outside `DialogContent`, following the established stacked-dialog pattern. `handleInlineCustomerCreated` sets `type: "existing"` so the already-persisted customer ID is used directly rather than triggering a second deferred creation.
- `CreateEstimateForm.tsx`: `CreateJobDialog` rendered as a sibling at lines 296-301 with `onJobCreated` callback wiring identical to `CreateQuoteForm`.
