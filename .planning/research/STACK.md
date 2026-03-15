# Stack Research

**Domain:** Quote email delivery, customer response flow, PDF generation, quote deletion
**Researched:** 2026-03-15
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

No new core frameworks needed. All new features build on the existing NestJS/MongoDB/React stack. The additions below are library-level, not framework-level.

### New Backend Libraries

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| pdfmake | ^0.3.6 | Server-side PDF generation for quotes | Declarative JSON document definitions fit perfectly for structured quote documents (line items table, totals, headers). No browser/Puppeteer dependency means no headless Chrome in production. Pure JS, zero native dependencies. Built on PDFKit but with a declarative API that maps naturally to quote data structures. |
| @types/pdfmake | ^0.2.x | TypeScript types for pdfmake | Required for type-safe document definitions in the strict TypeScript codebase. |
| handlebars | ^4.7.8 | Server-side HTML email template compilation | Compiles HTML templates with dynamic data (customer name, quote number, totals, action URLs). Same templating syntax SendGrid Dynamic Templates use internally (Handlebars), so the mental model is consistent. Mature, stable, zero-dependency templating. |

### No New Frontend Libraries

The customer-facing quote response page and any UI additions use the existing React/Vite/Tailwind/Radix stack. No new frontend dependencies are needed.

### No New Infrastructure

SendGrid is already integrated (`@sendgrid/mail` ^8.1.6 in package.json). The existing `EmailSenderService` already supports HTML email sending. The new features extend it, not replace it.

## Integration Points with Existing Code

### SendGrid Email (Existing -- Extend)

The existing `EmailSenderService` sends emails with inline HTML via `@sendgrid/mail`. For quote emails:

**Approach: Inline HTML compiled from Handlebars templates**
- Compile Handlebars templates server-side into HTML strings
- Pass compiled HTML to existing `EmailSenderService.sendEmail()` method
- Templates live as `.hbs` files in the API codebase (e.g., `src/quote/templates/quote-email.hbs`)
- Full control over templates in version control, no SendGrid dashboard dependency, testable in unit tests, works with the existing `sendEmail(html)` pattern

The existing `SendEmailDto` interface needs updating -- it currently has `senderName` and `wishlistUrl` fields that are artifacts from earlier work. These should be cleaned up or a new quote-specific DTO created.

### Quote Entity (Existing -- Extend)

The `IQuoteEntity` already has `sentAt`, `acceptedAt`, `rejectedAt` fields and `QuoteStatus` enum with DRAFT/SENT/ACCEPTED/REJECTED/EXPIRED values. The data model is ready for the send/respond flow. Only a new `responseToken` field needs to be added.

### Customer Entity (Existing -- Read)

`ICustomerEntity` has `email: string | null` and `name: string`. Email sending must validate that the customer has an email address before allowing quote send.

### Business Entity (Existing -- Read)

`BusinessEntity` has `name`, `currency`, `country`. These appear in the quote email and PDF header (business name, currency formatting).

### Authentication (Existing -- Bypass for Customer Response)

Firebase JWT auth protects all current endpoints. Customer accept/reject endpoints must be **unauthenticated** -- customers don't have Trade Flow accounts. Use secure token-based access instead (see below).

## New Architectural Components

### Secure Token for Customer Quote Response

**Approach: Cryptographic random token using Node.js built-in `crypto`**

Generate a token from `crypto.randomBytes(32).toString('hex')` stored on the quote document. The customer response URL includes this token as the sole authentication mechanism.

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Token generation | `crypto.randomBytes(32)` | Built-in Node.js, cryptographically secure, no additional dependency needed |
| Token storage | New `responseToken` field on Quote entity | Simple, queryable, one token per quote |
| Token format | 64-character hex string | URL-safe, no encoding issues |
| Expiration | Use existing `validUntil` field on quote | Already on the quote entity, natural business logic alignment |
| Endpoint pattern | `GET /quotes/respond/:token` (view) + `POST /quotes/respond/:token` (accept/reject) | Public endpoints, no Firebase auth guard |

