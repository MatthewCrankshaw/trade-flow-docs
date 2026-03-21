---
phase: 17-customer-quote-page
verified: 2026-03-20T20:30:00Z
status: human_needed
score: 5/5 must-haves verified
re_verification:
  previous_status: gaps_found
  previous_score: 3/5
  gaps_closed:
    - "Public quote endpoint uses @UseGuards(QuoteSessionAuthGuard) instead of inline token validation"
    - "Controller injects only PublicQuoteRetriever service, not repositories directly"
    - "Visit recording and viewedAt have test coverage"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Navigate to the customer-facing quote URL in a browser"
    expected: "Page renders with business name, customer name, job title, line items and totals; internal IDs and notes are not visible"
    why_human: "RESP-01 is explicitly a frontend requirement. This codebase is a pure NestJS REST API. A separate frontend application must call GET /v1/public/quote/:token and render the response. That frontend is not present in this repository."
  - test: "Open a quote link as a customer, then log in as the trader and view the same quote via GET /v1/quote/:quoteId"
    expected: "The trader's response includes a viewedAt timestamp matching when the customer first opened the link; opening the link a second time does not change the timestamp"
    why_human: "The end-to-end flow spans customer browser, public API endpoint, MongoDB write, authenticated API read, and trader UI — cannot be verified programmatically"
---

# Phase 17: Customer Quote Page Verification Report

**Phase Goal:** Customer can view a quote online and see that their visit was recorded
**Verified:** 2026-03-20T20:30:00Z
**Status:** human_needed
**Re-verification:** Yes — after gap closure (Plans 17-03 and 17-04)

---

## Re-verification Context

Previous verification (initial) scored 3/5 with three blocking gaps:

1. Inline token validation in controller instead of a proper QuoteSessionAuthGuard
2. Controller injecting 5 repositories directly instead of using a service (PublicQuoteRetriever)
3. Missing test coverage for visit recording and viewedAt

Plans 17-03 and 17-04 were executed to close these gaps. All three gaps are now closed. No regressions found on the two truths that were previously passing.

Commits verified:
- `e79fc09` feat(17-03): create QuoteSessionAuthGuard and PublicQuoteRetriever service
- `8a2dd94` refactor(17-03): slim PublicQuoteController to use guard and service
- `71d3dea` test(17-04): add QuoteSessionAuthGuard unit tests
- `2a96f07` test(17-04): add PublicQuoteRetriever spec and rewrite controller spec

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Customer can fetch a quote by token via a public unauthenticated endpoint | VERIFIED | `GET /v1/public/quote/:token` in `PublicQuoteController`; rate-limited 60 req/min; all tests pass |
| 2 | First visit is recorded atomically (only on first access) | VERIFIED | `QuoteTokenRepository.updateFirstViewedAt` uses MongoDB filter `{ firstViewedAt: { $exists: false } }`; guard calls it only when `!tokenDto.firstViewedAt` |
| 3 | Visit recording and viewedAt are covered by tests | VERIFIED | Guard spec: 6 tests including "should record first view when firstViewedAt is not set" and "should NOT record first view when firstViewedAt is already set", both asserting `updateFirstViewedAt` call count |
| 4 | Trader can see when a customer viewed a quote (authenticated endpoint) | VERIFIED | `viewedAt` field on `IQuoteResponse`; `QuoteController.enrichAndMapToResponse` calls `findLatestNonRevokedForQuote`; regression check confirms still intact |
| 5 | Public response excludes internal data (no data leak) | VERIFIED | `IPublicQuoteResponse` omits id, businessId, customerId, jobId, notes, sentAt, acceptedAt, rejectedAt, updatedAt; PublicQuoteRetriever service spec asserts absent keys |

**Score:** 5/5 truths verified

---

## Required Artifacts

