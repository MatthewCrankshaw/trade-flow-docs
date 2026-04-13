---
phase: 44-email-send-flow
plan: "04"
subsystem: trade-flow-ui
tags:
  - frontend
  - react
  - rtk-query
  - estimates
  - email
  - send-flow
dependency_graph:
  requires:
    - 44-01 (estimate-settings API endpoints GET/PATCH /v1/business/:id/estimate-settings)
    - 44-03 (POST /v1/estimates/:id/send endpoint)
    - 43-05 (EstimateActionStrip stub, EstimateDetailPage skeleton)
  provides:
    - EstimateEmailSettings component with disclaimer info Alert
    - Business > Documents tab (renamed from Quote Email, now stacks both templates)
    - useGetEstimateSettingsQuery and useUpdateEstimateSettingsMutation RTK Query hooks
    - SendEstimateDialog shell component
    - SendEstimateForm with To/Subject/Message, template variable resolution, validation hints
    - useSendEstimateMutation RTK Query hook
    - EstimateActionStrip with Send Estimate / Re-send buttons per status
    - EstimateDetailPage wired with estimate settings query
  affects:
    - trade-flow-ui/src/features/business/api/businessApi.ts
    - trade-flow-ui/src/features/business/components/BusinessDetails.tsx
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx
tech_stack:
  added: []
  patterns:
    - RTK Query injectEndpoints mirroring existing QuoteSettings pattern
    - SendEstimateForm mirrors SendQuoteForm with estimate-specific variables
    - Template variable resolution client-side on mount (resolveTemplateVariables)
    - Conditional send/re-send buttons keyed on estimate.status
    - vi.mock for RTK Query hooks in unit tests (same pattern as TrialBadge tests)
key_files:
  created:
    - trade-flow-ui/src/features/business/components/EstimateEmailSettings.tsx
    - trade-flow-ui/src/features/business/components/__tests__/EstimateEmailSettings.test.tsx
    - trade-flow-ui/src/features/estimates/components/SendEstimateDialog.tsx
    - trade-flow-ui/src/features/estimates/components/SendEstimateForm.tsx
    - trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx
  modified:
    - trade-flow-ui/src/types/estimate.ts (lastSentAt, lastSentTo, EstimateSettings, UpdateEstimateSettingsRequest)
    - trade-flow-ui/src/types/index.ts (EstimateSettings, UpdateEstimateSettingsRequest exports)
    - trade-flow-ui/src/features/business/api/businessApi.ts (getEstimateSettings, updateEstimateSettings endpoints)
    - trade-flow-ui/src/features/business/components/BusinessDetails.tsx (Documents tab with both settings)
    - trade-flow-ui/src/features/business/components/index.tsx (EstimateEmailSettings export)
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts (sendEstimate mutation)
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx (Send/Re-send buttons + SendEstimateDialog)
    - trade-flow-ui/src/features/estimates/components/index.ts (SendEstimateDialog, SendEstimateForm exports)
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx (useGetEstimateSettingsQuery wiring, settings passed to EstimateEditor)
decisions:
  - EstimateEmailSettings renders Info Alert (not warning variant) for the disclaimer note per plan D-DOC-03
  - EstimateActionStrip renders no send button for terminal statuses (converted/declined/expired/lost/deleted) matching plan D-UI-03
  - SendEstimateDialog guards form with `{open && <SendEstimateForm />}` to prevent stale state — mirrors SendQuoteDialog pattern
  - businessApi.ts updateEstimateSettings mutation reformatted by Prettier to single-line invalidatesTags (within 125-char print width)
  - EstimateDetailPage passes business/user/settings as optional props through EstimateEditor rather than fetching in EstimateEditor directly — keeps EstimateEditor testable
metrics:
  duration: ~25min
  completed_date: "2026-04-13"
  tasks: 2
  files: 14
---

# Phase 44 Plan 04: Frontend Send Experience Summary

Complete frontend send experience for estimates: EstimateEmailSettings component with mandatory disclaimer, Business > Documents tab stacking both email templates, SendEstimateDialog/Form with template variable resolution and inline validation hints, EstimateActionStrip Send/Re-send buttons per status, and EstimateDetailPage wired with settings query.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Business settings — Documents tab with EstimateEmailSettings + RTK Query hooks + unit tests | 7dc41f3 | 7 files (EstimateEmailSettings.tsx, __tests__/EstimateEmailSettings.test.tsx, BusinessDetails.tsx, business/components/index.tsx, businessApi.ts, types/estimate.ts, types/index.ts) |
| 2 | SendEstimateDialog + SendEstimateForm + EstimateActionStrip + EstimateDetailPage wiring + unit tests | 09a7699 | 9 files (SendEstimateDialog.tsx, SendEstimateForm.tsx, __tests__/SendEstimateForm.test.tsx, EstimateActionStrip.tsx, estimates/components/index.ts, estimateApi.ts, EstimateDetailPage.tsx, businessApi.ts prettier fix, EstimateEmailSettings.test.tsx prettier fix) |

