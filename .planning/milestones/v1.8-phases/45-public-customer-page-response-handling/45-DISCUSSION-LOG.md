# Phase 45: Public Customer Page & Response Handling - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-12
**Phase:** 45-public-customer-page-response-handling
**Areas discussed:** Customer page presentation, Response button UX, Terminal state handling, Backend mirroring strategy

---

## Customer Page Presentation

| Option | Description | Selected |
|--------|-------------|----------|
| 1:1 mirror with additions | Mirror PublicQuotePage layout with estimate-specific sections (contingency explanation, price range, uncertainty notes, disclaimer card). Same responsive single-column card layout. | ✓ |
| Fresh layout for estimates | New layout specific to estimates with different information hierarchy. | |
| You decide | Claude picks. | |

**User's choice:** 1:1 mirror with additions
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Follow displayMode from API | If 'range' show £X–£Y, if 'from' show From £X. Uses formatRange helper from Phase 43. | ✓ |
| Always show range | Always show full range regardless of displayMode. | |
| You decide | Claude picks. | |

**User's choice:** Follow displayMode from API
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Bordered card at top | Visually distinct bordered card with pale warning/info background, below business header, before estimate content. Matches Phase 44 D-LGL-02. | ✓ |
| Banner across full width | Full-width banner at very top of page, above business header. | |
| You decide | Claude picks. | |

**User's choice:** Bordered card at top
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Show explanation | Display contingency explanation near the price range. | |
| Price range only | Just show price range without mentioning contingency. | |
| You decide | Claude picks. | |

**User's choice:** Other — Visible as context, NOT as breakdown (extended free-text response)
**Notes:** Extensive rationale provided: Contingency percentage is NEVER shown to customers (invites unwanted negotiation). Instead, display "Why is this a range?" section using the tradesperson's uncertainty notes rendered as a sentence. If no notes selected, section doesn't appear. No mention of base price as separate figure. The percentage is an internal calculation tool; customers see only the resulting range and human-readable reasons for uncertainty. This builds trust and protects the tradesperson if final cost is at the high end.

---

## Response Button UX

**Pre-discussion input:** User provided extensive research on CTA design for both estimates and quotes, based on competitor analysis (Jobber, Powered Now, Payaca, YourTradebase). This fundamentally changed the response model from the original 4-button RESP-01..04 to a conversational 3-action model.

| Option | Description | Selected |
|--------|-------------|----------|
| Go Ahead = proceed intent | Customer signals "I want this work done." Trader decides next step. | ✓ |
| Send me a quote = convert intent | Customer signals "I want a formal quote." Original RESP-02 meaning. | |
| Both available | Both options plus Message. Four actions total. | |

**User's choice:** Go Ahead = proceed intent
**Notes:** None — recommended option aligned with research.

| Option | Description | Selected |
|--------|-------------|----------|
| General message only | One response type: 'message'. Customer writes whatever they want. | ✓ |
| Categorised message | Customer picks a category first (Question / Site visit / Changes) then writes. | |

**User's choice:** General message only
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Structured reasons | Friendly prompt with quick-tap reasons + optional freeform. Powered Now model. | ✓ |
| Optional freeform only | Just a freeform text box, no structured reasons. | |
| Immediate decline, no prompt | Tapping immediately declines with no reason capture. | |

**User's choice:** Structured reasons
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Happy to Proceed | More polite, very British. Mirrors language UK homeowners actually use. | ✓ |
| Go Ahead | Shorter, more direct. Two words. | |
| You decide | Claude picks at implementation time. | |

**User's choice:** Happy to Proceed
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Inline success state | Buttons area replaced with friendly confirmation. Page stays showing estimate. | ✓ |
| Redirect to thank-you page | Redirect to standalone page. Customer loses estimate reference. | |
| You decide | Claude picks. | |

**User's choice:** Inline success state
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| One response only | After submitting, buttons replaced with confirmation. Revisits show read-only. | ✓ |
| Multiple messages allowed | Customer can revisit and send additional messages. | |

**User's choice:** One response only
**Notes:** Back-and-forth happens via phone/email/WhatsApp, not through the estimate page.

---

## Terminal State Handling

| Option | Description | Selected |
|--------|-------------|----------|
| Estimate + response summary | Full estimate visible, response area shows what they submitted with date. | ✓ |
| Minimal confirmation only | Simple "You've already responded" message. No estimate content. | |
| You decide | Claude picks. | |

**User's choice:** Estimate + response summary
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Generic friendly message | All trader-triggered terminal states show same "no longer available" message. | ✓ |
| State-specific messages | Different messages for expired/converted/lost. | |
| You decide | Claude picks. | |

**User's choice:** Generic friendly message
**Notes:** Customer doesn't need to know the internal reason. Shows trader contact details.

---

## Backend Mirroring Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror structure, adapt handlers | Same file structure as quote public stack, adapted for 3 response types. | ✓ |
| Single respond endpoint with type | Same but explicitly single endpoint. | |
| You decide | Claude picks. | |

**User's choice:** Mirror structure, adapt handlers
**Notes:** None — both options were essentially the same (single respond endpoint is part of the mirror approach).

| Option | Description | Selected |
|--------|-------------|----------|
| Single template, type-aware content | One Maizzle template with conditional sections. Subject includes response type. | ✓ |
| Separate templates per type | Three distinct templates. More control but more maintenance. | |
| You decide | Claude picks. | |

**User's choice:** Single template, type-aware content
**Notes:** None

| Option | Description | Selected |
|--------|-------------|----------|
| Single response object | One response field on entity. Simpler but limits to one response. | |
| Response array | Array of responses. Future-proofs for potential multi-response. | ✓ |

**User's choice:** Response array
**Notes:** Chose array despite one-response-only UX decision. Future-proofs without adding meaningful complexity — array with at most one entry is functionally identical to a single object but doesn't require schema migration if the UX evolves.

| Option | Description | Selected |
|--------|-------------|----------|
| Simplify to 3 statuses | Drop SiteVisitRequested. RESPONDED + DECLINED only. Distinguish via response.type. | ✓ |
| Keep SiteVisitRequested | Retain all three statuses. | |
| You decide | Claude picks. | |

**User's choice:** Simplify to 3 statuses
**Notes:** Trader distinguishes intent via response.type, not the estimate status.

---

## Claude's Discretion

- Visual hierarchy and spacing
- Icon choices for response buttons
- Loading skeleton pattern
- Error state presentation
- Exact responsive breakpoints

## Deferred Ideas

- **Quote CTA overhaul**: Redesign existing public quote page with Accept Quote / Request Changes / Decline with reasons. Separate phase.
- **Optional e-signature toggle**: Tradesperson setting for quote acceptance. Click-to-accept default, optional signature. Separate phase.
