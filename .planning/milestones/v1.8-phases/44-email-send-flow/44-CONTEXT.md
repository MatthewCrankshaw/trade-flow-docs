# Phase 44: Email & Send Flow - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning

<domain>
## Phase Boundary

A trader reviews a pre-filled email in a new `SendEstimateDialog`, sends an estimate to a customer via Resend using a secure public `document-token` link, and can re-send — with or without a preceding Phase 42 revision — without regenerating the token and without creating a new revision on the re-send path. Mandatory non-binding legal language is baked into a new Maizzle template (`estimate-sent.html`) that lives outside any user-editable field.

Ships in the same phase:
- New standalone `estimate-settings` module (mirrors `quote-settings` 1:1 — separate module, separate collection, separate API). `quote-settings` is untouched.
- New standalone `estimate_email_sends` collection + `EstimateEmailSendRepository` / `EstimateEmailSendCreator` for the SND-04 audit artefact. Phase 46 follow-ups will append to this same collection.
- New `EstimateEmailRenderer` service + new `estimate-sent.html` Maizzle template in `trade-flow-api/src/email/templates/`.
- New `EstimateEmailSender` service that orchestrates: look-up-or-create token, render, send via Resend, persist HTML row, transition Draft→Sent (or Sent→Sent no-op), update `lastSentAt` / `lastSentTo` on the estimate, and — on the revised-send path — call `cancelAllFollowups` via the Phase 42 `IEstimateFollowupCanceller` binding.
- New `POST /v1/estimates/:id/send` endpoint on the existing Phase 41 `EstimateController` (flat shape, consistent with Phase 41 D-TXN-05 endpoint convention).
- Extensions to `DocumentTokenRepository` (from Phase 41): `findActiveByDocumentId(documentId, documentType)` and `extendExpiry(tokenId, newExpiresAt)` methods. The repository's existing `create` method stays. `documentId` for estimate tokens carries the `rootEstimateId`, not the per-revision id.
- Amendment to Phase 41's `BusinessCreator`: invoke the new `DefaultEstimateSettingsCreatorService.create(businessId)` alongside the existing `DefaultQuoteSettingsCreatorService.create(businessId)` call at business creation time.
- Amendment to Phase 41's `EstimateStatus` transition map: add `VIEWED → SENT`, `RESPONDED → SENT`, `SITE_VISIT_REQUESTED → SENT` as valid re-send transitions (the existing `SENT → SENT` no-op already covers the simplest case). Planner verifies exact rules against Phase 41 D-TXN-02.
- New `SendEstimateDialog` + `SendEstimateForm` components under `features/estimates/components/` (Phase 43 D-MIR-03 reserved these file paths for Phase 44). Mirror `SendQuoteDialog` / `SendQuoteForm` pattern 1:1.
- New `EstimateEmailSettings` UI component under `features/business/components/` (mirrors existing `QuoteEmailSettings` 1:1).
- Rename existing `quote-email` tab in `BusinessDetails.tsx` to `documents` with label **"Documents"**. Stack `QuoteEmailSettings` + new `EstimateEmailSettings` vertically inside the tab body.
- Extend `features/estimates/components/EstimateActionStrip.tsx` (Phase 43 shipped as a stub) with Send / Re-send buttons that open `SendEstimateDialog`.
- Update `openapi.yaml` with the new `estimate-settings` endpoints and the new `POST /v1/estimates/:id/send` endpoint.

**Out of scope (explicitly deferred):**
- Public customer estimate page that resolves the token (Phase 45).
- 4-button customer response flow (Phase 45).
- 3/10/21-day follow-up BullMQ scheduling (Phase 46 — but Phase 46 appends to the `estimate_email_sends` collection Phase 44 creates).
- Convert-to-quote / mark-as-lost flows (Phase 47).
- Any modification to `quote-settings` or the existing `quote-email` send path.
- Legal-review gate on the default template copy — gate is blocking but copy draft lives inside this phase; legal review happens pre-ship as a ROADMAP gate.

**Doc edits this phase must ship (in addition to code):**
- Update `.planning/ROADMAP.md` Phase 44 success criterion #2 wording from "Business > Templates tab" to "Business > Documents tab".
- Update `.planning/milestones/v1.8-ROADMAP.md` Phase 44 success criterion #2 to match.
- Update `.planning/REQUIREMENTS.md` SND-02 wording to match the new "Documents" tab name.

</domain>

<decisions>
## Implementation Decisions

### Legal Copy Enforcement (D-LGL)

- **D-LGL-01:** The non-binding legal disclaimer lives inside `src/email/templates/estimate-sent.html` (the Maizzle template) as a static HTML block. It is never stored in any DB field and never read from `estimate-settings`. The user literally cannot delete it because it does not exist in a field they can edit. SND-05 is satisfied structurally, not by validation.
- **D-LGL-02:** The disclaimer block appears **at the top of the email**, immediately under the business name header and before the trader's personal message. This matches Phase 45 CUST-03 which mandates "prominently at the top" for the customer-facing page — email and customer page stay coherent. The disclaimer is visually distinct (bordered card with a pale warning background, not just a sentence) so it reads as a legal notice rather than body copy.
- **D-LGL-03:** The disclaimer copy for the initial implementation is the exact wording from SND-05:
  > "This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit."
  Planner must ship this verbatim subject to the UK-consumer-law legal review gate. If legal review returns amended wording, update the template and this doc in the same PR. Do not paraphrase.
- **D-LGL-04:** `estimate-settings` exposes exactly two user-editable fields: `estimateEmailSubject?: string` and `estimateEmailBody?: string`. Same shape as `quote-settings`. No `legalCopy` field, no `signature` field, no `disclaimer` field. If a future requirement asks for richer template editing, the new fields are additive, not a replacement.
- **D-LGL-05:** Subject line enforcement for SND-07 ("Estimate" must appear in the subject): both `UpdateEstimateSettingsRequest` (business saves default) and `SendEstimateRequest` (trader edits ad-hoc in SendEstimateDialog) validate that the subject contains the word "Estimate" (case-insensitive) via a class-validator decorator. Validation failure throws `InvalidRequestError(ErrorCodes.ESTIMATE_SUBJECT_MISSING_KEYWORD, "Estimate email subject must contain the word 'Estimate'")` (or similar — planner picks consistent ErrorCode naming). System default subject is `"Estimate {{estimateNumber}} for {{jobTitle}}"`.
- **D-LGL-06:** Template variable resolution (`{{estimateNumber}}`, `{{jobTitle}}`, `{{customerName}}`, `{{businessName}}`, `{{userName}}`) happens client-side in `SendEstimateForm` (before the user edits), same pattern as `SendQuoteForm` lines 32-56. Server-side rendering is pure string-replace in `EstimateEmailRenderer` — it does not re-resolve variables that the client already resolved. This avoids double-resolution and the "user sees `{{jobTitle}}`" edge case.

### Rendered HTML Persistence (D-AUDIT)

- **D-AUDIT-01:** A new MongoDB collection `estimate_email_sends` stores one row per send event. Collection naming follows existing snake_case convention (e.g. `quote_counters`, `estimate_counters`, `document_tokens`). One row per send, never mutated once written, never soft-deleted. New `EstimateEmailSendRepository` + `EstimateEmailSendCreator` services live under `src/estimate/` (not under `src/email/`) because the entity is estimate-scoped.
- **D-AUDIT-02:** Row shape (`IEstimateEmailSendEntity`):
  ```typescript
  {
    _id: ObjectId;
    estimateId: ObjectId;          // the specific revision this send belongs to
    rootEstimateId: ObjectId;      // denormalized for chain-wide audit queries
    revisionNumber: number;        // denormalized from the estimate row
    businessId: ObjectId;          // for policy scoping
    type: EstimateEmailSendType;   // initial | resend | followup_3d | followup_10d | followup_21d
    to: string;                    // recipient email
    subject: string;               // exact subject sent
    html: string;                  // exact HTML passed to Resend, verbatim
    sentAt: Date;
    createdAt: Date;
    updatedAt: Date;
  }
  ```
  `EstimateEmailSendType` is a new enum in `src/estimate/enums/estimate-email-send-type.enum.ts`. Phase 44 writes `INITIAL` (first send of a revision), `RESEND` (subsequent send of the same revision). Phase 46 appends `FOLLOWUP_3D`, `FOLLOWUP_10D`, `FOLLOWUP_21D` on follow-up dispatch. Phase 44 defines all five enum values up front so Phase 46 only has to consume them.
