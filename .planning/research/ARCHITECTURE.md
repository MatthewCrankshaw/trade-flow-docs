# Architecture Research

**Domain:** Quote email delivery, customer-facing public pages, PDF generation, and quote deletion for existing Trade Flow system
**Researched:** 2026-03-15
**Confidence:** HIGH

## System Overview

```
                         AUTHENTICATED (existing)                    PUBLIC (new)
                    ┌──────────────────────────────┐         ┌──────────────────────┐
                    │    Trade Flow UI (React)      │         │  Customer Quote Page  │
                    │  /quotes/:id  (detail page)   │         │  /q/:token            │
                    │  Send Quote button             │         │  (no auth required)   │
                    │  Delete Quote button            │         │  Accept / Reject      │
                    └──────────┬───────────────────┘         └──────────┬───────────┘
                               │                                        │
                    Firebase JWT │                          Token in URL  │
                               │                                        │
┌──────────────────────────────┴────────────────────────────────────────┴──────────────┐
│                              Trade Flow API (NestJS)                                  │
│                                                                                       │
│  ┌─────────────────────────────────┐    ┌──────────────────────────────────┐           │
│  │  QuoteController (existing)     │    │  PublicQuoteController (NEW)     │           │
│  │  @UseGuards(JwtAuthGuard)       │    │  No auth guard                   │           │
│  │                                 │    │  Token-based access              │           │
│  │  POST .../send        (NEW)     │    │  GET  /v1/public/quote/:token    │           │
│  │  DELETE .../quote/:id  (NEW)    │    │  POST /v1/public/quote/:token/   │           │
│  │  GET .../quote/:id/pdf (NEW)    │    │       respond                    │           │
│  └────────┬────────────────────────┘    └────────┬─────────────────────────┘           │
│           │                                       │                                    │
│  ┌────────┴───────────────────────────────────────┴─────────────────────────────┐      │
│  │                           Quote Services Layer                                │      │
│  │                                                                               │      │
│  │  QuoteSender (NEW)          QuoteTokenService (NEW)                           │      │
│  │  QuotePdfGenerator (NEW)    QuotePublicRetriever (NEW)                        │      │
│  │  QuoteDeleter (NEW)         QuotePublicResponder (NEW)                        │      │
│  └──────────┬──────────────────────────────────┬────────────────────────────────┘      │
│             │                                   │                                      │
│  ┌──────────┴──────────┐             ┌─────────┴──────────┐                            │
│  │  EmailSenderService │             │  QuoteRepository   │                            │
│  │  (existing)         │             │  (existing)        │                            │
│  └─────────┬───────────┘             └─────────┬──────────┘                            │
│            │                                    │                                      │
└────────────┼────────────────────────────────────┼──────────────────────────────────────┘
             │                                    │
     ┌───────┴───────┐                  ┌────────┴────────┐
     │   SendGrid    │                  │    MongoDB      │
     │               │                  │  quotes         │
     └───────────────┘                  │  quote_tokens   │
                                        └─────────────────┘
```

### Component Responsibilities

| Component | Responsibility | New vs Existing |
|-----------|----------------|-----------------|
| **QuoteController** | Authenticated quote operations (send, delete, PDF download) | MODIFY -- add 3 endpoints |
| **PublicQuoteController** | Unauthenticated customer-facing quote view and respond | NEW controller |
| **QuoteSender** | Orchestrates sending: generate token, build email, send via SendGrid, transition status | NEW service |
| **QuoteTokenService** | Generate and validate HMAC-signed tokens for public quote access | NEW service |
| **QuotePublicRetriever** | Retrieve quote data for public display (no auth user, token-based) | NEW service |
| **QuotePublicResponder** | Accept/reject quote from customer response, update status | NEW service |
| **QuotePdfGenerator** | Generate PDF buffer from quote data using Puppeteer | NEW service |
| **QuoteDeleter** | Delete quote and associated line items | NEW service |
| **EmailSenderService** | Send emails via SendGrid | EXISTING -- no changes needed |
| **QuoteRepository** | Quote persistence | EXISTING -- add token field queries |
| **QuoteTokenRepository** | Token persistence and lookup | NEW repository |
| **Customer Quote Page** | Public React page at `/q/:token` showing quote + accept/reject buttons | NEW page (frontend) |

## Recommended Architecture

### 1. Token-Based Public Access (HMAC-Signed Tokens)

**Pattern:** HMAC-signed opaque token stored in a separate `quote_tokens` collection.

