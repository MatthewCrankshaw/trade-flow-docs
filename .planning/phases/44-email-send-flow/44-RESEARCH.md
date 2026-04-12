# Phase 44: Email & Send Flow - Research

**Researched:** 2026-04-12
**Domain:** NestJS backend module creation, email rendering (Maizzle), RTK Query frontend integration, MongoDB audit collection
**Confidence:** HIGH

## Summary

Phase 44 is a well-constrained mirror-and-extend phase. Every new backend module (`estimate-settings`), service (`EstimateEmailSender`, `EstimateEmailRenderer`), and frontend component (`SendEstimateDialog`, `SendEstimateForm`, `EstimateEmailSettings`) has a direct 1:1 quote-side equivalent already shipped and stable in the codebase. The primary research task was verifying that the existing patterns, DTOs, and integration points align with the CONTEXT.md decisions -- they do, with minor additions documented below.

The Phase 41 prerequisite has partially landed in code: `document-token` module exists with `documentType: "quote" | "estimate"` discriminator, `IDocumentTokenEntity`, and basic CRUD. The `estimate` module currently only contains the Phase 42 `IEstimateFollowupCanceller` interface and `NoopEstimateFollowupCanceller`. Phase 41's full CRUD (controller, services, repositories, entities, DTOs) has **not** been executed yet -- plans assume Phase 41 lands first, providing the `EstimateController`, `EstimateRetriever`, `EstimateTransitionService`, `EstimateRepository`, etc. that Phase 44 extends.

**Primary recommendation:** Mirror existing quote-send patterns 1:1 with renames, adding: (a) `estimate_email_sends` audit collection, (b) `findActiveByDocumentId` + `extendExpiry` on `DocumentTokenRepository`, (c) three new status transitions, (d) `@Matches(/estimate/i)` subject validation, (e) `estimate-sent.html` Maizzle template with static legal disclaimer block.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-LGL-01 through D-LGL-06:** Legal disclaimer is static HTML in Maizzle template, never stored in DB. Subject must contain "Estimate" (case-insensitive) validated via class-validator. Template variables resolved client-side.
- **D-AUDIT-01 through D-AUDIT-08:** New `estimate_email_sends` collection with immutable rows, no TTL, 3 indexes. Write ordering: render -> Resend -> persist audit -> transition -> update lastSentAt -> extend token -> cancel followups (if revised send).
- **D-TKN-01 through D-TKN-10:** Token stores `rootEstimateId` as `documentId`. Reuse existing token on re-send. Three send paths (first send, revised send, pure re-send) with distinct audit row types and follow-up cancellation rules.
- **D-DOC-01 through D-DOC-05:** Rename "Quote Email" tab to "Documents". Stack QuoteEmailSettings + EstimateEmailSettings vertically. Server-side subject validation is authoritative.
- **D-SET-01 through D-SET-07:** New standalone `estimate-settings` module mirroring `quote-settings` 1:1. `BusinessCreator` extended to call `DefaultEstimateSettingsCreatorService.create()`.
- **D-SND-01 through D-SND-06:** `EstimateEmailSender` mirrors `QuoteEmailSender` with audit row persistence, token reuse logic, and `IEstimateFollowupCanceller` injection. No onboarding step parity.
- **D-TPL-01 through D-TPL-05:** New `estimate-sent.html` Maizzle template with legal disclaimer card, summary card (price range, contingency note, uncertainty notes).
- **D-UI-01 through D-UI-06:** Mirror `SendQuoteDialog`/`SendQuoteForm` with estimate renames. Extend `EstimateActionStrip` with Send/Re-send buttons. Add `sendEstimate` mutation to `estimateApi.ts`.
- **D-API-01 through D-API-04:** `POST /v1/estimates/:id/send` on existing `EstimateController`. OpenAPI updates for 3 new endpoints.
- **D-GATE-01 through D-GATE-03:** CI must pass. Legal review gate is blocking. Structural test for disclaimer presence.

### Claude's Discretion
- Exact `ErrorCodes` enum value names
- Whether `format-range.utility.ts` lives under `@core/utilities/` or `@estimate/utilities/`
- Whether `contingencyNote` string is composed in service or Maizzle template
- Exact Tailwind classes for disclaimer card in email template
- Icon for "Documents" tab (`Mail` or `FileText`)
- Test fixture strategy (mirror quote-email spec structure)
- Client-side subject validation timing (keystroke vs blur)
- `BusinessCreator` test assertion for parallel vs sequential creator calls

