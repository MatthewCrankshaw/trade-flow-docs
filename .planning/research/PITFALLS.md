# Domain Pitfalls: Send Quotes (v1.3)

**Domain:** Quote email delivery, customer response flow, PDF generation, secure tokens for existing NestJS/React trade management system
**Researched:** 2026-03-15

## Critical Pitfalls

Mistakes that cause rewrites, security vulnerabilities, or major user-facing failures.

### Pitfall 1: Customer Email is Nullable -- Send Fails at Runtime

**What goes wrong:** The `ICustomerEntity.email` field is `string | null`. Attempting to send a quote email to a customer with no email address causes a SendGrid API error or, worse, sends to `null`/empty string which SendGrid may accept silently and count against your quota.
**Why it happens:** The quote creation flow does not require a customer email. The "Send Quote" button would appear regardless of whether the customer has an email.
**Consequences:** Runtime errors on send, confused users, wasted SendGrid credits, potential bounce reputation damage.
**Prevention:** Validate customer has a non-null, non-empty email before allowing the Draft-to-Sent transition. Disable/hide the "Send Quote" button in the UI when customer email is missing. Show a clear prompt: "Add an email address for [Customer Name] to send this quote."
**Detection:** Unit test the send service to reject sends when customer email is null. E2E test the send flow with a customer that has no email.

### Pitfall 2: Public Token Endpoint Bypasses Auth but Reuses Auth-Required Services

**What goes wrong:** The customer accept/reject endpoint must be public (no Firebase JWT) since customers are not app users. But the existing `QuoteTransitionService` calls `accessControllerFactory.create(this.quotePolicy)` which requires an `IUserDto` (the authenticated user). Passing `null` or a fake user object causes authorization checks to fail or, worse, silently bypasses business scoping.
**Why it happens:** Every existing service method takes `authUser: IUserDto` as the first parameter. The developer either (a) hacks around this by creating a synthetic user, which breaks the security model, or (b) duplicates the transition logic in a new service, which causes divergence.
**Consequences:** Security holes if a fake "system user" has overly broad permissions. Logic duplication if transition code is copied. Broken transitions if the workaround is fragile.
**Prevention:** Create a dedicated `QuotePublicResponseService` that performs the token-scoped transition directly via the repository, bypassing the policy layer entirely. The token itself IS the authorization (it proves the customer received the quote). This service should ONLY allow accept/reject transitions on quotes in SENT status -- hardcode this constraint rather than reusing the generic transition map. Keep the existing `QuoteTransitionService` untouched for internal (authenticated) status changes.
**Detection:** Code review flag: any public endpoint that constructs a fake `IUserDto`. Integration test: ensure public endpoint cannot transition quotes that are not in SENT status.

### Pitfall 3: Token Security -- Guessable, Never-Expiring, or Logged Tokens

**What goes wrong:** Using predictable tokens (e.g., base64-encoded quoteId, sequential IDs, or short tokens) allows anyone to accept/reject quotes by guessing URLs. Tokens that never expire remain valid forever, even after the quote is superseded. Tokens in URL query parameters get logged in server access logs, browser history, and analytics tools.
**Why it happens:** Developer uses a simple encoding scheme for speed ("just base64 the quote ID"), forgets to add expiry, or puts the token as a query parameter (`?token=abc`) which gets logged everywhere.
**Consequences:** Unauthorized quote acceptance/rejection. Stale tokens allow actions on quotes the tradesperson has already re-sent. Token leakage via logs or referrer headers.
**Prevention:**
- Generate tokens with `crypto.randomBytes(32).toString('hex')` -- 256-bit unguessable tokens.
- Store a hash of the token (not the token itself) in the database alongside the quoteId, expiry, and a `usedAt` timestamp.
- Set token expiry (30 days is reasonable for quotes -- matches typical quote validity).
- Use path-based tokens (`/quote/respond/:token`) not query params (`/quote?token=...`) to reduce logging exposure.
- Mark tokens as used after accept/reject (one-time use).
- When a quote is re-sent, invalidate previous tokens.
**Detection:** Security review: grep for `Buffer.from` or `btoa` on quote IDs. Check that tokens are stored hashed. Verify expiry is enforced in the lookup query.

### Pitfall 4: Race Condition on Quote Status Transition (Double Accept/Reject)

