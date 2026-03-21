# Phase 19: Customer Response - Context

**Gathered:** 2026-03-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Customer can accept or reject a quote from the public quote page and the tradesperson is immediately notified via email. Depends on Phase 17 (customer quote page) and Phase 18 (email sending infrastructure).

</domain>

<decisions>
## Implementation Decisions

### Accept/Reject UX Flow
- **D-01:** One-click accept — no confirmation dialog, no signature. Customer clicks Accept, transition happens immediately, inline success state shown.
- **D-02:** Decline shows an optional text field for a reason. Customer can submit with or without a reason.
- **D-03:** After responding, buttons are replaced with an inline status banner on the same page ("You accepted this quote" / "You declined this quote"). Quote details remain visible below.
- **D-04:** Accept/Decline buttons appear at the bottom of the quote card, below the totals section. Natural reading flow — customer reads quote top-to-bottom, then decides.

### Notification Email Content
- **D-05:** Simple factual tone — "[Customer] accepted your quote [Q-001] for [Job Title]." Links to the quote in the app.
- **D-06:** If the customer provided a decline reason, include it directly in the notification email body. Tradesperson gets feedback immediately.
- **D-07:** New simple notification HTML template — business branding header, message text, CTA button to view in app. Lighter than the quote email template.

### Response Window Rules
- **D-08:** Expired quotes (validUntil date passed) show "This quote has expired" instead of response buttons. Customer must contact tradesperson directly.
- **D-09:** Response is final — once accepted or declined, no reversal. Customer sees their decision on the page. Must contact tradesperson to change.
- **D-10:** Token stays active after response — customer can revisit the link to see the quote and their response status. Token still expires/revokes on the normal schedule.

### Claude's Discretion
- Button styling and colors (Accept = primary, Decline = secondary/destructive)
- Exact notification email template layout
- Loading states during response submission
- Error handling UX (network failure during response)
- Decline reason textarea placeholder text and character limit

</decisions>

<specifics>
## Specific Ideas

- Accept should feel instant and rewarding — no unnecessary friction to closing deals
- Decline reason is genuinely optional — no guilt-tripping or "are you sure?" patterns
- Notification emails should work well on mobile — tradesperson likely checks email on phone

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Public quote infrastructure (Phase 17)
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` — Existing public controller with QuoteSessionAuthGuard pattern, ThrottlerGuard
- `trade-flow-api/src/quote-token/guards/quote-session-auth.guard.ts` — Token validation, attaches IQuoteTokenDto to request
- `trade-flow-ui/src/pages/PublicQuotePage.tsx` — Public quote page component, already has status banners for accepted/rejected
- `trade-flow-ui/src/features/public-quote/components/PublicQuoteCard.tsx` — Card layout where buttons will be added

### Quote status transitions
- `trade-flow-api/src/quote/enums/quote-status.enum.ts` — ACCEPTED/REJECTED statuses already defined
- `trade-flow-api/src/quote/enums/quote-transitions.ts` — SENT → ACCEPTED/REJECTED transitions already valid
- `trade-flow-api/src/quote/services/quote-transition.service.ts` — Sets acceptedAt/rejectedAt timestamps on transition

### Email sending (Phase 18)
- `trade-flow-api/src/email/services/email-sender.service.ts` — Resend SDK email service
- `trade-flow-api/src/email/services/quote-email-renderer.service.ts` — HTML template renderer pattern (Maizzle inlining)
- `trade-flow-api/src/quote/services/quote-email-sender.service.ts` — Orchestration pattern for email sending

### Quote token
- `trade-flow-api/src/quote-token/data-transfer-objects/quote-token.dto.ts` — Token DTO with quoteId, recipientEmail, expiresAt
- `trade-flow-api/src/quote-token/responses/public-quote.response.ts` — IPublicQuoteResponse with status field

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `QuoteSessionAuthGuard` — Already handles token validation, expiry, revocation. Response endpoints can use the same guard.
- `QuoteTransitionService.transition()` — Already supports SENT → ACCEPTED/REJECTED. Just needs to be called from the new public endpoint.
- `EmailSenderService.sendEmail()` — Resend integration ready for notification emails.
- `QuoteEmailRenderer` pattern — Can create a similar `NotificationEmailRenderer` with a new HTML template.
- Status banners in `PublicQuoteCard.tsx` — Already render accepted/rejected states. Just need to wire the response flow.

### Established Patterns
- Public endpoints use `@Controller("v1/public")` with `ThrottlerGuard` + `QuoteSessionAuthGuard`
- Token-based auth attaches `IQuoteTokenDto` to request (not JWT `IUserDto`)
- Email templates live in `trade-flow-api/src/email/templates/` as HTML files, copied by nest-cli.json assets config
- Public API responses use `IResponse<T>` wrapper via `createResponse()`

### Integration Points
- New public POST endpoint(s) for accept/reject on the existing `PublicQuoteController`
- New notification email template in `src/email/templates/`
- New `NotificationEmailRenderer` or extend `QuoteEmailRenderer`
- RTK Query mutation in frontend for the public response endpoint
- `PublicQuoteCard` needs conditional Accept/Decline buttons when status is SENT
- Quote entity may need a `declineReason` field
- `IPublicQuoteResponse` may need to include `validUntil` for expiry check on frontend

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 19-customer-response*
*Context gathered: 2026-03-21*