### Deferred Ideas (OUT OF SCOPE)
- Onboarding step parity for estimates (no `FIRST_ESTIMATE_SENT`)
- Public GET endpoint for `estimate_email_sends`
- Lifting `sanitizeHtml` into `@core/utilities`
- Client-side subject validator with tooltip
- Chip reasons in email body
- Token revocation UI
- TTL index on `estimate_email_sends`
- Compensating transaction in BusinessCreator
- Follow-up template architecture (Phase 46 concern)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SND-01 | Send estimate to customer via email with secure public link | Verified: `DocumentTokenRepository` exists with `documentType` discriminator; `EmailSenderService` wraps Resend SDK; `QuoteEmailSender` pattern provides 1:1 template |
| SND-02 | New `estimate-settings` module with own API; Business > Documents tab | Verified: `quote-settings` module provides complete mirror pattern (module, controller, services, repository, entity, DTO, request, response, policy); `BusinessDetails.tsx` uses shadcn Tabs |
| SND-03 | Send dialog with editable subject and rich-text body | Verified: `SendQuoteForm.tsx` (161 lines) provides complete mirror including `RichTextEditor` integration, template variable resolution, customer email warning |
| SND-04 | Persist rendered HTML at send time for audit | New `estimate_email_sends` collection per D-AUDIT decisions; no existing pattern to mirror -- greenfield collection with immutable rows |
| SND-05 | Mandatory non-binding legal language in default template | Enforced structurally via static HTML in `estimate-sent.html` Maizzle template (D-LGL-01); verified `@Matches(/estimate/i)` decorator available for subject validation |
| SND-06 | Re-send without creating new revision or regenerating token | Verified: `DocumentTokenRepository.findLatestNonRevokedForDocument()` exists; Phase 44 adds `findActiveByDocumentId` (expiry check) and `extendExpiry`; transition map supports `SENT -> SENT` no-op |
| SND-07 | Subject line includes "Estimate" not "Quote" | Verified: `@Matches` decorator already used in codebase (visit-type requests); `@Matches(/estimate/i)` provides case-insensitive validation |
</phase_requirements>

## Standard Stack

### Core (already in project -- no new dependencies)

| Library | Version | Purpose | Verified |
|---------|---------|---------|----------|
| NestJS | 11.x | Backend framework | [VERIFIED: codebase package.json] |
| class-validator | 0.14.1 | DTO validation decorators (`@Matches`, `@IsEmail`, `@IsNotEmpty`) | [VERIFIED: codebase package.json] |
| mongodb | 7.0.0 | Native MongoDB driver | [VERIFIED: codebase package.json] |
| luxon | 3.5.1 | Date formatting for email (`estimateDate`) | [VERIFIED: codebase package.json] |
| @maizzle/framework | (installed) | HTML email rendering with Tailwind CSS inlining | [VERIFIED: codebase quote-email-renderer.service.ts] |
| React | 19.2.0 | UI framework | [VERIFIED: codebase package.json] |
| @reduxjs/toolkit | 2.11.2 | RTK Query for API endpoints | [VERIFIED: codebase package.json] |
| sonner | 2.0.7 | Toast notifications | [VERIFIED: codebase package.json] |
| lucide-react | 0.563.0 | Icons (`Send`, `FileText`, `AlertTriangle`, `Info`, `Loader2`) | [VERIFIED: codebase package.json] |

**No new npm packages required.** All dependencies are already installed.

## Architecture Patterns

### Recommended Project Structure (new files Phase 44 creates)