- **D-AUDIT-03:** Indexes on `estimate_email_sends`:
  - `{estimateId: 1, sentAt: -1}` — "all sends for this specific revision, most recent first"
  - `{rootEstimateId: 1, sentAt: -1}` — "all sends across the entire chain, most recent first" (for the Phase 43 / future History panel)
  - `{businessId: 1, sentAt: -1}` — business-scoped audit queries
  No unique indexes — a single revision legitimately has multiple rows (initial + multiple resends + 3 follow-ups).
- **D-AUDIT-04:** The estimate entity itself gains two new denormalized fields for quick-read display on the detail page and list: `lastSentAt: Date` and `lastSentTo: string`. Both nullable (nullable for Draft estimates that have never been sent). `EstimateEmailSender` updates these fields atomically alongside the status transition. SC #6 "estimate detail page reflects the updated lastSentAt" is satisfied by this field without a join to `estimate_email_sends`.
- **D-AUDIT-05:** HTML is stored **exactly as passed to Resend**. No re-sanitising pass, no Maizzle re-render, no strip-scripts. The already-sanitised-and-rendered string that `EstimateEmailRenderer.render()` returns is the same string passed to `EstimateEmailSendCreator.create()` and to `EmailSenderService.sendEmail()`. SC #5 "exact rendered HTML sent to the customer is persisted" is satisfied literally.
- **D-AUDIT-06:** Retention: **indefinite, no TTL index**. UK limitation period for contract disputes is 6 years (Limitation Act 1980). A TTL at 1 year would kill the audit trail for most real disputes. Phase 44 does not impose a TTL; a future phase can add one if storage becomes a concern. Expected storage: ~10 KB per row × ~5 rows per revision × typical revision count = tiny.
- **D-AUDIT-07:** Policy: `EstimateEmailSendRepository.findAllByEstimateId(estimateId)` and `findAllByRootEstimateId(rootEstimateId)` are both scoped by `businessId` via a policy check in the calling service. Phase 44 does NOT ship a GET endpoint for email sends (there is no UI yet that reads them). The repository exists so Phase 46 can append follow-up rows and so a future admin/audit tool can query. This is deliberate: no HTTP surface for a new collection until there's a real consumer.
- **D-AUDIT-08:** Write path ordering in `EstimateEmailSender.send()`:
  1. Resolve or create the document token (D-TKN extensions below).
  2. Render the email HTML via `EstimateEmailRenderer`.
  3. Call `EmailSenderService.sendEmail({to, subject, html, ...})` (Resend SDK).
  4. **After Resend returns success**, create the `estimate_email_sends` row (step 4 happens AFTER step 3 so we never persist an audit row for an email that was never sent).
  5. Transition Draft→Sent (or SENT→SENT no-op) via `EstimateTransitionService`.
  6. Update `estimate.lastSentAt` / `lastSentTo`.
  7. Extend token `expiresAt` to `now + 30 days` (D-TKN-03 below).
  8. On the revised-send path (see D-REV-01 below), call `cancelAllFollowups(oldRevisionId, oldRevisionNumber)` via the injected `IEstimateFollowupCanceller`.
  9. If any step 4-8 fails after Resend succeeded (step 3), log at `AppLogger.error` with full context and return 500. The email is already with the customer; the server-side state is inconsistent but the customer is not affected. This mirrors the "best-effort rollback; alert on failure" pattern from Phase 42 D-CONC-05.

### Token Lookup, Reuse, and Revision Handling (D-TKN)

- **D-TKN-01:** `document_tokens.documentId` for estimate tokens stores the `rootEstimateId`, **not** the per-revision id. Rationale: Phase 45 CUST-06 / SC #1 requires the public estimate controller to resolve the token to the **latest revision** in the chain, not to the specific revision that was originally emailed. Storing `rootEstimateId` on the token makes this resolution a simple `findOne({rootEstimateId, isCurrent: true})` call. For quotes, `documentId` continues to store the quote id as before (quotes do not revise).
- **D-TKN-02:** New repository method `DocumentTokenRepository.findActiveByDocumentId(documentId: string, documentType: "quote" | "estimate"): Promise<IDocumentTokenDto | null>`. "Active" means `deletedAt === null AND revokedAt === null AND expiresAt > now`. Returns at most one row — if multiple rows exist the behaviour is "return the most recent" (`sentAt: -1`), but in practice D-TKN-01 + D-TKN-04 guarantee exactly one active token per `rootEstimateId`. The method is a new addition; no refactor of existing `findByToken` / `create` methods.
- **D-TKN-03:** New repository method `DocumentTokenRepository.extendExpiry(tokenId: string, newExpiresAt: Date): Promise<IDocumentTokenDto>`. Used by `EstimateEmailSender` on every send to refresh the rolling 30-day window. Simple `findOneAndUpdate` with `$set: {expiresAt, updatedAt}`. Zero branching — even first send calls it (no-op if first send already set the window). Planner decides whether to also update `sentAt` on extend or treat `sentAt` as "first ever send" only; recommend **treat `sentAt` as first-ever** and surface "most recent send time" via the `estimate_email_sends` rows (D-AUDIT) + `estimate.lastSentAt` (D-AUDIT-04).
- **D-TKN-04:** `EstimateEmailSender.send()` token resolution flow:
  ```
  rootId = estimate.rootEstimateId
  existingToken = DocumentTokenRepository.findActiveByDocumentId(rootId, "estimate")
  if existingToken is null:
    token = DocumentTokenCreator.create(rootId, recipientEmail, "estimate")
  else:
    token = existingToken
  viewUrl = `${appUrl}/estimate/${token.token}`
  ```
  After the send succeeds, `DocumentTokenRepository.extendExpiry(token.id, now + 30 days)` is called. Every send extends the window — deliberate: trader's intent on re-send is "this is live again for the customer".
- **D-TKN-05:** The "edge case where an existing token is expired/revoked" path (`findActiveByDocumentId` returns null on a re-send) is handled by falling through to `DocumentTokenCreator.create()`. This technically regenerates a token, which is the scenario SND-06 rules out. Rationale: if a token has expired or been revoked, the original URL is already dead from the customer's point of view — reusing its string value is pointless. Phase 44 treats this as "degraded re-send" rather than refusing to send. The new token takes over. Planner flags this in the send response metadata (`regeneratedToken: boolean`) so the UI can surface it if we ever need to. No active revocation path exists in Phase 44 so in practice this branch is dead until a future phase adds revocation.
- **D-TKN-06:** On the **pure re-send path** (same revision, source status Sent/Viewed/Responded/SiteVisitRequested → Sent), the existing token is reused, its expiry is refreshed, and **no follow-up cancellation happens**. No "old revision" exists — the send is a reminder of the same content the customer already has the link to. Phase 46 follow-ups (when they ship) on the same revision continue on their original schedule.
- **D-TKN-07:** On the **revised-send path** (new Draft revision was born via Phase 42 `EstimateReviser`; trader now sends it for the first time; source status Draft → Sent), the existing token is still reused (because `documentId` is `rootEstimateId` and the Phase 42 reviser does NOT touch the token). Its expiry is refreshed. **And** `cancelAllFollowups(previousRevision._id, previousRevision.revisionNumber)` is called via the Phase 42 `IEstimateFollowupCanceller` binding (D-HOOK-04 from Phase 42 explicitly assigns this responsibility to Phase 44). `previousRevision` is found by walking the chain: `findOne({rootEstimateId, revisionNumber: current.revisionNumber - 1})`. Phase 42's NoopEstimateFollowupCanceller is the default binding; Phase 46 rebinds it to `BullMQEstimateFollowupCanceller`.
- **D-TKN-08:** Detection of which send path we're on (pure re-send vs first send of a revision vs revised send): `EstimateEmailSender` inspects the estimate's current state:
  - If `estimate.status === DRAFT AND estimate.revisionNumber === 1` → **first-ever send** (initial). Write `type: INITIAL` to the audit row. No follow-up cancellation.
  - If `estimate.status === DRAFT AND estimate.revisionNumber > 1` → **revised send**. Write `type: INITIAL` to the audit row (it IS the first send of this revision). DO call `cancelAllFollowups` on the previous revision.
  - If `estimate.status IN (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED)` → **pure re-send**. Write `type: RESEND`. No follow-up cancellation.
  - Any other status (terminal) → reject with `InvalidRequestError(ErrorCodes.ESTIMATE_NOT_SENDABLE, "Estimate cannot be sent in its current status")`.