**Why NOT JWT for tokens:** JWTs are self-contained and can't be revoked server-side without a blocklist. A simple random token stored in the database can be invalidated by clearing it. Simpler, more secure for this one-time-use case.

**Why NOT a separate tokens collection:** One token per quote, strict 1:1 relationship. Adding a field to the existing quote document is simpler than a separate collection with a foreign key.

### PDF Generation Service

**Approach: pdfmake with declarative document definitions**

A new `QuotePdfGenerator` service in the quote module that:
1. Takes a quote DTO with line items and totals
2. Builds a pdfmake document definition (JSON object with table, headers, footer)
3. Returns a `Buffer` containing the PDF bytes

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Library | pdfmake | Declarative JSON API maps directly to quote structure (header with business/customer info, line items table, totals row, notes section). No HTML/CSS rendering needed. |
| Output | Buffer (not file) | Stream directly to HTTP response or attach to email. No filesystem writes needed. |
| Fonts | pdfmake built-in Roboto | Professional, readable, no font file management required |
| Template | Code-defined document definition | Quote layout is structured tabular data, not freeform content. A JSON definition is more maintainable than HTML-to-PDF for invoice-style documents. |
| Storage | Generate on-the-fly, do not persist | PDFs are deterministic from quote data. Regenerate on demand rather than storing in MongoDB or S3. |

### Customer Response Page (Frontend)

**Approach: New public route in the existing React app**

A route like `/quote/respond/:token` that:
1. Fetches quote details from a public API endpoint (no auth required)
2. Displays quote summary (business name, line items, totals)
3. Provides Accept/Reject buttons that POST to the public API endpoint

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Hosting | Same React app, public route | No separate app to deploy. React Router already handles routing. |
| Auth bypass | Route outside protected route wrapper | Customer does not need a Trade Flow account |
| Styling | Same Tailwind/Radix components | Consistent professional look, no additional CSS framework |
| State | Local fetch or simple RTK Query endpoint | One-off page with minimal state requirements |

### Quote Deletion

**Approach: Hard delete for DRAFT quotes only**

No new libraries needed. This is pure business logic in the existing service layer. Only DRAFT status quotes should be deletable (sent/accepted/rejected quotes have business significance and should be preserved).

## Installation

```bash
# In trade-flow-api/
npm install pdfmake handlebars
npm install -D @types/pdfmake
```