```
trade-flow-api/src/
├── estimate-settings/                    # NEW MODULE (mirrors quote-settings 1:1)
│   ├── estimate-settings.module.ts
│   ├── controllers/
│   │   └── estimate-settings.controller.ts
│   ├── services/
│   │   ├── default-estimate-settings-creator.service.ts
│   │   ├── estimate-settings-retriever.service.ts
│   │   └── estimate-settings-updater.service.ts
│   ├── repositories/
│   │   └── estimate-settings.repository.ts
│   ├── entities/
│   │   └── estimate-settings.entity.ts
│   ├── data-transfer-objects/
│   │   └── estimate-settings.dto.ts
│   ├── requests/
│   │   └── update-estimate-settings.request.ts
│   ├── responses/
│   │   └── estimate-settings.response.ts
│   ├── policies/
│   │   └── estimate-settings.policy.ts
│   └── test/
│       ├── services/
│       └── mocks/
├── estimate/                             # EXTENDED (Phase 41 base + Phase 42 canceller)
│   ├── enums/
│   │   └── estimate-email-send-type.enum.ts      # NEW
│   ├── entities/
│   │   └── estimate-email-send.entity.ts         # NEW
│   ├── data-transfer-objects/
│   │   └── estimate-email-send.dto.ts            # NEW
│   ├── repositories/
│   │   └── estimate-email-send.repository.ts     # NEW
│   ├── services/
│   │   ├── estimate-email-send-creator.service.ts  # NEW
│   │   └── estimate-email-sender.service.ts        # NEW
│   ├── requests/
│   │   └── send-estimate.request.ts              # NEW
│   └── test/
│       └── services/
│           ├── estimate-email-sender.service.spec.ts  # NEW
│           └── estimate-email-send-creator.service.spec.ts  # NEW
├── email/
│   ├── services/
│   │   └── estimate-email-renderer.service.ts    # NEW
│   ├── templates/
│   │   └── estimate-sent.html                    # NEW
│   └── test/
│       └── services/
│           └── estimate-email-renderer.service.spec.ts  # NEW
├── document-token/
│   └── repositories/
│       └── document-token.repository.ts          # EXTENDED: +findActiveByDocumentId, +extendExpiry
└── core/
    ├── errors/
    │   └── error-codes.enum.ts                   # EXTENDED: +2 new codes
    └── utilities/
        └── format-range.utility.ts               # NEW (or @estimate/utilities/)

trade-flow-ui/src/
├── features/
│   ├── estimates/
│   │   ├── components/
│   │   │   ├── SendEstimateDialog.tsx            # NEW
│   │   │   ├── SendEstimateForm.tsx              # NEW
│   │   │   └── EstimateActionStrip.tsx           # EXTENDED (stub -> real buttons)
│   │   └── api/
│   │       └── estimateApi.ts                    # EXTENDED: +sendEstimate mutation
│   └── business/
│       ├── components/
│       │   ├── EstimateEmailSettings.tsx          # NEW
│       │   ├── BusinessDetails.tsx                # MODIFIED: tab rename + layout
│       │   └── index.tsx                          # MODIFIED: +EstimateEmailSettings export
│       └── api/
│           └── businessApi.ts                     # EXTENDED: +getEstimateSettings, +updateEstimateSettings
└── pages/
    └── EstimateDetailPage.tsx                     # EXTENDED: +useGetEstimateSettingsQuery, +SendEstimateDialog
```

### Pattern 1: Mirror-and-Rename (Primary Pattern)

**What:** Copy an existing quote-side file, rename `quote` -> `estimate` throughout, then add Phase 44-specific additions.
**When to use:** For every new file that has a direct quote-side counterpart.
**Source files verified in codebase:**

| New File | Mirror Source | Lines | Key Diffs |
|----------|-------------|-------|-----------|
| `estimate-settings.controller.ts` | `quote-settings.controller.ts` (77 lines) | Near-identical | Field names: `quoteEmail*` -> `estimateEmail*` |
| `estimate-settings.repository.ts` | `quote-settings.repository.ts` (76 lines) | Near-identical | Collection name, field names |
| `estimate-email-sender.service.ts` | `quote-email-sender.service.ts` (120 lines) | Significant additions | Token reuse, audit row, followup cancellation, status path detection |
| `estimate-email-renderer.service.ts` | `quote-email-renderer.service.ts` (48 lines) | Additional template variables | `priceDisplay`, `contingencyNote`, `uncertaintyNotes` |
| `estimate-sent.html` | `quote-email.html` (77 lines) | Legal disclaimer card, price range, contingency | Structural additions |
| `SendEstimateForm.tsx` | `SendQuoteForm.tsx` (161 lines) | Subject validation hint | `@Matches` inline error |
| `EstimateEmailSettings.tsx` | `QuoteEmailSettings.tsx` (131 lines) | Disclaimer info `<Alert>` | Additional UX hint |
| `send-estimate.request.ts` | `send-quote.request.ts` (18 lines) | `@Matches(/estimate/i)` on subject | Validation addition |

