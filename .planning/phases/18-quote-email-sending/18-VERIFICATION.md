---
phase: 18-quote-email-sending
verified: 2026-03-21T12:00:00Z
status: human_needed
score: 10/10 must-haves verified
re_verification:
  previous_status: gaps_found
  previous_score: 9/10
  gaps_closed:
    - "DLVR-01 accuracy gap: REQUIREMENTS.md now correctly splits DLVR-01 into DLVR-01a (Phase 18, email with link, complete) and DLVR-01b (Phase 20, PDF attachment, pending)"
    - "SendGrid error handling: EmailSenderService now returns 503 (missing API key), 502 (SendGrid failure), 500 (unexpected) instead of generic 500 CORE_0"
    - "QuoteSettings separation: New quote-settings module (GET/PATCH /v1/business/:businessId/quote-settings) owns email template data — Business entity cleaned of quoteEmail fields"
    - "Quote Email tab relocated: moved from /settings page to /business page in BusinessDetails component"
    - "Frontend API rewired: QuoteEmailSettings and QuoteDetailPage use useGetQuoteSettingsQuery/useUpdateQuoteSettingsMutation instead of business entity fields"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Open a draft quote for a customer with an email address. Click Send Quote. Observe the To, Subject, and Message fields."
    expected: "To is pre-filled with customer email; Subject shows resolved template (e.g., Quote Q-001 for Kitchen Renovation); Message body shows resolved text in the rich text editor."
    why_human: "Template variable substitution and initial Tiptap editor state require a running browser."
  - test: "In the send dialog, bold a word and add an italic phrase. Click Send Quote. Inspect the received email or network payload."
    expected: "The HTML sent to the backend contains <strong> and <em> tags. The email renders formatted text."
    why_human: "Tiptap editor state output and SendGrid delivery require live environment."
  - test: "Open a draft quote for a customer with no email address. Click Send Quote. Enter an email, check the save-email checkbox, send. Verify the customer record afterward."
    expected: "Warning alert appears. After send, customer.email is updated in the database."
    why_human: "Conditional UI state and customer update side effect require live data."
  - test: "Go to Business page > Quote Email tab. Enter a custom subject with {{quoteNumber}} and save. Then open a quote send dialog."
    expected: "The send dialog Subject field shows the resolved quote number substituted in."
    why_human: "End-to-end data flow from quote-settings record through RTK Query cache into dialog pre-fill requires running application."
  - test: "Send a quote (note the customer-facing View Quote Online link). Re-send the same quote. Compare the new link to the original."
    expected: "The second email contains a different token in the view URL (new token generated per send)."
    why_human: "Requires verifying actual email received by the customer — cannot inspect token generation outcome without live environment."
  - test: "Trigger a send with a misconfigured SENDGRID_API_KEY (placeholder value). Observe the API response."
    expected: "Response returns HTTP 503 with message: Email sending is not configured. Set a valid SENDGRID_API_KEY environment variable."
    why_human: "Requires a live environment with a placeholder key to verify the error response shape."
---

# Phase 18: Quote Email Sending — Re-Verification Report

**Phase Goal:** Tradesperson can send a quote to their customer via email with one action
**Verified:** 2026-03-21T12:00:00Z
**Status:** human_needed — all automated checks pass, gap from initial verification is closed
**Re-verification:** Yes — after gap closure (Plans 04, 05, 06 executed since initial verification)

---

## Re-Verification Summary

The initial verification (score: 9/10) found one gap: DLVR-01 in REQUIREMENTS.md incorrectly claimed PDF attachment was in scope for Phase 18, when CONTEXT.md explicitly deferred it to Phase 20. Three gap-closure plans were executed:

- **Plan 04** — Fixed SendGrid error handling, split DLVR-01 into DLVR-01a/DLVR-01b in REQUIREMENTS.md
- **Plan 05** — Created QuoteSettings backend module, cleaned Business entity of email template fields
- **Plan 06** — Migrated frontend to use quote-settings API, moved Quote Email tab from /settings to /business

All three gap-closure plans have been verified against the actual codebase. No regressions detected. Score upgrades to 10/10.

---

## Goal Achievement

