# Phase 45: Public Customer Page & Response Handling - Context

**Gathered:** 2026-04-12
**Status:** Ready for planning

<domain>
## Phase Boundary

A customer opens a secure estimate link (no login required), always sees the latest revision with non-binding language prominent, and responds via one of three conversational actions — triggering a notification email to the trader and an estimate status transition.

Ships in the same phase:
- New `PublicEstimateController` with `GET /v1/public/estimate/:token` (via `DocumentSessionAuthGuard`) returning the latest revision resolved through `rootEstimateId` + `isCurrent: true`.
- New `PublicEstimateRetriever` service resolving the token to the latest revision, setting `firstViewedAt` once, and triggering the Sent → Viewed transition.
- New `EstimateResponseHandler` service processing three response types: `proceed`, `message`, `decline` — each persisting a response object, transitioning status, and sending a trader notification email.
- New `POST /v1/public/estimate/:token/respond` endpoint with type-discriminated request body.
- New `estimate-response.html` Maizzle template for trader notification emails (single template, type-aware conditional sections).
- New public customer page at `/estimate/:token` in `trade-flow-ui` — mirrors the existing `PublicQuotePage` layout with estimate-specific additions (disclaimer card, price range, uncertainty explanation, 3-action response buttons).
- View tracking: `firstViewedAt` set on first token access; subsequent visits are no-ops for tracking.

**CTA model change from original requirements:** The original RESP-01..04 defined four structured buttons (Book a site visit, Send me a quote, I have a question, Not right now). Based on user research into UK homeowner behaviour and competitor analysis (Jobber, Powered Now, Payaca, YourTradebase), this is replaced with a conversational 3-action model. REQUIREMENTS.md RESP-01..04 will need updating to reflect the new CTA definitions. ROADMAP.md success criteria #3 needs rewriting.

**Status simplification:** The original status `SiteVisitRequested` is dropped. Site visits are now just messages captured via "Message [Name]". Two response-triggered statuses remain: `RESPONDED` and `DECLINED`. Phase 41's `EstimateStatus` enum and transition map need a corresponding update (remove `SITE_VISIT_REQUESTED` if already defined, or ensure it is never added).

**Out of scope (explicitly deferred):**
- Quote CTA overhaul (separate phase — see Deferred Ideas).
- Optional e-signature toggle (separate phase — see Deferred Ideas).
- Follow-up scheduling on response (Phase 46).
- Convert-to-quote / mark-as-lost flows (Phase 47).
- Multi-message / back-and-forth conversation through the estimate page.

</domain>

<decisions>
## Implementation Decisions

### Customer Page Presentation (D-PAGE)

- **D-PAGE-01:** 1:1 mirror of `PublicQuotePage` layout with estimate-specific additions. Same responsive single-column card layout. Business header at top, estimate content in the middle, response actions at the bottom.
- **D-PAGE-02:** Non-binding disclaimer appears as a visually distinct bordered card with pale warning/info background, placed immediately below the business header and before estimate content. Matches the email template placement from Phase 44 D-LGL-02 for consistency. Uses the same verbatim copy from SND-05: "This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit."
- **D-PAGE-03:** Price displayed following `displayMode` from the API. If `displayMode` is `"range"`, show `£X – £Y`. If `"from"`, show `From £X`. Uses the same `formatRange` helper from Phase 43. No contingency math performed on the customer page — API provides pre-computed `low` and `high` values.
- **D-PAGE-04:** Contingency percentage is **NEVER** shown to the customer. Showing "15% contingency" invites a negotiation the tradesperson doesn't want. Below the price range, the page displays a "Why is this a range?" section that renders the tradesperson's uncertainty notes (selected during estimate creation in Phase 43) as a human-readable sentence: "The final cost may vary depending on: [condition of existing pipework] · [material prices at time of purchase] · [access requirements to be confirmed on site visit]". If no uncertainty notes were selected, this section does not appear — just the range on its own. No mention of contingency percentages, no mention of the base price as a separate figure.
- **D-PAGE-05:** The customer page displays: scope/description, price range (with "Why is this a range?" when applicable), validity period, trader business info (name, contact details), and the non-binding disclaimer card. Line item breakdown is NOT shown on the customer-facing page — the customer sees the total range, not the per-item build-up. This matches how tradespeople communicate estimates verbally.
- **D-PAGE-06:** Zero non-essential cookies (PECR compliance, CUST-07). No analytics, no tracking pixels, no third-party scripts. The page is a pure server-rendered read + client interaction for the response buttons.

### Response Button UX (D-CTA)

