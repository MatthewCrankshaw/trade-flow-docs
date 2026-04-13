# Phase 47: Convert to Quote & Mark as Lost - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Two backend + frontend flows that terminate an estimate chain:

1. **Convert to Quote** — A trader converts a Sent or Responded estimate into a fully independent draft quote via `POST /v1/estimates/:id/convert`. The API pulls from the latest revision, copies line items with literal tax-rate percentages into `quote_line_items`, drops contingency, creates a new quote in Draft status, transitions the estimate to Converted, and writes `convertedToQuoteId`. The UI navigates the trader to the existing quote edit page for mandatory review. Idempotent within 24h via `Idempotency-Key` header.

2. **Mark as Lost** — A trader manually marks an estimate as Lost via `POST /v1/estimates/:id/mark-lost` with an optional structured reason (same taxonomy as customer decline). The API transitions the estimate to Lost, stores the reason, and cancels all pending follow-ups. The UI shows the reason prominently on the estimate detail page.

Both flows cancel pending follow-ups via `IEstimateFollowupCanceller` (Phase 46 implementation) and lock the estimate from further revisions.

</domain>

<decisions>
## Implementation Decisions

### Convert UX Flow (D-CONV)

- **D-CONV-01:** Convert is triggered by a **"Convert to Quote" button** in `EstimateActionStrip`, visible only when estimate status is `SENT` or `RESPONDED`. No other entry point (no status dropdown, no overflow menu).
- **D-CONV-02:** Click flow: (1) generate UUID v4 idempotency key in component state, (2) disable button and show spinner on the button itself (no overlay), (3) call `POST /v1/estimates/:id/convert` with `Idempotency-Key` header, (4) on success navigate to `/quotes/{quoteId}/edit`. The loading state is a button-level spinner only — no page-level loading overlay.
- **D-CONV-03:** The existing `QuoteEditPage` serves as the mandatory review step. No dedicated convert-review route, no extra confirm button, no banner. Landing on the edit page satisfies CONV-03's "mandatory review before saving" requirement. Trader edits if they want, then saves via the normal quote save flow.
- **D-CONV-04:** Idempotent double-submit within 24h returns the same `quoteId`. The UI treats this response identically to a first-time create — navigate to `/quotes/{quoteId}/edit` with no special toast or error. The button is disabled after first click anyway, making accidental double-submit unlikely.
- **D-CONV-05:** The `Idempotency-Key` UUID is generated at click time in component state. No localStorage persistence. If the trader navigates away and the POST already succeeded, they find the converted quote via the normal quote list page.

### Locked Estimate UI (D-LOCK)

- **D-LOCK-01:** **Converted state** — all action buttons (Edit, Revise, Send, Convert to Quote) are removed from `EstimateActionStrip`. A "Converted to Quote" chip or link replaces them, linking directly to the converted quote detail page (using `convertedToQuoteId`). Status badge shows Converted. Estimate content (line items, scope, price range) remains fully visible to the trader for reference.
- **D-LOCK-02:** **Lost state** — all action buttons removed from `EstimateActionStrip`. A reason card appears prominently in the content area (near the top, above line items), showing the structured reason and/or freeform text, and the timestamp. Status badge shows Lost. Estimate content remains visible.
- **D-LOCK-03:** Both locked states render no Edit, no Revise, no Send buttons. The estimate is read-only. No additional full-page locked banner beyond the action strip change and status badge.

### Mark as Lost Modal (D-LOST)

- **D-LOST-01:** Mark as Lost is triggered by a **"Mark as Lost" button** in `EstimateActionStrip`, visible when estimate status is `SENT`, `VIEWED`, or `RESPONDED`. Clicking opens a confirmation dialog.
- **D-LOST-02:** The dialog contains: a brief heading ("Mark estimate as lost?"), the structured reason picker using the same taxonomy as customer decline (same labels verbatim: "Too expensive" / "Going with someone else" / "Decided not to do the work" / "Just getting an idea of costs" / "Timing isn't right"), and an optional freeform textarea ("Anything else?"). Confirm button fires the POST.
- **D-LOST-03:** Both the structured reason and freeform text are **optional**. The trader can submit without selecting any reason. The API accepts `reason: null` and `notes: null`. LOST-01 says "structured reason or freeform text" — both are optional, consistent with how the customer decline reason is optional (Phase 45 D-CTA-04).
- **D-LOST-04:** Lost reason taxonomy is the **same labels verbatim** as customer decline (Phase 45 D-CTA-04) — no relabelling for trader perspective. The trader sees the same reason taxonomy that customers see because these are the real-world reasons that apply from both sides.

