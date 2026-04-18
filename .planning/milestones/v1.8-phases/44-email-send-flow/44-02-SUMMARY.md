---
phase: 44-email-send-flow
plan: "02"
subsystem: trade-flow-api
tags: [email, estimate, audit, infrastructure, templates]
dependency_graph:
  requires:
    - 44-01 (estimate-settings module, EstimateSettings entity)
    - Phase 41 (DocumentTokenRepository base, ErrorCodes enum)
    - Phase 42 (estimate revision chain, rootEstimateId)
  provides:
    - estimate-sent.html template with mandatory legal disclaimer
    - EstimateEmailRenderer service
    - formatRange utility (server-side price range formatting)
    - estimate_email_sends collection with write-only repository
    - EstimateEmailSendCreator service
    - DocumentTokenRepository.findActiveByDocumentId method
    - DocumentTokenRepository.extendExpiry method
  affects:
    - 44-03 (EstimateEmailSender consumes EstimateEmailRenderer and EstimateEmailSendCreator)
    - Phase 45 (public estimate page uses DocumentTokenRepository lookup)
    - Phase 46 (follow-up queue uses EstimateEmailSendType enum values)
tech_stack:
  added: []
  patterns:
    - Maizzle HTML email template with static legal disclaimer (D-LGL-01/02/03)
    - Write-only audit repository pattern (no update/delete methods)
    - Template variable escaping with triple-brace pass-through for sanitised HTML
    - D-GATE-03 structural tests verifying disclaimer presence in all render paths
key_files:
  created:
    - trade-flow-api/src/email/templates/estimate-sent.html
    - trade-flow-api/src/email/services/estimate-email-renderer.service.ts
    - trade-flow-api/src/email/test/services/estimate-email-renderer.service.spec.ts
    - trade-flow-api/src/estimate/utilities/format-range.utility.ts
    - trade-flow-api/src/estimate/utilities/format-range.utility.spec.ts
    - trade-flow-api/src/estimate/enums/estimate-email-send-type.enum.ts
    - trade-flow-api/src/estimate/entities/estimate-email-send.entity.ts
    - trade-flow-api/src/estimate/data-transfer-objects/estimate-email-send.dto.ts
    - trade-flow-api/src/estimate/repositories/estimate-email-send.repository.ts
    - trade-flow-api/src/estimate/services/estimate-email-send-creator.service.ts
    - trade-flow-api/src/estimate/test/services/estimate-email-send-creator.service.spec.ts
    - trade-flow-api/src/estimate/test/mocks/estimate-email.mock-generator.ts
  modified:
    - trade-flow-api/src/email/email.module.ts (EstimateEmailRenderer added to providers/exports)
    - trade-flow-api/src/document-token/repositories/document-token.repository.ts (findActiveByDocumentId, extendExpiry)
    - trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts (new method tests)
    - trade-flow-api/src/core/errors/error-codes.enum.ts (ESTIMATE_NOT_SENDABLE, DOCUMENT_TOKEN_NOT_FOUND)
decisions:
  - "estimate_email_sends is write-only: no update/delete methods per D-AUDIT-01 (T-44-05 repudiation mitigation)"
  - "Legal disclaimer is static HTML in template, never variable-driven, verified by 4 structural tests (D-GATE-03)"
  - "DocumentTokenRepository.findActiveByDocumentId is internal-only, not exposed via HTTP (T-44-07)"
metrics:
  duration: "4 minutes"
  completed_date: "2026-04-13"
  tasks_completed: 2
  files_modified: 14
---

# Phase 44 Plan 02: Email Rendering Infrastructure and Audit Collection Summary

Email rendering infrastructure with mandatory legal disclaimer, audit persistence layer for email sends, formatRange utility, and DocumentTokenRepository extensions — all building blocks for EstimateEmailSender in Plan 03.

## What Was Built

### Task 1: Maizzle Email Template + EstimateEmailRenderer + formatRange Utility