- **D-CTA-01:** Three conversational actions replace the original four structured buttons. This reflects how UK homeowners actually communicate with tradespeople — natural language, not contractual forms.
- **D-CTA-02:** Primary CTA: **"Happy to Proceed"** — a full-width primary button. Carries proceed intent: the customer signals "I want this work done." The trader then decides whether to send a formal quote (Phase 47 convert flow) or book the job directly. Status transitions to `RESPONDED`. Directly below the button: "This is an estimate — the final cost may differ from this approximate figure." This disclaimer is legally prudent and sets customer expectations.
- **D-CTA-03:** Secondary CTA: **"Message [Tradesperson First Name]"** — a full-width secondary/outline button. Opens an inline textarea expansion (no modal). Covers questions, revision requests, site visit requests through a single low-friction channel. Stored as response type `message` with the freeform text. Status transitions to `RESPONDED`. No category picker — just a general message.
- **D-CTA-04:** Tertiary CTA: **"Not right for me"** — a text link (not a button), muted styling. After tapping, the link area expands inline to show a friendly prompt: "Let [Name] know why — it helps them improve." with quick-tap structured reasons:
  - Too expensive
  - Going with someone else
  - Decided not to do the work
  - Just getting an idea of costs
  - Timing isn't right
  Plus an optional freeform textarea ("Anything else?"). Submit button: "Send feedback". Status transitions to `DECLINED`.
- **D-CTA-05:** All inline expansions (message textarea, decline reasons) expand directly below the tapped button/link, pushing other actions down. No modals. Customer stays in context. Matches the existing `PublicQuoteResponseButtons` inline pattern from v1.3.
- **D-CTA-06:** After any response is submitted, the entire action area is replaced with an inline success state: "Thanks! [Name] has been notified and will be in touch." No modal, no redirect. Estimate content remains visible above for reference. Uses optimistic local state pattern (`useState` + `onSuccess` callback) from the existing `PublicQuoteResponseButtons` implementation.
- **D-CTA-07:** One response only. After submitting any response, buttons are permanently replaced with the confirmation state. If the customer revisits the link later, they see the terminal read-only view (D-TERM below). Back-and-forth happens via phone, email, or WhatsApp — not through the estimate page.

### Terminal State Handling (D-TERM)

- **D-TERM-01:** Customer-triggered terminal states (customer previously responded): the estimate content remains fully visible (scope, price, disclaimer) but the response area shows a summary of what they submitted:
  - Proceed: "You told [Name] you're happy to proceed on [date]. He'll be in touch to arrange next steps."
  - Message: "You sent [Name] a message on [date]. He'll be in touch."
  - Decline: "You let [Name] know this estimate isn't right for you on [date]."
  Read-only, no active buttons. Customer can still reference the estimate details.
- **D-TERM-02:** Trader-triggered terminal states (Converted, Expired, Lost): all show the same generic friendly message: "This estimate is no longer available. Contact [Name] if you have any questions." with the trader's contact details (phone, email from business info). No distinction between expired/converted/lost — the customer doesn't need to know the internal reason. Estimate content is NOT shown (the estimate may have been superseded or the terms are no longer valid).
- **D-TERM-03:** Terminal state detection: the public endpoint checks the estimate status. If in a terminal state (`CONVERTED`, `DECLINED`, `EXPIRED`, `LOST`, `DELETED`), return the terminal view. If `RESPONDED` and the response array is non-empty, return the response summary view. If `VIEWED` or `SENT`, return the active view with response buttons.

### Backend Structure (D-API)

- **D-API-01:** Mirror the existing quote public stack file structure with adapted handler logic:
  - `src/estimate/controllers/public-estimate.controller.ts` — `PublicEstimateController`
  - `src/estimate/services/public-estimate-retriever.service.ts` — `PublicEstimateRetriever`
  - `src/estimate/services/estimate-response-handler.service.ts` — `EstimateResponseHandler`
  All within the existing `src/estimate/` module (no separate module for the public stack).