### Backend Structure (D-API)

- **D-API-01:** Two new endpoints on the existing `EstimateController` (Phase 41), following the flat endpoint convention (D-TXN-05 from Phase 41 CONTEXT):
  - `POST /v1/estimates/:id/convert` — accepts `Idempotency-Key` header, protected by `JwtAuthGuard`
  - `POST /v1/estimates/:id/mark-lost` — protected by `JwtAuthGuard`
- **D-API-02:** **Convert** — `EstimateToQuoteConverter` service: (1) find latest revision via `isCurrent: true`, (2) read estimate line items from `estimate_line_items`, (3) create a new quote via existing `QuoteCreator` pattern writing line items into `quote_line_items` with literal tax-rate percentages (no contingency), (4) transition estimate to `Converted` via `EstimateTransitionService`, (5) write `convertedToQuoteId` back-link, (6) cancel follow-ups via `IEstimateFollowupCanceller.cancelAllFollowupsForChain(rootEstimateId)`.
- **D-API-03:** **Idempotency key storage** — the `convertedToQuoteId` field on the estimate entity acts as the idempotency check: if already set, return the existing `quoteId` without creating a second quote. Store the raw `Idempotency-Key` header value on a new nullable `convertIdempotencyKey` field on the estimate entity for the 24h window check. Planner verifies the exact storage/lookup strategy against Phase 41 entity shape.
- **D-API-04:** **Mark as Lost** — `EstimateLostMarker` service: (1) validate status allows transition to `Lost`, (2) persist `{ reason: string | null, notes: string | null, markedLostAt: Date }` on the estimate entity, (3) transition to `Lost` via `EstimateTransitionService`, (4) cancel follow-ups via `IEstimateFollowupCanceller.cancelAllFollowupsForChain(rootEstimateId)`.
- **D-API-05:** Both services follow the existing single-responsibility naming convention (EstimateToQuoteConverter, EstimateLostMarker) living under `src/estimate/services/`. No new module — both register within the existing `EstimateModule`.

### Quote Back-Link (D-BACKLINK)

- **D-BACKLINK-01:** CONV-06 ("Converted from E-YYYY-NNN" back-link on the quote detail page) requires a new field on the quote entity: nullable `sourceEstimateId` and nullable `sourceEstimateNumber` (snapshot the number at convert time). The planner should verify whether `sourceEstimateId` was reserved on the quote entity in Phase 41 or if it needs to be added.
- **D-BACKLINK-02:** The "Converted from E-YYYY-NNN" back-link on the quote detail page links to the estimate detail page at `/estimates/{sourceEstimateId}`. Rendered as a small banner or info chip below the quote title, consistent with how other back-references are displayed in the app.

### Claude's Discretion

- Exact visual styling of the "Converted to Quote" chip/link in the action strip
- Exact visual design of the Lost reason card (color, border, icon)
- RTK Query cache invalidation strategy after convert/mark-lost (which tags to invalidate)
- Whether `EstimateToQuoteConverter` calls the existing `QuoteCreator` service or writes directly to the repository (planner decides based on current QuoteCreator interface)
- Unit test scope for `EstimateToQuoteConverter` and `EstimateLostMarker`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 47 Requirements
- `.planning/REQUIREMENTS.md` — CONV-01 through CONV-06, LOST-01, LOST-02