### Pattern 2: Immutable Audit Collection

**What:** `estimate_email_sends` -- write-once, never-update, never-soft-delete collection.
**When to use:** For the SND-04 rendered HTML audit trail.
**Key difference from other collections:** No `update` method, no `delete` method, no `updatedAt` field mutation after creation. Repository exposes only `create` and `findAll*` methods.

### Pattern 3: Token Reuse with Expiry Extension

**What:** Look up existing active token by `documentId` + `documentType`, reuse if found, create new if expired/revoked/missing, always extend expiry on successful send.
**When to use:** `EstimateEmailSender.send()` token resolution flow.
**Key difference from quote path:** Quotes always create a new token on each send. Estimates reuse existing tokens because the public page resolves to the latest revision via `rootEstimateId`.

### Anti-Patterns to Avoid

- **Sharing validators across quote/estimate boundaries:** The `@Matches(/estimate/i)` subject validator MUST live only in `SendEstimateRequest` and `UpdateEstimateSettingsRequest`. Never lift to a shared helper that quote requests import. [VERIFIED: D-SND-04 separation-over-DRY]
- **Storing legal disclaimer in DB:** The disclaimer is static HTML in the Maizzle template. No `legalCopy` field in `estimate-settings`. [VERIFIED: D-LGL-01]
- **Cancelling followups on pure re-send:** Only the revised-send path (Draft+revisionNumber>1 -> Sent) calls `cancelAllFollowups`. Pure re-send (Sent/Viewed/Responded/SiteVisitRequested -> Sent) does NOT cancel. [VERIFIED: D-TKN-06 vs D-TKN-07]
- **Using `as` type assertions for entity/DTO conversion:** Use mapping functions per CLAUDE.md directives. [VERIFIED: project constraints]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTML sanitisation | Custom parser | Allowlist-based tag stripping (copy from `QuoteEmailSender.sanitizeHtml`) | Already battle-tested in production |
| Email rendering | Custom template engine | Maizzle `render()` with dynamic import fallback | Already handles CSS inlining, Outlook VML, mobile rendering |
| Template variable resolution | Custom regex engine | `resolveTemplateVariables()` function from `SendQuoteForm` | Simple `{{var}}` replacement, already handles missing vars |
| Money formatting | Manual string formatting | Server-side `formatRange` utility mirroring Phase 43 frontend helper | Must produce byte-identical output for consistency |
| Date formatting | Custom date logic | `Luxon DateTime.toFormat("d MMMM yyyy")` | Already used in `QuoteEmailSender` for `quoteDate` |
| Subject validation | Custom validator function | `@Matches(/estimate/i)` class-validator decorator | Framework-integrated, auto-runs in `ValidationPipe` |

## Common Pitfalls

### Pitfall 1: Phase 41 Dependency
**What goes wrong:** Phase 41 (Estimate Module CRUD Backend) has not been executed yet. The `estimate` module currently only has the Phase 42 followup canceller interface.
**Why it happens:** Phase 44 depends on Phase 41 providing `EstimateController`, `EstimateRetriever`, `EstimateTransitionService`, `EstimateRepository`, `EstimateModule`, `IEstimateDto`, `IEstimateEntity`, `IEstimateResponse`, `EstimateStatus` enum, and the full `ALLOWED_TRANSITIONS` map.
**How to avoid:** Plans must explicitly state Phase 41 as a prerequisite. The first plan wave should assume Phase 41 artifacts exist (controller, services, repositories, entities, DTOs, enums, transition map).
**Warning signs:** Import errors for `@estimate/*` paths that don't exist yet.