- **D-API-02:** `GET /v1/public/estimate/:token` — guarded by `DocumentSessionAuthGuard`, asserts `documentType === "estimate"`. Calls `PublicEstimateRetriever` which: (1) resolves `rootEstimateId` from the token's `documentId`, (2) finds the latest revision via `findOne({rootEstimateId, isCurrent: true})`, (3) sets `firstViewedAt` on the token if null, (4) triggers Sent → Viewed transition if applicable, (5) returns the estimate with computed price range, uncertainty notes, business info, and response state.
- **D-API-03:** `POST /v1/public/estimate/:token/respond` — guarded by `DocumentSessionAuthGuard`. Request body: `{ type: "proceed" | "message" | "decline", message?: string, reason?: string }`. Calls `EstimateResponseHandler` which: (1) validates the estimate is in a respondable state (VIEWED or SENT — not already terminal), (2) pushes a response object to the estimate's responses array, (3) transitions status to RESPONDED or DECLINED, (4) sends trader notification email, (5) returns success.
- **D-API-04:** Response data stored as an array on the estimate entity: `responses: IEstimateResponse[]` where each entry is `{ type: "proceed" | "message" | "decline", message: string | null, reason: string | null, respondedAt: Date }`. Array (not single object) chosen for future-proofing. With the current one-response-only UX, the array will have at most one entry, but the data model supports evolution without schema migration.
- **D-API-05:** Status simplification: drop `SITE_VISIT_REQUESTED` from `EstimateStatus`. The response-triggered statuses are `RESPONDED` (for proceed and message) and `DECLINED` (for decline). The trader distinguishes intent via `response.type`, not the estimate status. Phase 41's enum and transition map updated accordingly.
- **D-API-06:** Customer-safe response filtering: the public `GET` endpoint returns only customer-relevant fields. Internal fields (businessId, policy data, audit trails) are stripped. Response shape mirrors the existing `PublicQuoteResponse` pattern — a dedicated response class for the public endpoint, not the authenticated detail response.

### Notification Emails (D-NOTIFY)

- **D-NOTIFY-01:** Single Maizzle template `estimate-response.html` with conditional sections based on response type. Renders in `EstimateNotificationEmailRenderer` service.
- **D-NOTIFY-02:** Subject line includes response type: "[Customer Name] responded to Estimate [E-YYYY-NNN] — Happy to Proceed" / "— Message" / "— Declined". Body includes: customer name, estimate reference, response type summary, customer's message or decline reason when applicable, and a "View estimate in Trade Flow" deep link to the authenticated estimate detail page.
- **D-NOTIFY-03:** Notification email is sent failure-tolerant: if email delivery fails, the response is still persisted and the status transition still happens. Log the delivery failure at `AppLogger.error`. The customer already saw the confirmation screen — the response is recorded regardless of email delivery. Mirrors the Pattern from Phase 44 D-AUDIT-08 step 9.

### Claude's Discretion

- Visual hierarchy and spacing of the customer page sections (beyond the structural decisions above)
- Exact icon choices for the response buttons (Lucide icon selection)
- Exact wording of the "Why is this a range?" section heading
- Loading skeleton/spinner pattern while the estimate loads
- Error state presentation (invalid token, network error)
- Exact responsive breakpoints for button layout

### Requirements Updates

Phase 45 implementation requires updates to these planning documents:
- **REQUIREMENTS.md RESP-01..04**: Rewrite to reflect the 3-action conversational model (proceed / message / decline) replacing the original 4-button model (site visit / send quote / question / decline).
- **ROADMAP.md Phase 45 SC #3**: Rewrite to describe the 3-action flow instead of the 4-button flow.
- **ROADMAP.md Phase 45 SC #4**: Update response types from the original set to `proceed | message | decline`.
- **Phase 41 EstimateStatus**: Remove `SITE_VISIT_REQUESTED` if already defined in Phase 41's enum/transition map, or ensure it is never added.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing public quote pattern (v1.3 — primary reference)
- `trade-flow-api/src/quote/controllers/public-quote.controller.ts` — PublicQuoteController endpoint structure
- `trade-flow-api/src/quote/services/public-quote-retriever.service.ts` — Token resolution and view tracking pattern
- `trade-flow-api/src/quote/services/quote-response-handler.service.ts` — Response processing and status transition pattern
- `trade-flow-ui/src/features/quotes/components/PublicQuotePage.tsx` — Customer-facing page layout
- `trade-flow-ui/src/features/quotes/components/PublicQuoteResponseButtons.tsx` — Inline response button pattern with optimistic state

### Phase 41 context (estimate entity and token foundation)
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` — D-TKN (token rename), D-ENT (entity shape including reserved response fields), D-CONT (price range/contingency)

### Phase 44 context (email send flow and token reuse)
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` — D-TKN (token lookup/reuse), D-LGL (legal copy enforcement and disclaimer placement), D-AUDIT (email send audit rows)

### Phase 43 context (frontend estimate patterns)
- `.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md` — D-MIR (features/estimates scaffolding), D-CONT (contingency slider, uncertainty chips, formatRange helper)