**Why not just a UUID?** A bare UUID is guessable with enough attempts and provides no cryptographic guarantee. An HMAC-signed token ties the token to the quote ID and a server-side secret, making forgery impossible without the secret key.

**Why not a JWT?** JWTs are self-contained and cannot be revoked without a blacklist. A database-backed token can be invalidated when the quote is re-sent, deleted, or expired. JWTs also expose payload data in the URL (base64-encoded), which is undesirable.

**Token design:**

```typescript
// quote_tokens collection
interface IQuoteTokenEntity {
  _id: ObjectId;
  token: string;          // HMAC-SHA256 hex string (64 chars)
  quoteId: ObjectId;
  businessId: ObjectId;
  expiresAt: Date;        // Token expiry (e.g., 30 days from send)
  createdAt: Date;
  revokedAt?: Date;       // Set when quote is re-sent or deleted
}
```

**Token generation:**

```typescript
import { createHmac, randomBytes } from 'crypto';

// Generate: HMAC-SHA256(secret, quoteId + nonce)
const nonce = randomBytes(16).toString('hex');
const token = createHmac('sha256', QUOTE_TOKEN_SECRET)
  .update(`${quoteId}:${nonce}`)
  .digest('hex');
```

**URL format:** `https://app.tradeflow.com/q/{token}`

- 64-character hex string -- long enough to be unguessable, short enough for email links
- Single path segment -- clean URL, no query parameters to strip
- Token is the only identifier -- no quote ID exposed in URL

**Validation flow:**

```
1. Customer clicks link -> GET /v1/public/quote/:token
2. API looks up token in quote_tokens collection
3. Checks: not revoked, not expired
4. Returns quote data (subset -- no internal IDs exposed)
```

### 2. Public API Endpoints (New Controller)

**Pattern:** Separate controller with NO auth guard, using token-based access instead.

```typescript
@Controller("v1/public")
export class PublicQuoteController {
  // NO @UseGuards(JwtAuthGuard) -- intentionally public

  @Get("quote/:token")
  async viewQuote(@Req() request: { params: { token: string } }) {
    // Token validation replaces auth
  }

  @Post("quote/:token/respond")
  async respondToQuote(
    @Req() request: { params: { token: string } },
    @Body() body: QuoteResponseRequest,  // { action: "accept" | "reject" }
  ) {
    // Token validation + one-time action
  }
}
```

**Why a separate controller?** Mixing public and authenticated endpoints in the same controller creates confusion about which guard applies. A dedicated `PublicQuoteController` makes the security boundary explicit and auditable.

**Public response format:** Same `{ data: T[] }` structure, but with a restricted response type:

```typescript
// Only expose what the customer needs to see
interface IPublicQuoteResponse {
  businessName: string;
  quoteNumber: string;
  quoteDate: string;
  validUntil?: string;
  title: string;
  notes?: string;
  status: string;
  lineItems: IPublicLineItemResponse[];
  totals: { subTotal: number; taxTotal: number; total: number };
}

// No internal IDs, no businessId, no customerId
interface IPublicLineItemResponse {
  description: string;      // Item name (resolved)
  quantity: number;
  unit: string;
  unitPrice: number;
  lineTotal: number;
  taxRate: number;
  components?: IPublicLineItemResponse[];
}
```

### 3. Quote Email Delivery

**Data flow:**

```
Tradesperson clicks "Send Quote" on QuoteDetailPage
    |
Frontend: POST /v1/business/:bid/quote/:qid/send
    |
QuoteController.sendQuote()
    |
QuoteSender.send(authUser, quoteId, businessId)
    |-- 1. Validate quote is in DRAFT or SENT status
    |-- 2. Fetch quote + customer + business data
    |-- 3. Validate customer has email address
    |-- 4. Generate token via QuoteTokenService (revoke old tokens)
    |-- 5. Build email HTML (inline CSS, business name, quote summary, CTA link)
    |-- 6. Send via EmailSenderService
    |-- 7. Transition quote to SENT status (via QuoteTransitionService)
    |
Response: updated quote with status=SENT, sentAt timestamp
```

**Email construction:** Build HTML server-side using template literals with inline CSS. No external template engine needed for a single transactional email template. SendGrid dynamic templates add UI management overhead that is not justified for one template.

**Email content:**
- From: Business name (via SendGrid sender identity)
- Subject: "Quote {Q-YYYY-NNN} from {Business Name}"
- Body: Business name, quote summary (number, date, total), "View Quote" CTA button linking to `/q/{token}`
- No PDF attachment in the email itself (customer views interactive page instead)