### Pitfall 2: Maizzle Dynamic Import ESM/CJS Interop
**What goes wrong:** The `Function('return import("@maizzle/framework")')()` dynamic import pattern in `QuoteEmailRenderer` is a workaround for CommonJS/ESM interop in NestJS.
**Why it happens:** NestJS uses CommonJS modules; Maizzle is ESM-only.
**How to avoid:** Copy the exact pattern from `quote-email-renderer.service.ts` lines 28-42 verbatim. Do not attempt to simplify or "clean up" the import. The fallback (`catch` returning populated template without CSS inlining) is the safety net.
**Warning signs:** `ERR_REQUIRE_ESM` at runtime; email renders without inline styles.

### Pitfall 3: Token documentId Must Be rootEstimateId
**What goes wrong:** Using the per-revision `estimateId` as the token's `documentId` breaks Phase 45's resolution model where the customer page resolves to the latest revision.
**Why it happens:** Intuitive to use the specific estimate ID, but the chain-resolution model requires the root ID.
**How to avoid:** Always pass `estimate.rootEstimateId` (or `estimate.id` if `revisionNumber === 1` and `rootEstimateId` is self-referential) when creating or looking up tokens.
**Warning signs:** Token lookup returns null on re-send of a revised estimate.

### Pitfall 4: Missing Status Transitions
**What goes wrong:** `EstimateTransitionService.transition()` rejects with `ESTIMATE_INVALID_TRANSITION` when re-sending from Viewed, Responded, or SiteVisitRequested status.
**Why it happens:** Phase 41's `ALLOWED_TRANSITIONS` map only includes `DRAFT -> SENT` and `SENT -> SENT`. Phase 44 must add three new transitions.
**How to avoid:** Add `VIEWED -> SENT`, `RESPONDED -> SENT`, `SITE_VISIT_REQUESTED -> SENT` to the `ALLOWED_TRANSITIONS` map in `EstimateTransitionService`. Add corresponding test assertions.
**Warning signs:** 422 errors when attempting to re-send a viewed or responded estimate.

### Pitfall 5: Resend SDK Error Mapping
**What goes wrong:** Resend API errors bubble as unstructured exceptions.
**Why it happens:** The `EmailSenderService` wraps Resend but error mapping was fixed in quick-task 260407-n0u.
**How to avoid:** Verify `EmailSenderService` error mapping is in place. `EstimateEmailSender` catches `HttpException` and re-throws; unknown errors become 500.
**Warning signs:** Unhandled promise rejections on email delivery failure.

### Pitfall 6: BusinessCreator Double-Creator Failure
**What goes wrong:** If `DefaultEstimateSettingsCreatorService.create()` fails after `DefaultQuoteSettingsCreatorService.create()` succeeds, the business has quote settings but no estimate settings.
**Why it happens:** No transaction wrapping both creators. Accepted risk per D-SET-06.
**How to avoid:** Accept the risk. The failure surfaces immediately. A retry or manual PATCH fixes it. The existing quote-settings path has the same non-transactional behavior.
**Warning signs:** Business creation returns 500 but business record exists.

### Pitfall 7: Subject Validation Case Sensitivity
**What goes wrong:** Using `@Contains('Estimate')` would require exact case match, rejecting "estimate" or "ESTIMATE".
**Why it happens:** `@Contains` is case-sensitive by default.
**How to avoid:** Use `@Matches(/estimate/i, { message: "Estimate email subject must contain the word 'Estimate'" })` for case-insensitive matching.
**Warning signs:** Users get validation errors for "Your estimate for..." subjects.

## Code Examples

### Subject Validation Decorator (SendEstimateRequest)
```typescript
// Source: verified against existing visit-type @Matches usage in codebase
import { IsEmail, IsString, IsNotEmpty, IsOptional, IsBoolean, Matches } from "class-validator";

export class SendEstimateRequest {
  @IsEmail()
  to: string;

  @IsString()
  @IsNotEmpty()
  @Matches(/estimate/i, { message: "Estimate email subject must contain the word 'Estimate'" })
  subject: string;

  @IsString()
  @IsNotEmpty()
  message: string;

  @IsOptional()
  @IsBoolean()
  saveEmail?: boolean;
}
```
[VERIFIED: `@Matches` pattern used in `visit-type/requests/create-visit-type.request.ts`]