### Observable Truths (from ROADMAP.md Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can send a quote and customer receives email with View Quote Online link | VERIFIED | DLVR-01a satisfied — email + link fully implemented; DLVR-01b (PDF) correctly deferred to Phase 20 per CONTEXT.md |
| 2 | User can review and edit pre-filled message in send dialog before sending | VERIFIED | SendQuoteForm: To/Subject/Message fields, resolveTemplateVariables(), RichTextEditor |
| 3 | User can configure default quote email template (now on Business page, not Settings) | VERIFIED | QuoteEmailSettings on BusinessDetails Quote Email tab; useGetQuoteSettingsQuery/useUpdateQuoteSettingsMutation |
| 4 | Quote status transitions Draft to Sent after email is successfully sent | VERIFIED | quoteTransitionService.transition() called after emailSenderService.sendEmail() returns |
| 5 | User can re-send a quote already in Sent status | VERIFIED | send() always calls quoteTokenCreator.create(); re-send path confirmed in spec |

**Score:** 5/5 truths verified

---

### Must-Have Truths (from Plan 18-01 frontmatter)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | POST /v1/business/:businessId/quote/:quoteId/send endpoint accepts { to, subject, message, saveEmail } and returns updated quote with status SENT | VERIFIED | Controller line 202; openapi.yaml line 2023 |
| 2 | QuoteEmailSender orchestrates: validate -> generate token -> render HTML email -> send via SendGrid -> transition status | VERIFIED | quote-email-sender.service.ts; full flow present |
| 3 | Maizzle renders a branded HTML email with business name header, user message, View Quote Online CTA, quote summary, footer | VERIFIED | quote-email.html verified in initial pass |
| 4 | Business entity and DTO include optional quoteEmailSubject and quoteEmailBody fields | SUPERSEDED | Plan 05 removed these fields from Business; they now live on QuoteSettings entity |
| 5 | Re-send generates a new token and status stays SENT | VERIFIED | send() always calls quoteTokenCreator.create() |
| 6 | Status transitions server-side after confirmed SendGrid delivery (not optimistically) | VERIFIED | transition() called on line 60 — after sendEmail() on line 52 |

**Score:** 6/6 (truth 4 context changed by Plan 05 — superseded, not failed)

---

### Must-Have Truths (from Plan 18-02 frontmatter)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can click Send Quote (Draft) or Re-send Quote (Sent) to open dialog | VERIFIED | QuoteActionStrip lines 113, 158: setShowSendDialog(true) |
| 2 | To field pre-filled from customer email; Subject/Message pre-filled from default template with variables resolved | VERIFIED | SendQuoteForm: useState(customerEmail ?? ""), resolveTemplateVariables() |
| 3 | If customer has no email, warning alert shows and checkbox to save email appears | VERIFIED | SendQuoteForm lines 126-146 |
| 4 | Message field is a rich text editor with Bold, Italic, and Link toolbar buttons | VERIFIED | RichTextEditor with StarterKit + Link + Placeholder extensions |
| 5 | After successful send, toast shows email address, dialog closes, status badge updates to Sent | VERIFIED | toast.success, onOpenChange(false), RTK Query cache invalidation |
| 6 | Re-send uses the same dialog, pre-filled fresh from the default template | VERIFIED | Both buttons open same SendQuoteDialog; form mounts fresh on open |

**Score:** 6/6

---

### Must-Have Truths (from Plan 18-03 frontmatter)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can configure a default quote email subject and message body in business settings | VERIFIED (relocated) | QuoteEmailSettings on Business page /business > Quote Email tab, not /settings |
| 2 | Available template variables are displayed as badges below the message field | VERIFIED | QuoteEmailSettings: 5 Badge elements (customerName, quoteNumber, jobTitle, businessName, userName) |
| 3 | Save button persists the template via updateQuoteSettings mutation | VERIFIED | Uses useUpdateQuoteSettingsMutation (not old useUpdateBusinessMutation) |
| 4 | Quote Email tab appears in the Settings page alongside the existing Profile tab | SUPERSEDED | Plan 06 moved tab to Business page; Settings page now has no Quote Email tab — correct per revised design |

