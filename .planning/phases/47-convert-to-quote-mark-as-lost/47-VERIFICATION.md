---
phase: 47-convert-to-quote-mark-as-lost
verified: 2026-04-14T09:00:00Z
status: human_needed
score: 4/4 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Convert to Quote flow end-to-end"
    expected: "Navigate to Sent estimate, click Convert to Quote, verify navigation to /quotes/{quoteId}/edit, verify line items copied without contingency, verify back-link on quote detail page, verify source estimate locked with EstimateConvertedLink"
    why_human: "Navigation, real-time API calls, multi-page flow, and visual locked state cannot be verified programmatically"
  - test: "Mark as Lost flow end-to-end"
    expected: "Navigate to Sent estimate, click Mark as Lost, verify dialog opens with reason chips and textarea, submit with reason and note, verify amber EstimateLostReasonCard appears with recorded data, verify no action buttons shown"
    why_human: "Dialog interaction, form submission, toast notification, and visual locked state rendering require human observation"
  - test: "Status-based button visibility"
    expected: "DRAFT shows only Send/Delete; SENT shows Re-send/Convert to Quote/Mark as Lost; VIEWED shows Re-send/Mark as Lost (no Convert); RESPONDED shows Re-send/Convert to Quote/Mark as Lost; CONVERTED shows only EstimateConvertedLink; LOST shows no buttons"
    why_human: "Visual rendering of status-based conditional button rendering requires running UI"
---

# Phase 47: Convert to Quote & Mark as Lost Verification Report

