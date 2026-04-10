# Feature Research: v1.8 Estimates

**Domain:** UK sole-trader SaaS — estimates (pre-site-visit "rough cost" documents) as a parallel document type to quotes, with price ranges, soft customer responses, automated follow-ups, and conversion to quotes.
**Researched:** 2026-04-10
**Confidence:** MEDIUM-HIGH (competitor product behaviour and UK legal framing HIGH; follow-up cadence and contingency percentages MEDIUM — drawn from sales industry and US plumbing sources, adapted for UK trade context)

## Executive Summary

The single most important finding: **every major competitor treats "estimate" as a cosmetic relabel of "quote"**. Tradify's own docs confirm it — switching to Estimate mode just swaps the word "Quote" for "Estimate" in customer-facing output; internally everything is still a quote, with the same fields, same structure, same customer response flow (accept/decline). Jobber, ServiceM8, YourTradebase and Powered Now all follow the same nomenclature-only pattern.

This is a major differentiation opportunity. UK tradespeople genuinely need a separate pre-site-visit document that (a) communicates price uncertainty honestly via ranges or "from £X", (b) legally signals non-binding status, (c) routes customers into a soft response flow ("book a site visit" rather than "accept £X"), and (d) nudges the trader with automated follow-ups because estimates are a lead-generation tool, not a signed agreement. None of the incumbents model estimates this way.

UK legal context reinforces the design: quotes are binding contracts under the Consumer Rights Act 2015; estimates are explicitly non-binding guide prices. Citizens Advice, the Dispute Resolution Ombudsman and Ralli Solicitors all draw the same line. If Trade Flow ships an "estimate" document that behaves like a quote (single price, accept/decline buttons), it exposes users to the same contract risk — defeating the purpose.

The existing v1.3 quote infrastructure (token-based public pages, Resend + Maizzle, view tracking) and v1.4 BullMQ worker (already used for Stripe webhooks) cover ~70% of the plumbing. What's genuinely new is the price-range data model, the soft-response taxonomy, the follow-up scheduler, and the convert-to-quote flow.

## Feature Landscape

### Table Stakes (Users Expect These)

