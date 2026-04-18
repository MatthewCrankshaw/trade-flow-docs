# Phase 47: Convert to Quote & Mark as Lost - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-13
**Phase:** 47-convert-to-quote-mark-as-lost
**Areas discussed:** Convert UX flow, Locked estimate UI, Mark as Lost modal, Idempotency key origin

---

## Convert UX Flow

| Option | Description | Selected |
|--------|-------------|----------|
| POST first, then navigate | Frontend calls POST, gets quoteId back, navigates to /quotes/{id}/edit | ✓ |
| Form-first, then POST | Pre-filled review page/modal before firing POST | |
| Confirmation dialog only | Simple "Are you sure?" then POST + navigate | |

**User's choice:** POST first, then navigate

| Option | Description | Selected |
|--------|-------------|----------|
| EstimateActionStrip button | Button in action strip, visible on Sent/Responded | ✓ |
| Status dropdown | Transition dropdown including convert | |
| Kebab/overflow menu | Hidden in ... menu | |

**User's choice:** EstimateActionStrip button

| Option | Description | Selected |
|--------|-------------|----------|
| Button loading state | Spinner on the button itself while POST in-flight | ✓ |
| Full-page loading overlay | Overlay covers detail page | |
| Immediate optimistic navigation | Navigate immediately, resolve after | |

**User's choice:** Button loading state

| Option | Description | Selected |
|--------|-------------|----------|
| Just land on quote edit page | Existing QuoteEditPage = mandatory review | ✓ |
| Dedicated convert review page | Distinct route with explicit Confirm & Save | |
| Edit page with a banner | Normal edit page + "Converted from..." banner | |

**User's choice:** Just land on quote edit page

---

## Locked Estimate UI

| Option | Description | Selected |
|--------|-------------|----------|
| Action strip clears + source link | Buttons removed, "Converted to Quote" chip/link added | ✓ |
| Locked banner + action strip clears | Prominent banner at top | |
| Status badge only | Action strip hides, no extra treatment | |

**User's choice (Converted):** Action strip clears + source link

| Option | Description | Selected |
|--------|-------------|----------|
| Reason card + action strip clears | Buttons removed, reason card shown prominently | ✓ |
| Action strip clears only | Reason visible only in status section | |
| Full locked banner + reason | Dramatic locked banner with reason inline | |

**User's choice (Lost):** Reason card + action strip clears

---

## Mark as Lost Modal

| Option | Description | Selected |
|--------|-------------|----------|
| Action strip button → confirmation dialog | Dialog with reason picker + freeform | ✓ |
| Action strip button → inline expansion | Inline reason picker below action strip | |
| Kebab menu entry → dialog | Hidden in overflow menu | |

**User's choice:** Action strip button → confirmation dialog

| Option | Description | Selected |
|--------|-------------|----------|
| Reason optional | Both structured and freeform optional | ✓ |
| Structured reason required, freeform optional | Must pick one structured reason | |

**User's choice:** Reason optional

---

## Idempotency Key Origin

| Option | Description | Selected |
|--------|-------------|----------|
| Frontend generates UUID at click time | UUID v4 in component state, sent as Idempotency-Key header | ✓ |
| Frontend generates + persists to localStorage | UUID survives page refresh within 24h window | |
| Backend derives key from estimateId | Deterministic server-side key, no frontend involvement | |

**User's choice:** Frontend generates UUID at click time

| Option | Description | Selected |
|--------|-------------|----------|
| Same flow — navigate to quote edit | Treat idempotent response same as first-time 201 | ✓ |
| Toast notification + navigate | Brief toast then navigate | |
| Error state | Show error (not appropriate) | |

**User's choice:** Same flow — navigate to quote edit

---

## Claude's Discretion

- Exact visual styling of the "Converted to Quote" chip/link in the action strip
- Exact visual design of the Lost reason card (color, border, icon)
- RTK Query cache invalidation strategy after convert/mark-lost
- Whether `EstimateToQuoteConverter` calls `QuoteCreator` directly or mirrors its write pattern
- Unit test scope for `EstimateToQuoteConverter` and `EstimateLostMarker`

## Deferred Ideas

None — discussion stayed within phase scope.
