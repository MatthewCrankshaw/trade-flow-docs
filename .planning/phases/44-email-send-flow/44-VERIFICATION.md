---
phase: 44-email-send-flow
verified: 2026-04-13T00:00:00Z
status: human_needed
score: 7/7 must-haves verified
overrides_applied: 0
deferred:
  - truth: "Re-sending from SITE_VISIT_REQUESTED status delivers email without creating a revision"
    addressed_in: "Phase 45"
    evidence: "EstimateStatus enum has 9 values — SITE_VISIT_REQUESTED was planned in Phase 41 but not implemented in the enum. Phase 45 (customer-facing public page) adds the SiteVisitRequested response path which will require the enum value and transition, at which point Phase 44's sender can be extended."
human_verification:
  - test: "Send estimate email end-to-end"
    expected: "Customer receives an email with the correct subject, business name, price range, and a working 'View Estimate Online' link. Legal disclaimer is visible. Link resolves to the public estimate page."
    why_human: "Requires live Resend API key, live MongoDB, and a browser to follow the token link. Cannot be verified programmatically without a running environment."
  - test: "Re-send from Sent/Viewed/Responded statuses"
    expected: "Re-send delivers email, reuses the existing document token (same URL), extends token expiry 30 days, and creates an audit row with type=RESEND. The estimate status remains unchanged (no new revision created)."
    why_human: "Requires live environment to verify token reuse vs new token creation, and to confirm Resend delivers the email."
  - test: "Subject validation in send dialog"
    expected: "Removing 'Estimate' from the subject field in SendEstimateForm shows an inline hint 'Subject must contain the word Estimate'. Submitting a subject without 'Estimate' is rejected by the API with a 422 error."
    why_human: "Client-side hint requires browser interaction. Server-side rejection requires a running API with ValidationPipe active."
  - test: "Business > Documents tab layout"
    expected: "Navigating to Business settings shows a 'Documents' tab (not 'Quote Email'). The tab body shows the Quote Email Template card followed immediately by the Estimate Email Template card, both in a vertical flex stack. The Estimate card shows the disclaimer info note."
    why_human: "Visual layout and tab rendering require a running UI."
---

# Phase 44: Email Send Flow — Verification Report

**Phase Goal:** Deliver the complete estimate send pipeline — backend send endpoint with token reuse, mandatory legal disclaimer, audit persistence, and the frontend send dialog plus Documents tab with parallel estimate email template management.

**Verified:** 2026-04-13

**Status:** HUMAN_NEEDED — all automated checks pass; live email delivery and UI rendering require human confirmation.

**Re-verification:** No — initial verification.

---

## Goal Achievement