### DocumentTokenRepository Extension Methods
```typescript
// Source: mirrors existing findLatestNonRevokedForDocument pattern in document-token.repository.ts
public async findActiveByDocumentId(
  documentId: string,
  documentType: "quote" | "estimate",
): Promise<IDocumentTokenDto | null> {
  const entity = await this.fetcher.findOne<IDocumentTokenEntity>(
    DocumentTokenRepository.COLLECTION,
    {
      documentId: new ObjectId(documentId),
      documentType,
      revokedAt: { $exists: false },
      expiresAt: { $gt: new Date() },
    } as never,
    { sort: { createdAt: -1 } },
  );
  if (!entity) return null;
  return this.toDto(entity);
}

public async extendExpiry(tokenId: string, newExpiresAt: DateTime): Promise<IDocumentTokenDto> {
  const result = await this.writer.findOneAndUpdate<IDocumentTokenEntity>(
    DocumentTokenRepository.COLLECTION,
    { _id: new ObjectId(tokenId) } as never,
    { $set: { expiresAt: newExpiresAt.toJSDate(), updatedAt: new Date() } } as never,
  );
  if (!result) {
    throw new ResourceNotFoundError(ErrorCodes.DOCUMENT_TOKEN_NOT_FOUND, "Document token not found");
  }
  return this.toDto(result as unknown as IDocumentTokenEntity);
}
```
[VERIFIED: pattern consistent with existing `DocumentTokenRepository` methods]

### EstimateEmailSendType Enum
```typescript
// Source: D-AUDIT-02 from CONTEXT.md
export enum EstimateEmailSendType {
  INITIAL = "initial",
  RESEND = "resend",
  FOLLOWUP_3D = "followup_3d",
  FOLLOWUP_10D = "followup_10d",
  FOLLOWUP_21D = "followup_21d",
}
```
[VERIFIED: enum convention matches existing enums like `QuoteStatus`, `BusinessStatus`]

### RTK Query Estimate Settings Endpoints (businessApi.ts extension)
```typescript
// Source: mirrors existing getQuoteSettings/updateQuoteSettings pattern in businessApi.ts
getEstimateSettings: builder.query<EstimateSettings, string>({
  query: (businessId) => ({
    url: `/v1/business/${businessId}/estimate-settings`,
  }),
  transformResponse: (response: StandardResponse<EstimateSettings>, _meta, businessId) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    return {
      id: "",
      businessId,
      estimateEmailSubject: undefined,
      estimateEmailBody: undefined,
    } as EstimateSettings;
  },
  providesTags: (_result, _error, businessId) => [
    { type: "Business", id: `estimate-settings-${businessId}` },
  ],
}),
updateEstimateSettings: builder.mutation<
  EstimateSettings,
  { businessId: string; data: UpdateEstimateSettingsRequest }
>({
  query: ({ businessId, data }) => ({
    url: `/v1/business/${businessId}/estimate-settings`,
    method: "PATCH",
    body: data,
  }),
  transformResponse: (response: StandardResponse<EstimateSettings>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No estimate settings data returned");
  },
  invalidatesTags: (_result, _error, { businessId }) => [
    { type: "Business", id: `estimate-settings-${businessId}` },
  ],
}),
```
[VERIFIED: pattern identical to existing `getQuoteSettings`/`updateQuoteSettings` in `businessApi.ts`]