**What goes wrong:** Customer clicks "Accept" twice quickly, or clicks accept while the tradesperson is simultaneously re-sending the quote. Two concurrent requests both read status as SENT, both pass the transition check, and both write. The quote ends up with inconsistent timestamps or the tradesperson's re-send overwrites the customer's acceptance.
**Why it happens:** The current `QuoteTransitionService` does a read-then-write: `findByIdOrFail` followed by `quoteRepository.update`. There is no atomic check-and-set. MongoDB does not provide document-level locking by default.
**Consequences:** Lost status transitions. Quote accepted but sentAt timestamp overwritten. Customer sees "accepted" but tradesperson sees "sent".
**Prevention:** Use MongoDB's atomic `findOneAndUpdate` with a status precondition:
```typescript
// Instead of: read quote, check status, then update
// Do: atomic update with status guard
const result = await this.quoteModel.findOneAndUpdate(
  { _id: quoteId, status: 'sent' },  // precondition
  { $set: { status: 'accepted', acceptedAt: new Date() } },
  { new: true }
);
if (!result) {
  // Either quote not found or status already changed
  throw new ConflictError('Quote has already been responded to');
}
```
This pattern ensures only one transition succeeds. The second request gets a null result and returns a clear error.
**Detection:** Load test: fire 10 concurrent accept requests at the same quote. Verify exactly one succeeds and nine get conflict errors.

### Pitfall 5: SendGrid "From" Address Domain Mismatch

**What goes wrong:** The existing `EmailSenderService` accepts a `from` address via `SendEmailDto`. When sending quote emails, developers use the tradesperson's personal email as the "from" address to make it feel personal. But SendGrid requires the from address domain to match an authenticated sender domain. Emails silently fail or land in spam.
**Why it happens:** Wanting the customer to see "from: mike@mikesplumbing.co.uk" instead of "from: noreply@tradeflow.app". The developer sets the from field to the business owner's email without checking SendGrid domain authentication.
**Consequences:** Emails rejected by SendGrid (403 Forbidden). Emails that do send land in spam because SPF/DKIM do not match. Deliverability reputation damaged.
**Prevention:** Always send from a verified domain (e.g., `quotes@tradeflow.app` or `noreply@tradeflow.app`). Use the `reply-to` header to set the tradesperson's email so customer replies go to the right place. This is the standard pattern for SaaS transactional email.
**Detection:** Integration test: verify the `from` address uses the app domain. Monitor SendGrid activity feed for 403 or bounce events after first deploy.

### Pitfall 6: PDF Money Formatting Mismatch with UI

**What goes wrong:** The PDF shows "$100" but the UI shows "$100.00". Or the PDF shows "100.5" instead of "100.50". Or bundle component line totals in the PDF do not match the expandable rows in the UI because the PDF calculates totals differently.
**Why it happens:** The existing system uses `Money` value objects (Dinero.js) with minor units on the API side. The API response returns `toMajorUnits()` which is a raw number (e.g., `100.5` not `100.50`). The UI formats this with `Intl.NumberFormat` or similar. The PDF generation code does its own formatting without using the same formatter.
**Consequences:** Customer sees different numbers on screen vs PDF. Loss of trust. Potential disputes over quoted amounts.
**Prevention:** Create a shared server-side formatting utility used by the PDF generator. Use `Intl.NumberFormat` with the business's currency and locale. Ensure the PDF generation receives the same `Money` objects (not pre-formatted strings) so rounding is identical. The business entity has `currency` field -- use it. Write snapshot tests comparing PDF output against expected formatted values.
**Detection:** Visual regression test: generate a PDF for a known quote and compare money values against expected formatted output. Test with edge cases: 0.10, 0.01, 1000.00, amounts requiring comma separators.

## Moderate Pitfalls

### Pitfall 7: Puppeteer/Chromium Deployment Headaches

**What goes wrong:** PDF generation works locally but fails in production. Puppeteer downloads a 150MB+ Chromium binary. Docker images balloon in size. Alpine Linux images lack required system libraries (libX11, libatk, etc.). Cloud Functions time out on cold start.
**Prevention:** Use PDFKit or pdfmake instead of Puppeteer for structured quote documents. Quotes are data-driven tabular documents (line items, totals, business info) -- not complex web pages. PDFKit generates PDFs directly from data without a browser, is lightweight (~2MB vs ~150MB), has no cold start penalty, and works in any Node.js environment without system dependencies. Only use Puppeteer if pixel-perfect HTML-to-PDF rendering of complex layouts is required (it is not for quotes).

### Pitfall 8: Email Send Failure Leaves Quote in Limbo Status

**What goes wrong:** The "Send Quote" action transitions the quote to SENT status, then attempts to send the email. The email send fails (SendGrid outage, invalid email, rate limit). The quote is now in SENT status but the customer never received the email. The tradesperson thinks it was sent.
**Prevention:** Send email first, then transition status. If email fails, status stays DRAFT and user gets a clear error. Downside: if the status update fails after email sends, email is sent but status is wrong -- but this is far less bad than the reverse (tradesperson can re-send, and the customer having received the email is the more important truth). Keep it simple for v1.3.