### 4. Customer Response Flow

**Data flow:**

```
Customer clicks "View Quote" link in email
    |
Browser: GET https://app.tradeflow.com/q/{token}
    |
React Router: renders CustomerQuotePage (public, no auth)
    |
CustomerQuotePage: GET /v1/public/quote/{token}
    |
PublicQuoteController.viewQuote()
    |-- QuoteTokenService.validateToken(token) -> quoteId
    |-- QuotePublicRetriever.getForPublicDisplay(quoteId) -> quote + business + items
    |-- Returns IPublicQuoteResponse
    |
Customer sees quote details + Accept/Reject buttons
    |
Customer clicks "Accept" (or "Reject")
    |
CustomerQuotePage: POST /v1/public/quote/{token}/respond { action: "accept" }
    |
PublicQuoteController.respondToQuote()
    |-- QuoteTokenService.validateToken(token) -> quoteId
    |-- QuotePublicResponder.respond(quoteId, action)
    |   |-- Validate quote is in SENT status
    |   |-- Transition to ACCEPTED or REJECTED
    |   |-- Set acceptedAt or rejectedAt timestamp
    |-- Returns success confirmation
    |
CustomerQuotePage shows confirmation message ("Quote accepted" / "Quote rejected")
```

**Idempotency:** If a quote has already been accepted/rejected, subsequent attempts return the current state rather than error. The customer sees "This quote has already been accepted" rather than a 422 error.

**No re-vote:** Once accepted or rejected, the status is terminal (matches existing transition rules where ACCEPTED and REJECTED have no valid outgoing transitions).

### 5. PDF Generation

**Technology:** Puppeteer (headless Chrome) for HTML-to-PDF conversion.

**Why Puppeteer over PDFKit:**
- Quote layout is already defined in HTML/CSS on the frontend
- Puppeteer renders real HTML/CSS to pixel-perfect PDF
- PDFKit requires manually constructing every element programmatically -- high effort for a layout that already exists as HTML
- Quote PDF is generated on-demand (not high-volume), so Puppeteer's memory overhead is acceptable

**Implementation pattern:**

```typescript
@Injectable()
export class QuotePdfGenerator {
  async generate(quote: IQuoteDto, business: IBusinessDto, customer: ICustomerDto): Promise<Buffer> {
    const html = this.buildQuoteHtml(quote, business, customer);
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();
    await page.setContent(html, { waitUntil: 'networkidle0' });
    const pdf = await page.pdf({ format: 'A4', printBackground: true });
    await browser.close();
    return Buffer.from(pdf);
  }
}
```

**Endpoint:** `GET /v1/business/:bid/quote/:qid/pdf` (authenticated -- tradesperson downloads PDF)

**Response:** Binary PDF with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="Q-2026-001.pdf"`

**Note:** This endpoint does NOT return the standard `{ data: T[] }` JSON format because it returns a binary file. This is an intentional exception documented in the controller.

**HTML template:** Self-contained HTML string with inline CSS. No external assets. Includes: business name, quote number/date, customer details, line items table, totals, notes.

### 6. Quote Deletion

**Pattern:** Hard delete (remove quote document and all associated line items from database).

**Why hard delete, not soft delete?** The existing line item soft delete (DELETED status enum) is for individual line items within an active quote. Quote-level deletion removes the entire quote because:
- A tradesperson deleting a quote means "this should not exist"
- No business requirement for deleted quote history (out of scope)
- Keeps the data model simple
- If audit logging is needed later (noted as out of scope), it can be added as a separate concern

**Data flow:**

```
Tradesperson clicks "Delete Quote" on QuoteDetailPage -> confirmation dialog
    |
Frontend: DELETE /v1/business/:bid/quote/:qid
    |
QuoteController.deleteQuote()
    |
QuoteDeleter.delete(authUser, quoteId, businessId)
    |-- 1. Fetch quote, validate ownership
    |-- 2. Validate quote is in deletable status (DRAFT only, or DRAFT + SENT)
    |-- 3. Revoke any active tokens (via QuoteTokenService)
    |-- 4. Delete all quote_line_items for this quoteId
    |-- 5. Delete the quote document
    |
