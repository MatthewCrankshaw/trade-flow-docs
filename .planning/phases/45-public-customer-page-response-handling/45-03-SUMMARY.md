---
phase: 45-public-customer-page-response-handling
plan: "03"
subsystem: trade-flow-api
tags: [estimate, public-api, response-handling, email-notification, guard]
dependency_graph:
  requires: ["45-02"]
  provides: ["RESP-01", "RESP-02", "RESP-03", "RESP-04", "RESP-05", "RESP-06"]
  affects: ["document-token-guard", "estimate-module"]
tech_stack:
  added: []
  patterns:
    - "Failure-tolerant email notification (try/catch around send, response persisted first)"
    - "DocumentType-branching in guard for multi-document-type token handling"
    - "forwardRef circular dependency resolution for EstimateModule <-> DocumentTokenModule"
key_files:
  created:
    - trade-flow-api/src/estimate/services/estimate-response-handler.service.ts
    - trade-flow-api/src/email/services/estimate-notification-email-renderer.service.ts
    - trade-flow-api/src/email/templates/estimate-response.html
    - trade-flow-api/src/estimate/test/services/estimate-response-handler.service.spec.ts
  modified:
    - trade-flow-api/src/estimate/controllers/public-estimate.controller.ts
    - trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
    - trade-flow-api/src/document-token/document-token.module.ts
    - trade-flow-api/src/estimate/estimate.module.ts
    - trade-flow-api/src/estimate/test/controllers/public-estimate.controller.spec.ts
    - trade-flow-api/src/document-token/test/guards/document-session-auth.guard.spec.ts
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
    - trade-flow-api/openapi.yaml
decisions:
  - "Used BusinessUserRepository + UserRepository to resolve trader email (same pattern as QuoteResponseHandler) — IBusinessDto has no email field, plan's business.email reference was incorrect"
  - "Used forwardRef(() => EstimateModule) in DocumentTokenModule to break circular dependency for EstimateRepository injection into DocumentSessionAuthGuard"
  - "EstimateNotificationEmailRenderer uses template string replacement (not Handlebars) with {{#if}} conditional blocks matching existing NotificationEmailRenderer pattern"
metrics:
  duration: "~25 minutes"
  completed_date: "2026-04-13"
  tasks: 3
  files: 9
---

# Phase 45 Plan 03: Estimate Response Handler and Notification Email Summary

One-liner: POST /respond endpoint with EstimateResponseHandler (respondable state check, response persistence, status transitions, failure-tolerant trader notification email) and guard fix for estimate tokens.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create EstimateResponseHandler and EstimateNotificationEmailRenderer | dbe634a | 6 created/modified |
| 2 | Add POST respond endpoint, fix guard, wire EstimateModule | 63ad10e | 5 modified |
| 3 | Full CI gate for backend | e07b8ac | 4 modified |

## What Was Built

### EstimateResponseHandler (`estimate-response-handler.service.ts`)

Injectable service that processes all three response types:

1. Resolves latest revision via `findCurrentInChainByRootId` (same pattern as `PublicEstimateRetriever`)
2. Validates respondable state: only `VIEWED` and `SENT` estimates can receive responses; throws `ESTIMATE_NOT_RESPONDABLE` (422) for all other statuses
3. Builds `IEstimateResponseEntity` with appropriate message/reason fields per response type
4. Persists response via `estimateRepository.pushResponse` (before email attempt)
5. Transitions status: proceed/message → `RESPONDED`, decline → `DECLINED` via `estimateTransitionService.publicTransition`
6. Sends trader notification email via failure-tolerant try/catch block — email failure is logged but does not propagate

### EstimateNotificationEmailRenderer (`estimate-notification-email-renderer.service.ts`)

Mirrors the existing `NotificationEmailRenderer` pattern: reads `estimate-response.html` template, applies string replacement for all variables, attempts Maizzle rendering with fallback to raw HTML. User-provided content (message, reason) is HTML-escaped via `escapeHtml` before template substitution.

### Email Template (`estimate-response.html`)