### Pitfall 9: Token Endpoint Exposes Internal Data

**What goes wrong:** The public endpoint that serves the customer-facing quote view returns full quote data including internal fields (businessId, jobId, internal notes, line item IDs, internal statuses). An attacker with a valid token can see business internals.
**Prevention:** Create a dedicated response DTO for the public quote view that includes ONLY what the customer needs: business name, business contact info, quote number, quote date, validity date, line items (description, quantity, unit price, line total), totals, and accept/reject actions. Never expose internal IDs, job details, or internal notes on the public endpoint.

### Pitfall 10: Customer-Facing Page Routing Conflicts with SPA Auth

**What goes wrong:** The customer quote response page (e.g., `/quote/respond/:token`) lives in the same React SPA as the authenticated tradesperson app. The React router renders the full app shell (nav bar, sidebar) before realizing this is a public page. Or worse, the auth check redirects the customer to the login page.
**Prevention:** Add a `/public/*` route prefix that renders a minimal layout (no nav, no auth check). The React router checks for `/public/` prefix before applying auth guards. Use a `PublicLayout` component with no auth wrapper -- distinct from the authenticated `AppLayout`. This is one codebase but two separate layout trees.

### Pitfall 11: Quote Deletion Ignoring Status Guards

**What goes wrong:** Allowing deletion of quotes in any status. A tradesperson accidentally deletes an ACCEPTED quote, losing the agreement record. Or deletes a SENT quote while the customer is reviewing it.
**Prevention:** Only allow deletion of DRAFT quotes. SENT/ACCEPTED/REJECTED quotes should not be deletable (they are business records). The existing `QuotePolicy.canDelete` returns `false` for all cases -- this needs to be updated to allow deletion of DRAFT quotes only. The UI should only show the delete button on DRAFT quotes.

### Pitfall 12: Re-Send Creates Duplicate Tokens Without Invalidating Old Ones

**What goes wrong:** Tradesperson sends a quote, customer does not respond, tradesperson edits the quote and sends again. Now two valid tokens exist -- the old one pointing to stale quote data (or the current data, which is different from what was originally sent). Customer uses the old link and accepts a quote with different terms than what they originally received.
**Prevention:** When a quote is re-sent, invalidate ALL previous tokens for that quote before generating a new one. The customer should only ever be able to respond via the most recent send. The transition map already allows SENT -> SENT (re-send). The token invalidation should happen as part of this transition.

### Pitfall 13: SendGrid Dynamic Template Variable Gotcha

