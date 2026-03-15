# Phase 16: Token Infrastructure and Public API - Context

**Gathered:** 2026-03-15
**Status:** Ready for planning

<domain>
## Phase Boundary

Secure token system for customers to access quotes without logging in. Backend-only infrastructure phase: token generation/validation services, a `quote_tokens` collection, and one public (unauthenticated) API endpoint that returns customer-safe quote data given a valid token. No UI in this phase.

</domain>

<decisions>
## Implementation Decisions

### Token design
- Crypto-random string (32-byte hex or base64url)
- 30-day expiry from generation
- Stored in a separate `quote_tokens` collection (not on the quote document)
- Token used as path segment in URL: `/quote/view/{token}` (frontend route, Phase 17) and `/v1/public/quote/:token` (API endpoint)
- Quote-specific tokens — no scoping or resource-type abstraction

### Token storage schema
- Token record includes: token, quoteId, expiresAt, revokedAt, sentAt, recipientEmail
- sentAt + recipientEmail link each token to a specific send event (for attribution)
- View tracking (viewedAt, viewCount) deferred to Phase 17
- No limit on active tokens per quote — each re-send creates a new token, all valid until expiry or revocation

### Public API endpoint
- Path: `GET /v1/public/quote/:token` — new `public` namespace for unauthenticated endpoints
- No `@UseGuards(JwtAuthGuard)` — first unguarded endpoint in the application
- Returns a customer-safe subset of quote data (see Public response shape below)

### Public response shape
- **Included:** business name, customer name, job title, quote number, quote title, quote date, validUntil, current status, full line items (quantity, unit, unitPrice, lineTotal, taxRate, type, bundle components), totals (subtotal, tax, total)
- **Hidden:** all internal IDs (businessId, customerId, jobId, itemId), status timestamps (sentAt, acceptedAt, rejectedAt, deletedAt, updatedAt), notes field, original prices / discount amounts (originalUnitPrice, originalLineTotal, discountAmount)
- Item type labels (material, labour, fee) included for transparency
- Show final prices only — no discount breakdown visible to customer

### Security boundaries
- Basic rate limiting on public endpoint (~60 requests/minute per IP)
- Expired token: return clear message "This quote link has expired. Please contact the tradesperson for an updated link." — HTTP 410 Gone
- Deleted quote: return "This quote is no longer available." — HTTP 410 Gone
- Invalid/unknown token: HTTP 404
- Log failed token lookups via Pino logger (existing pattern) — no database audit trail for failed attempts

### Lifecycle triggers
- Tokens generated on send (Phase 18 calls the token service) — no token for draft quotes
- Token generation service is internal only (no HTTP endpoint to create tokens) — Phase 18 wires it into the send flow
- On quote deletion: revoke all tokens for that quote (set revokedAt)
- New token generated on each re-send — old tokens remain valid until expiry or revocation
- No maximum active tokens per quote

### Claude's Discretion
- Exact token length and encoding (hex vs base64url)
- Rate limiting implementation approach (NestJS throttler, custom guard, etc.)
- Token service class structure and naming
- Public controller placement (new module vs within quote module)
- Index strategy on quote_tokens collection

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Quote system
- `trade-flow-api/src/quote/controllers/quote.controller.ts` — Existing quote endpoints, response mapping pattern, enrichAndMapToResponse()
- `trade-flow-api/src/quote/responses/quote.responses.ts` — IQuoteResponse interface (public response must be a customer-safe subset)
- `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts` — IQuoteDto with all fields (token service needs quoteId access)
- `trade-flow-api/src/quote/enums/quote-status.enum.ts` — QuoteStatus enum including DELETED

### Authentication (for contrast — public endpoint bypasses this)
- `trade-flow-api/src/auth/auth.guard.ts` — JwtAuthGuard pattern that the public endpoint must NOT use
- `trade-flow-api/src/auth/jwt.strategy.ts` — Existing auth strategy for reference

### Email (token consumer in Phase 18)
- `trade-flow-api/src/email/services/email-sender.service.ts` — EmailSenderService that Phase 18 will use alongside tokens

### Architecture
- `.planning/codebase/CONVENTIONS.md` — Naming patterns, module structure, service patterns
- `.planning/codebase/ARCHITECTURE.md` — Layered architecture, repository pattern, error handling

### Requirements
- `.planning/REQUIREMENTS.md` — RESP-01 (partial backend for this phase), DLVR-05 (view tracking, deferred to Phase 17)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `QuoteRetriever` service: Fetches quotes with business-scoping — public endpoint needs a bypass variant that finds by token instead
- `createResponse()` utility: Standard response wrapper — public endpoint should use the same format
- `createHttpError()` utility: Error mapping — public endpoint needs similar error handling
- `AppLogger`: Existing Pino logger for failed token attempts
- `MongoDbWriter` / `MongoDbFetcher`: Database access layer for the new quote_tokens collection

### Established Patterns
- Controller -> Service -> Repository layering — public endpoint follows the same pattern
- Feature-based modules: `src/quote/` contains all quote concerns — token module may be separate or nested
- Custom error classes (ResourceNotFoundError, InvalidRequestError) for structured errors
- Path aliases: `@quote/*`, `@core/*`, `@auth/*` for imports

### Integration Points
- `QuoteController` currently at `@Controller("v1")` — public endpoint needs a separate controller (no auth guard)
- `QuoteModule` — token service needs access to quote repository for status checks
- `quote-transition.service.ts` — DRAFT->DELETED transition should trigger token revocation
- `EmailModule` — Phase 18 will import the token service to generate tokens on send

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Follow existing NestJS patterns from the codebase.

</specifics>

<deferred>
## Deferred Ideas

- General-purpose scoped token system (resourceType + scope fields) — future if tokens needed for invoices or other resources
- Customer-facing notes field (separate from internal notes) — future consideration
- Token-based view tracking (viewedAt, viewCount on token record) — Phase 17

</deferred>

---

*Phase: 16-token-infrastructure-and-public-api*
*Context gathered: 2026-03-15*