Three conditional sections using `{{#if isProceed}}`, `{{#if isMessage}}`, `{{#if isDecline}}` blocks:
- PROCEED: green highlight "Great news! {customerName} is happy to proceed."
- MESSAGE: grey blockquote with customer's message
- DECLINE: red highlight with optional reason and optional decline message

"View Estimate" CTA button linking to `/estimates/{estimateId}`.

### POST /v1/public/estimate/:token/respond endpoint

Added to `PublicEstimateController` with:
- `@Throttle({ default: { limit: 10, ttl: 60000 } })` — 10 req/min (stricter than GET's 60/min)
- `@UseGuards(DocumentSessionAuthGuard)` — token session validation
- documentType assertion to prevent token confusion
- Delegates to `EstimateResponseHandler.handleResponse`, then re-fetches via `PublicEstimateRetriever.getPublicEstimate` to return consistent response shape

### DocumentSessionAuthGuard fix

The expired/revoked path now branches on `tokenDto.documentType`:
- `"estimate"` → looks up estimate then business for `businessName` context
- otherwise → existing quote path preserved

### Module wiring

- `EstimateModule`: added `PublicEstimateController`, `PublicEstimateRetriever`, `EstimateResponseHandler`, `EstimateNotificationEmailRenderer`
- `DocumentTokenModule`: added `forwardRef(() => EstimateModule)` to provide `EstimateRepository` to the guard

### ErrorCodes

Added `ESTIMATE_NOT_RESPONDABLE = "estimate_not_respondable"` to `ErrorCodes` enum and `ERRORS_MAP`.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Plan referenced `business.email` which does not exist on `IBusinessDto`**
- **Found during:** Task 1
- **Issue:** Plan's `EstimateResponseHandler` implementation called `to: business.email` but `IBusinessDto` has no email field — only `name`, `country`, `currency`, `status`, `trade`
- **Fix:** Used `BusinessUserRepository.findByBusinessId` + `UserRepository.findById` to resolve the trader's email, matching the established `QuoteResponseHandler` pattern exactly
- **Files modified:** `estimate-response-handler.service.ts`
- **Commit:** dbe634a

**2. [Rule 1 - Bug] Plan referenced `estimate.customerName` and `estimate.estimateNumber` which do not exist on `IEstimateDto`**
- **Found during:** Task 1
- **Issue:** `IEstimateDto` has `customerId` (not `customerName`) and `number` (not `estimateNumber`)
- **Fix:** Added `CustomerRepository` dependency to fetch customer by `estimate.customerId`; used `estimate.number` for the estimate number field
- **Files modified:** `estimate-response-handler.service.ts`
- **Commit:** dbe634a

**3. [Rule 3 - Blocking] Pre-existing prettier formatting errors blocked CI gate**
- **Found during:** Task 3
- **Issue:** `estimate-email-sender.service.ts` and `estimate-reviser.service.ts` had pre-existing prettier formatting errors that were not introduced by this plan but blocked `npm run ci`
- **Fix:** Ran `npx prettier --write` on those files to restore formatting compliance
- **Files modified:** `estimate-email-sender.service.ts`, `estimate-reviser.service.ts`
- **Commit:** e07b8ac

## Known Stubs

None. All data is wired end-to-end.

## Threat Flags

No new unplanned security surface introduced. All threats in the plan's threat model were mitigated:
- T-45-04: class-validator on `RespondToEstimateRequest` (implemented in Plan 02)
- T-45-05: respondable state check (VIEWED/SENT only) implemented in `EstimateResponseHandler`
- T-45-06: user-provided content HTML-escaped via `escapeHtml` before template rendering
- T-45-07: `@Throttle({ default: { limit: 10, ttl: 60000 } })` on POST endpoint
- T-45-08: Guard fix adds estimate token handling for expired/revoked path

## Self-Check: PASSED

All created files verified present on disk. All three task commits verified in git log.

| Item | Status |
|------|--------|
| estimate-response-handler.service.ts | FOUND |
| estimate-notification-email-renderer.service.ts | FOUND |
| estimate-response.html template | FOUND |
| estimate-response-handler.service.spec.ts | FOUND |
| Commit dbe634a (Task 1) | FOUND |
| Commit 63ad10e (Task 2) | FOUND |
| Commit e07b8ac (Task 3) | FOUND |
