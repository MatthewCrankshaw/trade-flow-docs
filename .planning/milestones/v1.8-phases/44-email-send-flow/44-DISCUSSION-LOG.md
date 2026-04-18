# Phase 44: Email & Send Flow - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-11
**Phase:** 44-email-send-flow
**Areas discussed:** Legal copy enforcement, Rendered HTML persistence, Token reuse on re-send, Business > Documents tab restructure

---

## Legal copy enforcement (SND-05, SND-07)

### Q1: How should the non-binding legal disclaimer be made structurally non-removable?

| Option | Description | Selected |
|--------|-------------|----------|
| In Maizzle template (Recommended) | Store the legal block directly in `src/email/templates/estimate-sent.html`, outside any user-editable field. Renderer unconditionally injects it alongside user's subject and custom message. User literally cannot delete it — never in a DB field they can edit. | ✓ |
| Two-field model at render time | Store `legalCopy` as a system-owned constant in code. Renderer concatenates `personalMessage` + `legalCopy` constant. Same effect as A but copy lives in a TS constant instead of HTML template. | |
| Required marker in stored body | DB-stored body must contain `{{legalDisclaimer}}` marker. Validator rejects any save that removes the marker. More fragile — user can still remove surrounding context. | |

**User's choice:** In Maizzle template
**Notes:** Template-owned legal block — never surfaced in any DB field the user can edit.

### Q2: Where does the disclaimer appear visually in the email?

| Option | Description | Selected |
|--------|-------------|----------|
| Top, above personal message (Recommended) | Immediately under the business name header, before the trader's personal message. Highest prominence. Consistent with Phase 45 CUST-03 'prominently at the top'. | ✓ |
| Near the CTA button | Placed directly above or below the 'View Estimate Online' CTA. Medium prominence. | |
| Footer, below summary card | Bottom of the email, after the summary card. Low prominence — risks being treated as fine print. | |

**User's choice:** Top, above personal message
**Notes:** Email and Phase 45 customer page must read coherently — both put the disclaimer at the top.

### Q3: What can the user edit in the Business > Templates tab for the estimate email?

| Option | Description | Selected |
|--------|-------------|----------|
| Subject + personal message (Recommended) | Two fields: `estimateEmailSubject` and `estimateEmailBody`. Same shape as `quote-settings`. Legal disclaimer and summary card owned by template/renderer. | ✓ |
| Subject only | Only subject user-editable; personal message is a fixed system default. Safer but strips expression. | |
| Subject + personal message + signature block | Three fields. Over-engineered. | |

**User's choice:** Subject + personal message

### Q4: How is SND-07 enforced (subject must include 'Estimate')?

| Option | Description | Selected |
|--------|-------------|----------|
| Validator on save + send (Recommended) | `UpdateEstimateSettingsRequest` rejects saves missing 'Estimate' (case-insensitive). Same validator on send endpoint catches ad-hoc edits in SendEstimateDialog. System default subject: `Estimate {{estimateNumber}} for {{jobTitle}}`. | ✓ |
| Validator on save only | Only settings save path enforces. SendEstimateDialog trusted. Would let a trader send an estimate with subject saying 'Quote'. | |
| Prepend at render time | Renderer unconditionally prepends `[Estimate] `. Ugly UX — results in `[Estimate] Quote 2026-001`. | |

**User's choice:** Validator on save + send
**Notes:** Two-point enforcement — both default save and ad-hoc send paths validate.

---

## Rendered HTML persistence (SND-04)

### Q1: Where should the rendered HTML audit artefact be stored on each send?

| Option | Description | Selected |
|--------|-------------|----------|
| New collection estimate_email_sends (Recommended) | One row per send: `{estimateId, revisionNumber, sentAt, to, subject, html, type}`. Keeps estimate docs thin, scales through Phase 46 follow-ups, gives natural audit query surface. New EstimateEmailSendRepository + EstimateEmailSendCreator. | ✓ |
| lastSentEmailHtml field on estimate entity | Single `lastSentEmailHtml: string` field on estimate, overwritten on each send. Simple but prior sends lost, Phase 46 follow-ups need another home. | |
| Array on estimate entity | `sentEmails: Array<...>` on estimate. Full history on one doc. Bloats doc, awkward to query individual sends. | |
| Store on document_tokens entity | Reuses existing infra. Couples audit data to token lifecycle. | |