Phase 44's goal is achieved at the code level. All seven SND requirements are implemented and wired. CI was green in both repositories at completion. Human verification is required to confirm live email delivery and UI visual correctness before the phase can be fully signed off.

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | GET /v1/business/:businessId/estimate-settings returns estimate settings | VERIFIED | Controller at `estimate-settings.controller.ts` line 21: `@Get("business/:businessId/estimate-settings")` with JwtAuthGuard. Null-safe stub returned when no row exists (D-SET-04 compliant). |
| 2 | PATCH /v1/business/:businessId/estimate-settings updates subject/body | VERIFIED | Controller line 49: `@Patch("business/:businessId/estimate-settings")`. Request class uses `@Matches(/estimate/i)` on subject field. |
| 3 | PATCH rejects subject missing "Estimate" (case-insensitive) | VERIFIED | `update-estimate-settings.request.ts`: `@Matches(/estimate/i, { message: "Estimate email subject must contain the word 'Estimate'" })` confirmed on `estimateEmailSubject` field. |
| 4 | Creating a new business creates default estimate-settings row | VERIFIED | `business-creator.service.ts` line 56: `await this.defaultEstimateSettingsCreator.create(createdBusiness.id)` called sequentially after quote settings. |
| 5 | estimate-settings module is independent of quote-settings module | VERIFIED | `estimate-settings.module.ts` imports only CoreModule, BusinessModule, and UserModule — no import of QuoteSettingsModule. Separate collection: `estimate_settings`. |
| 6 | POST /v1/estimates/:id/send sends email and transitions Draft to Sent | VERIFIED | `estimate.controller.ts` line 144: `@Post("estimates/:id/send")`. EstimateEmailSender orchestrates: authorize → token → render → Resend delivery → audit → transition → updateSendFields → extendExpiry. |
| 7 | Re-sending a Sent/Viewed/Responded estimate delivers email without creating a revision | VERIFIED (with deviation) | `determineSendPath()` handles SENT, VIEWED, RESPONDED as RESEND sendType. SITE_VISIT_REQUESTED is excluded because that status does not exist in the enum (Phase 41 gap — see Deferred Items). |
| 8 | Token is reused on re-send and expiry is extended to 30 days from now | VERIFIED | Sender calls `documentTokenRepository.findActiveByDocumentId(rootId, "estimate")` — reuses existing token if found, creates new only if null. `extendExpiry(token.id, DateTime.now().plus({ days: 30 }))` called post-send. |
| 9 | Rendered HTML is persisted in estimate_email_sends before status transition | VERIFIED | `estimateEmailSendCreator.create({ ... html ... })` called before `estimateTransitionService.transition()` in the post-send block. Write-only repository with no update/delete methods. |
| 10 | Legal disclaimer appears in every rendered email regardless of message content | VERIFIED | `estimate-sent.html` contains static disclaimer HTML. 4 structural tests (D-GATE-03) verify presence with empty message, simple message, large paragraph, and HTML comment injection. |
| 11 | Documents tab shows both Quote and Estimate email templates stacked vertically | VERIFIED | `BusinessDetails.tsx`: `TabsTrigger value="documents"`, `TabsContent value="documents"` containing `<div className="flex flex-col gap-6">` with both `<QuoteEmailSettings>` and `<EstimateEmailSettings>`. |
| 12 | SendEstimateForm pre-fills subject and message from estimate-settings | VERIFIED | `SendEstimateForm.tsx` uses `defaultSubject`/`defaultMessage` props resolved from settings, falls back to `DEFAULT_SUBJECT = "Estimate {{estimateNumber}} for {{jobTitle}}"`. Template variables resolved client-side on mount. |

**Score:** 7/7 SND requirements verified. 12/12 supporting truths confirmed.

### Deferred Items

| # | Item | Addressed In | Evidence |
|---|------|-------------|----------|
| 1 | Re-send from SITE_VISIT_REQUESTED status | Phase 45 | `EstimateStatus` enum has 9 values — SITE_VISIT_REQUESTED was designed in Phase 41 plan but not added to the actual enum (`estimate-transitions.spec.ts` asserts `ALLOWED_TRANSITIONS.size === 9`). Phase 45 ships the customer-facing public page with SiteVisitRequested response flow, which will add the status and require extending the sender. Not a Phase 44 regression. |

---

## Requirements Coverage

