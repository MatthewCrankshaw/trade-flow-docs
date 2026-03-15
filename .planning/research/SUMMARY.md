# Project Research Summary

**Project:** Trade Flow v1.3 — Send Quotes
**Domain:** Quote email delivery, customer response flow, PDF generation, quote deletion
**Researched:** 2026-03-15
**Confidence:** HIGH

## Executive Summary

Trade Flow v1.3 closes the quote delivery loop: a tradesperson sends a quote to a customer via email, the customer views and accepts/rejects it online without an account, and the tradesperson is notified. This is table-stakes functionality for any trade management tool — every major competitor (Tradify, Fergus, Jobber, ServiceM8, Xero) implements it this way. The recommended approach builds almost entirely on the existing NestJS/MongoDB/React stack with only two new backend libraries (pdfmake for PDF generation, handlebars for email templates) and no new infrastructure — SendGrid and all core quote entity fields are already in place.

The architecture centers on a clear security boundary between authenticated (tradesperson) and public (customer) surfaces. The customer-facing flow uses a cryptographically secure token stored in a new `quote_tokens` collection; possession of the token is the authorization. A dedicated `PublicQuoteController` with no auth guard keeps this boundary explicit and auditable. Quote deletion is a standalone feature with no external dependencies and is a natural first implementation target.

The highest-risk areas are token security (guessable, never-expiring, or logged tokens), race conditions on concurrent accept/reject requests, and the temptation to reuse auth-aware services in public endpoints. All three are preventable with established patterns: `crypto.randomBytes(32)`, atomic `findOneAndUpdate` with status preconditions, and a dedicated public service layer that treats the token itself as authorization. There is also a cross-file conflict: ARCHITECTURE.md suggests Puppeteer for PDF generation, but both STACK.md and PITFALLS.md agree on pdfmake — use pdfmake. Puppeteer introduces a 300MB+ Chromium dependency, cold-start penalties, and Alpine Linux compatibility issues that are not warranted for structured tabular quote documents.

## Key Findings

### Recommended Stack

No new frameworks are needed. The two new backend libraries are pdfmake (declarative JSON document definitions, pure JS, zero native dependencies) and handlebars (server-side HTML email template compilation). Both integrate cleanly into the existing NestJS/TypeScript codebase. The frontend adds zero new packages — the customer response page reuses Tailwind and Radix components within the existing React app.

See [STACK.md](STACK.md) for full details, alternatives considered, and version compatibility notes.

**Core technologies:**
- `pdfmake ^0.3.6`: Server-side PDF generation — declarative JSON API maps naturally to quote structure (line items table, headers, totals); no browser/Chromium dependency
- `@types/pdfmake ^0.2.x`: TypeScript types for pdfmake — required for strict TS codebase
- `handlebars ^4.7.8`: Email template compilation — stable, version-controlled, unit-testable; avoids SendGrid Dynamic Template dashboard dependency
- `crypto` (Node.js built-in): Token generation — `randomBytes(32)` provides 256-bit entropy; no additional dependency
- `@sendgrid/mail ^8.1.6`: Already installed — extend existing `EmailSenderService`, do not replace it

**What NOT to use:**
- Puppeteer/Playwright — 300MB+ Chrome, slow cold start, Alpine incompatibility; not warranted for tabular data
- SendGrid Dynamic Templates — live outside codebase, cannot be unit-tested, adds dashboard dependency
- JWTs for customer tokens — cannot be revoked server-side; simple random token with DB lookup is safer
- uuid for tokens — 122-bit entropy vs 256-bit from `crypto.randomBytes`; adds unnecessary dependency

### Expected Features

All competitors (Tradify, Fergus, Jobber, ServiceM8, Xero) implement the core send/respond loop. The research identifies a clear MVP boundary with strong competitive precedent.

See [FEATURES.md](FEATURES.md) for the full feature landscape, dependency graph, and competitor analysis.