## What Was Built

### EstimateEmailSettings (`src/features/business/components/EstimateEmailSettings.tsx`)

Mirror of `QuoteEmailSettings` with estimate-specific field names:
- Card title: "Estimate Email Template", description: "Configure the default email sent with your estimates."
- **Info Alert** (variant `default`, `Info` icon) with text: "The non-binding legal disclaimer is added automatically and cannot be edited." — satisfies D-DOC-03 / T-44-17
- Subject field placeholder: `"Estimate {{estimateNumber}} for {{jobTitle}}"`
- Message textarea placeholder with full estimate-specific copy
- Available variables as `Badge variant="outline"` elements: `{{customerName}}`, `{{estimateNumber}}`, `{{jobTitle}}`, `{{businessName}}`, `{{userName}}`
- Server-side subject validation error displayed inline beneath subject field on PATCH failure
- Calls `useGetEstimateSettingsQuery` / `useUpdateEstimateSettingsMutation`
- Success toast: `"Estimate email template saved"`, error toast: `"Failed to save template. Please try again."`

### businessApi.ts additions

Two new endpoints injected into existing `businessApi.injectEndpoints()`:
- `getEstimateSettings` — GET `/v1/business/:businessId/estimate-settings`, providesTags `estimate-settings-${businessId}`, null-safe stub when no row exists
- `updateEstimateSettings` — PATCH `/v1/business/:businessId/estimate-settings`, invalidatesTags `estimate-settings-${businessId}`
- Exported hooks: `useGetEstimateSettingsQuery`, `useUpdateEstimateSettingsMutation`

### EstimateSettings types (`src/types/estimate.ts`)

Added to `Estimate` interface: `lastSentAt?: string`, `lastSentTo?: string` — matches API response fields from Phase 44-03.

Added new interfaces:
```typescript
EstimateSettings { id, businessId, estimateEmailSubject?, estimateEmailBody? }
UpdateEstimateSettingsRequest { estimateEmailSubject?, estimateEmailBody? }
```

Both exported from `src/types/index.ts`.

### BusinessDetails.tsx — Documents tab

- `TabsTrigger value="quote-email"` renamed to `value="documents"`, label "Quote Email" → "Documents", icon `Mail` → `FileText`
- `TabsContent value="quote-email"` renamed to `value="documents"`
- Content: `<div className="flex flex-col gap-6">` containing `<QuoteEmailSettings>` then `<EstimateEmailSettings>` — satisfies plan acceptance criteria (flex gap, NOT space-y-6)

### SendEstimateDialog (`src/features/estimates/components/SendEstimateDialog.tsx`)

Dialog shell with `DialogHeader` containing title "Send Estimate" and description "Review and send this estimate to your customer." Guards `SendEstimateForm` with `{open && ...}` to prevent stale state on close.

### SendEstimateForm (`src/features/estimates/components/SendEstimateForm.tsx`)

Full send form mirroring `SendQuoteForm`:
- **To** field (type email): pre-filled from `customerEmail`, no-email Alert when null and field empty, "Save email to customer record" checkbox visible when To has value, checked by default when customerEmail was null
- **Subject** field: pre-filled from resolved template, inline validation hint `"Subject must contain the word 'Estimate'"` when subject doesn't contain "estimate" (case-insensitive) — T-44-15 mitigation (UX courtesy; server is authoritative)
- **Message** field: `RichTextEditor` pre-filled with resolved HTML from template
- Template variable resolution: `resolveTemplateVariables()` replaces `{{estimateNumber}}`, `{{jobTitle}}`, `{{customerName}}`, `{{businessName}}`, `{{userName}}` on mount
- `DEFAULT_SUBJECT = "Estimate {{estimateNumber}} for {{jobTitle}}"` fallback when settings are empty
- `DEFAULT_MESSAGE` fallback with guide-price language
- On success: `toast.success(\`Estimate sent to ${to}\`)`, close dialog
- On error: subject keyword error / status error / generic error toast variants
- Uses `useSendEstimateMutation` hook — T-44-18 mitigated via JWT auth in prepareHeaders

### estimateApi.ts — sendEstimate mutation

```typescript
sendEstimate: builder.mutation<Estimate, { estimateId, to, subject, message, saveEmail? }>({
  query: ({ estimateId, ...body }) => ({ url: `/v1/estimates/${estimateId}/send`, method: "POST", body }),
  invalidatesTags: [{ type: "Estimate", id: estimateId }, { type: "Estimate", id: "LIST" }],
})
```
Exported as `useSendEstimateMutation`.

### EstimateActionStrip (`src/features/estimates/components/EstimateActionStrip.tsx`)

Extended from Phase 43 Delete-only stub:
- `estimate.status === "draft"`: Shows "Send Estimate" button (`variant="default"`, `size="sm"`, `Send` icon)
- `estimate.status` in `["sent", "viewed", "responded", "site_visit_requested"]`: Shows "Re-send" button (`variant="outline"`, `size="sm"`, `Send` icon)
- Terminal statuses (`["converted", "declined", "expired", "lost", "deleted"]`): No send/re-send button rendered
- Delete button retained for draft status
- Renders `<SendEstimateDialog>` at bottom, controlled by `sendDialogOpen` state

