# Phase 18: Quote Email Sending - Context

**Gathered:** 2026-03-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Tradesperson can send a quote to their customer via email with one action. Includes: send dialog on quote detail page with editable To/Subject/Message fields pre-filled from a configurable default template, email delivery via SendGrid with a branded HTML email (built with Maizzle + Tailwind CSS), automatic Draft→Sent status transition on successful send, re-send capability for quotes already in Sent status, and default email template configuration in business settings. PDF attachment is NOT part of this phase (Phase 20).

</domain>

<decisions>
## Implementation Decisions

### Send dialog UX
- **D-01:** Full email preview dialog with three editable fields: To (email), Subject (text), Message (rich text with bold/italic/links)
- **D-02:** To field pre-filled from customer record email; Subject pre-filled from default template; Message pre-filled from default template with variables resolved
- **D-03:** If customer has no email on file, dialog opens with empty To field and a warning ("No email on file for this customer") plus a checkbox "Save email to customer record"
- **D-04:** To field is always editable — tradesperson can send to a different email than the one on file (e.g., customer's work email)
- **D-05:** Auto-focus lands on the message body textarea when dialog opens (most likely field to edit)
- **D-06:** After successful send: success toast ("Quote sent to john@example.com"), dialog closes, quote status badge updates to Sent, sentAt timestamp appears

### Email template system
- **D-07:** Default email template configured in Business Settings — new "Quote Email" section with default subject line and default message body
- **D-08:** Template variables: `{{customerName}}`, `{{quoteNumber}}`, `{{jobTitle}}`, `{{businessName}}`, `{{userName}}` — resolved when the send dialog opens
- **D-09:** If business has not configured a template, use a system-provided default (e.g., "Hi {{customerName}}, please find your quote {{quoteNumber}} for {{jobTitle}}. Kind regards, {{userName}}")

### Customer-facing email design
- **D-10:** Branded card layout: business name header, tradesperson's personal message, prominent "View Quote Online" CTA button, quote summary (number, date, total inc. tax), small "Sent via Trade Flow" footer
- **D-11:** HTML email built with Maizzle (npm package) using Tailwind CSS — templates live in trade-flow-api in a dedicated templates/ directory
- **D-12:** "View Quote Online" button links to the existing customer quote page at `/quote/view/{token}` (from Phase 17)

### Re-send behavior
- **D-13:** Re-send uses the same dialog as first send, pre-filled fresh from the default template (not the previous send's message)
- **D-14:** Re-send generates a new token — previous tokens remain active until their 30-day expiry (customer may have bookmarked the link)
- **D-15:** Status stays Sent on re-send (SENT→SENT transition already supported)

### Status automation
- **D-16:** Quote status transitions from Draft to Sent automatically when email is successfully sent (AUTO-01)
- **D-17:** Transition happens server-side after confirmed SendGrid delivery — not optimistically on the client

### Claude's Discretion
- Maizzle project structure and build configuration within trade-flow-api
- Rich text editor library choice for the message body in the send dialog
- Email template variable resolution approach (server-side vs client-side for preview)
- SendGrid dynamic template vs server-rendered HTML approach
- Quote email sender service class structure
- Business settings UI layout for the template configuration section
- Error handling for SendGrid failures (retry, user feedback)

</decisions>

<specifics>
## Specific Ideas

- Use Maizzle npm package to create the email layout and styling — it supports Tailwind CSS which aligns with the frontend design system
- Email should look professional enough that a tradesperson's customer takes it seriously — these are business quotes, not marketing emails
- The "View Quote Online" button is the primary CTA — make it prominent

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Email infrastructure (existing)
- `trade-flow-api/src/email/email.module.ts` — Email module definition, exports EmailSenderService
- `trade-flow-api/src/email/services/email-sender.service.ts` — SendGrid integration with sgMail, sendEmail(SendEmailDto) method
- `trade-flow-api/src/email/interfaces/send-email.dto.ts` — SendEmailDto interface (to, from, subject, html, senderName)

### Quote system
- `trade-flow-api/src/quote/controllers/quote.controller.ts` — Existing quote endpoints, transition endpoint at POST /v1/business/:businessId/quote/:quoteId/transition
- `trade-flow-api/src/quote/services/quote-transition.service.ts` — Status transition logic, sets sentAt timestamp on DRAFT→SENT
- `trade-flow-api/src/quote/enums/quote-transitions.ts` — Transition rules: DRAFT→SENT, SENT→SENT (re-send) already defined
- `trade-flow-api/src/quote/enums/quote-status.enum.ts` — QuoteStatus enum (DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED, DELETED)
- `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts` — IQuoteDto with fields needed for template variables

### Token infrastructure (Phase 16)
- `trade-flow-api/src/quote-token/services/quote-token-creator.service.ts` — create(quoteId, recipientEmail) generates 32-byte base64url token, 30-day expiry
- `trade-flow-api/src/quote-token/entities/quote-token.entity.ts` — Token entity with sentAt, recipientEmail fields

### Business entity (needs extension for template fields)
- `trade-flow-api/src/business/entities/business.entity.ts` — Business entity (name, country, currency) — needs quoteEmailSubject and quoteEmailBody fields
- `trade-flow-api/src/business/data-transfer-objects/business.dto.ts` — Business DTO to extend

### Customer data
- `trade-flow-api/src/customer/data-transfer-objects/customer.dto.ts` — Customer DTO with email field (string | null)

### Frontend patterns
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` — Where Send/Re-send buttons live, existing dialog patterns
- `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` — Dialog pattern reference (shadcn Dialog + form)
- `trade-flow-ui/src/features/quotes/api/quoteApi.ts` — RTK Query endpoints, mutation pattern for new send endpoint
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` — Quote detail page that renders QuoteActionStrip

### Architecture
- `.planning/codebase/CONVENTIONS.md` — Naming patterns, module structure, service patterns
- `.planning/codebase/ARCHITECTURE.md` — Layered architecture, repository pattern

### Requirements
- `.planning/REQUIREMENTS.md` — DLVR-01, DLVR-02, DLVR-03, DLVR-04, AUTO-01

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `EmailSenderService`: Ready-to-use SendGrid wrapper — needs extending for HTML email with Maizzle-rendered templates
- `QuoteTokenCreator.create(quoteId, recipientEmail)`: Generates secure tokens for "View Quote Online" links
- `QuoteTransitionService`: Handles DRAFT→SENT transition with sentAt timestamp — send flow calls this after email delivery
- `CreateQuoteDialog` pattern: shadcn Dialog with form state management — follow for SendQuoteDialog
- `QuoteActionStrip`: Already has Send/Re-send button slots for Draft and Sent states
- `quoteApi.ts` RTK Query: Add new `sendQuote` mutation endpoint

### Established Patterns
- Controller → Service → Repository layering
- Feature-based modules: `src/features/quotes/` for UI, `src/quote/` for API
- RTK Query mutations with optimistic updates and cache invalidation
- Toast notifications via sonner for success/error feedback
- shadcn Dialog component for modal interactions

### Integration Points
- New API endpoint: `POST /v1/business/:businessId/quote/:quoteId/send` — accepts { to, subject, message }
- Backend: QuoteEmailSender service orchestrates: validate → generate token → render email → send via SendGrid → transition status
- Frontend: SendQuoteDialog component opened from QuoteActionStrip Send/Re-send buttons
- Business settings: New "Quote Email" section for template defaults — extends existing business settings page
- Customer update: Optional save-back of email address if entered in send dialog

</code_context>

<deferred>
## Deferred Ideas

- Store sent message per send event for re-send pre-fill — adds complexity, template pre-fill is sufficient for now
- Email delivery status tracking (bounced, opened) — requires SendGrid webhook infrastructure
- Multiple email templates per business — sole tradespeople need one good default
- CC/BCC fields on send dialog — over-engineering for current use case
- "Powered by Trade Flow" link in email footer — future branding consideration

</deferred>

---

*Phase: 18-quote-email-sending*
*Context gathered: 2026-03-21*
