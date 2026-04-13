---
phase: 45-public-customer-page-response-handling
plan: "02"
subsystem: public-estimate-api
tags: [estimates, public-api, token-auth, view-tracking, terminal-states, response-handling]
dependency_graph:
  requires: [phase-45-01]
  provides: [public-estimate-get-endpoint, public-estimate-retriever, public-estimate-response, respond-to-estimate-request]
  affects: [phase-45-03, phase-45-04, phase-45-05]
tech_stack:
  added: []
  patterns: [document-session-auth-guard, throttler-guard, latest-revision-resolution, terminal-state-detection]
key_files:
  created:
    - trade-flow-api/src/estimate/responses/public-estimate.response.ts
    - trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts
    - trade-flow-api/src/estimate/controllers/public-estimate.controller.ts
    - trade-flow-api/src/estimate/requests/respond-to-estimate.request.ts
    - trade-flow-api/src/estimate/test/services/public-estimate-retriever.service.spec.ts
    - trade-flow-api/src/estimate/test/controllers/public-estimate.controller.spec.ts
  modified:
    - trade-flow-api/src/estimate/services/estimate-transition.service.ts
decisions:
  - "businessPhone and businessEmail return null because the business entity/DTO does not store contact details — deferred until business profile is extended"
  - "Latest-revision resolution uses findCurrentInChainByRootId (already on EstimateRepository) rather than a raw MongoDB query — avoids coupling to internals"
  - "publicTransition mirrors the authenticated transition method but skips policy/authUser checks — token auth already validated access at the guard layer"
  - "Controller has only GET handler — POST /respond deferred to Plan 03 where EstimateResponseHandler will be built"
metrics:
  duration: 25min
  completed: "2026-04-13"
  tasks: 2
  files: 7
---

# Phase 45 Plan 02: Public GET Endpoint and Token Resolution Summary

Token-to-estimate resolution with latest-revision lookup, firstViewedAt tracking, Sent->Viewed transition, terminal state detection, and customer-safe response — all behind DocumentSessionAuthGuard at GET /v1/public/estimate/:token.

## What Was Built

### Task 1: PublicEstimateRetriever, publicTransition, and IPublicEstimateResponse

- **IPublicEstimateResponse** (`public-estimate.response.ts`): Customer-safe response interface with all fields specified in plan: businessName, businessPhone (null), businessEmail (null), customerFirstName, traderFirstName, estimateNumber, estimateDate, validUntil, scope, notes, status, displayMode, priceRange (low/high inclTax in minor units), uncertaintyReasons, uncertaintyNotes, responses, respondable, terminalState.
- **IPublicEstimateResponseEntry**: Embedded interface for response entries with type, respondedAt, optional message and reason.
- **EstimateTransitionService.publicTransition** (`estimate-transition.service.ts`): New public method that validates the transition (via `isValidTransition`) and updates status + timestamp fields without policy/authUser checks. Mirrors the existing `transition()` method minus the access controller.
- **PublicEstimateRetriever** (`public-estimate-retriever.service.ts`): Injectable service with constructor injection of EstimateRepository, DocumentTokenRepository, BusinessRepository, CustomerRepository, EstimateTransitionService, and AppLogger.
  - `getPublicEstimate(token)`: Resolves latest revision, tracks first view, fetches business/customer, computes terminal state, maps to customer-safe response.
  - `resolveLatestRevision`: Uses `findCurrentInChainByRootId(documentId)` first; falls back to `findByIdOrFail(documentId)` for pre-revision estimates.
  - `trackFirstView`: Sets firstViewedAt on token if null; triggers Sent->Viewed transition if estimate is SENT.
  - `computeTerminalState`: Returns `trader_closed` for CONVERTED/EXPIRED/LOST/DELETED; `customer_responded` for DECLINED or RESPONDED-with-responses; `null` (respondable) for VIEWED/SENT.
  - `mapResponses`: Maps estimate responses[] to public-safe IPublicEstimateResponseEntry[].
- **Tests** (`public-estimate-retriever.service.spec.ts`): 14 unit tests covering all specified scenarios — valid token response, revision resolution, direct ID fallback, firstViewedAt set/not-set, Sent->Viewed transition trigger/not-trigger, all terminal states (CONVERTED, EXPIRED, LOST, DELETED, DECLINED, RESPONDED-with-responses), and respondable states.

### Task 2: PublicEstimateController and RespondToEstimateRequest