**Must have (table stakes for v1.3):**
- Quote PDF generation — professional PDF with business/customer/line item details; required by email and customer view
- Email quote to customer — SendGrid email with quote summary, view link, and PDF attachment
- Customizable email message — editable body text via send dialog before sending
- Customer quote view (public, no login) — token-based access to read-only quote page
- Customer accept button — one-click acceptance with automatic status update
- Customer decline button — one-click rejection with automatic status update
- Tradesperson notification email — email sent when customer accepts or rejects
- Quote deletion — DRAFT-only hard delete with confirmation dialog
- PDF download from customer view — download button on public page

**Should have — add after v1.3 validation:**
- Customer comment on decline — optional rejection reason; small scope, high value
- Re-send quote — "send again" for already-SENT quotes
- Quote viewed indicator — record `viewedAt` when customer loads page
- Days since sent — quote list aging display for manual follow-up prioritization

**Defer to v2+:**
- Quote versioning/options, electronic signature, automated follow-up reminders, customer portal/login, deposit collection on acceptance

**Key dependency order:** PDF generation must be built first (email attachment and customer download both depend on it). Customer view page is the gateway for accept/reject. Quote deletion has no dependencies and can be built first or in parallel.

### Architecture Approach

The architecture introduces a hard separation between two API surfaces: the existing authenticated controller (modified to add send, delete, and PDF endpoints) and a new `PublicQuoteController` with no auth guard. Token validation in the service layer replaces authentication for the public surface. All new backend operations follow the established service-per-action pattern. A new `quote_tokens` collection stores HMAC-signed tokens with expiry and revocation support. The frontend adds a `/q/:token` public route outside the `AuthenticatedLayout` wrapper, with a separate RTK Query slice (`publicQuoteApi.ts`) that does not inject Firebase auth headers.

See [ARCHITECTURE.md](ARCHITECTURE.md) for system diagrams, data flow sequences, anti-patterns, and the full suggested build order.

**Major components:**
1. `QuoteTokenService` (NEW) — generate HMAC-SHA256 tokens, validate, revoke on re-send/delete
2. `QuoteSender` (NEW) — orchestrate send: validate customer email, generate token, build email HTML, send via SendGrid, transition status
3. `QuotePdfGenerator` (NEW) — generate PDF buffer from quote data using pdfmake declarative definitions
4. `PublicQuoteController` (NEW) — unauthenticated endpoints: view quote, accept/reject; token replaces auth guard
5. `QuotePublicResponder` (NEW) — atomic status transition for customer response; bypasses auth-aware policy layer entirely; token is the authorization
6. `QuoteDeleter` (NEW) — hard delete quote + line items; DRAFT status guard; revoke tokens
7. `CustomerQuotePage` (NEW frontend) — public React page at `/q/:token`; outside `AuthenticatedLayout`
8. `publicQuoteApi.ts` (NEW frontend) — separate RTK Query slice with no auth headers

**New environment variables required:**
- `QUOTE_TOKEN_SECRET` — HMAC signing secret (generate with `openssl rand -hex 32`)
- `APP_BASE_URL` — frontend URL for constructing quote links in emails
- `QUOTE_TOKEN_EXPIRY_DAYS` — optional, default 30

### Critical Pitfalls

See [PITFALLS.md](PITFALLS.md) for all 17 pitfalls with detection, prevention, and phase-specific warnings.

1. **Public endpoint reuses auth-aware services** — The existing `QuoteTransitionService` requires `IUserDto`. Creating a fake system user breaks the security model. Use a dedicated `QuotePublicResponder` that performs the accept/reject transition directly via repository with the token as authorization. Keep the existing transition service untouched.

2. **Race condition on double accept/reject** — Read-then-write on status allows two concurrent requests to both pass the SENT check. Use atomic `findOneAndUpdate` with `{ _id: quoteId, status: 'sent' }` precondition. Second request gets a null result and returns a clear conflict error, not a corrupt state.

3. **Token security failures** — Guessable tokens (UUID/encoded IDs), never-expiring tokens, or tokens in query parameters (logged by servers and analytics). Use `crypto.randomBytes(32).toString('hex')`, HMAC-signed with server secret, stored with 30-day expiry and TTL index, path-based URL (`/q/:token`), invalidated on re-send.