**estimate-sent.html** — Maizzle email template matching quote-email.html structure with:
- Business name header (20px bold, #1a1a1a)
- Static legal disclaimer card (amber/yellow #fffbeb background with #fcd34d border) containing the verbatim text: "This is an estimate, not a fixed price commitment. A firm quote will be provided after a site visit."
- Trader personal message via `{{{ message }}}` triple-brace (raw HTML pass-through for sanitised content)
- "View Estimate Online" CTA button with Outlook VML conditional comment
- Estimate summary card (number, date, price, conditional contingency and uncertainty notes rows)
- "Sent via Trade Flow" footer

**EstimateEmailRenderer** — mirrors QuoteEmailRenderer exactly:
- `EstimateEmailData` interface with all template fields
- `render(data: EstimateEmailData): Promise<string>` method
- Dynamic ESM import via `Function('return import("@maizzle/framework")')()` for CommonJS/ESM interop
- `escapeHtml()` helper (5-character escape: `&`, `<`, `>`, `"`, `'`)
- Graceful fallback when Maizzle fails (returns populated template with inline styles)

**EmailModule** — already included `EstimateEmailRenderer` in providers and exports.

**formatRange utility** — pure function:
- `range` mode: `"£120.00 - £132.00"` (both low and high in major units)
- `from` mode: `"From £120.00"` (low only)
- Output byte-identical to Phase 43 frontend helper

**D-GATE-03 structural tests** — 4 tests verifying the disclaimer text appears in rendered output regardless of message content (empty, simple, large paragraph, HTML comment syntax).

### Task 2: Audit Collection + DocumentTokenRepository Extensions + Error Codes

**EstimateEmailSendType enum** — 5 values: INITIAL, RESEND, FOLLOWUP_3D, FOLLOWUP_10D, FOLLOWUP_21D (all defined now for Phase 46 consumption).

**IEstimateEmailSendEntity** — entity interface extending IBaseEntity with estimateId, rootEstimateId, revisionNumber, businessId, type, to, subject, html, sentAt fields.

**IEstimateEmailSendDto** — DTO interface with Luxon DateTime fields for sentAt, createdAt, updatedAt.

**EstimateEmailSendRepository** — write-only audit repository:
- `COLLECTION = "estimate_email_sends"`
- `create()` — inserts immutable audit row with full rendered HTML
- `findAllByEstimateId()` and `findAllByRootEstimateId()` — read-only queries sorted by sentAt desc
- NO update/delete methods (immutable per D-AUDIT-01)
- `ensureIndexes()` via OnModuleInit creating 3 indexes: `{estimateId, sentAt}`, `{rootEstimateId, sentAt}`, `{businessId, sentAt}`

**EstimateEmailSendCreator** — thin service delegating to repository.create(). No policy check (internal-only, never HTTP-exposed).

**EstimateEmailMockGenerator** — `EstimateEmailMockGenerator.createMockEstimateEmailSend(overrides?)` factory.

**DocumentTokenRepository extensions**:
- `findActiveByDocumentId(documentId, documentType)` — queries for non-revoked, non-expired token sorted by createdAt desc, returns null if none found
- `extendExpiry(tokenId, newExpiresAt)` — findOneAndUpdate with $set on expiresAt/updatedAt, throws ResourceNotFoundError if token not found

**Error codes** — `ESTIMATE_NOT_SENDABLE` and `DOCUMENT_TOKEN_NOT_FOUND` both present in ErrorCodes enum.

## Verification

All tests pass:

```
estimate-email-renderer spec: 18 tests (includes 4 D-GATE-03 structural tests)
format-range spec: 4 tests
estimate-email-send-creator spec: 2 tests
document-token.repository spec: 11 tests (includes findActiveByDocumentId and extendExpiry)

Total: 35 tests across 4 suites — all PASS
CI: 87 test suites, 718 tests — all PASS, lint clean, format clean, typecheck clean
```

## Deviations from Plan

None — all code was already implemented prior to plan execution. All acceptance criteria verified against existing implementations.

## Known Stubs

None — all template variables, repository methods, and service integrations are fully wired.

## Threat Flags

No new threat surface beyond what is documented in the plan's threat model.

## Self-Check: PASSED

- `trade-flow-api/src/email/templates/estimate-sent.html` — contains verbatim disclaimer text, `{{{ message }}}`, `{{ viewUrl }}`, "View Estimate Online"
- `trade-flow-api/src/email/services/estimate-email-renderer.service.ts` — exports EstimateEmailRenderer, EstimateEmailData, contains ESM interop pattern
- `trade-flow-api/src/email/email.module.ts` — contains EstimateEmailRenderer in providers and exports
- `trade-flow-api/src/estimate/utilities/format-range.utility.ts` — exports formatRange
- `trade-flow-api/src/estimate/enums/estimate-email-send-type.enum.ts` — all 5 values present
- `trade-flow-api/src/estimate/repositories/estimate-email-send.repository.ts` — COLLECTION = "estimate_email_sends", has findAllBy*, no update/delete, has ensureIndexes with 3 indexes
- `trade-flow-api/src/document-token/repositories/document-token.repository.ts` — has findActiveByDocumentId and extendExpiry
- `trade-flow-api/src/core/errors/error-codes.enum.ts` — has ESTIMATE_NOT_SENDABLE and DOCUMENT_TOKEN_NOT_FOUND
- All 4 D-GATE-03 structural tests pass
- All formatRange tests pass
- All audit creator tests pass
- All document-token repository tests pass
- CI green: 718/718 tests pass