| Artifact | Description | Status | Details |
|----------|-------------|--------|---------|
| `src/quote-token/guards/quote-session-auth.guard.ts` | QuoteSessionAuthGuard implementing CanActivate | VERIFIED | 55 lines; implements CanActivate; validates token, checks expiry/revocation, records first view, attaches tokenDto to `request.quoteToken` |
| `src/quote-token/services/public-quote-retriever.service.ts` | PublicQuoteRetriever service | VERIFIED | 105 lines; `getPublicQuote(quoteId)`, `mapToPublicResponse`, `mapPublicLineItems`, `mapPublicLineItemResponse`; QuoteStatus.DELETED check; bundle rolling-up |
| `src/quote-token/controllers/public-quote.controller.ts` | Slimmed controller | VERIFIED | 31 lines; single constructor parameter `publicQuoteRetriever`; `@UseGuards(QuoteSessionAuthGuard)` decorator; zero repository imports |
| `src/quote-token/quote-token.module.ts` | Module registering new providers | VERIFIED | `QuoteSessionAuthGuard` and `PublicQuoteRetriever` in both `providers` and `exports` arrays |
| `src/quote-token/test/guards/quote-session-auth.guard.spec.ts` | Guard unit tests | VERIFIED | 162 lines; 6 tests all passing: valid token attachment, 404 not found, 410 expired with business name, 410 revoked with business name, first view recorded, idempotent second view |
| `src/quote-token/test/services/public-quote-retriever.service.spec.ts` | Service unit tests | VERIFIED | 173 lines; 5 tests all passing: mapped response, deleted quote 410, deleted line item filtering, internal data exclusion, bundle components |
| `src/quote-token/test/controllers/public-quote.controller.spec.ts` | Slimmed controller spec | VERIFIED | 87 lines; 2 tests: delegation to service, HttpException propagation; mocks `PublicQuoteRetriever` only (not repositories) |
| `src/quote-token/repositories/quote-token.repository.ts` | `updateFirstViewedAt`, `findLatestNonRevokedForQuote` | VERIFIED | Both methods present; unchanged from initial verification |
| `src/quote-token/entities/quote-token.entity.ts` | `firstViewedAt?: Date` | VERIFIED | Field present; unchanged |
| `src/quote-token/data-transfer-objects/quote-token.dto.ts` | `firstViewedAt?: DateTime` | VERIFIED | Field present; unchanged |
| `src/quote/responses/quote.responses.ts` | `viewedAt?: string` | VERIFIED | Field present at line 34; unchanged |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `PublicQuoteController` | `QuoteSessionAuthGuard` | `@UseGuards(QuoteSessionAuthGuard)` decorator | WIRED | Line 17 of controller; guard validates token before handler runs |
| `PublicQuoteController` | `PublicQuoteRetriever` | Constructor injection | WIRED | Single constructor parameter; `getPublicQuote` called in handler |
| `QuoteSessionAuthGuard` | `QuoteTokenRetriever` | Constructor injection | WIRED | `findByToken`, `isExpired`, `isRevoked` called in `canActivate` |
| `QuoteSessionAuthGuard` | `QuoteTokenRepository.updateFirstViewedAt` | Constructor injection | WIRED | Called at guard line 49 when `!tokenDto.firstViewedAt` |
| `QuoteSessionAuthGuard` | `request.quoteToken` | `request.quoteToken = tokenDto` | WIRED | Guard line 52 attaches DTO; controller reads `(request as any).quoteToken` |
| `PublicQuoteRetriever` | `QuoteRepository`, `BusinessRepository`, `CustomerRepository`, `JobRepository` | Constructor injection | WIRED | All 4 repositories injected and called in `getPublicQuote` |
| `QuoteTokenModule` | `QuoteSessionAuthGuard`, `PublicQuoteRetriever` | `providers` array | WIRED | Both in providers; `PublicQuoteRetriever` also exported |
| `QuoteController` | `QuoteTokenRepository.findLatestNonRevokedForQuote` | Constructor injection via `forwardRef(QuoteTokenModule)` | WIRED | Line 217 of quote.controller.ts; regression check confirmed intact |

---

## Requirements Coverage

No REQUIREMENTS.md exists in this repository. Requirement descriptions are inferred from the requirement IDs and implementation evidence.

| Requirement | Plans | Description | Status | Evidence |
|-------------|-------|-------------|--------|----------|
| RESP-01 (frontend) | 17-03, 17-04 | Customer-facing quote page UI | NEEDS HUMAN — frontend in separate codebase | API provides `GET /v1/public/quote/:token` returning `IPublicQuoteResponse`. No frontend code in this repo. |
| RESP-05 | 17-03 | Public quote response contract / data safety | SATISFIED | `IPublicQuoteResponse` contract enforced by `PublicQuoteRetriever`; service spec asserts absent internal keys |
| DLVR-05 | 17-03, 17-04 | Record customer visit / first view tracking | SATISFIED | `updateFirstViewedAt` atomic, guard-controlled; 2 dedicated tests assert recording and idempotency |

No orphaned requirements found. All three IDs declared in plan frontmatter are accounted for.

---

## Anti-Patterns Found

No TODO, FIXME, placeholder comments, empty implementations, or stub return values found in any Phase 17 implementation files. The new guard (55 lines) and service (105 lines) are fully substantive.

---

## Human Verification Required

### 1. Customer quote page renders correctly (RESP-01)

**Test:** Navigate to the customer-facing quote URL in a browser
**Expected:** Page renders with business name, customer name, job title, line items and totals; internal IDs and notes are not visible
**Why human:** RESP-01 is explicitly a frontend requirement. This codebase is a pure NestJS REST API. A separate frontend application must call `GET /v1/public/quote/:token` and render the response. That frontend is not present in this repository.

### 2. End-to-end visit recording flow

**Test:** Open a quote link as a customer, then log in as the trader and view the same quote via `GET /v1/quote/:quoteId`
**Expected:** The trader's response includes a `viewedAt` timestamp matching when the customer first opened the link; opening the link a second time does not change the timestamp
**Why human:** The end-to-end flow spans customer browser, public API endpoint, MongoDB write, authenticated API read, and trader UI — cannot be verified programmatically

---

## Gap Closure Summary

All three gaps from the initial verification are confirmed closed:

**Gap 1 (CLOSED) — QuoteSessionAuthGuard exists and is wired.** `src/quote-token/guards/quote-session-auth.guard.ts` is a proper NestJS guard implementing `CanActivate`. It handles token lookup, expiry/revocation checks with 410 enrichment, atomic first-view recording, and attaches the token DTO to the request. The controller uses `@UseGuards(QuoteSessionAuthGuard)` and has zero inline validation logic.

**Gap 2 (CLOSED) — Controller uses service, not repositories.** `PublicQuoteController` has exactly one constructor parameter (`PublicQuoteRetriever`). Zero repository imports exist in the controller file. Business logic lives in the service layer as required by CLAUDE.md's strict Controller → Service → Repository rule.

**Gap 3 (CLOSED) — Visit recording and viewedAt have test coverage.** The guard spec includes two dedicated tests: one asserting `updateFirstViewedAt` is called on first visit, one asserting it is NOT called when `firstViewedAt` is already set. All 24 quote-token tests pass. The full project suite (271 tests, 39 suites per Plan 04 summary) passes.

---

_Verified: 2026-03-20T20:30:00Z_
_Verifier: Claude (gsd-verifier)_
_Re-verification: Yes — after closure of 3 gaps from initial verification_