4. **Email send failure leaves quote in SENT status** — Transitioning status before sending email means a SendGrid failure leaves the quote as SENT with no email delivered. Send email first, then transition status. Failure keeps quote in DRAFT with a clear user-facing error.

5. **PDF money formatting mismatch with UI** — The API returns raw floats from `toMajorUnits()`. Without a shared formatter, the PDF shows `$100.5` while the UI shows `$100.50`. Use a shared server-side `Intl.NumberFormat` utility with the business currency. PDF generator receives Money objects, not pre-formatted strings.

## Implications for Roadmap

Based on research, the dependency graph and pitfall profile suggest 6 phases:

### Phase 1: Quote Deletion
**Rationale:** Fully independent of all other v1.3 features. No external dependencies, no new libraries, no security complexity. Quick win that establishes the service-per-action file structure before more complex work begins.
**Delivers:** Tradesperson can delete DRAFT quotes from their quote list with a confirmation dialog.
**Addresses:** Quote deletion (table stakes), DRAFT-only status guard.
**Avoids:** Pitfall 11 (deleting non-DRAFT quotes) — the status guard is the entire implementation concern here.
**Research flag:** Standard patterns, no deeper research needed.

### Phase 2: Token Infrastructure and Public API
**Rationale:** Customer view page and email sending both depend on tokens. Building the token foundation first prevents rework in downstream phases. No UI needed — testable with curl/Postman.
**Delivers:** `quote_tokens` collection, `QuoteTokenService`, `PublicQuoteController`, `QuotePublicRetriever`; public GET endpoint returning restricted quote DTO.
**Addresses:** Secure customer access foundation for all downstream features.
**Implements:** Token-as-capability pattern; separate public controller with no auth guard; restricted `IPublicQuoteResponse` DTO (no internal IDs exposed).
**Avoids:** Pitfall 2 (auth bypass), Pitfall 3 (token security), Pitfall 9 (data over-exposure to public endpoint).
**Research flag:** HMAC token design has security nuance. Review ARCHITECTURE.md token design section carefully before writing implementation tasks. High confidence but security implications warrant care.

### Phase 3: Customer Quote Page (Frontend)
**Rationale:** The page the email links to must exist before sending emails is meaningful. Building the view before wiring accept/reject lets the token lookup flow be tested independently.
**Delivers:** Public React route `/q/:token`; `CustomerQuotePage` with `PublicQuoteView`; `publicQuoteApi.ts` RTK Query slice.
**Addresses:** Customer quote view (table stakes), PDF download button on public page.
**Implements:** Public route outside `AuthenticatedLayout`; separate RTK Query slice without auth headers.
**Avoids:** Pitfall 10 (SPA auth routing conflict), Pitfall 17 (no post-action feedback — build confirmation states and already-responded handling now).
**Research flag:** React Router public route pattern is well-established. No deeper research needed.

### Phase 4: Quote Email Sending
**Rationale:** Depends on token infrastructure (Phase 2) for link generation and the public page (Phase 3) as the landing destination. This is the core user-facing action of v1.3.
**Delivers:** `QuoteSender` service; POST `.../send` endpoint; Handlebars email template; `SendQuoteDialog` component; RTK Query `sendQuote` mutation.
**Addresses:** Email quote to customer (table stakes), customizable email message.
**Uses:** handlebars ^4.7.8 (install: `npm install handlebars`)
**Implements:** Send-then-transition order (email first, status update second); app domain + reply-to pattern for SendGrid from address; Handlebars template with data-only variables (no HTML passed as variables).
**Avoids:** Pitfall 1 (nullable customer email — validate before send, disable button in UI), Pitfall 5 (SendGrid from domain mismatch), Pitfall 8 (send failure status limbo), Pitfall 13 (SendGrid HTML escaping — data-only template variables).
**Research flag:** SendGrid integration is well-documented. Handlebars compilation is a solved problem. No deeper research needed.