### Legal Disclaimer Card (Maizzle HTML)
```html
<!-- Source: D-LGL-02, specifics section of CONTEXT.md -->
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0"
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
[VERIFIED: follows `quote-email.html` table-based email pattern; inline styles for email client compatibility]

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| SendGrid (`@sendgrid/mail`) | Resend SDK | v1.3 (quick-task 260407-n0u) | `EmailSenderService` wraps Resend; error mapping already in place |
| `quote-token` module | `document-token` module with `documentType` discriminator | Phase 41 (partial) | Token creation already accepts `"estimate"` type |
| Per-send token creation (quotes) | Token reuse with expiry extension (estimates) | Phase 44 | New pattern: estimates reuse tokens because public page resolves latest revision |

**Note on Resend vs SendGrid:** The INTEGRATIONS.md still references SendGrid, but the actual codebase uses Resend (verified in `email-sender.service.ts` and quick-task 260407-n0u). The `EmailSenderService` interface is the same regardless of provider. [VERIFIED: codebase inspection]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 41 will provide `EstimateController`, `EstimateRetriever`, `EstimateTransitionService`, `EstimateRepository`, `IEstimateDto`, `IEstimateEntity`, `IEstimateResponse`, `EstimateStatus` enum, `ALLOWED_TRANSITIONS` map, `EstimateModule` with full DI wiring | Architecture Patterns | HIGH -- Phase 44 cannot execute without these artifacts. Plans must gate on Phase 41 completion. |
| A2 | Phase 43 will provide `EstimateDetailPage`, `EstimateActionStrip` (stub), `estimateApi.ts` (base endpoints), `Estimate` type, `features/estimates/index.ts` barrel | Architecture Patterns | HIGH -- Frontend send UI cannot be wired without the Phase 43 page and component stubs. |
| A3 | `@Matches` class-validator decorator supports regex flags (case-insensitive `/i`) | Code Examples | LOW -- verified `@Matches` is used in codebase with regex; regex flags are standard JS. |
| A4 | `MongoDbWriter.findOneAndUpdate` returns the updated document (returnDocument: 'after' default) | Code Examples | LOW -- already relied upon in `quote-settings.repository.ts` upsert pattern. |

## Open Questions

1. **Phase 41 execution status**
   - What we know: Phase 41 plans exist (41-01 through 41-08). Phase 41 code has NOT been executed -- `estimate` module only has followup canceller interface.
   - What's unclear: When Phase 41 will land. All Phase 44 plans assume it's complete.
   - Recommendation: Plans should explicitly state "assumes Phase 41 artifacts exist" in prerequisites. If Phase 41 hasn't landed when Phase 44 execution begins, execute Phase 41 first.

2. **`format-range.utility.ts` location**
   - What we know: D-TPL-02 says it should produce byte-identical output to Phase 43's frontend helper. It's estimate-specific today.
   - What's unclear: Whether any non-estimate code will ever need range formatting.
   - Recommendation: Place under `@estimate/utilities/` since it's domain-specific. If needed elsewhere later, move to `@core/utilities/`.

3. **Path alias for estimate-settings module**
   - What we know: The codebase uses `@quote-settings/*` for `src/quote-settings/`. Phase 44 needs `@estimate-settings/*`.
   - What's unclear: Whether Phase 41 already registered this alias in `tsconfig.json`.
   - Recommendation: Verify `tsconfig.json` paths include `@estimate-settings/*` -> `src/estimate-settings/*`. If missing, add it.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies beyond what the project already requires). All tools (Node.js 22, MongoDB, npm, TypeScript) are already verified by existing project CI. No new runtime dependencies.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 |
| Framework (UI) | Vitest 4.1.3 |
| Config file (API) | `trade-flow-api/jest.config.ts` (or package.json jest config) |
| Config file (UI) | `trade-flow-ui/vite.config.ts` (extends via vitest) |
| Quick run command (API) | `cd trade-flow-api && npm run test -- --testPathPattern="estimate"` |
| Quick run command (UI) | `cd trade-flow-ui && npm run test -- --testPathPattern="estimate"` |
| Full suite command (API) | `cd trade-flow-api && npm run ci` |
| Full suite command (UI) | `cd trade-flow-ui && npm run ci` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SND-01 | EstimateEmailSender.send() sends email and transitions status | unit | `npm run test -- estimate-email-sender` | Wave 0 |
| SND-02 | EstimateSettingsController GET/PATCH endpoints | unit | `npm run test -- estimate-settings` | Wave 0 |
| SND-02 | EstimateEmailSettings UI renders and saves | unit | `npm run test -- EstimateEmailSettings` | Wave 0 |
| SND-03 | SendEstimateForm pre-fills and submits | unit | `npm run test -- SendEstimateForm` | Wave 0 |
| SND-04 | EstimateEmailSendCreator persists audit row after send | unit | `npm run test -- estimate-email-send-creator` | Wave 0 |
| SND-05 | EstimateEmailRenderer output always contains disclaimer | unit | `npm run test -- estimate-email-renderer` | Wave 0 |
| SND-06 | EstimateEmailSender reuses token on re-send | unit | `npm run test -- estimate-email-sender` | Wave 0 |
| SND-07 | SendEstimateRequest rejects subject without "Estimate" | unit | `npm run test -- send-estimate.request` | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern="estimate"` (both repos)
- **Per wave merge:** `npm run ci` (both repos)
- **Phase gate:** Full `npm run ci` green in both repos before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `trade-flow-api/src/estimate/test/services/estimate-email-sender.service.spec.ts` -- covers SND-01, SND-06
- [ ] `trade-flow-api/src/estimate/test/services/estimate-email-send-creator.service.spec.ts` -- covers SND-04
- [ ] `trade-flow-api/src/email/test/services/estimate-email-renderer.service.spec.ts` -- covers SND-05
- [ ] `trade-flow-api/src/estimate-settings/test/services/*` -- covers SND-02
- [ ] `trade-flow-api/src/estimate/test/mocks/estimate-email.mock-generator.ts` -- shared test fixtures
- [ ] `trade-flow-api/src/document-token/test/repositories/document-token.repository.spec.ts` -- extend with findActiveByDocumentId, extendExpiry tests

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | Yes | `JwtAuthGuard` on all new endpoints (estimate-settings GET/PATCH, estimates/:id/send) |
| V3 Session Management | No | No session state introduced; token-based document access is Phase 45 scope |
| V4 Access Control | Yes | `EstimatePolicy` / `EstimateSettingsPolicy` gates all mutations; business-scoped |
| V5 Input Validation | Yes | `class-validator` decorators on `SendEstimateRequest`, `UpdateEstimateSettingsRequest`; `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true` |
| V6 Cryptography | No | No new crypto; document token generation reuses existing `crypto.randomUUID()` in `DocumentTokenCreator` |

### Known Threat Patterns for This Phase

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| HTML injection in email body | Tampering | `sanitizeHtml()` allowlist stripping (`p`, `br`, `strong`, `em`, `a`, `ul`, `ol`, `li`) |
| XSS in template variables | Tampering | `escapeHtml()` applied to all `{{ }}` double-brace variables; only `{{{ }}}` triple-brace is raw (sanitised message only) |
| Unauthorised estimate send | Elevation of Privilege | `JwtAuthGuard` + `EstimatePolicy.canUpdate()` checks business ownership |
| Email to arbitrary recipients | Information Disclosure | `@IsEmail()` validates format; business-scoped policy ensures only business owner can trigger sends |
| Audit row tampering | Repudiation | `estimate_email_sends` collection is write-once; no update/delete endpoints exposed; no TTL |

## Sources

### Primary (HIGH confidence)
- Codebase inspection of `trade-flow-api/src/quote/services/quote-email-sender.service.ts` (120 lines)
- Codebase inspection of `trade-flow-api/src/email/services/quote-email-renderer.service.ts` (48 lines)
- Codebase inspection of `trade-flow-api/src/email/templates/quote-email.html` (77 lines)
- Codebase inspection of `trade-flow-api/src/quote-settings/` module (all files)
- Codebase inspection of `trade-flow-api/src/document-token/` module (all files)
- Codebase inspection of `trade-flow-ui/src/features/quotes/components/SendQuoteForm.tsx` (161 lines)
- Codebase inspection of `trade-flow-ui/src/features/business/components/QuoteEmailSettings.tsx` (131 lines)
- Codebase inspection of `trade-flow-ui/src/features/business/api/businessApi.ts` (98 lines)
- Codebase inspection of `trade-flow-api/src/core/errors/error-codes.enum.ts` (error code naming convention)
- Phase 44 CONTEXT.md (all decisions D-LGL, D-AUDIT, D-TKN, D-DOC, D-SET, D-SND, D-TPL, D-UI, D-API, D-GATE)
- Phase 44 UI-SPEC.md (component inventory, layout contracts, copywriting)

### Secondary (MEDIUM confidence)
- [class-validator @Contains/@Matches documentation](https://github.com/typestack/class-validator) -- verified `@Matches` supports regex with flags

### Tertiary (LOW confidence)
- None -- all claims verified against codebase or official documentation.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies; all libraries already in use
- Architecture: HIGH -- 1:1 mirror of existing quote-send patterns with verified diffs
- Pitfalls: HIGH -- drawn from actual codebase inspection and CONTEXT.md decisions

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable patterns, no fast-moving dependencies)