### Prior Phase Decisions
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` §D-ENT — `IEstimateEntity` field inventory (convertedToQuoteId, Lost/Converted reserved fields), D-TXN transition map
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` §D-LI — line-item duplication pattern; `quote_line_items` is untouched, `estimate_line_items` is separate
- `.planning/phases/42-revisions/42-CONTEXT.md` §D-HOOK — `IEstimateFollowupCanceller` interface, `cancelAllFollowups` vs `cancelAllFollowupsForChain`, DI token `ESTIMATE_FOLLOWUP_CANCELLER`
- `.planning/phases/45-public-customer-page-response-handling/45-CONTEXT.md` §D-CTA-04 — decline reason taxonomy (verbatim labels used for Mark as Lost picker)
- `.planning/phases/45-public-customer-page-response-handling/45-CONTEXT.md` §D-TERM-02 — customer page terminal state for Converted/Lost ("no longer available" message)
- `.planning/phases/46-follow-up-queue-automation/46-CONTEXT.md` §D-EXP-04 — `cancelAllFollowups` cancels 4 jobIds; `cancelAllFollowupsForChain` cancels across all revisions

### Existing code patterns (primary references)
- `trade-flow-api/src/quote/` — existing quote module structure for EstimateToQuoteConverter reference
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — Phase 43 stub that Phase 47 extends with Convert and Mark as Lost buttons

### Planning files
- `.planning/ROADMAP.md` §Phase 47 — Goal and success criteria
- `.planning/milestones/v1.8-ROADMAP.md` §Phase 47 — Full success criteria

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **`EstimateActionStrip`** (Phase 43): stub component where Convert and Mark as Lost buttons will be added — Phase 47 extends this directly
- **`IEstimateFollowupCanceller`** (Phase 42): interface with `cancelAllFollowupsForChain(rootEstimateId)` — both Convert and Mark as Lost call this
- **`EstimateTransitionService`** (Phase 41): validates and applies status transitions including → `Converted` and → `Lost`
- **`QuoteCreator`** service (existing quote module): existing pattern for creating quotes — `EstimateToQuoteConverter` either calls this directly or mirrors its approach writing to `quote_line_items`
- **Dialog pattern** (existing UI): shadcn/Radix `Dialog` component used throughout app (e.g., `SendEstimateDialog` from Phase 44) — `MarkAsLostDialog` follows the same pattern

### Established Patterns
- **Action strip buttons**: existing `EstimateActionStrip` stub from Phase 43 is the extension point; buttons appear/hide based on current status
- **RTK Query mutations**: `estimateApi.ts` (Phase 43) is the extension point for `useConvertEstimateMutation` and `useMarkEstimateLostMutation`
- **POST-then-navigate**: standard pattern in this codebase for create flows (matches how quote creation works)

### Integration Points
- `quote_line_items` collection — written by `EstimateToQuoteConverter` via the existing quote line-item stack (no new collection)
- `sourceEstimateId` / `sourceEstimateNumber` fields on quote entity — new nullable fields needed on `IQuoteEntity` / `IQuoteDto` / `IQuoteResponse` (planner verifies whether already reserved)
- `convertedToQuoteId` field on estimate entity — reserved in Phase 41; Phase 47 writes it
- `lostReason` / `lostNotes` / `markedLostAt` fields on estimate entity — reserved as nullable in Phase 41 (D-ENT-01); Phase 47 writes them

</code_context>

<specifics>
## Specific Ideas

- "Convert to Quote" button in `EstimateActionStrip` should only appear on `SENT` or `RESPONDED` estimates
- "Mark as Lost" button should appear on `SENT`, `VIEWED`, and `RESPONDED` estimates
- After convert, the estimate detail page shows a "Converted to Quote" chip/link pointing to `/quotes/{convertedToQuoteId}`
- The lost reason dialog uses the exact same reason labels as the customer decline picker (Phase 45 D-CTA-04): "Too expensive" / "Going with someone else" / "Decided not to do the work" / "Just getting an idea of costs" / "Timing isn't right"
- The lost reason card on the estimate detail page shows both structured reason and freeform notes, with a `markedLostAt` timestamp

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 47-convert-to-quote-mark-as-lost*
*Context gathered: 2026-04-13*