Response: 200 with empty data array { data: [] }
Frontend: invalidate quote cache, navigate to /quotes list
```

**Deletable statuses:** Only DRAFT quotes should be deletable. Once a quote is SENT (customer may have seen it), ACCEPTED, or REJECTED, deletion should be blocked. This prevents the tradesperson from deleting a quote the customer is actively viewing.

## Data Flow

### New MongoDB Collections

```
quote_tokens
|-- _id: ObjectId
|-- token: string (indexed, unique)
|-- quoteId: ObjectId (indexed)
|-- businessId: ObjectId
|-- expiresAt: Date (TTL index for automatic cleanup)
|-- createdAt: Date
|-- revokedAt: Date | null
```

### Modified Collections

**quotes** -- No schema changes needed. The existing entity already has `sentAt`, `acceptedAt`, `rejectedAt`, and `status` fields.

**quote_line_items** -- No schema changes needed. Deletion will use `deleteMany({ quoteId })`.

### New Indexes

| Collection | Index | Purpose |
|------------|-------|---------|
| `quote_tokens` | `{ token: 1 }` unique | Token lookup for public access |
| `quote_tokens` | `{ quoteId: 1 }` | Revoke tokens when quote is re-sent/deleted |
| `quote_tokens` | `{ expiresAt: 1 }` TTL | Automatic cleanup of expired tokens |

## Architectural Patterns

### Pattern 1: Separate Public Controller

**What:** A dedicated controller for unauthenticated endpoints, clearly separated from authenticated controllers.
**When to use:** When adding public-facing endpoints to an otherwise fully authenticated API.
**Trade-offs:** Slight duplication in response mapping vs. crystal-clear security boundary.

### Pattern 2: Token-as-Capability

**What:** The token itself grants access to a specific resource and action. Possession of the token = authorization.
**When to use:** When unauthenticated users need scoped access to a single resource (email links, share links).
**Trade-offs:** Token leakage = unauthorized access (mitigated by expiry and HTTPS). Cannot revoke in transit (mitigated by server-side revocation check).

### Pattern 3: Service-per-Action

**What:** Each new operation (send, delete, PDF generate, public respond) gets its own service class, following the existing Creator/Retriever/Updater/Deleter pattern.
**When to use:** Always -- this is the established project convention.
**Trade-offs:** More files, but each is small, testable, and single-responsibility.

## Frontend Architecture

### New Components (trade-flow-ui)

| Component | Location | Purpose |
|-----------|----------|---------|
| **CustomerQuotePage** | `src/pages/CustomerQuotePage.tsx` | Public page at `/q/:token` |
| **PublicQuoteView** | `src/features/quotes/components/PublicQuoteView.tsx` | Quote display for customers |
| **QuoteResponseButtons** | `src/features/quotes/components/QuoteResponseButtons.tsx` | Accept/Reject CTA |
| **QuoteResponseConfirmation** | `src/features/quotes/components/QuoteResponseConfirmation.tsx` | Post-response message |
| **SendQuoteDialog** | `src/features/quotes/components/SendQuoteDialog.tsx` | Confirmation before sending |
| **DeleteQuoteDialog** | `src/features/quotes/components/DeleteQuoteDialog.tsx` | Confirmation before deleting |

### Modified Components

| Component | Change |
|-----------|--------|
| **QuoteActionStrip** | Add "Send Quote" and "Delete Quote" buttons |
| **QuoteDetailPage** | Wire up send/delete actions, add PDF download button |
| **quoteApi.ts** | Add `sendQuote`, `deleteQuote`, `downloadQuotePdf` mutations/queries |

### Routing Changes

```tsx
// In App.tsx -- add BEFORE the catch-all redirect
<Route path="/q/:token" element={<CustomerQuotePage />} />