- **D-TKN-09:** Source statuses valid for the send endpoint: `DRAFT`, `SENT`, `VIEWED`, `RESPONDED`, `SITE_VISIT_REQUESTED`. Terminal states (`CONVERTED`, `DECLINED`, `EXPIRED`, `LOST`, `DELETED`) are rejected. The transition service handles the `XXX → SENT` validation:
  - `DRAFT → SENT` already exists in Phase 41 D-TXN-02.
  - `SENT → SENT` already exists in Phase 41 D-TXN-03 (explicit no-op for re-send).
  - `VIEWED → SENT`, `RESPONDED → SENT`, `SITE_VISIT_REQUESTED → SENT` are **new** transitions Phase 44 adds to `ALLOWED_TRANSITIONS`. Rationale: a trader who just got a response (even a "I have a question" reply) can legitimately re-send the estimate as a reminder. The status technically regresses but the underlying document is the same. Planner verifies this doesn't break any Phase 41 transition test and adds positive assertions for the new transitions.
- **D-TKN-10:** `document_tokens` table schema: no changes needed in Phase 44. Phase 41 D-TKN already added `documentType` discriminator and renamed `quoteId → documentId`. `documentId` for existing quote tokens continues to hold the quote `_id`; new estimate tokens hold the `rootEstimateId`. The discriminator resolves ambiguity between the two.

### Business > Documents Tab (D-DOC)

- **D-DOC-01:** Rename the existing tab in `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` from `quote-email` to `documents`. Tab label changes from "Quote Email" to **"Documents"**. Tab icon: planner's discretion — existing is likely `Mail`; consider `Mail` still (covers both), or `FileText` (document-centric). Not `Settings2` because that reads as a sub-settings panel.
- **D-DOC-02:** Tab body renders two components stacked vertically inside a `<div className="space-y-6">` (or similar spacing container):
  1. `<QuoteEmailSettings businessId={businessId} />` — existing component, unchanged.
  2. `<EstimateEmailSettings businessId={businessId} />` — new component, 1:1 mirror of `QuoteEmailSettings` with `quote → estimate` renames.
  Each form has its own section heading ("Quote Email Template" / "Estimate Email Template"), its own `useQuery`, its own save button, its own toast. Zero cross-coupling.
- **D-DOC-03:** New `EstimateEmailSettings` component lives at `trade-flow-ui/src/features/business/components/EstimateEmailSettings.tsx`. Structure mirrors `QuoteEmailSettings.tsx` line-for-line with renames:
  - `useGetQuoteSettingsQuery` → `useGetEstimateSettingsQuery`
  - `useUpdateQuoteSettingsMutation` → `useUpdateEstimateSettingsMutation`
  - `quoteEmailSubject` → `estimateEmailSubject`
  - `quoteEmailBody` → `estimateEmailBody`
  - "Quote email template saved" toast → "Estimate email template saved"
  One deliberate addition: an informational `<Alert>` or small note in the estimate form body that reads "The non-binding legal disclaimer is added automatically and cannot be edited." This is a **UX hint**, not a security enforcement — D-LGL-01 handles the enforcement. The hint prevents traders from wondering "where do I put the legal bit?"
- **D-DOC-04:** Export `EstimateEmailSettings` from `features/business/components/index.tsx` alongside the existing `QuoteEmailSettings` export. Import both into `BusinessDetails.tsx`.
- **D-DOC-05:** Subject-line validation on the client: `EstimateEmailSettings` form catches `InvalidRequestError` from the PATCH and displays the error message inline beneath the subject field (e.g. "Estimate email subject must contain the word 'Estimate'"). Client-side validation via a `valibot` / `react-hook-form` resolver is **not** added in Phase 44 — server-side validation is the source of truth and the toast+inline display pattern matches `QuoteEmailSettings` today.

### estimate-settings Module (D-SET)

- **D-SET-01:** New module `src/estimate-settings/` mirroring `src/quote-settings/` 1:1:
  ```
  estimate-settings/
  ├── estimate-settings.module.ts
  ├── controllers/
  │   └── estimate-settings.controller.ts
  ├── services/
  │   ├── default-estimate-settings-creator.service.ts
  │   ├── estimate-settings-retriever.service.ts
  │   └── estimate-settings-updater.service.ts
  ├── repositories/
  │   └── estimate-settings.repository.ts
  ├── entities/
  │   └── estimate-settings.entity.ts
  ├── data-transfer-objects/
  │   └── estimate-settings.dto.ts
  ├── requests/
  │   └── update-estimate-settings.request.ts
  ├── responses/
  │   └── estimate-settings.response.ts
  ├── policies/
  │   └── estimate-settings.policy.ts
  └── test/
      ├── services/
      └── mocks/
  ```
- **D-SET-02:** Entity shape `IEstimateSettingsEntity` (mirrors `QuoteSettingsEntity`):
  ```typescript
  {
    _id: ObjectId;
    businessId: ObjectId;
    estimateEmailSubject?: string;
    estimateEmailBody?: string;
    createdAt: Date;
    updatedAt: Date;
  }
  ```
  No `legalCopy` field (D-LGL-01). No `documentType` discriminator — the collection is estimate-only.
- **D-SET-03:** MongoDB collection: `estimate_settings` (snake_case, consistent with `quote_settings`). One-to-one with `businesses` via a unique index on `{businessId: 1}`. Planner verifies exact index definition against `quote-settings` repo.
- **D-SET-04:** Endpoints (mirror `quote-settings` 1:1):
  - `GET /v1/business/:businessId/estimate-settings` → `JwtAuthGuard`, returns `IEstimateSettingsResponse` wrapped in `createResponse([...])`. If no row exists (new business that somehow slipped past the eager-create path), returns a stub `{id: "", businessId, estimateEmailSubject: undefined, estimateEmailBody: undefined}` — same null-safe behaviour as `QuoteSettingsController` lines 29-38.
  - `PATCH /v1/business/:businessId/estimate-settings` → `JwtAuthGuard`, accepts `UpdateEstimateSettingsRequest` with `estimateEmailSubject?: string` + `estimateEmailBody?: string`, both optional, subject validated via the SND-07 class-validator decorator (D-LGL-05).
- **D-SET-05:** `DefaultEstimateSettingsCreatorService.create(businessId)` writes a new row with both template fields set to `undefined` (i.e. system defaults apply at send time). The row exists so the unique index is populated and PATCH can upsert cleanly.
- **D-SET-06:** `BusinessCreator` is extended: it already calls `DefaultQuoteSettingsCreatorService.create(createdBusiness.id)`. Phase 44 adds a parallel `DefaultEstimateSettingsCreatorService.create(createdBusiness.id)` call in the same service method. Both creators are injected; both are called; both are awaited. Failures in either are propagated to the business creation response (rejecting the business creation) — this matches the existing quote-settings behaviour and is a deliberate "don't leave a business in a partially-initialised state" choice. The `business-creator.service.spec.ts` test suite gains a mirror assertion for the new creator call.
- **D-SET-07:** Module imports: `EstimateSettingsModule` imports `CoreModule`. `BusinessModule` imports `EstimateSettingsModule` (needed for the `DefaultEstimateSettingsCreatorService` injection in `BusinessCreator`). `EstimateModule` imports `EstimateSettingsModule` (needed so `EstimateEmailSender` can read the template defaults via `EstimateSettingsRetriever`). `AppModule` registers `EstimateSettingsModule`. No circular dependencies expected — the same DI shape quote-settings already has.

### EstimateEmailSender Service (D-SND)