### Phase 5: Customer Response (Accept/Reject)
**Rationale:** Customers need to have received emails (Phase 4) before accept/reject is meaningful in end-to-end testing. `QuotePublicResponder` is the most security-sensitive new service and is best built with the other public endpoint infrastructure already in place.
**Delivers:** `QuotePublicResponder` service; POST `.../respond` endpoint; `QuoteResponseButtons` and confirmation UI; tradesperson notification email on response.
**Addresses:** Customer accept/decline (table stakes), tradesperson notification (table stakes), automatic status update.
**Implements:** Atomic `findOneAndUpdate` with status precondition; dedicated service that bypasses auth-aware policy layer; idempotent response (already-responded quotes show current state, not an error).
**Avoids:** Pitfall 2 (auth bypass breaks services), Pitfall 4 (race condition double accept/reject), Pitfall 12 (re-send creates duplicate tokens — invalidate old tokens as part of re-send here).
**Research flag:** MongoDB atomic update pattern is well-documented. No deeper research needed.

### Phase 6: PDF Generation
**Rationale:** Standalone feature with the heaviest new dependency. Building last means the send flow (Phases 4–5) can be tested with a link-only email first, then PDF attachment is added once generation is verified working. Keeps new library risk isolated from the critical path.
**Delivers:** `QuotePdfGenerator` service using pdfmake; GET `.../pdf` authenticated endpoint; PDF attachment added to quote email; download button on `QuoteDetailPage`.
**Addresses:** Quote PDF generation (table stakes), PDF download from customer view.
**Uses:** pdfmake ^0.3.6 and @types/pdfmake ^0.2.x (install: `npm install pdfmake && npm install -D @types/pdfmake`)
**Avoids:** Pitfall 6 (money formatting mismatch — shared `Intl.NumberFormat` utility), Pitfall 7 (Puppeteer deployment issues — use pdfmake, not Puppeteer), Pitfall 14 (missing business branding — use available fields, do not block on adding new entity fields).
**Research flag:** pdfmake declarative API is well-documented. No deeper research needed.

### Phase Ordering Rationale

- **Deletion first** because it has zero dependencies and zero security complexity; establishes the new service-per-action file pattern before more complex work begins
- **Token infrastructure second** because both email sending and the customer view depend on tokens; the foundation must exist before either consumer is built
- **Frontend public page before email** because the email links to this page; having the destination working makes the email feature testable end-to-end on first integration
- **Email before customer response** because the response flow requires customers to have received emails; also mirrors the natural user journey order
- **PDF last** because it is independent of the accept/reject flow, has the heaviest new dependency (pdfmake), and can be bolted onto the email as an attachment without blocking the core delivery loop
- This order exactly matches the dependency graph in FEATURES.md and the suggested build order in ARCHITECTURE.md

### Research Flags

Phases with well-documented patterns (skip `/gsd:research-phase`):
- **Phase 1 (Deletion):** Standard NestJS service-per-action + MongoDB hard delete. No new patterns.
- **Phase 3 (Frontend public page):** React Router public route + RTK Query without auth. Well-documented.
- **Phase 4 (Email sending):** Handlebars + SendGrid inline HTML. Established pattern.
- **Phase 5 (Customer response):** MongoDB atomic update + NestJS controller. Well-documented.
- **Phase 6 (PDF generation):** pdfmake declarative API. Well-documented.

Phases that may benefit from deeper research during planning:
- **Phase 2 (Token infrastructure):** HMAC token signing and the `quote_tokens` collection schema design have security nuance (TTL indexes, revocation logic, raw vs hashed token storage). Worth reviewing ARCHITECTURE.md token design section in detail before writing implementation tasks.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All recommendations verified against existing codebase and npm registry. pdfmake and handlebars are mature, stable, zero-native-dependency libraries. One conflict resolved: ARCHITECTURE.md suggests Puppeteer; STACK.md and PITFALLS.md both recommend pdfmake — pdfmake wins. |
| Features | HIGH | Validated against 5 direct competitors (Tradify, Fergus, Jobber, ServiceM8, Xero) via official help documentation. MVP boundary is well-supported. Existing quote entity already has all required status and timestamp fields. |
| Architecture | HIGH | All integration points verified against actual codebase files. Existing `QuoteTransitionService`, `EmailSenderService`, and quote entity confirmed compatible. Token pattern sourced from established security literature. |
| Pitfalls | HIGH | Most critical pitfalls sourced from high-confidence official documentation and codebase analysis. MongoDB atomic update pattern sourced from multiple independent sources. Puppeteer deployment pitfall is well-documented in production deployment guides. |