// This route sits OUTSIDE AuthenticatedLayout (no ProtectedRoute wrapper)
```

**Important:** The `/q/:token` route must NOT be inside the `AuthenticatedLayout` wrapper. It needs:
- Redux Provider (for RTK Query to work)
- ErrorBoundary
- But NOT AuthProvider/ProtectedRoute/OnboardingProvider

### RTK Query for Public Endpoints

```typescript
// New: publicQuoteApi.ts -- separate from quoteApi.ts
// Does NOT inject Firebase auth token (public endpoints)
const publicApiSlice = createApi({
  reducerPath: 'publicApi',
  baseQuery: fetchBaseQuery({ baseUrl: VITE_API_BASE_URL }),  // No auth headers
  endpoints: (builder) => ({
    getPublicQuote: builder.query({
      query: (token: string) => `/v1/public/quote/${token}`,
    }),
    respondToQuote: builder.mutation({
      query: ({ token, action }) => ({
        url: `/v1/public/quote/${token}/respond`,
        method: 'POST',
        body: { action },
      }),
    }),
  }),
});
```

**Why a separate API slice?** The existing `apiSlice` injects Firebase auth tokens in `prepareHeaders`. Public endpoints must NOT send auth tokens (the customer does not have a Firebase account). A separate slice with no auth header injection keeps the separation clean.

## Anti-Patterns

### Anti-Pattern 1: Reusing JwtAuthGuard with Optional Auth

**What people do:** Make the auth guard optional (`@UseGuards(OptionalJwtAuthGuard)`) and check `request.user` inside the handler.
**Why it is wrong:** Muddies the security contract. Every developer reading the code must understand "optional" means "sometimes unauthenticated." The existing `JwtAuthGuard` also creates users on first auth -- running it on public endpoints would fail.
**Do this instead:** Use a completely separate controller with no guard. Token validation is explicit in the service layer.

### Anti-Pattern 2: Exposing Internal IDs in Public Response

**What people do:** Return the same `IQuoteResponse` (with `businessId`, `customerId`, `jobId`) to public endpoints.
**Why it is wrong:** Leaks internal identifiers to unauthenticated users. A customer should not see MongoDB ObjectIds for business/customer/job records.
**Do this instead:** Create a restricted `IPublicQuoteResponse` that only includes display data (business name, quote number, line items with descriptions, totals).

### Anti-Pattern 3: PDF Generation as Background Job for On-Demand Downloads

**What people do:** Generate PDFs asynchronously and store them, then serve from storage.
**Why it is wrong for this use case:** Over-engineering. The tradesperson clicks "Download PDF" and expects it immediately. Quote data may change between generation and download. Storing PDFs creates a cache invalidation problem.
**Do this instead:** Generate on-demand when the download endpoint is hit. Puppeteer can generate a simple quote PDF in under 2 seconds.

### Anti-Pattern 4: Sending PDF as Email Attachment

**What people do:** Attach the PDF to the email so the customer has it immediately.
**Why it is wrong:** Email attachments increase email size (deliverability risk), PDFs cannot contain interactive accept/reject buttons, and the customer cannot respond without visiting the app. The PDF also becomes stale if the quote is updated and re-sent.
**Do this instead:** Email contains a link to the interactive quote page. PDF download is available on that page if the customer wants a copy.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| **SendGrid** | Existing `EmailSenderService.sendEmail()` | Add QuoteModule import of EmailModule. Build HTML in QuoteSender, pass to EmailSenderService |
| **Puppeteer** | New dependency, launched per PDF request | `npm install puppeteer`. Headless Chrome downloads ~300MB on first install. In production, use `puppeteer-core` + system Chrome to reduce bundle size |

### Internal Module Dependencies (New)

| Boundary | Communication | Notes |
|----------|---------------|-------|
| QuoteModule -> EmailModule | Direct service injection | QuoteModule imports EmailModule to access EmailSenderService |
| QuoteModule -> BusinessModule | Direct service injection | QuoteSender needs business name for email. Already used pattern (CustomerModule imports exist) |
| QuoteModule -> CustomerModule | Direct service injection | Already imported. QuoteSender needs customer email address |
| PublicQuoteController -> QuoteTokenService | Direct injection | Token validation on every public request |
| PublicQuoteController -> QuotePublicRetriever | Direct injection | Fetches quote without auth user |
| PublicQuoteController -> QuotePublicResponder | Direct injection | Handles accept/reject |

### New Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `QUOTE_TOKEN_SECRET` | HMAC signing secret for quote access tokens | YES -- generate with `openssl rand -hex 32` |
| `QUOTE_TOKEN_EXPIRY_DAYS` | Token expiry in days (default: 30) | NO -- sensible default |
| `APP_BASE_URL` | Frontend URL for constructing quote links in emails | YES -- e.g., `https://app.tradeflow.com` |

### New API Endpoints

| Endpoint | Method | Auth | Purpose | Returns |
|----------|--------|------|---------|---------|
| `/v1/business/:bid/quote/:qid/send` | POST | JWT | Send quote email to customer | Updated quote (status=SENT) |
| `/v1/business/:bid/quote/:qid` | DELETE | JWT | Delete quote and line items | Empty data `{ data: [] }` |
| `/v1/business/:bid/quote/:qid/pdf` | GET | JWT | Download quote as PDF | Binary PDF |
| `/v1/public/quote/:token` | GET | None (token) | Customer views quote | Public quote response |
| `/v1/public/quote/:token/respond` | POST | None (token) | Customer accepts/rejects | Confirmation |

