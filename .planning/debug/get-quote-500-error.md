---
status: diagnosed
trigger: "500 error on GET /v1/quote/:quoteId"
created: 2026-03-14T00:00:00Z
updated: 2026-03-14T00:00:00Z
---

## Current Focus

hypothesis: QuoteLineItemRetriever is missing @Injectable() decorator, causing NestJS DI to instantiate it with undefined dependencies
test: Compared QuoteLineItemRetriever class definition against all other services
expecting: Missing @Injectable() means no design:paramtypes metadata, so NestJS injects nothing
next_action: Report diagnosis

## Symptoms

expected: GET /v1/quote/:quoteId returns 200 with quote data
actual: Returns 500 Internal Server Error for all single-quote requests
errors: "ERROR: Internal server error encountered: {req:{method:GET,url:/v1/quote/69b5ab20ec48d9e986f3dc6e},context:{}}"
reproduction: Any GET /v1/quote/:quoteId request
started: After Phase 13 added quote API integration

## Eliminated

- hypothesis: Wrong URL pattern in UI (missing /business/:businessId)
  evidence: Controller defines route as GET quote/:quoteId (no businessId), matching the UI call at quoteApi.ts line 24
  timestamp: 2026-03-14

- hypothesis: Invalid ObjectId parsing
  evidence: The example quoteId 69b5ab20ec48d9e986f3dc6e is a valid 24-char hex ObjectId
  timestamp: 2026-03-14

- hypothesis: Response mapping error in mapToResponse
  evidence: All DateTime/Money conversions use optional chaining or null coalescing safely
  timestamp: 2026-03-14

## Evidence

- timestamp: 2026-03-14
  checked: Error log format vs createHttpError utility
  found: The 500 hits the catch-all at handle-error.utility.ts line 70-81 (unknown error type), meaning the thrown error is NOT ResourceNotFoundError, ForbiddenError, InvalidRequestError, or InternalServerError
  implication: A raw Error/TypeError is being thrown somewhere in the call chain

- timestamp: 2026-03-14
  checked: QuoteRetriever.findByIdOrFail vs findAllByBusinessId code paths
  found: findByIdOrFail (line 29) calls quoteLineItemRetriever.findAllByQuoteId(), but findAllByBusinessId does NOT use quoteLineItemRetriever at all
  implication: The bug is isolated to the code path that uses QuoteLineItemRetriever, explaining why list works but single fetch fails

- timestamp: 2026-03-14
  checked: QuoteLineItemRetriever class definition at quote-line-item-retriever.service.ts
  found: Class is exported WITHOUT @Injectable() decorator (line 8: "export class QuoteLineItemRetriever"). Has 3 constructor-injected dependencies (QuoteLineItemRepository, QuoteLineItemPolicy, AccessControllerFactory)
  implication: Without @Injectable(), TypeScript does not emit design:paramtypes metadata. NestJS cannot determine constructor parameters and instantiates the class with zero arguments, leaving all dependencies as undefined

- timestamp: 2026-03-14
  checked: All other services in quote module for comparison
  found: QuoteRetriever, QuoteCreator, QuoteUpdater, QuoteTotalsCalculator, QuoteTransitionService all have @Injectable(). QuoteLineItemRetriever is the only one missing it
  implication: This is a simple omission, not a pattern issue

- timestamp: 2026-03-14
  checked: Runtime error chain
  found: When findAllByQuoteId runs, this.quoteLineItemRepository is undefined -> calling .findAllByQuoteId() on undefined throws TypeError -> TypeError is not a recognized error type in createHttpError -> falls through to catch-all -> returns 500
  implication: Root cause confirmed

## Resolution

root_cause: QuoteLineItemRetriever (trade-flow-api/src/quote/services/quote-line-item-retriever.service.ts line 8) is missing the @Injectable() decorator. Without it, NestJS dependency injection cannot resolve its constructor parameters (QuoteLineItemRepository, QuoteLineItemPolicy, AccessControllerFactory), leaving them as undefined. When the getQuote endpoint calls quoteRetriever.findByIdOrFail -> quoteLineItemRetriever.findAllByQuoteId, the undefined this.quoteLineItemRepository causes a TypeError that is not handled by the custom error handler, resulting in a 500.
fix: Add @Injectable() decorator to QuoteLineItemRetriever class
verification:
files_changed: []