Props extended: `businessName?`, `userName?`, `defaultSubject?`, `defaultMessage?`

### EstimateDetailPage wiring

- Added `useGetEstimateSettingsQuery(businessId!, { skip: !businessId })` call in `EstimateDetailContent`
- Extended `useCurrentBusiness()` destructure to include `business` and `user`
- Added `businessName`, `userName`, `defaultSubject`, `defaultMessage` to `EstimateEditorProps` and `EstimateEditor` function signature
- Passes all four down from `EstimateDetailContent` → `EstimateEditor` → `EstimateActionStrip`

## Verification

```
Task 1 tests: 5/5 PASS (EstimateEmailSettings: title, disclaimer, variables, server error, pre-fill)
Task 2 tests: 7/7 PASS (SendEstimateForm: To/Subject/Message, no-email warning, subject hint, save checkbox, disabled send, send call, checkbox default)
All tests: 94/94 PASS across 12 test files
Lint: 0 errors (1 pre-existing warning in BusinessStep.tsx — out of scope)
Format: all files Prettier-clean
TypeScript: zero errors
CI: PASS in trade-flow-ui
```

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Prettier formatting violations after initial Task 1 and 2 commits**
- **Found during:** CI `format:check` run after both task commits
- **Issue:** `businessApi.ts` (multi-line mutation type parameter), `EstimateEmailSettings.test.tsx`, `SendEstimateForm.test.tsx`, and `EstimateDetailPage.tsx` had formatting issues detected by Prettier
- **Fix:** Ran `npm run format` to auto-fix all four files before final CI confirmation
- **Files modified:** `businessApi.ts`, `__tests__/EstimateEmailSettings.test.tsx`, `__tests__/SendEstimateForm.test.tsx`, `EstimateDetailPage.tsx`
- **Commit:** Included in 09a7699 (staged after format run)

## Known Stubs

**Pre-existing stub (out of scope for this plan):**
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` line 271: Customer response card shows "No customer response yet. Response handling ships in Phase 45." — intentional Phase 43 stub, resolved by Phase 45.

No new stubs introduced by this plan. All API hooks, form fields, and status buttons are fully wired.

## Threat Flags

No new threat surfaces beyond those documented in the plan's threat model. All four mitigations confirmed:
- T-44-15: Subject inline validation hint when "Estimate" removed (UX); `@Matches(/estimate/i)` on server is authoritative
- T-44-16: `sanitizeHtml()` in EstimateEmailSender (Phase 44-03) strips disallowed tags from client-submitted HTML
- T-44-17: `useGetEstimateSettingsQuery` / `useUpdateEstimateSettingsMutation` pass auth JWT via `prepareHeaders`; business ID scoped
- T-44-18: `useSendEstimateMutation` passes auth JWT; server rejects unauthenticated requests

## Self-Check: PASSED

Files verified present:
- `trade-flow-ui/src/features/business/components/EstimateEmailSettings.tsx` — contains "Estimate Email Template", "The non-binding legal disclaimer is added automatically and cannot be edited.", `useGetEstimateSettingsQuery`, `useUpdateEstimateSettingsMutation`
- `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` — contains `value="documents"` (not `value="quote-email"`), "Documents", `QuoteEmailSettings`, `EstimateEmailSettings`, `flex flex-col gap-6`
- `trade-flow-ui/src/features/business/api/businessApi.ts` — contains `getEstimateSettings`, `updateEstimateSettings`, `estimate-settings-${businessId}` tag
- `trade-flow-ui/src/features/estimates/components/SendEstimateDialog.tsx` — contains "Send Estimate", "Review and send this estimate to your customer."
- `trade-flow-ui/src/features/estimates/components/SendEstimateForm.tsx` — contains `useSendEstimateMutation`, "Estimate {{estimateNumber}} for {{jobTitle}}", "Subject must contain the word", "No email on file for this customer", "Save email to customer record"
- `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` — contains "Send Estimate", "Re-send", `SendEstimateDialog`
- `trade-flow-ui/src/features/estimates/api/estimateApi.ts` — contains `sendEstimate`, `useSendEstimateMutation`, `/v1/estimates/${estimateId}/send`
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx` — contains `useGetEstimateSettingsQuery`
- `trade-flow-ui/src/features/business/components/__tests__/EstimateEmailSettings.test.tsx` — 5 tests all PASS
- `trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx` — 7 tests all PASS

Commits verified in UI git history:
- 7dc41f3: feat(44-04): add EstimateEmailSettings, Documents tab, RTK Query hooks, and unit tests
- 09a7699: feat(44-04): add SendEstimateDialog, SendEstimateForm, EstimateActionStrip send buttons, and EstimateDetailPage wiring