**Overall confidence:** HIGH

### Gaps to Address

- **Token storage: raw vs hashed** — STACK.md stores the raw token directly on the quote entity (simpler); ARCHITECTURE.md uses a separate `quote_tokens` collection and suggests storing the token as-is (not hashed). Recommendation: use the separate `quote_tokens` collection for revocability and cleanliness, store the raw token (hashing adds complexity without meaningful benefit when the DB is not the attack surface and the token is already 256-bit random). Confirm this choice explicitly in Phase 2 planning before writing code.

- **SendGrid "from" address configuration** — Emails must be sent from a verified SendGrid sender domain (e.g., `quotes@tradeflow.app`). Confirm this sender identity exists in the SendGrid account before Phase 4 begins. This is an ops concern, not a code concern, but it blocks Phase 4 testing.

- **Business entity branding fields** — `BusinessEntity` lacks address, phone, and logo. The PDF will use business name + tradesperson email as the from header for v1.3. Accepted as a known limitation. A "business profile" milestone is the correct fix; do not block PDF generation on adding new entity fields.

- **Re-send token invalidation** — The existing `ALLOWED_TRANSITIONS` map already allows SENT -> SENT. Token invalidation on re-send must be explicitly built into `QuoteSender` (not assumed). Confirm this is in the Phase 5 implementation tasks.

## Sources

### Primary (HIGH confidence)
- Trade Flow codebase: `quote.entity.ts`, `quote-transitions.ts`, `quote-transition.service.ts`, `email-sender.service.ts`, `customer.entity.ts`, `business.entity.ts`, `auth.guard.ts`, `App.tsx`, `package.json` — all integration points verified against actual code
- [pdfmake npm](https://www.npmjs.com/package/pdfmake) — version 0.3.6 confirmed
- [Node.js crypto docs](https://nodejs.org/api/crypto.html) — randomBytes API
- [Handlebars official site](https://handlebarsjs.com/) — API reference
- [SendGrid Deliverability Best Practices](https://support.sendgrid.com/hc/en-us/articles/360041790453-Best-Practices-for-Email-Deliverability)
- [Token Security Best Practices](https://auth0.com/docs/secure/tokens/token-best-practices)
- [NestJS Authorization docs](https://docs.nestjs.com/security/authorization) — guard patterns

### Secondary (MEDIUM confidence)
- [Tradify help docs](https://help.tradifyhq.com/), [Fergus features](https://fergus.com/features/quoting/), [Jobber quote approvals](https://help.getjobber.com/hc/en-us/articles/115012715008-Quote-Approvals), [ServiceM8 quote acceptance](https://support.servicem8.com/hc/en-us/articles/115000282726-How-to-use-a-quote-acceptance), [Xero send quotes](https://www.xero.com/us/accounting-software/send-quotes/) — competitor feature analysis
- [MongoDB atomic operations for race conditions](https://medium.com/tales-from-nimilandia/handling-race-conditions-and-concurrent-resource-updates-in-node-and-mongodb-by-performing-atomic-9f1a902bd5fa) — atomic findOneAndUpdate pattern
- [URL protection through HMAC](https://blog.cyril.email/posts/2025-03-12/url-protection-through-hmac.html) — HMAC token signing pattern
- [PDF generation in Node.js tips](https://joyfill.io/blog/integrating-pdf-generation-into-node-js-backends-tips-gotchas) — pdfmake vs Puppeteer comparison
- [Puppeteer vs PDFKit comparison](https://www.leadwithskills.com/blogs/pdf-generation-nodejs-puppeteer-pdfkit) — library selection rationale

---
*Research completed: 2026-03-15*
*Ready for roadmap: yes*