- **D-SND-01:** New service `src/estimate/services/estimate-email-sender.service.ts`. Mirrors `QuoteEmailSender` but with the following differences (beyond the `quote → estimate` renames):
  - Injects `EstimateSettingsRetriever` (not `QuoteSettingsRetriever`).
  - Injects `EstimateEmailRenderer` (not `QuoteEmailRenderer`).
  - Injects `EstimateEmailSendCreator` (new — persists the audit row).
  - Injects `DocumentTokenCreator` + `DocumentTokenRepository` (not `QuoteTokenCreator`).
  - Injects the `IEstimateFollowupCanceller` binding via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` — the Phase 42 wiring point.
  - Does NOT call `OnboardingProgressUpdater.markStepComplete(FIRST_QUOTE_SENT)`. Onboarding step parity for estimates is **deferred** to a follow-up phase. Quote send remains the onboarding anchor.
- **D-SND-02:** Service method signature:
  ```typescript
  public async send(
    authUser: IUserDto,
    estimateId: string,
    request: SendEstimateRequest
  ): Promise<IEstimateDto>
  ```
  Note: no `businessId` parameter — the flat Phase 41 endpoint pattern passes only `:id` and the policy enforces business scoping via the estimate's own `businessId` field.
- **D-SND-03:** `SendEstimateRequest` shape (new class at `src/estimate/requests/send-estimate.request.ts`):
  ```typescript
  {
    to: string;           // validated as email
    subject: string;      // validated non-empty + must contain "Estimate" (D-LGL-05)
    message: string;      // rich-text HTML body (Tiptap output); sanitised server-side same as quote path
    saveEmail?: boolean;  // if true, save `to` back to customer record
  }
  ```
  Mirrors `SendQuoteRequest` 1:1 with the subject validator added. class-validator decorators.
- **D-SND-04:** The `sanitizeHtml()` helper in `QuoteEmailSender` (lines 108-119) is copied verbatim into `EstimateEmailSender` with no refactor into a shared utility. This deliberately duplicates the tag allowlist (`["p", "br", "strong", "em", "a", "ul", "ol", "li"]`) in two places. Rationale: D-01 separation-over-DRY at the entity boundary. The next time either sanitiser changes, both should change. Planner may flag a follow-up ticket to lift into a shared `@core/utilities` helper, but Phase 44 ships duplicated.
- **D-SND-05:** Error handling mirrors `QuoteEmailSender` lines 75-80: HttpExceptions bubble; any other unknown error becomes `HttpException("Failed to send estimate email", HttpStatus.INTERNAL_SERVER_ERROR)`. The step-4-or-later failure modes from D-AUDIT-08 log at error level before re-throwing.
- **D-SND-06:** `EstimateEmailSender` does NOT handle the `customerEmail === null` "no email on file" edge case at the service layer. The `SendEstimateForm` UI catches this and displays the warning alert + "save email to customer record" checkbox (mirrors `SendQuoteForm` lines 107-125). The service just validates the incoming `to` is a well-formed email and proceeds.

### EstimateEmailRenderer and Maizzle Template (D-TPL)

- **D-TPL-01:** New service `src/email/services/estimate-email-renderer.service.ts`. Mirrors `QuoteEmailRenderer` (lines 1-48 of `quote-email-renderer.service.ts`) with the following changes:
  - Reads `src/email/templates/estimate-sent.html` (new template file, Phase 44 creates).
  - Template variables: `businessName`, `message` (raw HTML via `{{{ message }}}` triple-brace), `viewUrl`, `estimateNumber`, `estimateDate`, `priceDisplay` (string like `£120 - £132` or `From £120`), `contingencyNote` (optional — only rendered if contingencyPercent > 0), `uncertaintyNotes` (optional — only rendered if non-empty).
  - Same Maizzle dynamic-import dance with the Function-eval fallback (lines 28-42 of `quote-email-renderer.service.ts`). Same `escapeHtml` helper.
- **D-TPL-02:** `EstimateEmailData` interface (new, in the same file as the renderer):
  ```typescript
  export interface EstimateEmailData {
    businessName: string;
    message: string;              // sanitised rich-text HTML
    viewUrl: string;              // `${appUrl}/estimate/${token}`
    estimateNumber: string;       // "E-2026-001"
    estimateDate: string;         // "15 April 2026" via Luxon
    priceDisplay: string;         // formatRange output, server-side computed
    contingencyPercent: number;   // for display-only ("Includes 10% uncertainty buffer")
    contingencyNote?: string;     // pre-composed string; undefined hides the block
    uncertaintyNotes?: string;    // user's freeform notes from Phase 43; undefined hides
  }
  ```
  The server computes `priceDisplay` using the same `{low, high, displayMode}` values that the API returns on `IEstimateDto.priceRange`. The server-side `formatRange` is a small utility in `src/core/utilities/format-range.utility.ts` that mirrors the Phase 43 frontend helper (D-FMT-01) — Phase 44 ships it. It operates on the `priceRange.low.total` / `priceRange.high.total` values as minor units (dinero.js), formats with currency symbol, and returns a string. Planner verifies field-for-field against the Phase 43 frontend helper so both produce byte-identical output for the same input.
- **D-TPL-03:** `estimate-sent.html` Maizzle template layout (top to bottom):
  1. Business name header (same pattern as `quote-email.html`).
  2. **Legal disclaimer card** (D-LGL-02) — bordered, pale warning background, contains the SND-05 wording verbatim.
  3. Trader's personal message (`{{{ message }}}`, raw HTML).
  4. "View Estimate Online" CTA button → `{{ viewUrl }}`.
  5. Summary card with:
     - `Estimate: {{ estimateNumber }}`
     - `Date: {{ estimateDate }}`
     - `Price: {{ priceDisplay }}` (rendered **bold** — this is the headline number)
     - If `{{ contingencyNote }}` present: a small caption line (e.g. "Includes a 10% uncertainty buffer").
     - If `{{ uncertaintyNotes }}` present: a small "Notes:" block with the trader's freeform uncertainty notes.
  6. Footer: "Sent via Trade Flow" small text.
- **D-TPL-04:** The Phase 43 `UNCERTAINTY_CHIP_LABELS` map (D-CHIP-03) is **not** surfaced in the estimate email body. Rationale: the email is a high-level "here's your estimate, here's the range" message — the chip reasons are more useful on the customer page (Phase 45) where the customer is actively deciding. The freeform `uncertaintyNotes` IS surfaced because it's the trader's own words. Planner may revisit if legal review requests the chip reasons be shown in the email for transparency.
- **D-TPL-05:** Phase 44 does NOT ship follow-up templates. Phase 46 owns `estimate-followup-3d.html`, `estimate-followup-10d.html`, `estimate-followup-21d.html`. Phase 44's template file count: exactly one (`estimate-sent.html`).

### SendEstimateDialog and SendEstimateForm (D-UI)

- **D-UI-01:** New components under `trade-flow-ui/src/features/estimates/components/`:
  - `SendEstimateDialog.tsx` — mirrors `SendQuoteDialog.tsx` 1:1 (7-line shell wrapping `SendEstimateForm`).
  - `SendEstimateForm.tsx` — mirrors `SendQuoteForm.tsx` 1:1 with the following diffs:
    - Imports `useSendEstimateMutation` from `features/estimates/api/estimateApi.ts` (new mutation Phase 44 adds).
    - System default subject constant: `"Estimate {{estimateNumber}} for {{jobTitle}}"`.
    - System default message constant: `"Hi {{customerName}},\n\nPlease find your estimate {{estimateNumber}} for {{jobTitle}}. This is a guide price — a firm quote will follow after a site visit.\n\nKind regards,\n{{userName}}"`.
    - Template variable resolution uses `estimate.number` instead of `quote.number`.
    - Client-side subject validation: before enabling the Send button, check that the resolved subject contains "Estimate" (case-insensitive). Show an inline error tip if the user has edited it out. The server-side validator (D-LGL-05) is the authoritative check; the client-side hint is UX courtesy.
    - Dialog title: "Send Estimate" (not "Send Quote").
    - Success toast: `` `Estimate sent to ${to}` ``.
- **D-UI-02:** `SendEstimateForm` receives the same prop shape as `SendQuoteForm` with `quote` replaced by `estimate`:
  ```typescript
  interface SendEstimateFormProps {
    onOpenChange: (open: boolean) => void;
    estimate: Estimate;
    customerEmail: string | null;
    customerName: string;
    businessName: string;
    userName: string;
    businessId: string;
    defaultSubject: string;  // from estimate-settings
    defaultMessage: string;  // from estimate-settings
  }
  ```
- **D-UI-03:** `EstimateActionStrip.tsx` (stub from Phase 43 D-MIR-02) is extended in Phase 44 with:
  - A "Send Estimate" button, visible when `estimate.status === "draft"`.
  - A "Re-send" button, visible when `estimate.status IN ["sent", "viewed", "responded", "site_visit_requested"]`.
  - Clicking either opens `<SendEstimateDialog open estimate={estimate} ... />`.
  - Icon: `Send` from lucide-react (matches quote pattern).
  - Disabled state when `customerEmail === null` AND the user can't edit it in the dialog — actually no, the dialog handles null email via the "no email on file" warning + save-back checkbox pattern (D-SND-06), so the button is never disabled for missing email. It's only hidden on terminal statuses.
- **D-UI-04:** `estimateApi.ts` (from Phase 43 D-MIR-04) gains a new RTK Query endpoint:
  ```typescript
  sendEstimate: builder.mutation<Estimate, {
    estimateId: string;
    to: string;
    subject: string;
    message: string;
    saveEmail?: boolean;
  }>({
    query: ({ estimateId, ...body }) => ({
      url: `/v1/estimates/${estimateId}/send`,
      method: "POST",
      body,
    }),
    invalidatesTags: (result, error, { estimateId }) => [
      { type: "Estimate", id: estimateId },
      { type: "Estimate", id: "LIST" },
    ],
  }),
  ```
  Exported as `useSendEstimateMutation`. Invalidates the estimate's cache entry and the list (because `lastSentAt` / status changed). Does NOT invalidate `document_tokens` or `estimate_email_sends` caches (neither is fetched by the frontend in Phase 44).
- **D-UI-05:** `QuoteDetailPage` pattern for reading the default template is mirrored on `EstimateDetailPage`:
  ```typescript
  const { data: estimateSettings } = useGetEstimateSettingsQuery(businessId!, {
    skip: !businessId,
  });
  // ...
  defaultSubject={estimateSettings?.estimateEmailSubject ?? ""}
  defaultMessage={estimateSettings?.estimateEmailBody ?? ""}
  ```
  `EstimateDetailPage` is a Phase 43 file — Phase 44 extends it with the `useGetEstimateSettingsQuery` + `SendEstimateDialog` wiring. Planner confirms `EstimateDetailPage` has a props-open-state for the send dialog or adds one.
- **D-UI-06:** New `useSendEstimateMutation` hook exported from `features/estimates/index.ts` barrel (from Phase 43 D-MIR-04).

### Endpoint Shape and Controller Wiring (D-API)

- **D-API-01:** `POST /v1/estimates/:id/send` — new route handler on the existing Phase 41 `EstimateController`. Consistent with Phase 41's flat endpoint pattern (D-TXN-05). NOT business-scoped (no `/business/:businessId/` prefix).
- **D-API-02:** Handler shape (mirrors Phase 41 controller conventions):
  ```typescript
  @UseGuards(JwtAuthGuard)
  @Post("v1/estimates/:id/send")
  public async send(
    @Req() request: { user: IUserDto; params: { id: string } },
    @Body() body: SendEstimateRequest,
  ): Promise<IResponse<IEstimateResponse>> {
    try {
      const estimate = await this.estimateEmailSender.send(
        request.user,
        request.params.id,
        body,
      );
      return createResponse([this.mapToResponse(estimate)]);
    } catch (error) {
      throw createHttpError(error);
    }
  }
  ```
- **D-API-03:** `EstimateModule` wiring: `EstimateEmailSender`, `EstimateEmailSendRepository`, `EstimateEmailSendCreator` are added to the `providers` array. `EstimateEmailRenderer` is added to `EmailModule` providers (it lives in `@email/services/` per existing convention). `EstimateModule` imports `EmailModule`, `EstimateSettingsModule`, `DocumentTokenModule` (from Phase 41), `BusinessModule`, `CustomerModule`. Planner verifies no circular imports — the `BusinessModule` ↔ `EstimateSettingsModule` relationship already exists in the parallel quote-settings case.
- **D-API-04:** `openapi.yaml` updates:
  - New endpoints: `POST /v1/estimates/:id/send`, `GET /v1/business/:businessId/estimate-settings`, `PATCH /v1/business/:businessId/estimate-settings`.
  - New schemas: `IEstimateSettingsResponse`, `UpdateEstimateSettingsRequest`, `SendEstimateRequest`.
  - Updated `IEstimateResponse` schema: add `lastSentAt: string | null` and `lastSentTo: string | null` fields.

### CI Gate and Legal Review

- **D-GATE-01:** `npm run ci` passes in both `trade-flow-api` and `trade-flow-ui` as a blocking condition before the phase is considered done. Standard CI gate policy from `.planning/CLAUDE.md`.
- **D-GATE-02:** **Legal review gate (from ROADMAP):** The default estimate email template copy (both the Maizzle HTML disclaimer block and the default subject/message constants) must pass a targeted UK-consumer-law copy review before this phase ships. The review is a blocking gate, not a soft constraint. If legal requests edits, Phase 44 ships an additional commit with the amended copy and re-runs the CI gate. Planner captures this as an explicit final task in the plan.
- **D-GATE-03:** Non-removable disclaimer enforcement is verified by a **structural test** (not a validation test): a new test in `src/email/test/services/estimate-email-renderer.service.spec.ts` asserts that the rendered HTML for an estimate always contains the exact SND-05 wording, regardless of any `estimateEmailBody` input. Test matrix: (1) default message empty, (2) message is "Hello", (3) message is a large paragraph, (4) message contains HTML tags that try to comment out / escape the disclaimer. All four cases assert the disclaimer string is in the rendered output.

### Claude's Discretion (planner may override with rationale)

- Exact DI token naming for `IEstimateFollowupCanceller` injection (`ESTIMATE_FOLLOWUP_CANCELLER` per Phase 42 D-HOOK-02 — already fixed).
- Exact `ErrorCodes` enum value names (`ESTIMATE_SUBJECT_MISSING_KEYWORD`, `ESTIMATE_NOT_SENDABLE`, etc.) — planner picks consistent with existing naming in `src/core/errors/error-codes.enum.ts`.
- Whether the server-side `format-range.utility.ts` lives under `@core/utilities/` or `@estimate/utilities/` — pick based on whether anything else in `@core` needs range formatting (unlikely — it's estimate-specific).
- Whether the `contingencyNote` string ("Includes a 10% uncertainty buffer") is composed in the service layer or inside the Maizzle template via a conditional — planner's call based on which is more readable.
- Exact Tailwind classes for the disclaimer card — follow the existing `quote-email.html` visual language.
- Whether `EstimateEmailSender` injects `OnboardingProgressUpdater` or not. Phase 44 decision is **not** (D-SND-01 explicitly defers onboarding step parity). Planner confirms no dead import.
- Location of the `send-estimate.request.ts` file — `src/estimate/requests/` per existing convention.
- Test fixture strategy for `EstimateEmailRenderer` and `EstimateEmailSender`: mirror the existing `quote-email-renderer.service.spec.ts` and `quote-email-sender.service.spec.ts` structure. Copy the mock factories and rename.
- Whether `SendEstimateForm`'s client-side subject validation triggers on every keystroke or only on blur — follow whatever `SendQuoteForm` does today (likely on-submit via the disabled-button check).
- Icon choice for the "Documents" tab trigger — `Mail` reads as "email stuff" which is accurate; `FileText` reads as "document stuff" which is also accurate. Either is fine.
- Whether the `BusinessCreator` test update asserts that both creators are called in parallel or sequentially — planner verifies against the existing implementation.

### Folded Todos

None — `.planning/STATE.md` shows "Pending Todos: None." No todos were folded into this phase.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Milestone roadmap and restructure decisions
- `.planning/milestones/v1.8-ROADMAP.md` — Phase 44 goal, requirements (SND-01..07), success criteria (#1-6), dependency on Phase 41 and Phase 43, legal-review gate wording. Note: SC #2 must be rewritten as part of Phase 44 to replace "Business > Templates tab" with "Business > Documents tab".
- `.planning/ROADMAP.md` — same SC #2 update required.
- `.planning/notes/2026-04-11-v1.8-restructure-decisions.md` — D-01 separation-over-DRY. Phase 44 applies this to `estimate-settings`, `EstimateEmailRenderer`, and `EstimateEmailSender` (all new, not shared with quote equivalents).

### Requirements
- `.planning/REQUIREMENTS.md` — SND-01 through SND-07 (Phase 44's direct requirements). SND-02 wording needs the Templates → Documents tab rename.

### Phase 41 decisions that Phase 44 must honour
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` — canonical source for the Phase 41 contract. Specifically:
  - **D-TKN-04** — `IDocumentTokenEntity` shape; `documentId` is the field Phase 44 populates with `rootEstimateId` for estimates.
  - **D-TXN-01** — `EstimateStatus` values and spelling (`site_visit_requested` with underscore, etc.).
  - **D-TXN-02** — `ALLOWED_TRANSITIONS` map Phase 44 extends with `VIEWED → SENT`, `RESPONDED → SENT`, `SITE_VISIT_REQUESTED → SENT` (D-TKN-09).
  - **D-TXN-03** — Existing `SENT → SENT` no-op rule for pure re-send (D-TKN-06).
  - **D-TXN-04** — `EstimateTransitionService` is the only path to change status; Phase 44 calls it from `EstimateEmailSender`.
  - **D-TXN-05** — Flat endpoint pattern (`/v1/estimates/:id/...`, no `/business/:businessId/` scoping). Phase 44 ships `POST /v1/estimates/:id/send` (D-API-01).
  - **D-ENT-01** — `IEstimateEntity` shape; Phase 44 adds `lastSentAt`, `lastSentTo` (D-AUDIT-04).
