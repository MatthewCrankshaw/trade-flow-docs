# Feature Research

**Domain:** Quote delivery and customer response for trade/contractor business management
**Researched:** 2026-03-15
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Email quote to customer | Every competitor (Tradify, Fergus, Jobber, ServiceM8, Xero) sends quotes via email. This is the core action -- "send the quote" must work. | MEDIUM | Existing `EmailSenderService` with SendGrid already handles transactional emails. Need: HTML email template with quote summary, link to view full quote, PDF attachment. SendGrid recommends link-to-hosted over attachment-only for deliverability, but tradespeople expect a PDF in the email too -- do both. |
| Customer can view quote online (no login) | All competitors use a token-based link -- customer clicks link in email, sees the quote immediately. No account creation, no password. Jobber calls it "client hub", ServiceM8 uses a "branded online quote acceptance portal". Friction kills quote acceptance rates. | MEDIUM | New public (unauthenticated) API endpoint with a secure token. Token stored on quote entity. Customer-facing page is a standalone React route (no app chrome/navigation). Must show: business name/logo, quote number, line items, totals, validity date. |
| Customer can accept quote online | Every competitor has an "Accept" button on the customer view. Fergus, ServiceM8, Jobber, and Xero all support one-click acceptance. This is the whole point -- reduce back-and-forth phone/text/email chasing. | LOW | Button on customer view page hits a public API endpoint that transitions quote from SENT to ACCEPTED. Existing `quote-transition.service.ts` already validates SENT -> ACCEPTED. `acceptedAt` field already exists on entity. |
| Customer can decline quote online | Xero, Jobber, and ServiceM8 all include a decline/reject option alongside accept. Tradespeople need to know a "no" quickly so they can move on. Without it, quotes just go silent. | LOW | Same pattern as accept. SENT -> REJECTED transition already exists in `quote-transitions.ts`. `rejectedAt` field already exists on entity. |
| Tradesperson notified when customer responds | Tradify: "notified instantly". Fergus: "notification as soon as customer approves". ServiceM8: "prompt a notification in your account". Xero: "instantly get notified". Universal expectation across all competitors. | LOW | Send notification email to tradesperson when customer accepts or rejects. Reuse existing `EmailSenderService`. In-app real-time notification is future scope -- email is sufficient for v1.3. |
| Quote PDF generation | Every competitor generates PDF quotes. Tradify, ServiceM8, and Xero all produce professional branded PDFs. Customers expect a document they can save, print, or forward to a partner/spouse for approval. | MEDIUM | Generate PDF server-side. Must include: business details, customer details, quote number/date, line items with quantities/prices, bundle summaries, subtotal/tax/total, validity date, notes. Serve via API endpoint for both email attachment and customer download. |
| Automatic status update from customer action | When customer clicks Accept/Reject on the online view, the quote status must update automatically in the tradesperson's app. Manual status toggling after customer email response defeats the entire purpose of digital delivery. | LOW | Already handled by accept/reject endpoints updating the quote entity. UI refreshes via RTK Query cache invalidation. No new infrastructure needed. |
| Quote deletion | Tradespeople create test quotes, duplicate quotes, or quotes for jobs that fall through. Need to clean up their quote list. Every competitor supports this. | LOW | Existing `DELETED` status pattern established in PROJECT.md ("Soft delete via DELETED status enum"). Only allow deletion from DRAFT status to prevent deleting sent/accepted quotes with audit significance. Exclude DELETED from default list queries. |
| Customizable email message | Tradify and Fergus let you edit the email body before sending. Xero pre-fills but allows editing. Tradespeople want to add a personal note ("Hi Dave, here's the quote for the bathroom reno we discussed"). | LOW | Pre-fill email with template (greeting, quote summary, link) but let tradesperson edit the message body before sending via a send dialog. |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Customer comment on decline | Xero lets customers leave a comment when declining. Knowing WHY a quote was rejected (too expensive, wrong scope, went with competitor) is gold for improving future quotes. Most competitors skip this. | LOW | Optional text field on the decline action. Store as `rejectionReason` on quote entity. Display to tradesperson on quote detail. Small add with outsized value. |
| PDF download from customer view | Let the customer download the PDF directly from the online quote view page. Customers often need to forward to a spouse/partner or print for records. | LOW | Add download button to customer view page that serves the same PDF generated for the email. The PDF endpoint already exists for the email attachment -- just expose it publicly via the same token auth. |
| Re-send quote | Emails get lost, go to spam, or customer asks to see it again. "Send again" button on a quote that is already in SENT status. | LOW | Same send flow, available when status is already SENT. Tradify re-populates all previous recipients. Keep simple: same customer email, re-send with fresh link. The transition map already allows SENT -> SENT. |
| Quote viewed indicator | Know if the customer actually opened the quote vs. ignoring the email. Reduces "did they get it?" anxiety that leads to awkward follow-up calls. | LOW | Record `viewedAt` timestamp when customer loads the quote view page. Show a "Viewed" badge or timestamp on the quote detail in the tradesperson's app. Cheap to implement alongside the customer view page. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Customer account/login system | "Customers should have a portal" | Massive scope. Customers interact with a tradesperson's quote once or twice per job. Nobody wants another login. Jobber's "client hub" works specifically because it does NOT require a password. | Token-based access link with cryptographically secure token. No account, no password, no friction. |
| Quote versioning/options | Fergus and ServiceM8 support multiple quote versions. "Give customer 3 options to choose from." | Significant UI and data model complexity. Sole tradespeople typically send one quote, maybe revise once. Multi-version is a team/enterprise feature. | Edit the quote in draft, re-send. If versioning demand is validated later, add as future milestone. |
| Electronic signature on acceptance | ServiceM8 supports remote signature. "Make it legally binding." | Legal complexity varies by jurisdiction. Adds signature pad UI. A simple Accept click with timestamp and IP address is sufficient evidence for trade quote values (not construction megaprojects). | Record acceptance timestamp, customer email, and IP address as proof of acceptance. Signature is a future add-on. |
| Automated follow-up reminders | ServiceM8 has quote follow-up automation. "Automatically chase customers who haven't responded." | Adds scheduling infrastructure (cron jobs, timers), risk of annoying customers, and configuration UI. Premature for a solo tradesperson tool. | Show "days since sent" on quote list so tradesperson can manually follow up. Automation is a future milestone. |
| Real-time notifications (WebSocket/push) | "I want to know the instant a customer accepts" | WebSocket infrastructure is significant new territory. Email notification covers 95% of the need. Sole tradespeople check their phone periodically, not staring at a dashboard. | Email notification on accept/reject. In-app status updates on next page load via normal RTK Query refetch. |
| Rich HTML quote template builder | "Let me design my own quote layout" | Template builders are entire products themselves (think Canva/Mailchimp). Tradespeople want professional-looking quotes, not a design tool. | One well-designed, clean PDF template with business branding (name, contact info). Template customization limited to what is already in the business entity. |