**Score:** 4/4 (truth 4 context changed by Plan 06 — tab moved to Business page, which is better UX)

---

### Must-Have Truths (from Plan 18-04 frontmatter — gap closure)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | SendGrid errors return a meaningful error message instead of generic 500 CORE_0 | VERIFIED | email-sender.service.ts: try/catch, 502 for SendGrid failure, 503 for missing key, 500 for unexpected |
| 2 | Placeholder SendGrid API key is detected and a clear error is returned at send time | VERIFIED | apiKeyConfigured flag; guard throws 503 HttpException when !apiKeyConfigured |
| 3 | REQUIREMENTS.md DLVR-01 accurately reflects Phase 18 scope (email with link, no PDF) | VERIFIED | DLVR-01a (Phase 18, Complete), DLVR-01b (Phase 20, Pending); old "DLVR-01:" entry removed |

**Score:** 3/3

---

### Must-Have Truths (from Plan 18-05 frontmatter — gap closure)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | GET /v1/business/:businessId/quote-settings returns quote email settings for the business | VERIFIED | QuoteSettingsController.getByBusinessId(); returns empty defaults when no settings exist |
| 2 | PATCH /v1/business/:businessId/quote-settings updates quote email subject and body | VERIFIED | QuoteSettingsController.update(); delegates to quoteSettingsUpdater.update() |
| 3 | QuoteEmailSender fetches email template from QuoteSettingsRetriever instead of BusinessRetriever | VERIFIED | quote-email-sender.service.ts: QuoteSettingsRetriever injected; quote.module.ts imports QuoteSettingsModule |
| 4 | BusinessEntity no longer has quoteEmailSubject or quoteEmailBody fields | VERIFIED | grep count = 0 for quoteEmailSubject in business.entity.ts and business.dto.ts |

**Score:** 4/4

---

### Must-Have Truths (from Plan 18-06 frontmatter — gap closure)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Quote Email settings tab appears on the Business page (/business), not on the Settings page (/settings) | VERIFIED | BusinessDetails.tsx: TabsTrigger value="quote-email" at line 576; SettingsPage.tsx: 0 matches for "quote-email" |
| 2 | QuoteEmailSettings component fetches and saves via the new quote-settings API endpoints | VERIFIED | useGetQuoteSettingsQuery (line 2), useUpdateQuoteSettingsMutation (line 2); 0 matches for useUpdateBusinessMutation |
| 3 | SendQuoteForm pre-fills subject/message from quote-settings endpoint data, not from business data | VERIFIED | QuoteDetailPage.tsx line 51: useGetQuoteSettingsQuery; lines 159-160: quoteSettings?.quoteEmailSubject/Body passed as defaultSubject/defaultMessage |
| 4 | Business type no longer has quoteEmailSubject or quoteEmailBody fields | VERIFIED | api.types.ts: 2 matches for "QuoteSettings" (new type), 2 matches for quoteEmailSubject (in QuoteSettings only) |