### Phase 42 context (revisions)
- `.planning/phases/42-revisions/42-CONTEXT.md` — Revision chain resolution (rootEstimateId + isCurrent), how revisions affect the public page

### Requirements
- `.planning/REQUIREMENTS.md` — CUST-01..07, RESP-01..07 (noting RESP-01..04 need rewriting per D-CTA decisions)

### Project retrospective (v1.3 quote public page lessons)
- `.planning/RETROSPECTIVE.md` — v1.3 Send Quotes section: patterns for token-based public access, Maizzle templates, optimistic UI, notification emails

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **PublicQuotePage**: 1:1 layout reference for the estimate customer page — responsive single-column card layout
- **PublicQuoteResponseButtons**: Inline expansion pattern with optimistic local state (`useState` + `onSuccess`) — direct adaptation for 3-action model
- **DocumentSessionAuthGuard**: Already supports `documentType: "estimate"` (Phase 41 D-TKN-02) — no changes needed for the guard itself
- **formatRange helper** (Phase 43): Renders `£X – £Y` or `From £X` based on displayMode — reuse on customer page
- **Maizzle template tooling**: `estimate-sent.html` template from Phase 44 establishes the build pattern for `estimate-response.html`
- **QuoteResponseHandler**: Response processing pattern (validate respondable state, persist response, transition status, send notification) — adapt for 3 types

### Established Patterns
- **Public controller layering**: Controller → Retriever (read) / ResponseHandler (write) — no direct repository access from controller
- **Customer-safe response filtering**: Dedicated public response class strips internal fields
- **Optimistic UI for responses**: `useState` + `onSuccess` callback pattern for immediate feedback after mutation
- **Failure-tolerant notification**: Email failure doesn't block the response persistence/transition

### Integration Points
- **Route**: New `/estimate/:token` route in `trade-flow-ui` App.tsx (public, no auth wrapper)
- **Controller registration**: `PublicEstimateController` registered in existing `EstimateModule`
- **Token resolution**: `DocumentTokenRepository.findByToken()` → `documentType === "estimate"` → `PublicEstimateRetriever`
- **Status transitions**: New VIEWED → RESPONDED and VIEWED → DECLINED transitions in `EstimateTransitionService`
- **Email rendering**: New `EstimateNotificationEmailRenderer` using Maizzle dynamic import pattern from Phase 44

</code_context>

<specifics>
## Specific Ideas

### CTA Research Context
The conversational CTA model was derived from competitor analysis of Jobber, Powered Now, Payaca, and YourTradebase. Key insight: UK homeowners respond to tradespeople using natural language ("happy to go ahead", "can you come round Tuesday?"), not contractual actions ("Accept", "Request Quote"). The CTA wording must match this natural register.

### "Why is this a range?" Section
The uncertainty explanation is the key trust-building element. Without it, a range of £1,200–£1,320 invites "why can't you just tell me how much it costs?" With context — "final cost depends on condition of existing pipework" — the uncertainty is reframed as a specific, legitimate reason rather than vagueness. This also protects the tradesperson: if the final invoice is at the high end, the customer can reference the upfront explanation. The contingency percentage is purely an internal calculation tool; the customer sees only the resulting range and the human-readable reasons.

### Decline Reason Prompt
"Let [Name] know why — it helps them improve" is friendlier than a blunt "Reject" or "Decline". The structured reasons (Too expensive / Going with someone else / Decided not to do the work / Just getting an idea of costs / Timing isn't right) provide actionable feedback the tradesperson can use to improve their quoting. This follows the Powered Now model.

</specifics>

<deferred>
## Deferred Ideas

### Quote CTA Overhaul (future phase)
Redesign the existing public quote page CTAs based on the same research:
- Primary: "Accept Quote" with brief clarifying text ("By accepting, you agree to the work and price detailed above")
- Secondary: "Request Changes" (Jobber model) — opens message to tradesperson for modifications
- Tertiary: "Decline" as subtle text link with structured rejection reasons (Powered Now model: "This quote isn't right — let [Name] know why")
This is a separate phase touching the already-shipped v1.3 public quote page.

### Optional E-Signature Toggle (future phase)
Tradesperson setting (not default) for quote acceptance. Default to click-to-accept, with optional e-signature for tradespeople who want formal sign-off. Based on Payaca (required for all) vs Jobber/YourTradebase (toggleable) analysis. Trade Flow's "simplest to use" positioning favours click-to-accept as default.

</deferred>

---

*Phase: 45-public-customer-page-response-handling*
*Context gathered: 2026-04-12*