- `.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md` — Phase 41's research into the backend contract.
- `.planning/phases/41-estimate-module-crud-backend/41-*-PLAN.md` — Phase 41's plans, specifically `41-08-controller-module-wiring-and-docs-PLAN.md` for controller shape.

### Phase 42 decisions that Phase 44 must honour
- `.planning/phases/42-revisions/42-CONTEXT.md` — canonical source for the revision contract. Specifically:
  - **D-HOOK-01/02/03/04/05** — `IEstimateFollowupCanceller` interface and the Phase 42 wiring. D-HOOK-04 explicitly assigns Phase 44 the responsibility of calling `cancelAllFollowups` at the moment a new revision transitions Draft → Sent (D-TKN-07 here).
  - **D-REV-02** — Revisions are allowed from SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED; Phase 44's re-send-on-revised-draft path must understand that the previous revision still holds its existing status.
  - **D-CHAIN-01/02** — `rootEstimateId` chain identity and `parentEstimateId` walk. Phase 44 uses `rootEstimateId` as the token's `documentId` (D-TKN-01).
  - **D-DET-01** — `GET /v1/estimates/:id` returns current row even when `:id` is a non-current revision id. Phase 44's controller inherits this behaviour.

### Phase 43 decisions that Phase 44 must honour
- `.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md` — frontend conventions and component shape.
  - **D-MIR-02** — Component mirror list; `SendEstimateDialog` + `SendEstimateForm` + `EstimateActionStrip` are reserved for Phase 44 to fill in.
  - **D-MIR-03** — Send dialog/form are explicitly deferred to Phase 44.
  - **D-MIR-04** — `estimateApi.ts` exists; Phase 44 adds `useSendEstimateMutation` to it.
  - **D-MIR-07** — `Estimate.priceRange` shape Phase 44's email renderer reads.
  - **D-MIR-08** — `EstimateStatus` values. Phase 44 uses these for the send-path status check.
  - **D-CHIP-03** — Uncertainty chip labels. Phase 44 decision (D-TPL-04): not surfaced in the email body.
  - **D-FMT-01/02** — Frontend `formatRange` helper. Phase 44 ships a server-side mirror at `src/core/utilities/format-range.utility.ts` (D-TPL-02) that produces byte-identical output.