## Suggested Build Order

Build order based on dependency analysis:

```
Phase 1: Quote Deletion (no external deps)
    |-- QuoteDeleter service
    |-- DELETE endpoint on QuoteController
    |-- DeleteQuoteDialog + QuoteActionStrip changes (frontend)
    |-- RTK Query deleteQuote mutation

Phase 2: Token Infrastructure + Public API
    |-- quote_tokens collection + QuoteTokenRepository
    |-- QuoteTokenService (generate, validate, revoke)
    |-- PublicQuoteController + PublicQuoteResponse types
    |-- QuotePublicRetriever service
    |-- No frontend yet (test with curl/Postman)

Phase 3: Customer Quote Page (frontend)
    |-- Public route /q/:token in App.tsx
    |-- publicQuoteApi.ts (separate RTK Query slice)
    |-- CustomerQuotePage + PublicQuoteView components
    |-- Accept/Reject UI (no backend wiring yet if Phase 4 not done)

Phase 4: Quote Email Sending
    |-- QuoteSender service (orchestrates token + email + transition)
    |-- POST .../send endpoint on QuoteController
    |-- Email HTML template construction
    |-- SendQuoteDialog + QuoteActionStrip changes (frontend)
    |-- RTK Query sendQuote mutation

Phase 5: Customer Response (Accept/Reject)
    |-- QuotePublicResponder service
    |-- POST .../respond endpoint on PublicQuoteController
    |-- QuoteResponseButtons + confirmation UI (frontend)
    |-- Wire respondToQuote mutation in publicQuoteApi

Phase 6: PDF Generation
    |-- Install Puppeteer
    |-- QuotePdfGenerator service
    |-- GET .../pdf endpoint on QuoteController
    |-- PDF HTML template
    |-- Download button on QuoteDetailPage (frontend)
```

**Rationale for this order:**
1. **Deletion first** -- standalone feature, no deps on other new features, quick win
2. **Token infra before email** -- email needs tokens to generate links, so token system must exist first
3. **Public page before email** -- the page the email links to must exist before sending emails makes sense
4. **Email sending next** -- depends on tokens (Phase 2) and public page (Phase 3)
5. **Customer response after email** -- customers need to receive emails before they can respond
6. **PDF last** -- standalone feature, heaviest dependency (Puppeteer), lowest priority

## Sources

- [NestJS Authorization Documentation](https://docs.nestjs.com/security/authorization) -- guard patterns, public decorator approach
- [URL Protection Through HMAC](https://blog.cyril.email/posts/2025-03-12/url-protection-through-hmac.html) -- HMAC token signing pattern
- [HMAC Verification Tokens](https://rotational.io/blog/hmac-verification-tokens/) -- token-as-capability pattern
- [Puppeteer HTML to PDF](https://blog.risingstack.com/pdf-from-html-node-js-puppeteer/) -- PDF generation with Puppeteer
- [NestJS + Puppeteer PDF Generation](https://medium.com/@mprasad96/from-html-templates-to-well-formatted-pdfs-using-puppeteer-and-nestjs-1263bdff641c) -- NestJS-specific implementation
- [SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates) -- evaluated but not recommended for single template
- [Best HTML to PDF Libraries for Node.js](https://blog.logrocket.com/best-html-pdf-libraries-node-js/) -- library comparison
- Existing codebase patterns (HIGH confidence -- all integration points verified against actual code):
  - `trade-flow-api/src/quote/controllers/quote.controller.ts` -- existing controller structure, response mapping
  - `trade-flow-api/src/quote/enums/quote-transitions.ts` -- existing ALLOWED_TRANSITIONS map (SENT->SENT already allowed for re-send)
  - `trade-flow-api/src/quote/services/quote-transition.service.ts` -- existing transition with timestamp setting
  - `trade-flow-api/src/email/services/email-sender.service.ts` -- existing SendGrid integration
  - `trade-flow-api/src/auth/auth.guard.ts` -- existing JwtAuthGuard with user creation side effects
  - `trade-flow-ui/src/App.tsx` -- existing routing structure with AuthenticatedLayout wrapper

---
*Architecture research for: Trade Flow v1.3 Send Quotes*
*Researched: 2026-03-15*