## Feature Dependencies

```
[Quote PDF Generation]
    +--required by--> [Email Quote to Customer] (PDF attached to email)
    +--required by--> [Customer Quote View] (download button)

[Customer Quote View (public token-based page)]
    +--required by--> [Customer Accept Quote]
    +--required by--> [Customer Decline Quote]
    +--required by--> [Quote Viewed Indicator]

[Customer Accept/Decline]
    +--required by--> [Tradesperson Notification Email]
    +--triggers-----> [Automatic Status Update] (same operation)

[Email Quote to Customer]
    +--requires--> [Quote PDF Generation]
    +--requires--> [Customizable Email Message] (send dialog)
    +--triggers---> [Quote Status: DRAFT -> SENT]
    +--generates--> [Customer View Token] (stored on quote)

[Quote Deletion]
    +--independent--> (no dependencies on other new features)
```

### Dependency Notes

- **PDF generation must come first:** Both the email (attachment) and the customer view (download) depend on having a working PDF. Build this before the email sending flow.
- **Customer view page is the gateway:** Accept/reject buttons live on this page. Must be built before the customer response flow works end-to-end.
- **Quote deletion is fully independent:** No dependencies on other v1.3 features. Good first or parallel task.
- **Notification depends on customer response:** Tradesperson notification emails are triggered by accept/reject actions, so those endpoints must exist first.
- **Email sending triggers status transition:** Sending a quote should automatically transition it from DRAFT to SENT. The transition logic and allowed transitions already exist in `quote-transitions.ts`.
- **Existing infrastructure reduces scope:** `EmailSenderService`, `QuoteTransitionService`, quote entity fields (`sentAt`, `acceptedAt`, `rejectedAt`, `validUntil`) are all already in place.

## MVP Definition

### Launch With (v1.3)

Minimum viable milestone -- what is needed to close the "send quote" loop.

- [ ] Quote PDF generation -- professional PDF with business/customer/line item details
- [ ] Email quote to customer -- SendGrid email with quote summary, view link, and PDF attachment
- [ ] Customizable email message -- editable body text via send dialog before sending
- [ ] Customer quote view (public, no login) -- token-based access to read-only quote page
- [ ] Customer accept button -- one-click acceptance with automatic status update
- [ ] Customer decline button -- one-click rejection with automatic status update
- [ ] Tradesperson notification email -- email sent when customer accepts or rejects
- [ ] Quote deletion -- soft delete for DRAFT quotes
- [ ] PDF download from customer view -- download button on the public page