### Phase 18 (shipped) reference implementations for mirror
- `trade-flow-api/src/quote/services/quote-email-sender.service.ts` — canonical reference for `EstimateEmailSender` structure (D-SND-01..06). Mirror with renames + the Phase 44 additions.
- `trade-flow-api/src/email/services/quote-email-renderer.service.ts` — canonical reference for `EstimateEmailRenderer` (D-TPL-01).
- `trade-flow-api/src/email/templates/quote-email.html` — canonical reference for `estimate-sent.html` structure (D-TPL-03).
- `trade-flow-api/src/quote/requests/send-quote.request.ts` — canonical reference for `SendEstimateRequest` (D-SND-03).
- `trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts` — canonical reference for the spec shape; mirror the mock providers and test cases.
- `trade-flow-api/src/email/test/services/quote-email-renderer.service.spec.ts` — canonical reference for the renderer spec.

### quote-settings module to mirror
- `trade-flow-api/src/quote-settings/quote-settings.module.ts` — module registration pattern.
- `trade-flow-api/src/quote-settings/controllers/quote-settings.controller.ts` — GET + PATCH handler shape, null-safe stub on empty GET (D-SET-04).
- `trade-flow-api/src/quote-settings/services/default-quote-settings-creator.service.ts` — default-settings creator pattern (D-SET-05).
- `trade-flow-api/src/quote-settings/services/quote-settings-retriever.service.ts` — retriever pattern.
- `trade-flow-api/src/quote-settings/services/quote-settings-updater.service.ts` — updater pattern + upsert behaviour.
- `trade-flow-api/src/quote-settings/repositories/quote-settings.repository.ts` — repository pattern, unique-on-businessId index.
- `trade-flow-api/src/quote-settings/entities/quote-settings.entity.ts` — entity shape mirror (D-SET-02).
- `trade-flow-api/src/quote-settings/data-transfer-objects/quote-settings.dto.ts` — DTO shape mirror.
- `trade-flow-api/src/quote-settings/requests/update-quote-settings.request.ts` — request class with class-validator decorators.
- `trade-flow-api/src/quote-settings/responses/quote-settings.response.ts` — response interface (independent of DTO).
- `trade-flow-api/src/quote-settings/policies/quote-settings.policy.ts` — policy class.

### BusinessCreator extension point
- `trade-flow-api/src/business/services/business-creator.service.ts` — exact lines where `DefaultQuoteSettingsCreatorService.create(businessId)` is called. Phase 44 adds a parallel `DefaultEstimateSettingsCreatorService.create(businessId)` call.
- `trade-flow-api/src/business/test/services/business-creator.service.spec.ts` — existing assertion that the default quote-settings creator is called (line 226). Phase 44 adds a mirror assertion for the estimate-settings creator.
- `trade-flow-api/src/business/business.module.ts` — imports list; Phase 44 adds `EstimateSettingsModule` import.

### DocumentToken module (from Phase 41)
- `trade-flow-api/src/document-token/repositories/document-token.repository.ts` (Phase 41-renamed from `quote-token`) — Phase 44 adds `findActiveByDocumentId` (D-TKN-02) and `extendExpiry` (D-TKN-03) methods.
- `trade-flow-api/src/document-token/services/document-token-creator.service.ts` — existing `create(documentId, recipientEmail, documentType)` method used by `EstimateEmailSender` on the null-token path (D-TKN-05).
- `trade-flow-api/src/document-token/entities/document-token.entity.ts` — entity shape; Phase 44 does NOT add new fields.
- `trade-flow-api/src/document-token/guards/document-session-auth.guard.ts` — referenced by Phase 45; Phase 44 does not use it directly.

### EmailModule and Resend integration
- `trade-flow-api/src/email/email.module.ts` — providers list; Phase 44 adds `EstimateEmailRenderer`.
- `trade-flow-api/src/email/services/email-sender.service.ts` — Resend wrapper used unchanged (`sendEmail({to, subject, html, from, senderName})`).
- `trade-flow-api/src/email/interfaces/send-email.dto.ts` — the DTO shape used by both the quote and estimate send paths.