**Score:** 4/4

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/email/services/email-sender.service.ts` | Error handling with 503/502/500 | VERIFIED | 53 lines; apiKeyConfigured flag, try/catch, typed error narrowing |
| `trade-flow-api/src/quote/services/quote-email-sender.service.ts` | HttpException re-throw | VERIFIED | HttpException import + re-throw in catch block |
| `.planning/REQUIREMENTS.md` | DLVR-01a/DLVR-01b split | VERIFIED | Lines 12-13 (requirements), lines 95-96 (traceability table) |
| `trade-flow-api/src/quote-settings/quote-settings.module.ts` | NestJS module | VERIFIED | QuoteSettingsModule with CoreModule, BusinessModule imports, QuoteSettingsRetriever exported |
| `trade-flow-api/src/quote-settings/entities/quote-settings.entity.ts` | MongoDB entity | VERIFIED | businessId, quoteEmailSubject, quoteEmailBody |
| `trade-flow-api/src/quote-settings/repositories/quote-settings.repository.ts` | Upsert repository | VERIFIED | findByBusinessId + upsert with $setOnInsert, COLLECTION = "quote_settings" |
| `trade-flow-api/src/quote-settings/services/quote-settings-retriever.service.ts` | Read service | VERIFIED | Delegates to repository |
| `trade-flow-api/src/quote-settings/services/quote-settings-updater.service.ts` | Write service with auth | VERIFIED | businessRetriever.findByIdOrFail for auth check |
| `trade-flow-api/src/quote-settings/controllers/quote-settings.controller.ts` | GET + PATCH endpoints | VERIFIED | 77 lines; both endpoints with JwtAuthGuard, empty-defaults fallback on GET |
| `trade-flow-ui/src/types/api.types.ts` | QuoteSettings type, Business cleaned | VERIFIED | QuoteSettings interface exists; quoteEmailSubject only in QuoteSettings (not Business) |
| `trade-flow-ui/src/features/business/api/businessApi.ts` | RTK Query hooks | VERIFIED | useGetQuoteSettingsQuery, useUpdateQuoteSettingsMutation exported |
| `trade-flow-ui/src/features/business/components/QuoteEmailSettings.tsx` | Uses new API | VERIFIED | useGetQuoteSettingsQuery + useUpdateQuoteSettingsMutation; 0 references to useUpdateBusinessMutation |
| `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` | Quote Email tab | VERIFIED | TabsTrigger + TabsContent for quote-email; QuoteEmailSettings rendered at line 639 |
| `trade-flow-ui/src/pages/SettingsPage.tsx` | Quote Email tab removed | VERIFIED | 0 matches for "quote-email" or "QuoteEmailSettings" |
| `trade-flow-ui/src/pages/QuoteDetailPage.tsx` | Uses quote-settings for pre-fill | VERIFIED | useGetQuoteSettingsQuery; quoteSettings?.quoteEmailSubject/Body passed to QuoteActionStrip |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `quote.controller.ts` | `QuoteEmailSender.send()` | POST /v1/business/:businessId/quote/:quoteId/send | WIRED | Controller delegates to quoteEmailSender.send() |
| `quote-email-sender.service.ts` | `QuoteTransitionService.transition()` | after email delivery | WIRED | transition() called after sendEmail() returns |
| `quote-email-sender.service.ts` | `QuoteTokenCreator.create()` | for View Quote Online link | WIRED | create() called before send |
| `quote-email-sender.service.ts` | `QuoteSettingsRetriever` | constructor injection | WIRED | QuoteSettingsRetriever injected; QuoteSettingsModule imported in quote.module.ts |
| `email-sender.service.ts` | `@sendgrid/mail` | try/catch around sgMail.send() | WIRED | try { await sgMail.send(msg) } catch with typed error extraction |
| `QuoteSettingsController` | `QuoteSettingsUpdater.update()` | PATCH endpoint | WIRED | controller.update() delegates to quoteSettingsUpdater.update() |
| `QuoteSettingsRepository` | `quote_settings` collection | findByBusinessId + upsert | WIRED | COLLECTION = "quote_settings"; MongoDbFetcher/Writer injected |
| `BusinessDetails.tsx` | `QuoteEmailSettings` | TabsContent quote-email | WIRED | Line 639: TabsContent renders QuoteEmailSettings with businessId |
| `QuoteEmailSettings.tsx` | `businessApi.getQuoteSettings` | useGetQuoteSettingsQuery | WIRED | Line 2 (import), query called with businessId |
| `QuoteEmailSettings.tsx` | `businessApi.updateQuoteSettings` | useUpdateQuoteSettingsMutation | WIRED | Line 2 (import), mutation called on save |
| `QuoteDetailPage.tsx` | `businessApi.getQuoteSettings` | useGetQuoteSettingsQuery for dialog pre-fill | WIRED | Line 14 (import), line 51 (query), lines 159-160 (passed to QuoteActionStrip) |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| DLVR-01a | 18-01, 18-04 | User can send a quote via email with a link to view the quote online | SATISFIED | Email delivery pipeline + View Quote Online link fully implemented and tested |
| DLVR-01b | 18-04 | Sent quote emails include PDF as an attachment | DEFERRED (Phase 20) | Intentionally out of scope per CONTEXT.md; REQUIREMENTS.md correctly marks as Phase 20 Pending |
| DLVR-02 | 18-03, 18-05, 18-06 | Configure default quote email template in business settings with variable placeholders | SATISFIED | QuoteEmailSettings on Business page; quote-settings API; 5 variable badges |
| DLVR-03 | 18-02, 18-06 | Review and edit pre-filled email in send dialog with template variables auto-resolved | SATISFIED | SendQuoteForm: resolveTemplateVariables(), pre-filled To/Subject/Message, RichTextEditor |
| DLVR-04 | 18-01, 18-02 | Re-send a quote already in Sent status | SATISFIED | Re-send Quote button opens same dialog; full token+email+transition always executed |
| AUTO-01 | 18-01 | Quote status transitions Draft to Sent automatically on successful send | SATISFIED | quoteTransitionService.transition() called after emailSenderService.sendEmail() confirms delivery |

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None found | — | — | — | No stubs, placeholders, empty handlers, or TODO comments found in any phase 18 files. Three grep matches for "placeholder" in email-sender.service.ts are intentional — they are the placeholder API key detection string values, not code stubs. |

---

## Human Verification Required

### 1. Send dialog pre-fill with template variable resolution

**Test:** Open a draft quote for a customer with an email address. Click "Send Quote". Observe the To, Subject, and Message fields.
**Expected:** To is pre-filled with customer email; Subject shows resolved template (e.g., "Quote Q-001 for Kitchen Renovation"); Message body shows the resolved text in the rich text editor.
**Why human:** Template variable substitution and initial Tiptap editor state require a running browser.

### 2. Rich text formatting is preserved in sent email

**Test:** In the send dialog, bold a word and add an italic phrase. Click "Send Quote". Inspect the received email or network payload.
**Expected:** The HTML sent to the backend contains `<strong>` and `<em>` tags. The email renders formatted text.
**Why human:** Tiptap editor state output and SendGrid delivery require live environment.

### 3. No-email customer warning and save-email side effect

**Test:** Open a draft quote for a customer with no email address. Click "Send Quote". Enter an email, check the save-email checkbox, send. Verify the customer record afterward.
**Expected:** Warning alert "No email on file for this customer" appears. After send, customer.email is updated in the database.
**Why human:** Conditional UI state and customer update side effect require live data.

### 4. Business page Quote Email tab flows through to send dialog

**Test:** Go to Business page > Quote Email tab. Enter a custom subject "Quote {{quoteNumber}} ready!" and save. Then open a quote send dialog.
**Expected:** The send dialog's Subject field shows "Quote Q-001 ready!" (with resolved quote number).
**Why human:** End-to-end data flow from quote-settings record through RTK Query cache into dialog pre-fill requires running application.

### 5. Re-send delivers new View Quote Online link

**Test:** Send a quote (note the customer-facing link). Re-send the same quote. Compare the new link to the original.
**Expected:** The second email contains a different token in the view URL (new token generated per send).
**Why human:** Requires verifying actual email received by the customer.

### 6. SendGrid error returns 503 for placeholder API key

**Test:** Configure a placeholder SENDGRID_API_KEY (e.g., "your-key"). Attempt to send a quote.
**Expected:** API returns HTTP 503 with message: "Email sending is not configured. Set a valid SENDGRID_API_KEY environment variable."
**Why human:** Requires a live environment with a placeholder key configured.

---

## Gaps Summary

**No gaps.** The one gap from the initial verification (DLVR-01 requirements accuracy) has been resolved.

DLVR-01 was split into DLVR-01a and DLVR-01b:
- DLVR-01a (email with link, Phase 18) — marked Complete in REQUIREMENTS.md, fully implemented
- DLVR-01b (PDF attachment, Phase 20) — marked Pending in REQUIREMENTS.md, correctly deferred

Three additional gap-closure plans (04, 05, 06) also improved quality beyond the minimum gap closure: they added meaningful SendGrid error handling, introduced a cleaner QuoteSettings separation of concerns, and relocated the Quote Email settings UI to the more appropriate Business page.

All 6 plans' artifacts are present, substantive, and wired. Phase 18 goal is achieved.

---

_Verified: 2026-03-21_
_Re-verification: Yes (initial score 9/10, re-verification score 10/10)_
_Verifier: Claude (gsd-verifier)_