No new packages needed in `trade-flow-ui/`.

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| pdfmake (declarative JSON) | Puppeteer (headless Chrome) | When PDF must pixel-perfectly match an HTML/CSS design mockup. Not this case -- quotes are structured tabular data. Puppeteer adds ~300MB Chrome dependency and is significantly slower. |
| pdfmake (declarative JSON) | PDFKit (imperative drawing API) | When you need pixel-level control over every drawn element. Not this case -- pdfmake wraps PDFKit with a declarative API that is better suited for document-style PDFs with tables. |
| pdfmake (declarative JSON) | @react-pdf/renderer | When PDF generation happens client-side or you want JSX syntax for templates. Not this case -- generation is server-side in NestJS. |
| Handlebars (server-side compile) | SendGrid Dynamic Templates | When non-developers need to edit email templates via a visual UI dashboard. Not this case -- solo dev, templates should live in version control and be unit-testable. |
| Handlebars (server-side compile) | MJML (responsive email framework) | When building a large library of responsive email templates with complex layouts. Overkill for 1-2 transactional quote emails. Adds a compilation step and new DSL. |
| crypto.randomBytes token | JWT token in URL | When token needs to carry claims without a DB lookup. Not this case -- revocability matters and the DB lookup is cheap (indexed field). |
| Public route in existing React app | Separate static page / standalone app | When the response page needs to be extremely lightweight or hosted on a different domain. Not this case -- reusing existing UI components is faster to build and maintain. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Puppeteer / Playwright for PDF | 300MB+ Chrome dependency, slow cold start, heavy memory usage, overkill for structured documents | pdfmake -- zero native dependencies, fast, declarative |
| MJML for email templates | Adds build step, new template DSL to learn, overkill for 1-2 transactional emails | Handlebars compiling inline HTML -- simpler, already sufficient |
| SendGrid Dynamic Templates | Templates live outside codebase, cannot be version-controlled, cannot be unit tested, adds dashboard dependency | Handlebars server-side compilation with existing inline HTML sending pattern |
| Separate microservice for PDF | Architecture overhead for a single feature within an existing module | Service class within existing quote module |
| nodemailer | Already using SendGrid SDK; adding nodemailer creates two competing email paths | Existing @sendgrid/mail integration |
| uuid for tokens | 122 bits of entropy vs 256 bits from crypto.randomBytes; adds unnecessary dependency | crypto.randomBytes(32) -- built-in Node.js, more secure |
| Storing PDFs in MongoDB/S3 | Adds storage management complexity; PDFs are deterministic and can be regenerated on demand from quote data | Generate on-the-fly from quote data |
| html-pdf / wkhtmltopdf | Deprecated, relies on QtWebKit, security issues, no active maintenance | pdfmake -- actively maintained, pure JS |

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| pdfmake@0.3.6 | Node.js 22.x | Pure JS, no native dependencies, broad Node.js compatibility |
| handlebars@4.7.8 | Node.js 22.x | Stable for years, no known compatibility issues. Last published 2023 but fully functional -- templating is a solved problem. |
| @types/pdfmake@0.2.x | TypeScript 5.9.x | Verify type definitions match pdfmake 0.3.x API after install |
| @sendgrid/mail@8.1.6 | Already installed | No changes needed; existing version supports inline HTML and dynamic template features |

## Summary of Changes by Repo

### trade-flow-api (2 new packages)
- `pdfmake` + `@types/pdfmake` for PDF generation
- `handlebars` for email template compilation
- New `responseToken` field on quote entity
- New public (unauthenticated) controller endpoints for customer response
- New `QuotePdfGenerator` service
- New `QuoteEmailSender` service (uses existing `EmailSenderService`)
- Handlebars template files in `src/quote/templates/`

### trade-flow-ui (0 new packages)
- New public route `/quote/respond/:token`
- New `QuoteResponsePage` component (uses existing Tailwind/Radix components)
- Quote detail page additions (Send button, PDF download button, Delete button)

## Sources

- [pdfmake on npm](https://www.npmjs.com/package/pdfmake) -- version 0.3.6 verified, HIGH confidence
- [pdfmake GitHub](https://github.com/bpampuch/pdfmake) -- capability and API verification, HIGH confidence
- [SendGrid Dynamic Templates docs](https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates) -- template API reference, HIGH confidence
- [SendGrid Node.js transactional templates](https://github.com/sendgrid/sendgrid-nodejs/blob/main/docs/use-cases/transactional-templates.md) -- code examples, HIGH confidence
- [Handlebars official site](https://handlebarsjs.com/) -- API reference, HIGH confidence
- [Node.js Crypto documentation](https://nodejs.org/api/crypto.html) -- randomBytes API, HIGH confidence
- [JS PDF library comparison (DEV Community)](https://dev.to/handdot/generate-a-pdf-in-js-summary-and-comparison-of-libraries-3k0p) -- pdfmake vs alternatives, MEDIUM confidence
- [Top JS PDF libraries 2026 (Nutrient)](https://www.nutrient.io/blog/top-js-pdf-libraries/) -- ecosystem overview, MEDIUM confidence
- Codebase: `trade-flow-api/src/email/services/email-sender.service.ts` -- existing SendGrid integration pattern, HIGH confidence
- Codebase: `trade-flow-api/src/quote/entities/quote.entity.ts` -- existing quote data model, HIGH confidence
- Codebase: `trade-flow-api/package.json` -- existing dependency versions, HIGH confidence

---
*Stack research for: Trade Flow v1.3 -- Send Quotes*
*Researched: 2026-03-15*