Features a UK sole trader would consider broken if absent. These set the baseline of a credible estimates feature.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Separate document type with distinct numbering | Legal and psychological separation from quotes; users need to tell them apart in lists | LOW | E-YYYY-NNN via new `estimate_counters` collection mirroring `quote_counters` pattern from v1.2 |
| Non-binding language on customer-facing page and email | UK legal distinction — estimates are guide prices, quotes are contracts. Mislabelling exposes the user to unintended binding contracts under CRA 2015 | LOW | One paragraph of disclaimer copy in Maizzle template; include "This is a guide price only and does not form a contract" |
| Line items shared with quotes (same data model) | Users already know how to add items/bundles; would be jarring to have two different editors | LOW | Reuse existing line item schema; `documentType: 'quote' \| 'estimate'` discriminator on parent doc |
| Customer-facing page via token | Direct parity with quotes — customers expect to click a link, see the estimate, respond | LOW | Reuse v1.3 `QuoteSessionAuthGuard` generalised to `DocumentSessionAuthGuard` (or dedicated estimate variant) |
| Email delivery with configurable template | Mirrors the quote send flow users already learned in v1.3 | LOW | Clone QuoteSettings module into EstimateSettings; reuse Resend + Maizzle infra |
| View tracking (firstViewedAt) | Shipped in v1.3 for quotes; users expect the "has customer seen this?" signal everywhere | LOW | Reuse existing tracking logic |
| Status lifecycle | Need to know which estimates are still live vs dead | MEDIUM | Draft → Sent → Viewed → Responded → (Site Visit Requested / Converted / Declined / Expired). Expired = auto-transition after N days with no response |
| Edit and resend | Customers ask questions, traders revise. Without this, every change is a new document | MEDIUM | Versioned revisions (parent_estimate_id + revision_number) under the hood; UI reads as "edit and resend" with a collapsed History section |
| Convert to Quote action | The whole point of an estimate is it becomes a quote after the site visit. Competitors missing this force manual re-keying | MEDIUM | Pulls from latest revision, drops contingency, sets `convertedFromEstimateId`, back-links both ways |
| Decline with reason capture | Quotes already do this; estimate parity expected | LOW | Reuse existing decline-reason text field, extended with the structured taxonomy below |
| Price range OR "From £X" display | Without this, estimates are just quotes with a different word on them (Tradify's mistake) | MEDIUM | Base price + contingency percent → display as "£900 – £1,100" (range) or "From £900" (floor only, for diagnostic/repair jobs) |

### Differentiators (Competitive Advantage)

Features that no UK competitor does well, that genuinely help traders win more work.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| True separate document type (not nomenclature relabel) | Tradify, Jobber, ServiceM8, YourTradebase all treat estimate as a relabel. Trade Flow's estimate has different semantics: price range, soft responses, follow-ups, no binding | MEDIUM | This is the core strategic move. Everything else flows from it |
| Contingency slider with transparent range display | Tradespeople already add 10-25% mental buffers informally. Exposing it as a slider lets them explain uncertainty to the customer without writing prose | LOW | 0-30% in 5% steps, default 10%. Matches the 10-25% construction industry norm from US sources, conservatively adjusted |
| Soft customer response flow (4 buttons, not accept/decline) | The hardest bit of the sales conversation is "£X, yes or no" when the trader hasn't seen the job. Replacing binary accept/decline with "Book a site visit / Send me a quote / I have a question / Not right now" matches real-world dynamics | MEDIUM | Custom button routes on public estimate page; each captures structured data + triggers appropriate tradesperson notification |
| Structured decline reasons | Competitors use free-text. Structured reasons (Too expensive / Going with someone else / Not the right time / Decided against the work / Other) unlock future pipeline analytics milestone | LOW | Enum on `estimate.declineReason`; see Reason Taxonomy section below |
| Quick-tap uncertainty notes | One-tap reasons for contingency ("Pipework behind walls", "Access unclear", "Material spec TBD") tell the customer *why* the range exists — more persuasive than a blank range | LOW | Chip selector in create dialog, persisted as string array; rendered as bullet list on customer page |
| Automated follow-up sequence | Yesware/Apollo data shows 50% of responses come from the first follow-up; 80% of sales need 5+ touches. Competitors leave this to the trader's willpower, which means it never happens | MEDIUM | Three BullMQ delayed jobs at 3/10/21 days (see Cadence Evidence below). Cancel on response/revision/convert. Reset on revision |
| Invisible versioning with visible history | "Edit and resend" UX with a collapsed audit trail. Users don't want to manage "v2" explicitly — they want the document to stay correct. But they occasionally need to prove what they sent last Tuesday | MEDIUM | parent_estimate_id + revision_number on the entity, but list view only shows latest; collapsed History section in detail view |
| "Book a site visit" with pre-populated availability prompt | Not a full calendar integration. A lightweight "Here are the next 5 working days, what suits?" prompt the customer replies to, captured as a site visit request the trader manually schedules | LOW-MEDIUM | Customer sends 1-3 preferred windows as free text or date-picker. Creates a SiteVisitRequest row; trader confirms via existing v1.0 schedule entries. **Avoids a full calendar-sync build** |
| Back-link on converted quote | When a quote was born from an estimate, the quote detail page shows "Converted from E-2026-0042". Context preserved for the trader months later | LOW | One extra field on Quote entity + a link in the UI |

### Anti-Features (Commonly Requested, Often Problematic)

Things that sound reasonable but would bloat scope, confuse the data model, or clone the worst patterns from competitors.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Just call the existing Quote feature "Estimate" when user chooses | Simplest possible implementation; matches what Tradify/Jobber do | Defeats the purpose — same semantics, same contract risk, same accept/decline flow. Users will say "why is this different?" and churn | Ship estimates as a true parallel type with the soft-response flow |
| Calendar integration for site visit booking (Google/Outlook sync) | "Customer should pick a slot directly from my calendar" | Two-way calendar sync is a multi-milestone feature with OAuth, timezone, edge cases, stale-slot problems. BUILT Booking, Nabooki and Tradease are *entire products* doing just this | Capture the customer's preferred windows as a message; trader creates the schedule entry manually via the existing v1.0 job-page schedule UI |
| Visible version numbers (E-2026-0042 v2, v3) | Audit trail feels professional | Cognitive overhead for the trader; customers get confused seeing "v2" on a document and ask "what was v1?". Creates version-management UX that nobody wants | Invisible `revision_number` + collapsed History section; latest revision is always the live document |
| Deposit collection on estimate acceptance | Pattern copied from quote feature discussions | Estimates are explicitly non-binding. Taking a deposit on a non-binding document is legally incoherent and operationally risky (refunds, disputes) | Deposits belong on quotes or invoices, not estimates |
| Electronic signature on estimate | "Makes it feel real" | Doubles down on binding semantics, contradicting the whole point of an estimate. Also legal complexity (eIDAS in UK) | Keep signatures for the future quote feature, not estimates |
| Estimate → Invoice direct conversion | "Skip the quote step for simple jobs" | Invoicing isn't built yet (out of scope per PROJECT.md). Also collapses the quote-as-contract step that legally protects the trader | Convert to Quote, then (future milestone) Quote to Invoice |
| AI-generated contingency suggestions | ServiceM8's AI angle; "pick the right buffer automatically" | Hallucination risk on money. Trader needs to own the number. Training data for UK sole trader jobs doesn't exist at useful granularity | Sensible default (10%) + clear slider; trust the user |
| Email tracking pixel as the primary view signal | Matches the v1.3 quote pattern — already built | UK PECR/GDPR implications if used without disclosure. DMA guidance says pixels need clear notice; ICO has issued warnings. Low risk for B2C transactional but not zero | Continue using the link-click / page-load approach from v1.3 (customer visits the token URL = view event). Document the mechanism in privacy policy. Avoid hidden pixel. **Confirm with PITFALLS.md review** |
| Customer login for estimate responses | "Secure customer portal" | Adds signup friction on the customer side. v1.3 decision was token-based for exactly this reason | Reuse token-based public access |
| Auto-convert estimate to quote when trader marks "site visit done" | "One less click" | The whole point of the site visit is the trader revises the numbers. Auto-conversion with old numbers creates binding contracts the trader doesn't intend | Explicit Convert to Quote action; pre-fill from latest revision but require trader review |
| Rich-text uncertainty notes with formatting toolbar | v1.3 used Tiptap for the email template editor | Overkill for 3-word chips. Slows the create flow. Traders don't want to format on their phone between jobs | Quick-tap chip selector with maybe 6-8 presets + custom text field |

## Feature Dependencies

```
Estimate entity + data model (document type toggle)
    ├──requires──> Quote module (reuse line item schema, bundle handling)
    └──requires──> MongoDB atomic counter pattern (v1.2 quote_counters)

Customer-facing estimate page
    ├──requires──> Token infra (v1.3)
    └──requires──> Public RTK Query slice (v1.3 publicQuoteApi → publicDocumentApi)

Email send with Maizzle template
    ├──requires──> Resend SDK (v1.3)
    ├──requires──> Maizzle pipeline (v1.3)
    └──requires──> EstimateSettings module (clone QuoteSettings pattern)

Automated follow-up sequence
    ├──requires──> BullMQ worker (v1.4)
    ├──requires──> Delayed job pattern (already used in v1.6 Stripe webhooks)
    └──requires──> Estimate status tracking (for cancellation on response)

Soft response flow (4 buttons)
    ├──requires──> Customer-facing estimate page
    └──enhances──> Structured decline reasons (persists for future BI milestone)

Convert to Quote action
    ├──requires──> Estimate entity
    ├──requires──> Quote module (v1.2)
    └──requires──> Versioned revisions (pull from latest)

Versioned revisions (invisible)
    └──requires──> Estimate entity with parent_estimate_id self-reference

Site visit request (no calendar sync)
    ├──requires──> Customer-facing estimate page
    ├──enhances──> v1.0 Schedule entries (trader creates manually from request)
    └──conflicts──> Full calendar integration (anti-feature)
```

### Dependency Notes

- **Estimate entity leverages 70% of quote infrastructure:** Line items, bundles, tax rates, token public access, email rendering, BullMQ — all already built. Genuinely new work is the range/contingency fields, soft-response buttons, follow-up scheduler, convert flow, and decline taxonomy.
- **Follow-up sequence depends on BullMQ delayed jobs:** Same pattern as Stripe webhook processing from v1.6, so the team has done this before. Cancellation on customer response requires storing jobIds on the estimate.
- **Site visit request deliberately avoids calendar sync:** The dependency graph is much smaller if we capture a message-style preference rather than syncing two calendars. Existing v1.0 schedule entries absorb the output.
- **Versioned revisions enable convert-to-quote correctness:** Pull from latest revision, not the original. Without versioning, converting a revised estimate silently uses stale numbers.

## MVP Definition

### Launch With (v1.8)

The minimum coherent estimates feature. Ship less than this and it's just quotes with a new label.

- [x] Document type toggle (Quote / Estimate) on create dialog with shared line-item data model — core differentiation
- [x] Base price + contingency slider (0-30%, default 10%, 5% steps) — data model for the range
- [x] Range / "From £X" display modes on customer page — the honest-uncertainty signal
- [x] Optional uncertainty notes with quick-tap chips (site inspection, pipework, materials, access) — adds persuasion to the range
- [x] Separate E-YYYY-NNN numbering via `estimate_counters` — legal and psychological separation
- [x] Versioned revisions (parent + revision_number) invisible to users with collapsed History section — enables edit-and-resend without "v2" UX baggage
- [x] Customer-facing page via token (reuses v1.3 infra) — table stakes
- [x] Four soft response buttons (Book site visit / Send quote / I have a question / Not right now) with structured decline reasons — the differentiator
- [x] View tracking (firstViewedAt) parity with quotes — table stakes
- [x] Status lifecycle (Draft → Sent → Viewed → Responded → Site Visit Requested / Converted / Declined / Expired) — state machine
- [x] Convert to Quote action (pulls latest revision, drops contingency, back-links both ways) — the pay-off
- [x] Revise Estimate action (resets follow-up sequence) — lifecycle completeness
- [x] Mark as Lost action with structured reason — captures future BI signal
- [x] Automated follow-up sequence at 3 / 10 / 21 days via BullMQ delayed jobs — cadence-backed differentiator
- [x] Estimate email template in Business settings with Maizzle HTML rendering — table stakes
- [x] Non-binding legal language in template and customer page — UK legal safety

All of the above are already listed as v1.8 target features in PROJECT.md. Research confirms the scope is correct.

### Add After Validation (v1.8.x / v1.9)

Features to add once v1.8 is in user hands and validated.

- [ ] Trader-initiated follow-up cadence override (pause, resume, custom intervals) — ship defaults only in v1.8; customize later if users ask
- [ ] Estimate templates / presets ("Bathroom refit starting estimate", "Boiler install starting estimate") — wait for usage to reveal the right presets
- [ ] Customer counter-offer on estimate ("I'm thinking more like £800") — interesting signal but complicates the soft-response flow; validate need first
- [ ] Estimate PDF export — deferred from v1.3 for quotes too; solve once for both documents in a later milestone

### Future Consideration (v2+)

Defer until product-market fit is clear.

- [ ] Reporting on decline reasons (which uncertainty categories are lost most often, which jobs convert best) — feeds off structured data v1.8 captures, but dashboard is its own milestone
- [ ] Calendar sync for "Book a site visit" — only if users demand it and the message-based flow proves insufficient
- [ ] Electronic signatures — only if an enterprise customer asks, and only on quotes
- [ ] Multi-currency estimates — already have GBP infra; revisit for non-UK expansion

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Document type toggle + E-YYYY-NNN numbering | HIGH | LOW | P1 |
| Contingency slider + range / "From £X" display | HIGH | MEDIUM | P1 |
| Non-binding legal language on template + page | HIGH | LOW | P1 |
| Customer page via token (reuse v1.3) | HIGH | LOW | P1 |
| Four-button soft response flow | HIGH | MEDIUM | P1 |
| Structured decline reason taxonomy | MEDIUM | LOW | P1 |
| Automated follow-up sequence (3/10/21d) | HIGH | MEDIUM | P1 |
| Convert to Quote action | HIGH | MEDIUM | P1 |
| Versioned revisions (invisible) + History section | MEDIUM | MEDIUM | P1 |
| Uncertainty notes quick-tap chips | MEDIUM | LOW | P1 |
| View tracking parity | MEDIUM | LOW | P1 |
| Revise Estimate + Mark as Lost actions | MEDIUM | LOW | P1 |
| Estimate email template with Maizzle | MEDIUM | LOW | P1 |
| Estimate PDF export | MEDIUM | MEDIUM | P3 (defer with quotes) |
| Trader follow-up cadence customization | LOW | MEDIUM | P2 |
| Estimate templates/presets | LOW | LOW | P2 (wait for usage data) |
| Calendar sync for site visits | LOW | HIGH | P3 (anti-feature for now) |
| AI contingency suggestions | LOW | HIGH | P3 (anti-feature) |

## Competitor Feature Analysis

| Feature | Tradify | Jobber | ServiceM8 | YourTradebase | Powered Now | Fergus | Trade Flow v1.8 |
|---------|---------|--------|-----------|---------------|-------------|--------|-----------------|
| Estimate as distinct document type | **Nomenclature only** — Document Theme swap replaces "Quote" with "Estimate" in outgoing communications; internally still a quote | Nomenclature — "estimates" and "quotes" are interchangeable in docs | Nomenclature — quoting flow is the same | Listed as "quotes, estimates, and invoices" — separate UI, shared data | Not clearly separated | Not clearly separated ("quotes and estimates" grouped) | **True parallel type** with different semantics |
| Price range / contingency | No | No | No | No | No | No | **Yes (slider)** |
| "From £X" display | No | No | No | No | No | No | **Yes** |
| Soft customer response (not binary accept/decline) | No — customer accepts/declines | No — accept/decline | No — accept/decline | No | No | No — accept/decline | **Yes (4 buttons)** |
| Structured decline reasons | Free text | Free text | Free text | Free text | Unknown | Free text | **Enum taxonomy** |
| Automated follow-up sequence | Manual reminders | Manual reminders | Manual reminders, some automation | Manual | Unknown | Manual | **Automated 3/10/21d** |
| Convert estimate → quote | N/A (same object) | Request → Quote → Job conversion built-in | N/A (same object) | N/A (same object) | N/A | N/A | **Yes (explicit action)** |
| Customer-facing online view | Yes (Tradify online quotes) | Yes | Yes | Yes | Yes | Yes | Yes (reuse v1.3) |
| View tracking | Yes | Yes | Yes | Yes | Unknown | Unknown | Yes (reuse v1.3) |
| Non-binding legal language | User's responsibility | User's responsibility | User's responsibility | User's responsibility | User's responsibility | User's responsibility | **Built into template** |
| Versioned revisions | Limited — recreate or edit | Limited | Limited | Limited | Unknown | Limited | **Invisible revisions + History** |
| Calendar integration for site visits | Yes (schedule) | Yes (calendar) | Yes (online booking) | Basic | Yes | Yes | **No — message-based** |

**Takeaway:** Across every axis that matters to a UK sole trader doing pre-site-visit estimates, competitors either do not differentiate at all (document type, price range, soft responses, follow-ups, legal language) or over-engineer features the solo operator doesn't need (full calendar sync, online booking widgets). Trade Flow has a clear wedge: treat the pre-site-visit stage as a genuinely different workflow, not a word-swap.

## UK Legal Context

UK sources consistently draw the quote/estimate line the same way:

- **Quote = legally binding contract.** Under the Consumer Rights Act 2015 and English common law, a written quote that is accepted by the customer forms a binding agreement. The tradesperson cannot then raise the price unless both parties agree in writing.
- **Estimate = non-binding guide price.** The Dispute Resolution Ombudsman explicitly frames it this way: "an estimate is a rough guide to how much the work will cost. It is not a fixed price and can change."
- **Language matters in disputes.** Ralli Solicitors note that courts look at how the document was worded and presented when deciding if a binding contract formed. Labelling a document "estimate" while listing a single fixed price and an accept button arguably creates a quote regardless of the label.
- **Consumer Rights Act 2015 clarity requirement.** Requires pricing and terms to be clear and fair; both quotes and estimates must not mislead. A "from £X" or range display must not hide material costs the trader already knows about.

### Required legal language (suggested defaults for Trade Flow templates)

On the customer-facing estimate page and in the email body, include at minimum:

> This is an estimate — a guide price for the work described. It is not a fixed quote and does not form a binding contract. The final cost may be higher or lower once we have inspected the work. We will confirm the final price in a formal quote before starting.

This pattern aligns with Citizens Advice and Ombudsman guidance. Trade Flow defaults this copy in the Maizzle template; the trader can edit it but at their own risk. Consider a small inline note in settings warning against removing the legal disclaimer.

## Contingency Percentage Evidence

Research sources (US-dominant but broadly applicable):

- **Construction general:** 10-25% contingency is the recommended industry range, adjusted to the age and condition of the property (Ultimate Calculators plumbing guide).
- **Plumbing specifically:** 10-15% is typical, reaching 25% for older properties or renovations with unknowns (Simpro, Housecall Pro, Bookipi templates).
- **Residential repair:** 5-10% for simpler repair jobs where the scope is mostly visible (FreshBooks).
- **Diagnostic jobs:** A different pricing model entirely — fixed diagnostic fee (e.g., £75-£150 plus first hour) rather than a range, because the uncertainty is "we don't know what's wrong yet" not "we don't know how long it'll take."

**Trade Flow implementation:**

- Slider range: **0-30% in 5% steps** (0% allows the trader to send a plain estimate with no buffer; 30% handles worst-case renovations; 5% steps keep it fast to adjust on mobile).
- Default: **10%** (middle of the plumbing range, safe starting point for most jobs).
- Display modes:
  - **Range** ("£900 – £1,100"): use when base price and upper bound are both meaningful.
  - **"From £X"**: use when the trader genuinely doesn't know the ceiling — diagnostic, fault-finding, investigative work. The lower bound is the minimum; the upper bound is undefined until inspection.
- The trader chooses the display mode per estimate; don't try to infer it.

**Confidence:** MEDIUM. US sources dominate the numbers, and UK sole traders are a tighter market segment than the commercial/residential new-build industry the sources describe. The 10-25% band is likely correct; exact trade-by-trade defaults are not worth pre-setting in v1.8.

## Follow-up Cadence Evidence

Sales cadence research (not trade-specific, but the psychology transfers):

- **31.5% of replies come from the first follow-up** (Yesware, Belkins 2025 study).
- **~50% of sales require 5+ follow-ups**, but ~44% of reps give up after one (multiple sources).
- **Optimal spacing:** 2-3 business days for the first follow-up, then extending to 4-7 days for subsequent touches. Fibonacci-style spacing (Day 1, 3, 10, 17, 31) is a common recommendation.
- **Three-week total window** is the most effective range for B2B sales cadences.

**Trade Flow implementation: 3 / 10 / 21 days** from the Sent date.

- **Day 3** — gentle nudge ("Just checking you saw the estimate"). Matches the 2-3 business day first-follow-up recommendation.
- **Day 10** — value reframe ("Happy to answer questions or book a site visit"). Matches the 7-10 day second touch in Fibonacci-style cadences.
- **Day 21** — final check-in ("If the timing isn't right, no worries — just let me know so I can free up the slot"). Matches the ~3 week window for B2B cadence effectiveness.

After Day 21, auto-transition to **Expired** (configurable default, user can extend). Cancellation of remaining jobs must happen on: customer response (any of the 4 buttons), tradesperson marks as Lost, tradesperson revises (reset cadence from new Sent date), tradesperson converts to Quote.

**Confidence:** MEDIUM. The data is from B2B SaaS sales cadence research, not UK trade-to-homeowner communications. Three-week total and three-touch structure are defensible defaults; adjust based on user feedback post-launch. Avoid over-promising "research-backed" — the research backs the *structure*, not the *exact days*.

## Decline Reason Taxonomy

Based on CRM lost-deal taxonomy research (HubSpot, Zendesk, Petavue) adapted to the trade context:

**Proposed enum for Trade Flow:**

1. **Too expensive** — customer cites price as the reason. Captures price-sensitivity signal.
2. **Going with another trader** — lost to competitor. Captures competitive pressure.
3. **Not the right time** — timing mismatch (moving house, wrong season, budget cycle). Deal is alive later.
4. **Decided not to do the work** — customer abandoned the project itself, not just the trader.
5. **Work out of scope** — trader or customer realised it's not the right fit (specialist work, access issues, out of area).
6. **Other** — free text, for everything else.

Why this taxonomy:

- **Five structured + one "other"** matches the "four or five specific categories" guideline from sales CRM research.
- **Each category maps to a different business action:** "Too expensive" → pricing review; "Going with another trader" → differentiation work; "Not the right time" → re-engagement cadence later; "Decided not to do the work" → no action; "Work out of scope" → process improvement.
- **Avoids overlap** (HubSpot research warns about ambiguous categories making filtering unreliable).
- **Feeds the future reporting milestone** — structured data from day one means the dashboard can be built later without data migration.

Customers see these as four or five short radio-button labels on the "Not right now" flow. Traders see structured reasons in the estimate detail and (future) in the pipeline dashboard.

## Site Visit Booking Approach (Message-Based, Not Calendar Sync)

Research shows every serious online-booking product for trades (BUILT Booking, Nabooki, Tradease, SimplyBook, Acuity) is itself a full product, priced £15-£200/month. Two-way calendar sync with Google/Outlook involves OAuth, timezone handling, stale-slot invalidation, and notification loops. **This is a multi-milestone effort if built properly, and a footgun if built partially.**

**Trade Flow v1.8 approach — capture the request, not the booking:**

- Customer clicks "Book a site visit" on the estimate page.
- Form asks: "When's good for you?" with either free text ("weekday afternoons next week") or a simple multi-select of the next 10 working days.
- Submits a **SiteVisitRequest** row attached to the estimate.
- Trader is notified by email (reuse Resend transactional path).
- Trader manually creates a v1.0 schedule entry on the related job (or creates a new job if the estimate isn't linked yet).
- Estimate status transitions to **Site Visit Requested**.
- Follow-up sequence cancels.

This gives the trader full control (which is what sole operators want — see Powered Now user research in other sources), avoids the calendar-sync rabbit hole, and leverages the already-shipped v1.0 schedule feature. **Users who demand real calendar sync in the future can get it in a dedicated calendar-integration milestone.**

## View Tracking and UK PECR/GDPR Considerations

Existing v1.3 quote feature tracks `firstViewedAt` via the customer visiting the token URL (server-side page-load event). **This is the right approach for v1.8 as well.**

**Why not an email tracking pixel?**

- PECR (the UK Privacy and Electronic Communications Regulations) and UK GDPR together require notice and, in many cases, consent for tracking pixels in emails.
- ICO guidance and DMA Email Council guidance both flag tracking pixels as requiring recipient awareness.
- EU EDPB guidelines go further, treating pixels as requiring informed consent.
- Risk for a sole trader SaaS is low but non-zero; the user's customers are UK consumers, so UK GDPR applies.
- The link-click approach Trade Flow already uses is on stronger legal footing because the customer actively clicks a link — no hidden tracking.

**Recommendation:** Keep the v1.3 pattern unchanged. `firstViewedAt` = timestamp when customer first loads the token URL. Document the mechanism in the privacy policy. **Do not add a hidden email tracking pixel in v1.8.** This is called out in PITFALLS.md as well.

**Confidence:** MEDIUM-HIGH on the regulatory interpretation; HIGH on the recommendation (avoid the risk when the existing approach already works).

## Dependencies on Existing Trade Flow Infrastructure

| Existing Capability | Used For | Source |
|---------------------|----------|--------|
| Line item schema and bundles | Estimate line items (shared data model) | v1.2 |
| Atomic counter collection pattern | `estimate_counters` for E-YYYY-NNN | v1.2 (quote_counters) |
| Token-based public access | Customer-facing estimate page | v1.3 |
| `QuoteSessionAuthGuard` | Generalise to `EstimateSessionAuthGuard` | v1.3 |
| Resend SDK + Maizzle | Estimate email delivery | v1.3 |
| First-view tracking | Estimate view tracking | v1.3 |
| Rich text editor for templates | Estimate email template settings | v1.3 (Tiptap) |
| BullMQ delayed jobs | Follow-up cadence scheduler | v1.4 + v1.6 (Stripe webhooks) |
| Schedule entries on jobs | Output target for site visit requests | v1.0 |
| Trade-specific defaults on business creation | Default estimate email template | v1.0/v1.7 pattern |
| `@SkipSubscriptionCheck` decorator | Public estimate endpoints | v1.7 |

**Net new infrastructure:**

- `estimate` entity with document-type discriminator + contingency fields
- `estimate_counters` atomic counter collection
- `EstimateSettings` module (cloned from QuoteSettings)
- Follow-up cadence BullMQ processor
- `SiteVisitRequest` entity + controller
- Structured decline reason enum
- Maizzle estimate template with non-binding legal copy
- Convert-to-quote service + back-link fields on Quote entity

Approximate split: **70% reuse of existing v1.2-v1.7 infrastructure, 30% net new code.** This is a mid-sized milestone — smaller than v1.7 onboarding, comparable to v1.3 quote sending.

## Sources

### Competitor Analysis
- [Sending an Estimate instead of a Quote — Tradify Help Centre](https://help.tradifyhq.com/hc/en-us/articles/22034309269785-Sending-an-Estimate-instead-of-a-Quote) — confirms Tradify treats estimates as nomenclature-only relabel via Document Themes
- [Create an Estimate — Tradify Help Centre](https://help.tradifyhq.com/hc/en-us/articles/15070931267609-Create-an-Estimate)
- [Converting a Request to a Quote or Job — Jobber Help Center](https://help.getjobber.com/hc/en-us/articles/360056871013-Converting-a-Request-to-a-Quote-or-Job)
- [Quoting for Work — ServiceM8](https://www.servicem8.com/us/articles/quoting-for-work-write-winning-job-estimates)
- [YourTradebase Features — Capterra UK](https://www.capterra.co.uk/software/140632/yourtradebase)
- [Fergus Job Management Software](https://fergus.com/)
- [ServiceM8 vs Tradify vs Jobber comparison — tpsTech](https://tpstech.au/blog/service-scheduling-software-showdown-servicem8-vs-tradify-vs-jobber/)

### UK Legal Context
- [Difference Between a Quote and Estimate: UK Legal Guide — Go Legal AI](https://go-legal.ai/difference-between-a-quote-and-estimate-uk-legal-guide/)
- [Estimates and Quotations — Ralli Solicitors LLP](https://ralli.co.uk/estimates-and-quotations/)
- [What's the Difference Between a Quote and Estimate — MyJobQuote](https://www.myjobquote.co.uk/tradesadvice/difference-between-quote-and-estimate)
- [Dispute Resolution Ombudsman — Estimate vs Quotation](https://www.disputeresolutionombudsman.org/blogs/q-what-is-the-difference-between-and-estimate-and-a-quotation-and-why-is-it-important)
- [Quotes and estimated prices in UK — Admintech](https://admintech.uk/services-and-works/fixed-price-estimated-price-and-fee-quote/)
- [Zoopla — Quotes from tradespeople guide](https://www.zoopla.co.uk/discover/improving-a-home/quotes-from-tradespeople-everything-you-need-to-know/)

### Contingency Percentages
- [Plumbing Cost Calculator — Ultimate Calculators](https://ultimatecalculators.com/calculator/plumbing-cost-calculator/)
- [Plumbing Estimate Template — Simpro](https://www.simprogroup.com/blog/plumbing-estimate-template)
- [2026 Plumbing Price Guide — Housecall Pro](https://www.housecallpro.com/resources/marketing/how-to/how-to-price-plumbing-jobs/)
- [Plumbing Cost Estimator — FreshBooks](https://www.freshbooks.com/hub/estimates/plumbing-estimate)
- [How to Price Plumbing Jobs — Jobber Academy](https://www.getjobber.com/academy/plumbing/how-to-price-a-plumbing-job/)

### Diagnostic Fee Model (for "From £X" use case)
- [Free Estimate vs Diagnostic Fee — ABQ Plumbing](https://www.abqplumb.com/free-estimate-vs-diagnostic-fee/)
- [Perfect Diagnostic Dispatch Fee — ServiceTitan](https://www.servicetitan.com/field-service-management/perfect-diagnostic-fee)
- [London Heating Expert pricing](https://www.londonheating.expert/pricing) — UK example of diagnostic-fee-plus-hourly model

### Follow-up Cadence Research
- [Sales Follow-Up Statistics 2025 — Belkins](https://belkins.io/blog/sales-follow-up-statistics)
- [Sales Follow-Up Statistics — Yesware](https://www.yesware.com/blog/sales-follow-up-statistics/)
- [Email Cadence Best Practices — Outreach](https://www.outreach.ai/resources/blog/email-cadence)
- [Sales Cadence Playbook — Instantly](https://instantly.ai/blog/sales-follow-up-cadence-playbook/)
- [What Is a Sales Follow-Up Email — Apollo](https://www.apollo.io/insights/sales-follow-up-email)

### CRM Lost-Deal Taxonomy
- [What is Closed Lost — DealHub](https://dealhub.io/glossary/closed-lost/)
- [Uncovering 9 Closed Lost Reasons — The Sales Blog](https://www.thesalesblog.com/blog/uncovering-9-closed-lost-reasons-winning-sales-strategies)
- [Creating deal loss reasons — Zendesk](https://support.zendesk.com/hc/en-us/articles/4408828162330-Creating-and-using-deal-loss-reasons)
- [Closed-Lost stage in sales pipeline — HubSpot Community](https://community.hubspot.com/t5/Tips-Tricks-Best-Practices/Closed-lost-stage-in-sales-pipeline/m-p/949918)

### PECR / UK GDPR / Tracking Pixels
- [ICO — Direct marketing guidance](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-direct-marketing-using-electronic-mail/what-else-do-we-need-to-consider/)
- [DMA Email Council on Tracking Pixels](https://dma.org.uk/article/dma-email-council-understanding-email-tracking-pixels)
- [PECR overview — Sprintlaw UK](https://sprintlaw.co.uk/articles/pecr-privacy-and-electronic-communications-in-the-uk/)
- [GDPR Email Marketing UK — Data Protection Network](https://dpnetwork.org.uk/electronic-communications/)

### Calendar / Site Visit Booking (anti-feature research)
- [BUILT Booking](https://www.builtfortrades.co.uk/services/built-booking)
- [Tradease](https://www.tradease.uk)
- [Nabooki trade booking](https://www.nabooki.com/booking-system/commercial-trade-services/)
- [Online Booking System Cost Comparison — Sitethreesixty](https://sitethreesixty.com/us/booking-system-comparison)

### Document Versioning Patterns
- [Document Version Control — Accruent](https://www.accruent.com/resources/blog-posts/document-version-control-101-everything-you-need-know)
- [Construction document versioning — PMWeb](https://pmweb.com/understanding-the-difference-between-versions-and-revisions-in-issued-for-construction-ifc-drawings/)

---
*Feature research for: Trade Flow v1.8 Estimates*
*Researched: 2026-04-10*
*Confidence: MEDIUM-HIGH — Competitor behaviour and UK legal framing HIGH; cadence and contingency percentages MEDIUM (adapted from US/sales sources)*