### Add After Validation (v1.x)

Features to add once core send/respond flow is working and validated.

- [ ] Customer comment on decline -- optional rejection reason field (small scope, high value)
- [ ] Re-send quote -- "send again" for quotes already in SENT status
- [ ] Quote viewed indicator -- record and display when customer opened the quote
- [ ] "Days since sent" display -- show aging on quote list for manual follow-up prioritization

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] Quote versioning/options -- multiple quote versions per job
- [ ] Electronic signature -- signature capture on acceptance
- [ ] Automated follow-up reminders -- scheduled reminder emails for unresponded quotes
- [ ] Customer portal/login -- persistent customer access to all their quotes/invoices
- [ ] SMS quote delivery -- send via text message as well as email
- [ ] Deposit collection on acceptance -- requires invoicing system (not yet built)

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Quote PDF generation | HIGH | MEDIUM | P1 |
| Email quote to customer | HIGH | MEDIUM | P1 |
| Customer quote view (public) | HIGH | MEDIUM | P1 |
| Customer accept/decline | HIGH | LOW | P1 |
| Automatic status update | HIGH | LOW | P1 |
| Tradesperson notification | HIGH | LOW | P1 |
| Customizable email message | MEDIUM | LOW | P1 |
| Quote deletion | MEDIUM | LOW | P1 |
| PDF download from customer view | MEDIUM | LOW | P1 |
| Customer comment on decline | MEDIUM | LOW | P2 |
| Re-send quote | MEDIUM | LOW | P2 |
| Quote viewed indicator | LOW | LOW | P3 |
| Days since sent display | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for v1.3 launch
- P2: Should have, add if time allows within v1.3 or next patch
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Tradify | Fergus | Jobber | ServiceM8 | Xero | Our Approach (v1.3) |
|---------|---------|--------|--------|-----------|------|---------------------|
| Email delivery | Templates with variables | Editable subject/body | Via client hub link | PDF attached to email | With file attachments | Email with editable body + PDF attachment + view link |
| Customer view (no login) | Online approval page | Email link, no account | Client hub, email-only auth | Branded portal via unique URL | Link in email | Token-based public page, no login |
| Accept online | Yes, instant notification | Yes, auto-creates site visit | Yes, with optional deposit | Yes, with optional signature | Yes, one click | One-click accept, auto status update |
| Decline online | Implied | Implied | Yes, can request changes | Implied | Yes, with comment | One-click decline with optional reason |
| Notification | Instant notification | Notification on approval | Email notification | In-app notification | Email notification | Email to tradesperson |
| PDF generation | Yes | Yes | Yes | Yes (template-based) | Yes (branded) | Yes, clean professional template |
| Quote versioning | No | Yes (multiple versions) | No | Yes (versions + options) | No | No (future milestone) |
| Signature | No | No | Yes (on approval) | Yes (remote signature) | No | No (future milestone) |
| Follow-up automation | No | No | Yes (reminders) | Yes (automation add-on) | No (third-party) | No (future milestone) |
| Deposit on accept | No | Yes (auto invoice) | Yes (Jobber Payments) | No | No | No (requires invoicing) |

## Sources

- [Tradify: Email a Quote](https://help.tradifyhq.com/hc/en-us/articles/360034729013-Email-a-Quote)
- [Tradify: Email Templates](https://help.tradifyhq.com/hc/en-us/articles/360015909494-Email-Templates-in-Tradify)
- [Fergus: Quoting Software](https://fergus.com/features/quoting/)
- [Fergus: Quote Versions](https://help.fergus.com/en/articles/10614319-quote-versions)
- [Jobber: Quote Approvals](https://help.getjobber.com/hc/en-us/articles/115012715008-Quote-Approvals)
- [Jobber: Client Hub](https://help.getjobber.com/hc/en-us/articles/1500011237822-What-Do-Your-Clients-See-in-Client-Hub)
- [ServiceM8: How to use quote acceptance](https://support.servicem8.com/hc/en-us/articles/115000282726-How-to-use-a-quote-acceptance)
- [ServiceM8: Send quotes for remote signature](https://support.servicem8.com/hc/en-us/articles/360002067216-How-to-send-quotes-for-remote-signature)
- [Xero: Send Quotes](https://www.xero.com/us/accounting-software/send-quotes/)
- [Xero: Mark quote as accepted or declined](https://central.xero.com/s/article/Mark-a-quote-as-accepted-or-declined)
- [SendGrid: Best Practices for Deliverability](https://support.sendgrid.com/hc/en-us/articles/360041790453-Best-Practices-for-Email-Deliverability)

---
*Feature research for: Quote delivery and customer response in Trade Flow v1.3*
*Researched: 2026-03-15*