**User's choice:** New collection estimate_email_sends
**Notes:** Scales to Phase 46 follow-ups cleanly. Thin estimate docs preserved.

### Q2: What fields does the estimate entity itself carry to satisfy 'persisted on the estimate at send time' + SC #6 'lastSentAt'?

| Option | Description | Selected |
|--------|-------------|----------|
| lastSentAt + lastSentTo only (Recommended) | Estimate entity gets `lastSentAt: Date` and `lastSentTo: string`. HTML in estimate_email_sends collection. SC #6 satisfied. Thin entity, denormalized quick-read. | ✓ |
| lastSentAt only | Just `lastSentAt`. Detail page shows age but not recipient without extra query. | |
| Full last-send metadata | Duplicates the send row. Violates single source of truth. | |

**User's choice:** lastSentAt + lastSentTo only

### Q3: How is the stored HTML sanitised/treated for the audit trail?

| Option | Description | Selected |
|--------|-------------|----------|
| Store exactly as sent (Recommended) | Exact HTML passed to Resend stored verbatim. No re-sanitising. SC #5 satisfied literally. | ✓ |
| Strip scripts/iframes before store | Defensive but means stored HTML differs from sent — defeats audit purpose. | |
| Store both pre- and post-Maizzle | Doubles storage, unclear use case. | |

**User's choice:** Store exactly as sent

### Q4: What is the retention policy for estimate_email_sends rows?

| Option | Description | Selected |
|--------|-------------|----------|
| Indefinite, no TTL (Recommended) | Audit artefacts for dispute trails. UK limitation period 6 years. Future phase can add TTL if needed. | ✓ |
| TTL index at 1 year | Bounded storage but kills audit trail for anything older than 12 months. | |
| TTL index at 90 days | Matches estimate lifecycle roughly. Too aggressive for legal audit. | |

**User's choice:** Indefinite, no TTL

---

## Token reuse on re-send (SND-06)

### Q1: How does EstimateEmailSender find the existing token on re-send?

| Option | Description | Selected |
|--------|-------------|----------|
| findActiveByDocumentId lookup (Recommended) | New `DocumentTokenRepository.findActiveByDocumentId(documentId, documentType)`. Service: look up, reuse if found, create if not. One repo method, zero Phase 41 re-open. | ✓ |
| Generate at create-time | Phase 41 creates token on estimate creation. Rejected — reopens Phase 41, wastes tokens on never-sent drafts. | |
| Track tokenId on estimate entity | Add `currentTokenId: ObjectId` denormalized field. Faster lookup but couples entity. | |

**User's choice:** findActiveByDocumentId lookup

### Q2: On re-send, does the token's expiresAt refresh (rolling 30-day window) or stay fixed from first send?

| Option | Description | Selected |
|--------|-------------|----------|
| Refresh to now + 30 days (Recommended) | Each send extends expiresAt. Rationale: trader's intent on re-send is 'this estimate is live again'. Requires new `DocumentTokenRepository.extendExpiry` method. | ✓ |
| Keep original expiry fixed | Expiry pinned to first send. Re-send 25 days in means 5 days left — almost pointless. | |
| Extend only if close to expiry | Refresh only when `expiresAt - now < 7 days`. Clever without clear benefit. | |

**User's choice:** Refresh to now + 30 days

### Q3: When Phase 42 creates a revision, does the token point at the root estimate, specific revision, or current revision?

| Option | Description | Selected |
|--------|-------------|----------|
| Token carries rootEstimateId, resolves to current revision (Recommended) | `document_tokens.documentId` stores `rootEstimateId`. Public controller resolves rootEstimateId → latest isCurrent: true. Re-send after revision reuses same token unchanged. Matches Phase 45 SC #1. | ✓ |
| Token carries specific revision id | Each revision needs new token. Contradicts SND-06 and Phase 45 SC #1. | |
| Token carries specific revision, resolver walks chain | More joins per request, same end result as A. | |

**User's choice:** Token carries rootEstimateId, resolves to current revision

### Q4: Which source statuses does the re-send path accept, and how is 'no new revision' enforced?