| Requirement | Plan | Description | Status | Evidence |
|-------------|------|-------------|--------|----------|
| SND-01 | 44-03 | User can send estimate via email with secure public link | SATISFIED | POST /v1/estimates/:id/send endpoint wired. EstimateEmailSender creates/reuses DocumentToken, calls EmailSenderService, persists audit row. |
| SND-02 | 44-01 + 44-04 | estimate-settings module API + Business > Documents tab | SATISFIED | Backend: GET/PATCH /v1/business/:businessId/estimate-settings with independent collection. Frontend: Documents tab with both templates stacked via `flex flex-col gap-6`. |
| SND-03 | 44-04 | Send dialog with editable subject and rich-text body | SATISFIED | SendEstimateDialog shell + SendEstimateForm with To/Subject/RichTextEditor fields, template variable resolution, and subject validation hint. |
| SND-04 | 44-02 + 44-03 | Rendered email HTML persisted at send time | SATISFIED | `estimate_email_sends` write-only collection. `EstimateEmailSendCreator.create()` called with full rendered HTML before status transition. |
| SND-05 | 44-02 | Mandatory non-binding legal disclaimer in email template | SATISFIED | Static disclaimer in `estimate-sent.html`: "This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit." 4 structural tests verify invariant. |
| SND-06 | 44-03 | Re-send without creating a new revision | SATISFIED | `determineSendPath()` returns sendType=RESEND for SENT/VIEWED/RESPONDED. No revision creation on re-send path. Token reused. SITE_VISIT_REQUESTED excluded (pre-existing Phase 41 gap — status not in enum). |
| SND-07 | 44-01 + 44-03 | Subject must contain "Estimate" | SATISFIED | `@Matches(/estimate/i)` on both `UpdateEstimateSettingsRequest.estimateEmailSubject` and `SendEstimateRequest.subject`. Client-side hint in SendEstimateForm. |

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/estimate-settings/estimate-settings.module.ts` | NestJS module with @Module | VERIFIED | Contains `@Module`, exports `EstimateSettingsRetriever` and `DefaultEstimateSettingsCreatorService`. Uses forwardRef for circular dependency resolution. |
| `trade-flow-api/src/estimate-settings/controllers/estimate-settings.controller.ts` | GET + PATCH endpoints | VERIFIED | Both endpoints present with JwtAuthGuard. Null-safe stub for missing settings. |
| `trade-flow-api/src/estimate-settings/requests/update-estimate-settings.request.ts` | @Matches(/estimate/i) validation | VERIFIED | Exact pattern confirmed in source. |
| `trade-flow-api/src/estimate-settings/repositories/estimate-settings.repository.ts` | estimate_settings collection | VERIFIED | `private static readonly COLLECTION = "estimate_settings"` confirmed. Upsert pattern present. |
| `trade-flow-api/src/email/templates/estimate-sent.html` | Legal disclaimer + template variables | VERIFIED | Contains static disclaimer text, `{{{ message }}}` triple-brace, `{{ viewUrl }}`, "View Estimate Online" CTA. |
| `trade-flow-api/src/email/services/estimate-email-renderer.service.ts` | EstimateEmailRenderer + ESM interop | VERIFIED (from SUMMARY) | ESM interop pattern confirmed in summary self-check. 18 tests including 4 structural disclaimer tests all pass. |
| `trade-flow-api/src/estimate/utilities/format-range.utility.ts` | formatRange pure function | VERIFIED | `range` mode: `"£X.XX - £Y.YY"`, `from` mode: `"From £X.XX"`. 4 test cases all pass per summary. |
| `trade-flow-api/src/estimate/repositories/estimate-email-send.repository.ts` | Write-only audit repo, estimate_email_sends | VERIFIED | `COLLECTION = "estimate_email_sends"`, 3 indexes created via OnModuleInit, no update/delete methods, create + findAllBy* only. |
| `trade-flow-api/src/document-token/repositories/document-token.repository.ts` | findActiveByDocumentId + extendExpiry | VERIFIED | Both methods confirmed at lines 78 and 100 via grep. |
| `trade-flow-api/src/estimate/services/estimate-email-sender.service.ts` | Full send orchestrator | VERIFIED | Contains: `findActiveByDocumentId`, `extendExpiry`, `cancelAllFollowups`, `sanitizeHtml`, `ESTIMATE_FOLLOWUP_CANCELLER`, `estimatePolicy`, `import { formatRange }`. |
| `trade-flow-api/src/estimate/requests/send-estimate.request.ts` | @Matches subject validation | VERIFIED | `@Matches(/estimate/i)` confirmed on `subject` field. |
| `trade-flow-api/src/estimate/controllers/estimate.controller.ts` | POST send endpoint | VERIFIED | `@Post("estimates/:id/send")` confirmed at line 144. `estimateEmailSender.send` called in handler. |
| `trade-flow-ui/src/features/business/components/EstimateEmailSettings.tsx` | Estimate template card with disclaimer note | VERIFIED | "Estimate Email Template" title confirmed, "The non-binding legal disclaimer is added automatically and cannot be edited." confirmed. `useGetEstimateSettingsQuery` + `useUpdateEstimateSettingsMutation` hooks used. |
| `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` | Documents tab with both templates | VERIFIED | `value="documents"`, "Documents" label, `flex flex-col gap-6` containing both `<QuoteEmailSettings>` and `<EstimateEmailSettings>`. |
| `trade-flow-ui/src/features/estimates/components/SendEstimateDialog.tsx` | Dialog shell | VERIFIED | "Send Estimate" title, "Review and send this estimate to your customer." description. `{open && <SendEstimateForm />}` guard prevents stale state. |
| `trade-flow-ui/src/features/estimates/components/SendEstimateForm.tsx` | Full send form | VERIFIED | `useSendEstimateMutation`, DEFAULT_SUBJECT, subject hint, "No email on file" warning, "Save email to customer record" checkbox all confirmed. |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | Send/Re-send buttons | VERIFIED | "Send Estimate" (draft), "Re-send" (sent/viewed/responded/site_visit_requested), `SendEstimateDialog` at bottom. |
| `trade-flow-ui/src/features/estimates/api/estimateApi.ts` | sendEstimate mutation | VERIFIED | `/v1/estimates/${estimateId}/send` POST, `useSendEstimateMutation` exported. |
| `trade-flow-ui/src/pages/EstimateDetailPage.tsx` | estimate settings query wiring | VERIFIED | `useGetEstimateSettingsQuery(businessId!, { skip: !businessId })` at line 314. |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `business-creator.service.ts` | DefaultEstimateSettingsCreatorService | DI injection + `.create()` call | WIRED | Line 56: `await this.defaultEstimateSettingsCreator.create(createdBusiness.id)` |
| `app.module.ts` | EstimateSettingsModule | imports array | WIRED | `EstimateSettingsModule` present in AppModule imports at line 57 |
| `estimate.controller.ts` | EstimateEmailSender | DI injection + `.send()` call | WIRED | `estimateEmailSender.send(request.user, request.params.id, body)` in send handler |
| `estimate-email-sender.service.ts` | EstimateEmailRenderer | DI + `.render()` call | WIRED | `await this.estimateEmailRenderer.render({ ... })` at line 89 |
| `estimate-email-sender.service.ts` | EstimateEmailSendCreator | DI + `.create()` call | WIRED | `await this.estimateEmailSendCreator.create({ ... })` post-send |
| `estimate-email-sender.service.ts` | DocumentTokenRepository | findActiveByDocumentId + extendExpiry | WIRED | Both calls confirmed in source |
| `EstimateActionStrip.tsx` | SendEstimateDialog | open state toggle | WIRED | `SendEstimateDialog` rendered at bottom, controlled by `sendDialogOpen` state |
| `SendEstimateForm.tsx` | estimateApi.ts | useSendEstimateMutation | WIRED | Hook imported and called on form submit |
| `EstimateDetailPage.tsx` | businessApi.ts | useGetEstimateSettingsQuery | WIRED | Hook called at line 314 with businessId |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|--------------------|--------|
| `EstimateEmailSettings.tsx` | `estimateSettings` | `useGetEstimateSettingsQuery` → GET /v1/business/:id/estimate-settings | Yes — repository queries `estimate_settings` collection, null-safe stub for missing rows | FLOWING |
| `SendEstimateForm.tsx` | `defaultSubject`, `defaultMessage` | `EstimateDetailPage` → `useGetEstimateSettingsQuery` props | Yes — settings from API propagated through EstimateEditor → EstimateActionStrip → SendEstimateDialog → form | FLOWING |
| `estimate-email-sender.service.ts` | `html` | `estimateEmailRenderer.render()` → Maizzle template + real estimate data | Yes — estimate entity data, business name from DB, formatRange computed from real price range | FLOWING |
| `estimate_email_sends` | `html` field | Full rendered HTML from render step | Yes — write-only create, no static returns | FLOWING |

---

## Behavioral Spot-Checks

Step 7b: SKIPPED — requires running servers (NestJS API + MongoDB + Resend) for the send flow. Checks delegated to human verification section.

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `EstimateDetailPage.tsx` | 271 | "No customer response yet. Response handling ships in Phase 45." placeholder card | INFO | Pre-existing Phase 43 stub, intentionally deferred to Phase 45. Not introduced by Phase 44. |

No new stubs or TODO/FIXME/placeholder patterns introduced by Phase 44.

---

## CI Confirmation

Both repositories reported green CI at phase completion:

- **trade-flow-api:** 730 tests across 88 suites — all PASS. Lint: 0 errors. Format: clean. Typecheck: zero errors.
- **trade-flow-ui:** 94 tests across 12 test files — all PASS. Lint: 0 errors (1 pre-existing warning in BusinessStep.tsx, out of scope). Format: clean. TypeScript: zero errors.

Commit hashes documented in summaries:
- 44-01 API: `500e943`, `ab70586`, `3a391e6`
- 44-02 API: committed as part of 44-03 wave (44-02 parallel agent created files but did not commit all)
- 44-03 API: `4796c54`, `594374c`, `96c7c0f`, `7aee5f5`
- 44-04 UI: `7dc41f3`, `09a7699`

---

## Human Verification Required

### 1. Live Email Delivery

**Test:** Create a test estimate in Draft status. Click "Send Estimate" in the action strip. Fill in a recipient email, an "Estimate..." subject, and a short message. Submit.

**Expected:** Recipient receives an HTML email containing: business name header, amber disclaimer card ("This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit."), the composed message, a "View Estimate Online" button that navigates to `/estimate/{token}` and resolves correctly, estimate number and price range in the summary card.

**Why human:** Requires live Resend API key, MongoDB, and a browser to verify email content and link resolution.

### 2. Re-send Behaviour

**Test:** After sending an estimate (it moves to Sent status), click "Re-send". Submit with the same or different recipient.

**Expected:** Email is delivered. The token URL is the same as the initial send (token reused). Estimate status remains Sent (no status change). An audit row appears in `estimate_email_sends` with `type = "resend"`. Token expiry is extended.

**Why human:** Requires live environment to verify token identity, audit trail, and email receipt.

### 3. Subject Validation

**Test:** Open SendEstimateDialog. Clear "Estimate" from the subject field.

**Expected:** An inline hint "Subject must contain the word 'Estimate'" appears below the field immediately. Attempting to submit triggers a 422 from the API.

**Why human:** Browser interaction required for client-side hint; running API required for server rejection.

### 4. Business > Documents Tab

**Test:** Navigate to Business settings. Click the "Documents" tab (previously called "Quote Email").

**Expected:** Tab renders two vertically stacked cards: "Quote Email Template" above, "Estimate Email Template" below. The Estimate card shows a blue info alert with "The non-binding legal disclaimer is added automatically and cannot be edited." Saving a template in either card does not affect the other.

**Why human:** Visual rendering and tab layout require a running UI.

---

## Gaps Summary

No blocking gaps. All SND-01 through SND-07 requirements are implemented and wired in both repositories. One pre-existing Phase 41 issue (SITE_VISIT_REQUESTED status never added to the enum) means re-send from that specific status is not handled; this is deferred to Phase 45 which ships the full customer response flow requiring that status. Phase 44's SND-06 commitment ("re-send from Sent or revised status") is met for all statuses that actually exist.

---

_Verified: 2026-04-13_
_Verifier: Claude (gsd-verifier)_