**Phase Goal:** A trader can idempotently convert a Sent/Responded estimate into a fully independent quote with mandatory review, and can manually mark an estimate as Lost with a structured reason -- both flows cancel any pending follow-ups and lock the source estimate.
**Verified:** 2026-04-14T09:00:00Z
**Status:** human_needed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths (Roadmap Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | POST /v1/estimates/:id/convert accepts Idempotency-Key header, copies estimate line items as literal snapshot, drops contingency, creates draft quote, transitions estimate to Converted, sets convertedToQuoteId back-link | VERIFIED | `estimate.controller.ts:222-235` has `@Post("estimates/:id/convert")` reading `request.headers["idempotency-key"]`; `estimate-to-quote-converter.service.ts` calls `quoteLineItemCreator.create()` with snapshot values (taxRate, unitPrice, quantity directly), no contingency fields referenced; `EstimateStatus.CONVERTED` transition at line 65; `convertedToQuoteId` written at line 70 |
| 2 | Trader opens convert from estimate detail page, new quote opens in edit mode (mandatory review), double-click with same Idempotency-Key returns same quoteId | VERIFIED (code) / NEEDS HUMAN (UI flow) | `EstimateActionStrip.tsx:53` generates `crypto.randomUUID()`, calls `convertEstimate` mutation, navigates to `/quotes/${result.quoteId}/edit` (line 60). Idempotency: service line 45 checks `if (estimate.convertedToQuoteId)` and returns existing ID immediately. |
| 3 | Converted quote detail page shows "Converted from E-YYYY-NNN" back-link; source estimate shows locked state (no Edit, no Revise) | VERIFIED (code) / NEEDS HUMAN (visual) | `QuoteSourceEstimateLink.tsx` renders "Converted from" + `sourceEstimateNumber` linking to `/estimates/${sourceEstimateId}`; `QuoteDetailPage.tsx:112-115` conditionally renders it. `EstimateActionStrip.tsx:66-71` returns `<EstimateConvertedLink>` when `isConverted`, `null` for all other terminal states. |
| 4 | POST /v1/estimates/:id/mark-lost transitions estimate to Lost with structured reason/freeform text, cancels follow-ups, renders locked detail page | VERIFIED (code) / NEEDS HUMAN (visual) | `estimate.controller.ts:237-247`; `EstimateLostMarker` writes `lostReason`/`lostNotes`, transitions to `EstimateStatus.LOST`, calls `cancelAllFollowups`; `MarkAsLostDialog.tsx` shows 5 reason chips + textarea; `EstimateLostReasonCard.tsx` renders amber card with reason/notes/timestamp on detail page. |

**Score:** 4/4 truths verified (automated code checks pass; 3 items also need human E2E verification)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts` | Convert estimate to quote service | VERIFIED | 159 lines, `@Injectable()` class `EstimateToQuoteConverter`, `convert()` method, `quoteCreator.create`, idempotency check, `cancelAllFollowups` |
| `trade-flow-api/src/estimate/services/estimate-lost-marker.service.ts` | Mark estimate as lost service | VERIFIED | 53 lines, `@Injectable()` class `EstimateLostMarker`, `markAsLost()` method, status validation, `lostReason`/`lostNotes` write, `cancelAllFollowups` |
| `trade-flow-api/src/estimate/requests/mark-estimate-lost.request.ts` | Request DTO with optional reason/notes | VERIFIED | `@IsEnum(LostReason)` + `@IsOptional()` on reason; `@MaxLength(500)` on notes |
| `trade-flow-api/src/estimate/enums/lost-reason.enum.ts` | Shared lost/decline reason enum | VERIFIED | All 5 values: `TOO_EXPENSIVE`, `GOING_WITH_SOMEONE_ELSE`, `DECIDED_NOT_TO_DO_WORK`, `JUST_GETTING_IDEA`, `TIMING_NOT_RIGHT` |
| `trade-flow-api/src/estimate/test/services/estimate-to-quote-converter.service.spec.ts` | Converter unit tests (min 80 lines) | VERIFIED | 244 lines, 8 test cases covering: SENT, RESPONDED, idempotency, CONVERTED transition, convertedToQuoteId write, cancelAllFollowups, invalid status rejection, literal tax rate copy |
| `trade-flow-api/src/estimate/test/services/estimate-lost-marker.service.spec.ts` | Lost marker unit tests (min 40 lines) | VERIFIED | 173 lines, 7 test cases covering: LOST transition, lostReason/lostNotes write, cancelAllFollowups, null reason/notes, invalid status rejection, VIEWED allowed, RESPONDED allowed |
| `trade-flow-api/src/estimate/controllers/estimate.controller.ts` | Convert and mark-lost endpoints | VERIFIED | `@Post("estimates/:id/convert")` at line 223, `@Post("estimates/:id/mark-lost")` at line 237, both with `@UseGuards(JwtAuthGuard)` |
| `trade-flow-api/src/estimate/estimate.module.ts` | Module wiring for new services | VERIFIED | `EstimateToQuoteConverter` and `EstimateLostMarker` in providers (lines 78-79) and exports (lines 87-88); `forwardRef(() => QuoteModule)` at line 54 |
| `trade-flow-api/openapi.yaml` | API documentation for new endpoints | VERIFIED | `/v1/estimates/{estimateId}/convert` at line 3667 with `Idempotency-Key` header; `/v1/estimates/{estimateId}/mark-lost` at line 3717 with `too_expensive` enum values |
| `trade-flow-ui/src/features/estimates/components/MarkAsLostDialog.tsx` | Dialog with reason picker and textarea | VERIFIED | 119 lines, `Mark estimate as lost?` title, 5 reason chips with `too_expensive` etc., `useMarkEstimateLostMutation`, `MAX_NOTES_LENGTH = 500` |
| `trade-flow-ui/src/features/estimates/components/EstimateLostReasonCard.tsx` | Read-only card for lost reason | VERIFIED | 32 lines, `AlertTriangle` icon, `bg-warning` styling, "Marked as lost on" timestamp |
| `trade-flow-ui/src/features/estimates/components/EstimateConvertedLink.tsx` | Chip/link to converted quote | VERIFIED | 20 lines, `ArrowRightLeft` icon, navigates to `/quotes/${convertedToQuoteId}` |
| `trade-flow-ui/src/features/quotes/components/QuoteSourceEstimateLink.tsx` | Back-link from converted quote to source estimate | VERIFIED | 19 lines, `ArrowLeft` icon, "Converted from" text, `Link to="/estimates/${sourceEstimateId}"` |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `estimate-to-quote-converter.service.ts` | `QuoteCreator` | `quoteCreator.create()` | WIRED | Line 54: `const createdQuote = await this.quoteCreator.create(authUser, quoteDto)` |
| `estimate-to-quote-converter.service.ts` | `EstimateRepository` | atomic convertedToQuoteId write | WIRED | Line 73: `await this.estimateRepository.update(withConvertFields)` with `convertedToQuoteId` and `convertIdempotencyKey` |
| `estimate-lost-marker.service.ts` | `EstimateTransitionService` | transition to LOST | WIRED | Line 37: `await this.estimateTransitionService.transition(authUser, estimateId, EstimateStatus.LOST)` |
| `estimate.controller.ts` | `EstimateToQuoteConverter` | DI injection and convert() call | WIRED | Line 229: `await this.estimateToQuoteConverter.convert(request.user, request.params.id, idempotencyKey)` |
| `estimate.controller.ts` | `EstimateLostMarker` | DI injection and markAsLost() call | WIRED | Line 243: `await this.estimateLostMarker.markAsLost(request.user, request.params.id, body.reason, body.notes)` |
| `EstimateActionStrip.tsx` | `estimateApi.ts` | `useConvertEstimateMutation` hook | WIRED | Line 42: `const [convertEstimate, { isLoading: isConverting }] = useConvertEstimateMutation()` |
| `EstimateActionStrip.tsx` | `/quotes/{quoteId}/edit` | `useNavigate` after successful convert | WIRED | Line 60: `navigate(\`/quotes/${result.quoteId}/edit\`)` |
| `QuoteSourceEstimateLink.tsx` | `/estimates/{sourceEstimateId}` | Link component | WIRED | Line 14: `<Link to={\`/estimates/${sourceEstimateId}\`}>` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `EstimateLostReasonCard.tsx` | `reason`, `notes`, `lostAt` props | API response via `markEstimateLost` mutation → `EstimateDetailPage` | Yes -- fetched from MongoDB via `EstimateLostMarker`, stored as `lostReason`/`lostNotes` on estimate document | FLOWING |
| `EstimateConvertedLink.tsx` | `convertedToQuoteId` prop | API response via estimate retrieval → `EstimateActionStrip` | Yes -- set by `EstimateToQuoteConverter` on convert, stored on estimate document | FLOWING |
| `QuoteSourceEstimateLink.tsx` | `sourceEstimateId`, `sourceEstimateNumber` | API response via `useGetQuoteQuery` → `QuoteDetailPage` | Yes -- set by converter on quote document via `quoteRepository.update()` | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED (requires running servers for API endpoint tests; both repos are not in a runnable state from docs directory)

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| CONV-01 | 47-01, 47-02, 47-03 | User can convert a Sent or Responded estimate to a new quote | SATISFIED | Backend service validates `SENT`/`RESPONDED` only; frontend `CONVERTIBLE_STATUSES = ["sent", "responded"]`; controller endpoint exists |
| CONV-02 | 47-01, 47-03 | Convert copies line items with literal tax rate percentages, drops contingency | SATISFIED | `mapToQuoteLineItem()` copies `taxRate`, `unitPrice`, `quantity` directly from estimate line items; no contingency fields referenced in converter |
| CONV-03 | 47-03 | Convert opens new quote in edit mode for mandatory review | SATISFIED | `navigate(\`/quotes/${result.quoteId}/edit\`)` on success |
| CONV-04 | 47-01, 47-02 | Convert endpoint accepts Idempotency-Key header, returns same quote on double-submit | SATISFIED | Header read at controller line 228; service returns existing `convertedToQuoteId` at line 45-47 immediately on re-call |
| CONV-05 | 47-01, 47-02, 47-03 | Converted estimate transitions to Converted status with convertedToQuoteId back-link, locked from revisions | SATISFIED | `EstimateStatus.CONVERTED` transition in service; `convertedToQuoteId` written; `EstimateActionStrip` returns `EstimateConvertedLink` only (no action buttons) for converted estimates |
| CONV-06 | 47-04 | Converted quote shows "Converted from E-YYYY-NNN" back-link on detail page | SATISFIED | `QuoteSourceEstimateLink` component rendered in `QuoteDetailPage` when `sourceEstimateId && sourceEstimateNumber` present |
| LOST-01 | 47-01, 47-02, 47-03 | User can manually mark estimate as Lost with structured reason or freeform text | SATISFIED | `EstimateLostMarker.markAsLost()` accepts optional `reason` (LostReason enum) and `notes`; `MarkAsLostDialog` has 5 reason chips + textarea |
| LOST-02 | 47-01, 47-02 | Marking as Lost cancels all pending follow-ups | SATISFIED | `followupCanceller.cancelAllFollowups(estimate.id, estimate.revisionNumber)` called in `EstimateLostMarker.markAsLost()` |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `estimate-to-quote-converter.service.ts` | 122, 128 | Variable named `placeholderId` | Info | Local variable naming only; used as actual ObjectId for new quote line items -- not a stub |

No blockers or warnings found.

### Human Verification Required

#### 1. Convert to Quote End-to-End Flow

**Test:** With both trade-flow-api and trade-flow-ui running locally:
1. Navigate to Estimates list, find or create a SENT estimate
2. Open the estimate detail page
3. Verify "Convert to Quote" button visible (blue primary with ArrowRightLeft icon)
4. Click "Convert to Quote" -- verify "Converting..." spinner during call
5. Verify navigation to `/quotes/{quoteId}/edit` after success
6. Verify the new quote has the same line items as the estimate
7. Navigate to quote detail page, verify "Converted from E-YYYY-NNN" back-link appears
8. Click back-link, verify navigation to source estimate
9. Verify source estimate shows "Converted to Quote" chip with no action buttons

**Expected:** Seamless end-to-end flow, correct navigation, locked source estimate
**Why human:** Multi-page navigation, real-time API mutations, visual locked state cannot be verified programmatically

#### 2. Mark as Lost End-to-End Flow

**Test:** With both repos running locally:
1. Navigate to a SENT estimate detail page
2. Verify "Mark as Lost" button visible (red outline with XCircle icon)
3. Click "Mark as Lost" -- verify dialog opens with title "Mark estimate as lost?"
4. Select a reason chip (e.g., "Timing isn't right"), verify it highlights
5. Type a note in the textarea, verify character counter appears
6. Click "Mark as Lost" in dialog footer
7. Verify success toast "Estimate marked as lost"
8. Verify amber EstimateLostReasonCard appears above line items with reason, note, and timestamp
9. Verify no action buttons visible on the estimate

**Expected:** Dialog works correctly, estimate locked with reason card displayed
**Why human:** Dialog interaction, toast notification, and visual card rendering require observation

#### 3. Status-Based Button Visibility

**Test:** Verify button visibility across all estimate statuses:
- DRAFT: only Send + Delete (no Convert, no Mark as Lost)
- SENT: Re-send + Convert to Quote + Mark as Lost
- VIEWED: Re-send + Mark as Lost (no Convert -- CONV-01 constraint)
- RESPONDED: Re-send + Convert to Quote + Mark as Lost
- CONVERTED: only "Converted to Quote" chip/link
- LOST: no buttons (reason card in content area instead)

**Expected:** Precise status-based visibility matching requirements
**Why human:** Requires running UI with test data in each status state

### Gaps Summary

No code-level gaps found. All must-haves are verified with substantive implementations, correct wiring, and real data flows. Both backend services have comprehensive unit test coverage (244 and 173 lines respectively). Both CI gates are reported passed by Plan 02 and Plan 03 summaries.

The phase is blocked on human verification (Task 2 in Plan 04) which is a planned checkpoint -- the PLAN marked it as `type: checkpoint:human-verify` with `gate: blocking`. Automated checks are complete.

---

_Verified: 2026-04-14T09:00:00Z_
_Verifier: Claude (gsd-verifier)_