| Option | Description | Selected |
|--------|-------------|----------|
| Sent, Viewed, Responded, SiteVisitRequested (Recommended) | Re-send accepts any non-terminal post-Draft status. EstimateEmailSender calls EstimateTransitionService. No revision created because EstimateReviser (Phase 42) is the only path that writes new revision rows. Phase 44 adds VIEWED→SENT, RESPONDED→SENT, SITE_VISIT_REQUESTED→SENT to ALLOWED_TRANSITIONS. | ✓ |
| Sent only | Trader with Viewed estimate can't re-send reminder without revising first. Too restrictive. | |
| Any status except terminal states | Includes weird intermediate states unnecessarily. | |

**User's choice:** Sent, Viewed, Responded, SiteVisitRequested

---

## Business > Documents tab restructure (SND-02)

### Q1: How should the Business > Templates tab be structured in BusinessDetails.tsx?

| Option | Description | Selected |
|--------|-------------|----------|
| Rename tab to 'Templates', stack both forms (Recommended) | Rename `quote-email` tab to `templates`. Stack QuoteEmailSettings + new EstimateEmailSettings vertically. SC #2 literal match. | ✓ |
| Add sibling 'Estimate Email' tab | Keep existing tab, add sibling. Doesn't read as 'Templates tab'. | |
| Rename to 'Templates' with nested sub-tabs | Nested Tabs inside a tab. Inconsistent with existing design system patterns. | |

**User's choice:** Rename tab — BUT with a notable override: name the tab **"Documents"** instead of "Templates".
**Notes:** User's freeform override: "Lets name this Documents instead." CONTEXT.md reflects this — tab value `documents`, label "Documents", doc references (ROADMAP, REQUIREMENTS) updated accordingly.

### Q2: What endpoints does the new estimate-settings module expose?

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror quote-settings 1:1 (Recommended) | `GET/PATCH /v1/business/:businessId/estimate-settings`. Same URL shape, DTO shape, stub-on-empty behaviour. Enforces D-01 separation-over-DRY. | ✓ |
| Flat /v1/estimate-settings without businessId scope | Matches Phase 41's flat /v1/estimates pattern. Diverges from quote-settings convention. | |

**User's choice:** Mirror quote-settings 1:1

### Q3: When is the EstimateSettings document first created for a business?

| Option | Description | Selected |
|--------|-------------|----------|
| Lazy on first GET (Recommended) | GET returns stub if no document. PATCH upserts. Matches quote-settings controller lines 29-38. | |
| Eager on business creation | `BusinessCreator` calls DefaultEstimateSettingsCreator alongside DefaultQuoteSettingsCreator. Every business has row from day one. | ✓ |
| Eager on first estimate send | EstimateEmailSender creates row if missing. Ties creation to real user action. | |

**User's choice:** Eager on business creation
**Notes:** User overrode the "recommended" — this is actually what quote-settings already does (confirmed in `business-creator.service.spec.ts` line 226). Phase 44 amendment to CONTEXT.md reflects the corrected recommendation: mirror the existing eager pattern.

### Q4: How is the Documents tab wired to the two APIs?

| Option | Description | Selected |
|--------|-------------|----------|
| Two independent RTK Query hooks, one component each (Recommended) | `useGetEstimateSettingsQuery` / `useUpdateEstimateSettingsMutation` alongside existing quote-settings hooks. New `EstimateEmailSettings` component mirrors `QuoteEmailSettings` 1:1. | ✓ |
| Single form component with dual API calls | Couples the two templates into one save button. Violates D-MIR. | |
| Two hooks, one combined component | Middle ground but less clean than A. | |

**User's choice:** Two independent RTK Query hooks, one component each

---

## Claude's Discretion

Ares not discussed (deferred to planner / downstream agents):
- Email content / summary card details (price line, contingency note, uncertainty notes presentation)
- Onboarding step parity for estimates — CONTEXT.md D-SND-01 explicitly defers this (no FIRST_ESTIMATE_SENT)
- Endpoint shape — already anchored at `POST /v1/estimates/:id/send` by Phase 41 D-TXN-05 flat-endpoint convention
- SendEstimateDialog / SendEstimateForm specifics — 1:1 mirror of SendQuoteDialog / SendQuoteForm per D-UI-01

## Deferred Ideas

None raised during discussion beyond those captured in CONTEXT.md `<deferred>`.