### Frontend components to mirror
- `trade-flow-ui/src/features/quotes/components/SendQuoteDialog.tsx` — 7-line shell, Phase 44 mirrors as `SendEstimateDialog.tsx` (D-UI-01).
- `trade-flow-ui/src/features/quotes/components/SendQuoteForm.tsx` — full form shape including client-side template variable resolution, Phase 44 mirrors as `SendEstimateForm.tsx` (D-UI-01).
- `trade-flow-ui/src/features/business/components/QuoteEmailSettings.tsx` — Phase 44 mirrors as `EstimateEmailSettings.tsx` (D-DOC-03).
- `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` — tab structure (lines 441-525); Phase 44 renames `quote-email` → `documents`, restructures tab body (D-DOC-01/02).
- `trade-flow-ui/src/features/business/components/index.tsx` — export list; Phase 44 adds `EstimateEmailSettings`.
- `trade-flow-ui/src/features/business/api/businessApi.ts` — existing quote-settings RTK Query endpoints (lines 59-86). Phase 44 adds parallel `getEstimateSettings` / `updateEstimateSettings` endpoints OR lifts them into a new `features/estimates/api/estimateSettingsApi.ts` file. Planner picks the cleaner location; recommendation is to keep them together in `businessApi.ts` because they are business-scoped settings.
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` — lines 40-124 show how `useGetQuoteSettingsQuery` and `SendQuoteDialog` are wired. Phase 44 mirrors this pattern on `EstimateDetailPage` (D-UI-05).
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — Phase 43 stub; Phase 44 extends with Send/Re-send buttons (D-UI-03).
- `trade-flow-ui/src/features/estimates/api/estimateApi.ts` — Phase 43 file; Phase 44 adds `sendEstimate` mutation (D-UI-04).
- `trade-flow-ui/src/features/estimates/index.ts` — Phase 43 barrel; Phase 44 exports new send components and the new hook.

### Codebase maps
- `.planning/codebase/CONVENTIONS.md` — backend + frontend naming standards.
- `.planning/codebase/ARCHITECTURE.md` — layered architecture rules.
- `.planning/codebase/STRUCTURE.md` — where modules live.
- `.planning/codebase/TESTING.md` — vitest + jest conventions.

### OpenAPI
- `trade-flow-api/openapi.yaml` — canonical spec; Phase 44 adds the three new endpoints and updates `IEstimateResponse` (D-API-04).

### Research context
- `.planning/research/ARCHITECTURE.md` §10 (Milestone mapping) — original Phase-A-B-C rationale. Note: the 2026-04-11 restructure flipped the ARCHITECTURE.md recommendation to extend `quote-settings` and locked in the separation-over-DRY path Phase 44 follows.
- `.planning/research/FEATURES.md` §11 (UK legal context) — Consumer Rights Act 2015, Dispute Resolution Ombudsman guidance on estimate non-binding status. Informs D-LGL-03.
- `.planning/research/PITFALLS.md` — Pitfall 4 (Redis AOF, Phase 46 concern). Not directly applicable to Phase 44.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- **`QuoteEmailSender`, `QuoteEmailRenderer`, `quote-email.html`** — the template for `EstimateEmailSender`, `EstimateEmailRenderer`, `estimate-sent.html`. The existing files are clean, small, and self-contained. Lifting them 1:1 into the estimate namespace is a low-risk copy-and-rename.
- **`EmailSenderService`** — Resend wrapper already in place. Phase 44 calls `sendEmail({to, subject, html, from, senderName})` unchanged. The "from address resolution" logic (`resolveFromEmail` in `QuoteEmailSender` lines 92-106) is duplicated into `EstimateEmailSender` (D-SND-01).
- **`DocumentTokenCreator`** (Phase 41) — already accepts `documentType: "quote" | "estimate"`. Phase 44 calls it for estimates in the "no active token exists" path (D-TKN-05).
- **`EstimateTransitionService`** (Phase 41) — already handles Draft → Sent and Sent → Sent. Phase 44 adds three new transitions (Viewed → Sent, Responded → Sent, SiteVisitRequested → Sent) to the map and its test.
- **`BusinessCreator`** — already wires `DefaultQuoteSettingsCreatorService`. Phase 44 wires the parallel `DefaultEstimateSettingsCreatorService` in the same method. Confirmed in `business-creator.service.spec.ts` line 226.
- **`SendQuoteDialog` + `SendQuoteForm`** — 7-line dialog shell + rich-text form. Template for Phase 44's estimate equivalents. `SendQuoteForm` already handles the customer-email-null warning, save-back checkbox, template variable resolution, client-side sanitisation boundary, and Tiptap RichTextEditor integration. Mirror 1:1.
- **`QuoteEmailSettings`** — the template for `EstimateEmailSettings` (D-DOC-03). Already handles the upsert pattern, toast, undefined-to-empty-string conversion, and the 'no saved settings yet' empty-state.
- **`BusinessDetails.tsx` Tabs component** — already uses shadcn `Tabs` / `TabsList` / `TabsTrigger` / `TabsContent`. Phase 44 renames the `quote-email` trigger and content values to `documents` and re-arranges the tab content body.
- **`QuoteDetailPage.tsx`** — exact template for `EstimateDetailPage.tsx`'s send-wiring: `useGetQuoteSettingsQuery(businessId)` → `defaultSubject` / `defaultMessage` props → `SendQuoteDialog`.
- **Phase 42 `IEstimateFollowupCanceller` interface + `NoopEstimateFollowupCanceller` binding** — already wired via DI token `ESTIMATE_FOLLOWUP_CANCELLER`. Phase 44 injects the interface and calls `cancelAllFollowups` on the revised-send path (D-TKN-07).

### Established Patterns

- **Controller → Service → Repository layering** — Phase 44 follows strictly. `EstimateController.send` → `EstimateEmailSender.send` → `EstimateEmailSendRepository.create` / `DocumentTokenRepository.findActiveByDocumentId` / `EstimateTransitionService.transition` / etc.
- **Policy-based authorization** — `EstimatePolicy.canUpdate` (or equivalent) gates the send because sending mutates estimate state. Planner verifies which policy method is correct — creating/updating is covered by existing Phase 41 policies.
- **Separation over DRY at entity boundary** — enforced for `estimate-settings` (new module), `EstimateEmailSender` (new service, not shared with quote), `EstimateEmailRenderer` (new service), `estimate-sent.html` (new template), `sanitizeHtml` helper (duplicated inside `EstimateEmailSender`, D-SND-04), `formatRange` helper (new server-side mirror of the Phase 43 frontend helper, D-TPL-02). The shared `EmailSenderService` / `DocumentTokenCreator` are NOT entity-scoped — they remain shared infrastructure, consistent with the separation-over-DRY feedback memory ("exception: security infra").
- **Null-safe empty-settings GET stub** — `QuoteSettingsController` lines 29-38 return a stub object when no DB row exists. Phase 44 mirrors this in `EstimateSettingsController` even though the eager-creation path (D-SET-06) should mean the stub is never hit in practice.
- **Rich-text sanitisation at service layer** — `QuoteEmailSender.sanitizeHtml` (lines 108-119) applies an allowlist of tags. `EstimateEmailSender` duplicates this verbatim.
- **Client-side template variable resolution** — `SendQuoteForm` lines 32-56 resolve `{{customerName}}` etc. client-side before display. Server's `EstimateEmailRenderer` is pure string-replace for different variables (business-level context); it does NOT re-resolve client-side variables.
- **Structural tests for HTML output** — `quote-email-renderer.service.spec.ts` already tests rendered output. Phase 44's `estimate-email-renderer.service.spec.ts` extends this with the disclaimer-presence test (D-GATE-03).
- **RTK Query mutation with tag invalidation** — `sendEstimate` invalidates the estimate cache entry and the list. No optimistic updates (the send can fail and we want clean rollback).

### Integration Points

- **New module `src/estimate-settings/`** — registered in `AppModule`. Imported by `BusinessModule` (for `DefaultEstimateSettingsCreatorService`) and `EstimateModule` (for `EstimateSettingsRetriever`).
- **`EstimateModule` extension** — providers list gains `EstimateEmailSender`, `EstimateEmailSendRepository`, `EstimateEmailSendCreator`. Imports list gains `EmailModule`, `EstimateSettingsModule`, `BusinessModule`, `CustomerModule`, `DocumentTokenModule` (if not already imported from Phase 41).
- **`EmailModule` extension** — providers list gains `EstimateEmailRenderer`.
- **`BusinessCreator` extension** — injects `DefaultEstimateSettingsCreatorService` alongside the existing quote-settings creator; calls `.create(createdBusiness.id)` in the same service method. `BusinessModule` imports `EstimateSettingsModule`.
- **`EstimateController` extension** — adds `POST /v1/estimates/:id/send` handler.
- **`EstimateTransitionService.ALLOWED_TRANSITIONS`** — three new entries (D-TKN-09). Test updates mirror.
- **`DocumentTokenRepository` extension** — two new methods (D-TKN-02, D-TKN-03). Spec updates.
- **`estimate` entity extension** — `lastSentAt?: Date`, `lastSentTo?: string`. `EstimateRepository.toDto` maps these through `toOptionalDateTime`. DTO / response interfaces gain the fields.
- **`EstimateActionStrip`** — Phase 43 stub extended with Send / Re-send buttons.
- **`EstimateDetailPage`** — Phase 43 page extended with `useGetEstimateSettingsQuery` + `<SendEstimateDialog>` integration.
- **`estimateApi.ts`** — Phase 43 file extended with `sendEstimate` mutation.
- **`businessApi.ts`** — extended with `getEstimateSettings` / `updateEstimateSettings` hooks.
- **`BusinessDetails.tsx`** — tab rename + content restructure.
- **`features/business/components/index.tsx`** — `EstimateEmailSettings` export added.
- **`openapi.yaml`** — 3 new endpoints, 3 new schemas, updated `IEstimateResponse`.
- **`ROADMAP.md` / `v1.8-ROADMAP.md` / `REQUIREMENTS.md`** — SND-02 and Phase 44 SC #2 wording updated from "Templates" to "Documents".

### Known Pitfalls That Apply

- **Resend SDK failure modes** — recent quick-task 260407-n0u mapped Resend errors to proper HTTP response codes. Phase 44's `EstimateEmailSender` reuses the same `EmailSenderService` so error mapping is already in place. Planner verifies.
- **Maizzle dynamic import fallback** — `quote-email-renderer.service.ts` uses a `Function('return import(...)')` eval to work around CommonJS/ESM interop. `EstimateEmailRenderer` copies this verbatim (D-TPL-01). The fallback to inline-styled HTML on Maizzle failure is the same safety net.
- **Token reuse + revision chain** — storing `rootEstimateId` as the token's `documentId` (D-TKN-01) is a deliberate schema choice that commits Phase 44 and Phase 45 to the same chain-resolution model. Any future phase that wants per-revision tokens (e.g. for audit of "which revision did the customer actually see first") would require reopening this.
- **Follow-up cancellation on pure re-send** — D-TKN-06 explicitly does NOT cancel follow-ups on a pure re-send. Rationale: same revision, same schedule, the re-send is a reminder. A buggy cancellation here would silently lose Phase 46 follow-ups. The integration test matrix must cover pure-re-send + revised-send + first-send separately.
- **Subject validation regression risk** — adding "subject must contain 'Estimate'" validation could break existing quote flows if the validator is accidentally applied at the shared-helper level. D-SND-04 keeps the validator scoped to estimate requests only. Planner must not lift the subject validator into a shared helper that quote requests also use.
- **`BusinessCreator` double-creator path** — if `DefaultEstimateSettingsCreatorService.create` fails after `DefaultQuoteSettingsCreatorService.create` succeeds, the business is in a partially-initialised state (has quote settings, no estimate settings). Planner decides whether to wrap both creators in a try/catch-and-compensate block or accept the risk. Recommendation: accept — the failure would surface immediately to the user, the business record exists, and a retry of the PATCH would fix it. The `BusinessCreator` today does not use transactions for the quote-settings creator call.
- **Template variable escaping** — the Maizzle template uses `{{{ message }}}` (triple-brace) for the raw HTML body and `{{ field }}` (double-brace) for escaped values. Phase 44 must escape the `priceDisplay`, `contingencyNote`, `uncertaintyNotes` fields via the `escapeHtml` helper in the renderer because they pass through `{{ }}` not `{{{ }}}`.

</code_context>

<specifics>
## Specific Ideas

### Legal disclaimer card visual design

The disclaimer is a **visually distinct card** at the top of the email, not a plain paragraph. Approximate Maizzle markup:

```html
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0"
       class="bg-amber-50 border border-amber-200 rounded-md"
       style="background-color: #fffbeb; border: 1px solid #fcd34d; border-radius: 6px; margin-bottom: 24px;">
  <tr>
    <td style="padding: 16px;">
      <p style="margin: 0; font-size: 14px; line-height: 1.5; color: #78350f; font-weight: 600;">
        This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit.
      </p>
    </td>
  </tr>