**What goes wrong:** Developer builds quote email content as HTML and passes it as a template variable. SendGrid escapes HTML in dynamic template variables -- the customer sees raw `<br>` tags and `<table>` markup instead of formatted content.
**Prevention:** Use SendGrid Dynamic Templates for the email layout. Pass ONLY data values (strings, numbers) as template variables -- never HTML fragments. The template itself (designed in SendGrid's visual editor) handles the HTML structure and cross-client compatibility (Outlook, Gmail, Apple Mail all render differently). Template variables should be: `customerName`, `businessName`, `quoteNumber`, `quoteTotal`, `quoteLinkUrl`, `validUntilDate`. The template renders these into the pre-designed layout.

## Minor Pitfalls

### Pitfall 14: Missing Business Branding Data for PDF

**What goes wrong:** The PDF is generated with quote data but no business address, phone number, or logo. The customer receives a quote with no indication of who sent it beyond the business name.
**Prevention:** The `BusinessEntity` currently has name, country, currency, and trade. It does NOT have address, phone, or logo fields. For v1.3, use what is available: business name + tradesperson email (from user record) as the "from" info on the PDF. Accept minimal branding for v1.3 and plan a "business profile" enhancement as a future milestone. Do not block PDF generation on adding new fields to the business entity.

### Pitfall 15: Missing Customer Email Validation on Save

**What goes wrong:** Customer email field contains a malformed address (e.g., "john@" or "not-an-email"). SendGrid rejects it or sends to the malformed address, causing a hard bounce that damages sender reputation.
**Prevention:** Validate email format when saving customer records (verify this exists). Additionally validate before sending -- SendGrid errors on malformed addresses should be caught and surfaced as user-friendly errors ("Invalid email address for this customer").

### Pitfall 16: Two-Repo Coordination -- Feature Branch Drift

**What goes wrong:** API and UI feature branches get out of sync. API ships the send endpoint but the UI branch is not ready. Or the UI calls an endpoint signature that changed in a later API commit. Integration testing catches this late.
**Prevention:** Define the API contract (endpoint paths, request/response shapes) before coding either side. Document in the planning docs. Deploy API changes first (backward compatible), then UI changes. Test against the deployed API, not mocked responses, before merging UI changes.

### Pitfall 17: Accept/Reject Page Has No Feedback After Action

**What goes wrong:** Customer clicks "Accept" on the public quote page. The request succeeds, but the page does not update. Customer clicks again (creating race condition, see Pitfall 4). Or the page shows a generic "success" with no context -- customer is unsure what just happened.
**Prevention:** After a successful accept/reject action: (a) disable both buttons immediately on click (optimistic disable), (b) show a clear confirmation state: "You have accepted this quote. [Business Name] has been notified." (c) On page load, if the token is already used or the quote is already accepted/rejected, show the current state rather than the action buttons. The public endpoint should return the quote status so the page can render appropriately for already-responded quotes.

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Email delivery (SendGrid) | From address domain mismatch (#5), customer email null (#1), send failure status limbo (#8), template variable escaping (#13) | Validate customer email before send, use app domain + reply-to, send-then-transition, data-only template variables |
| Token generation and security | Guessable tokens (#3), never-expiring tokens (#3), re-send stale tokens (#12) | crypto.randomBytes, stored hashed with expiry, invalidate on re-send |
| Public customer endpoint | Auth bypass breaks services (#2), data over-exposure (#9), SPA routing conflict (#10), no post-action feedback (#17) | Dedicated public service, scoped response DTO, PublicLayout route group, clear confirmation states |
| Quote status transitions | Race condition double-accept (#4), email failure status mismatch (#8) | Atomic findOneAndUpdate with status precondition, send-first pattern |
| PDF generation | Money formatting mismatch (#6), Puppeteer deployment (#7), missing business info (#14) | Shared formatters, use PDFKit not Puppeteer, accept minimal branding for v1.3 |
| Quote deletion | Deleting non-draft quotes (#11) | Status guard: only DRAFT quotes deletable |
| Two-repo coordination | Feature branch drift (#16) | Define API contract first, deploy API before UI |

## Summary: Top 5 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Public endpoint reuses auth-required services (#2) | HIGH | Security hole or authorization failure | Dedicated QuotePublicResponseService with token-as-auth |
| Race condition on double accept/reject (#4) | HIGH | Inconsistent quote state; lost transitions | Atomic findOneAndUpdate with status precondition |
| Guessable or never-expiring tokens (#3) | HIGH | Unauthorized quote manipulation | crypto.randomBytes, hashed storage, expiry, one-time use |
| Email send failure leaves quote in SENT status (#8) | MEDIUM | Tradesperson thinks quote was sent; customer never received it | Send email first, then transition status |
| PDF money formatting mismatch with UI (#6) | MEDIUM | Customer distrust; disputes over quoted amounts | Shared formatting utility; snapshot tests |

## Sources

- Trade Flow codebase analysis: `quote-transitions.ts`, `quote-transition.service.ts`, `auth.guard.ts`, `quote.policy.ts`, `email-sender.service.ts`, `customer.entity.ts`, `business.entity.ts`, `money.value-object.ts`
- [SendGrid Deliverability Best Practices](https://support.sendgrid.com/hc/en-us/articles/360041790453-Best-Practices-for-Email-Deliverability) - HIGH confidence
- [SendGrid HTML Formatting Issues](https://docs.sendgrid.com/ui/sending-email/formatting-html) - HIGH confidence
- [Dynamic Template Dropped Emails](https://support.sendgrid.com/hc/en-us/articles/41862241312155-Dynamic-Template-Issue-Leading-to-Dropped-Emails) - HIGH confidence
- [NestJS Public Route Decorator Pattern](https://dev.to/dannypule/exclude-route-from-nest-js-authgaurd-h0) - MEDIUM confidence
- [MongoDB Atomic Operations for Race Conditions](https://medium.com/tales-from-nimilandia/handling-race-conditions-and-concurrent-resource-updates-in-node-and-mongodb-by-performing-atomic-9f1a902bd5fa) - MEDIUM confidence
- [Token Security Best Practices](https://auth0.com/docs/secure/tokens/token-best-practices) - HIGH confidence
- [Session Token in URL Risks](https://medium.com/@jaycees10000/session-token-in-url-11e85b613010) - MEDIUM confidence
- [PDF Generation in Node.js Tips and Gotchas](https://joyfill.io/blog/integrating-pdf-generation-into-node-js-backends-tips-gotchas) - MEDIUM confidence
- [Puppeteer vs PDFKit Comparison](https://www.leadwithskills.com/blogs/pdf-generation-nodejs-puppeteer-pdfkit) - MEDIUM confidence
- [SendGrid Deliverability 2026](https://www.mailmodo.com/guides/sendgrid-deliverability/) - MEDIUM confidence

---
*Research completed: 2026-03-15*