- **RespondToEstimateRequest** (`respond-to-estimate.request.ts`): class-validator DTO with type-discriminated validation:
  - `type`: `@IsEnum(EstimateResponseType)` — required, rejects unknown values
  - `message`: `@ValidateIf(type === MESSAGE)`, `@IsString()`, `@IsNotEmpty()`, `@MaxLength(2000)` — rejects empty strings per D-CTA-03
  - `reason`: `@ValidateIf(type === DECLINE)`, `@IsOptional()`, `@IsEnum(EstimateDeclineReason)`
  - `declineMessage`: `@ValidateIf(type === DECLINE)`, `@IsOptional()`, `@IsString()`, `@MaxLength(500)`
- **PublicEstimateController** (`public-estimate.controller.ts`): `@Controller("v1/public")` with `@UseGuards(ThrottlerGuard)` at class level. Single GET handler:
  - `@Throttle({ default: { limit: 60, ttl: 60000 } })` — 60 req/min per IP (T-45-03 mitigation)
  - `@UseGuards(DocumentSessionAuthGuard)` — validates token authenticity
  - `@Get("estimate/:token")` — route
  - Asserts `documentToken.documentType !== "estimate"` and throws ResourceNotFoundError with DOCUMENT_TOKEN_TYPE_MISMATCH (T-45-01 mitigation)
  - Delegates to `publicEstimateRetriever.getPublicEstimate(token)` and wraps errors in `createHttpError`
- **Controller tests** (`public-estimate.controller.spec.ts`): 4 unit tests — happy path returns data, type mismatch throws HttpException, delegation to retriever, error wrapping.

## Deviations from Plan

### Auto-fixed Issues

None — plan executed exactly as written.

### Notes

**businessPhone/businessEmail**: The plan's `IPublicEstimateResponse` includes `businessPhone: string | null` and `businessEmail: string | null`. The business entity/DTO (`IBusinessDto`) does not have phone or email fields. These are returned as `null` in the current implementation. This is documented as a Known Stub (see below).

## Known Stubs

| Stub | File | Line | Reason |
|------|------|------|--------|
| `businessPhone: null` | `public-estimate-retriever.service.ts` | 49 | `IBusinessDto` has no phone field — business profile extension is a future enhancement |
| `businessEmail: null` | `public-estimate-retriever.service.ts` | 50 | `IBusinessDto` has no email field — business profile extension is a future enhancement |

These stubs return `null` to the UI, which must handle `null` gracefully. D-TERM-02 references "trader's contact details (phone, email from business info)" — the UI in Plan 05 should conditionally render contact details only when non-null.

## Threat Flags

| Flag | File | Description |
|------|------|-------------|
| threat_flag: information_disclosure | `public-estimate-retriever.service.ts` | New public endpoint returns estimate data without JWT auth — mitigated by DocumentSessionAuthGuard (T-45-01), documentType assertion (T-45-01), and customer-safe field filtering in IPublicEstimateResponse (T-45-02) |

## Self-Check: PASSED

- [x] `public-estimate.response.ts` exports `IPublicEstimateResponse` with all required fields
- [x] `public-estimate-retriever.service.ts` has `@Injectable()` and `getPublicEstimate` method
- [x] `public-estimate-retriever.service.ts` calls `findCurrentInChainByRootId` for latest-revision resolution
- [x] `public-estimate-retriever.service.ts` calls `documentTokenRepository.updateFirstViewedAt`
- [x] `public-estimate-retriever.service.ts` calls `publicTransition` with `EstimateStatus.VIEWED`
- [x] `estimate-transition.service.ts` has `publicTransition` method
- [x] `public-estimate.controller.ts` has `@Controller("v1/public")` decorator
- [x] `public-estimate.controller.ts` has `@Get("estimate/:token")` route
- [x] `public-estimate.controller.ts` has `@UseGuards(DocumentSessionAuthGuard)`
- [x] `public-estimate.controller.ts` asserts `documentType !== "estimate"` before delegating
- [x] `respond-to-estimate.request.ts` has `@IsEnum(EstimateResponseType)` on type
- [x] `respond-to-estimate.request.ts` has `@IsNotEmpty()` on message
- [x] `respond-to-estimate.request.ts` has `@MaxLength(2000)` on message and `@MaxLength(500)` on declineMessage
- [x] `respond-to-estimate.request.ts` has `@ValidateIf` conditional validators
- [x] `public-estimate-retriever.service.spec.ts` has 14 passing tests
- [x] `public-estimate.controller.spec.ts` has 4 passing tests
- [x] `npx tsc --noEmit` exits with zero errors

## Commits

**trade-flow-api repo:**
- `b88e42b`: feat(45-02): add PublicEstimateRetriever, publicTransition, and IPublicEstimateResponse
- `bc85ab4`: feat(45-02): add PublicEstimateController GET endpoint and RespondToEstimateRequest DTO