</table>
```

Pale amber background + warning-colour text reads as "legal notice, please read" without being alarming. Planner may refine the exact colour tokens against the trade-flow-ui design system.

### System default subject and message for estimate-settings

**Subject:** `"Estimate {{estimateNumber}} for {{jobTitle}}"`
**Message (plain text, converted to HTML paragraphs client-side):**

```
Hi {{customerName}},

Please find your estimate {{estimateNumber}} for {{jobTitle}}. This is a guide price — a firm quote will follow after a site visit.

Kind regards,
{{userName}}
```

The message deliberately echoes the disclaimer in plain English ("a guide price — a firm quote will follow") without repeating the exact legal wording. The disclaimer carries the legal weight; the personal message carries the conversational tone. Both together reinforce the non-binding nature.

### Estimate email summary card

```
Estimate: E-2026-001
Date: 15 April 2026
Price: £120.00 - £132.00           ← bold, headline number
Includes a 10% uncertainty buffer  ← small caption, only if contingency > 0
Notes: Customer mentioned older    ← only if uncertaintyNotes non-empty
       pipes may be lead, would
       confirm on site visit.
```

The `Price:` line uses `formatRange(priceRange.low, priceRange.high, displayMode, currencyCode)` server-side. In `from` mode it renders `From £120.00`. In `range` mode it renders `£120.00 - £132.00`. The server utility `format-range.utility.ts` is byte-identical to the Phase 43 frontend `formatRange` helper — this is tested indirectly by the D-GATE-03 structural test.

### Pure re-send vs revised send vs first send — state table

| Incoming state | Path | Token action | Audit row type | Follow-up cancellation | Status transition |
|----------------|------|---------------|----------------|-------------------------|-------------------|
| Draft, revisionNumber=1 | First send | Create | INITIAL | No (nothing scheduled yet) | Draft → Sent |
| Draft, revisionNumber>1 | Revised send | Reuse + extend expiry | INITIAL | YES — cancel previous revision's follow-ups via IEstimateFollowupCanceller | Draft → Sent |
| Sent | Pure re-send | Reuse + extend expiry | RESEND | No | Sent → Sent (no-op) |
| Viewed | Pure re-send | Reuse + extend expiry | RESEND | No | Viewed → Sent (new) |
| Responded | Pure re-send | Reuse + extend expiry | RESEND | No | Responded → Sent (new) |
| SiteVisitRequested | Pure re-send | Reuse + extend expiry | RESEND | No | SiteVisitRequested → Sent (new) |
| Converted / Declined / Expired / Lost / Deleted | — | Rejected | — | — | Rejected with ESTIMATE_NOT_SENDABLE |

### Test matrix for D-GATE-03 (disclaimer presence)

1. Default message empty → rendered HTML contains SND-05 wording.
2. Default message = "Hello" → rendered HTML contains SND-05 wording AND "Hello".
3. Large paragraph as default message → both present.
4. Message contains HTML comment `<!-- -->` wrapping the disclaimer placeholder → disclaimer still present (because the disclaimer is NOT a placeholder, it's static template content).
5. Message contains the exact disclaimer string verbatim → disclaimer appears twice (once from user, once from template). This is acceptable — the legal one is present and structural.

### estimate_email_sends collection usage beyond Phase 44

Phase 46 follow-up processors (3/10/21 day) will append rows with `type: FOLLOWUP_3D` / `FOLLOWUP_10D` / `FOLLOWUP_21D`. Querying "all emails ever sent for this estimate chain" becomes `db.estimate_email_sends.find({rootEstimateId}).sort({sentAt: -1})`. A future History panel on the trader's detail page can render this chronologically without Phase 44 shipping any read endpoint today.

### BusinessCreator modification — exact shape

Approximate diff:

```typescript
// BusinessCreator.create()
constructor(
  // ...
  private readonly defaultQuoteSettingsCreator: DefaultQuoteSettingsCreatorService,
  private readonly defaultEstimateSettingsCreator: DefaultEstimateSettingsCreatorService,  // NEW
) {}

public async create(authUser, dto): Promise<IBusinessDto> {
  const createdBusiness = await this.repository.create(dto);
  await this.defaultQuoteSettingsCreator.create(createdBusiness.id);
  await this.defaultEstimateSettingsCreator.create(createdBusiness.id);  // NEW
  return createdBusiness;
}
```

The test `business-creator.service.spec.ts` gains a new `jest.Mocked<DefaultEstimateSettingsCreatorService>` provider and an assertion `expect(defaultEstimateSettingsCreator.create).toHaveBeenCalledWith(createdBusiness.id)` mirroring line 226.

</specifics>

<deferred>
## Deferred Ideas

- **Onboarding step parity for estimates** — `QuoteEmailSender` marks `FIRST_QUOTE_SENT` on first send. Phase 44 does NOT add `FIRST_ESTIMATE_SENT`. Estimates are a secondary flow in v1.8; adding a parallel onboarding milestone now would split attention and complicate the onboarding wizard. Future phase can add it alongside a "first estimate sent" onboarding card.
- **Public read endpoint for `estimate_email_sends`** — no HTTP surface in Phase 44. A future admin/audit tool or a trader-facing "email history" panel would need a GET endpoint. Repository methods exist; controller does not.
- **Lifting `sanitizeHtml` into `@core/utilities`** — D-SND-04 ships it duplicated per D-01. A follow-up refactor could consolidate if a third consumer appears (invoice send? customer notification?). Not blocking.
- **Subject-line editor with a "must contain Estimate" hint in the UI** — Phase 44 does server-side validation + catch-and-display pattern. A cleaner UX would be a client-side validator with a tooltip ("Subject must include the word 'Estimate'"). Planner may add during implementation if trivial; otherwise defer.
- **Chip reasons surfaced in the email body** — D-TPL-04 decided against. If legal review requests it for transparency, add in a follow-up. Requires mapping `UNCERTAINTY_CHIP_LABELS` to the email template data.
- **Token revocation path** — no UI to revoke a token in Phase 44. D-TKN-05 handles the edge case (fall through to create). Future phase could add an admin-level revoke.
- **TTL index on `estimate_email_sends`** — D-AUDIT-06 deliberately no TTL. If storage becomes a concern post-launch, add a long TTL (5-7 years to cover UK limitation period).
- **Compensating transaction in BusinessCreator double-creator path** — the "partially initialised business" failure mode is accepted. A future refactor could wrap both default-settings creators in a compensating block. Not blocking; the quote-settings side today also doesn't compensate.
- **Test harness script that dumps a Phase 41 real API response for `estimate-sent.html` golden-file comparison** — Phase 43 D-FMT-04 uses a copy-pasted fixture. Phase 44 could extend the same fixture for email rendering tests. Planner's call at plan time.
- **Follow-up template architecture** — Phase 46 will ship `estimate-followup-3d.html` / `-10d.html` / `-21d.html`. The shape they share with `estimate-sent.html` is a potential shared-layout concern. Phase 44 does not pre-build an abstraction; Phase 46 decides at that time.
- **Onboarding `FIRST_ESTIMATE_SENT` OR generic `FIRST_DOCUMENT_SENT`** — if the product team wants one metric covering both, a future phase could introduce a document-type-agnostic onboarding step.

### Reviewed Todos (not folded)
None — `.planning/STATE.md` shows "Pending Todos: None."

</deferred>

---

*Phase: 44-email-send-flow*
*Context gathered: 2026-04-11*
