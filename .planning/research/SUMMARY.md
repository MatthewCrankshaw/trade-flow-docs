# Research Summary — v1.8 Estimates

**Project:** Trade Flow
**Milestone:** v1.8 Estimates (brownfield, subsequent milestone on shipped v1.7 SaaS)
**Domain:** UK sole-trader SaaS — estimates as a parallel document type to quotes
**Researched:** 2026-04-10
**Confidence:** HIGH overall (HIGH for stack/architecture, HIGH for pitfalls/UK legal, MEDIUM-HIGH for features)

## Executive Summary

v1.8 Estimates is a **pattern-level milestone, not an infrastructure milestone**: zero new runtime dependencies, ~70% reuse of existing v1.2–v1.7 code (quote module shape, token infrastructure, Maizzle/Resend email pipeline, BullMQ worker, atomic counters, Radix UI primitives), and ~30% genuinely new code for contingency ranges, soft-response flow, follow-up scheduler, revisions, and convert-to-quote. Every capability the milestone needs — delayed jobs via `Queue.add({ delay })`, a single-thumb slider via `radix-ui@1.4.3`, self-referential Mongoose documents, range-formatted money via the existing `dinero.js` wrapper — is already satisfied by libraries installed in earlier milestones.

The **strategic wedge** is that every major competitor (Tradify, Jobber, ServiceM8, YourTradebase, Powered Now, Fergus) treats "estimate" as cosmetic nomenclature — a relabel of the quote feature with identical semantics and the same binary accept/decline flow. Trade Flow's v1.8 does something none of them do: ships estimates as a **true parallel document type** with price ranges, a four-button soft-response flow (Book site visit / Send quote / I have a question / Not right now), automated follow-ups at 3/10/21 days, invisible-to-user versioned revisions, and non-binding legal language baked into the default template. UK law reinforces this design — the Consumer Rights Act 2015 and Dispute Resolution Ombudsman treat estimates as non-binding guide prices, but case law is clear that a badly worded estimate (single "Total: £X", an "Accept" button, no disclaimer) can become a binding offer on unconditional acceptance, exposing Trade Flow users to contract risk if we ship estimates that behave like quotes.

The **highest-consequence risks** are UK legal copy (binding-offer exposure via customer-facing wording), BullMQ delayed-job hygiene (orphaned follow-ups after revise/convert/decline → brand-damaging emails, silent data loss if Redis has no AOF persistence), public token semantics (must resolve to the latest revision, not the one the customer was emailed), estimate→quote conversion correctness (snapshot, don't reference; idempotent on double-click), and rounding artifacts between the v1-API and v2-alpha-UI dinero.js versions on contingency math. Each of these must be addressed as a **requirements-level constraint** in the roadmap, not left as a phase-level implementation detail.

## Key Findings

### Recommended Stack

See **[STACK.md](./STACK.md)** for full detail. **No new npm packages.** The work is entirely pattern-level on top of infrastructure shipped in v1.2–v1.7.

**Core technologies (all already installed):**
- **BullMQ 5.71.0 + @nestjs/bullmq 11.0.4 + ioredis 5.10.1** — delayed follow-up jobs via native `Queue.add(jobName, data, { delay, jobId })`; no cron library needed
- **Mongoose 9.1.5** — self-referential `parentEstimateId` + `rootEstimateId` + `revisionNumber` for invisible versioning
- **radix-ui 1.4.3 (meta package)** — contingency slider via `import { Slider } from "radix-ui"`; standalone `@radix-ui/react-slider` must NOT be added (the v1.7 shadcn/ui unified-package migration is the current idiom)
- **dinero.js** — v1.9.1 on API for server-side money, v2.0.0-alpha.14 on UI for display; range math computed on the API only, UI just renders what the API returns
- **luxon 3.5.1** — already standardized in v1.6; `DateTime.plus({ days: 3 })` for follow-up offsets
- **Resend + Maizzle** — existing v1.3 email pipeline; add estimate Maizzle templates alongside existing quote templates
- **react-hook-form + valibot** — existing form stack handles the create dialog's document-type toggle and conditional fields

**Explicit non-additions:** `@nestjs/schedule`/cron (BullMQ already handles delay), `@radix-ui/react-slider` standalone, currency-formatting libraries, `mongoose-sequence` / `mongoose-version` plugins, `xstate`/`ts-pattern` for the eight-state lifecycle. Each is called out in STACK.md with the rationale.

### Expected Features

See **[FEATURES.md](./FEATURES.md)** for full competitor analysis and legal framing. All of the v1.8 target features in PROJECT.md are validated by research — **the scope is correct, not speculative.**

**Must have (table stakes — users expect these):**
- Separate document type with distinct E-YYYY-NNN numbering
- Non-binding legal language on customer-facing page and email body
- Line items shared with quotes (same data model)
- Customer-facing page via token (reuse v1.3 infra)
- Email delivery with configurable template
- View tracking (firstViewedAt) parity with quotes
- Status lifecycle: Draft → Sent → Viewed → Responded → (SiteVisitRequested / Converted / Declined / Expired)
- Edit and resend (versioned revisions under the hood)
- Convert to Quote action with back-link
- Decline with structured reason

**Should have (the differentiators — Trade Flow's wedge):**
- True parallel document type with different semantics (not a nomenclature relabel — the single biggest competitive gap)
- Contingency slider 0–30% in 5% steps, default 10%, with range / "From £X" display modes
- Four-button soft customer response flow: Book a site visit / Send me a quote / I have a question / Not right now
- Structured decline reason enum (Too expensive / Going with another trader / Not the right time / Decided not to do the work / Work out of scope / Other) — feeds future BI milestone
- Quick-tap uncertainty notes (chips for site inspection, pipework, materials, access)
- Automated follow-up sequence at 3 / 10 / 21 days via BullMQ delayed jobs, defaults only, resets on revision
- Invisible versioning + collapsed History section (not visible "v2" UX)
- Message-based "Book a site visit" (capture preferred windows as a request, not a full calendar sync)
- Back-link on converted quote ("Converted from E-2026-0042")

**Defer (v1.8.x / v1.9 / v2+ — explicitly out of scope for v1.8):**
- Trader-initiated follow-up cadence override (ship defaults only first)
- Estimate templates/presets (wait for usage data)
- Customer counter-offer on estimate
- Estimate PDF export (deferred with quote PDF)
- Calendar sync for site visits (message-based flow may prove sufficient)
- Electronic signatures on estimates (doubles down on binding semantics)
- Deposit collection on estimates (legally incoherent on non-binding doc)
- AI-generated contingency suggestions
- Email tracking pixel (PECR/GDPR risk; link-click tracking is fine)
- Customer login for estimate responses

### Architecture Approach

See **[ARCHITECTURE.md](./ARCHITECTURE.md)** for full detail grounded in the `trade-flow-api` source. The research is HIGH confidence because the researcher read the existing module files directly.

**Headline decision: a separate `EstimateModule` that mirrors `QuoteModule`, not a polymorphic discriminator on the existing quote module.** Quote status lifecycles, field surfaces, and transition graphs are genuinely different from estimates; unifying them would force nullable-field bloat, `documentType` branching across every service, and silent correctness bugs. Separate modules keep strict Controller → Service → Repository layering intact and match the existing precedent (`quote-token` and `quote-settings` are already sibling feature modules, not embedded in `quote`).

**Major components:**
1. **New `src/estimate/` module** — full mirror of `src/quote/` (controllers, creator/retriever/updater/deleter services, repository, policy, DTOs, responses, entity, status enum, transition service) plus estimate-specific services: `EstimateNumberGenerator` (E-YYYY-NNN via `estimate_counters`), `EstimateTotalsCalculator` (with contingency math), `EstimateReviser`, `EstimateFollowupScheduler`, `EstimateToQuoteConverter`, `EstimateResponseHandler`, `EstimateEmailSender`.
2. **Renamed `document-token` module** (from `quote-token`) — generalised with `documentType: "quote" | "estimate"` discriminator and `documentId` (renamed from `quoteId`). Guard becomes `DocumentSessionAuthGuard`. One-shot MongoDB migration renames the collection. **Duplicating quote-token is explicitly rejected** because the guard encodes the most-tested customer-facing logic (rate limiting, expiry, revocation, `firstViewedAt`).
3. **Renamed `document-settings` module** (from `quote-settings`) — adds `estimateEmailTemplate` alongside `quoteEmailTemplate` in one settings API and one Settings > Templates tab. **(Open question: is this rename in v1.8 scope or deferred?)**
4. **Shared `quote_line_items` collection with `parentType` widening** — line items are the same thing for quotes and estimates, and the existing bundle/tax/pricing factories already operate on "a line item attached to a parent." Add `parentType: "quote" | "estimate"` + `estimateId?` fields; backfill existing rows with `updateMany({}, { $set: { parentType: "quote" } })`. Rejected alternative: separate `estimate_line_items` collection (would duplicate ~15 files of bundle/tax logic for zero gain).
5. **New `ESTIMATE_FOLLOWUPS` BullMQ queue + `EstimateFollowupProcessor` in WorkerModule** — deterministic `jobId` pattern `estimate-followup:${estimateId}:${revisionNumber}:${step}` for O(1) idempotent cancellation via `queue.remove()`. Defence-in-depth: worker re-reads estimate state at execution time and silently no-ops on `CONVERTED`/`DECLINED`/`EXPIRED`/`SITE_VISIT_REQUESTED`/deleted/non-current.
6. **Revisions: flat collection, `parentEstimateId` + `revisionNumber` + `isCurrent`** — compound indexes `{ parentEstimateId: 1, revisionNumber: -1 }` and partial unique `{ parentEstimateId: 1, isCurrent: 1 }` guarantee one current per chain and O(log n) latest-revision lookup. Rejected embedded-array alternative (16MB doc limit, poor query patterns, line items already external).
7. **New `src/features/estimates/` frontend feature module** mirroring `src/features/quotes/`, plus a tiny new `src/features/documents/` module owning the shared `CreateDocumentDialog` with the Quote/Estimate type toggle. Separate `publicEstimateApi` RTK Query slice (no Firebase JWT headers, same pattern as v1.3 `publicQuoteApi`). New public page `CustomerEstimatePage.tsx` at `/estimate/:token`.
8. **Convert-to-Quote lives in the estimate module, not the quote module** — `EstimateToQuoteConverter` depends on `QuoteCreator` via standard DI (one-way; quote knows nothing about estimates). No `forwardRef`. Back-link stored as `convertedToQuoteId` on the estimate side.

### Critical Pitfalls

See **[PITFALLS.md](./PITFALLS.md)** for all 21 pitfalls. These five are the highest-consequence and **must be encoded as requirements-level constraints**, not left as phase implementation details.

1. **UK legal binding-offer exposure (Pitfall 1)** — default email template and customer page must include explicit non-binding disclaimer ("This is an estimate, not a fixed price… A firm quote will be provided after a site visit."); customer page MUST use "From £X" or range display (never a single "Total"); response buttons MUST NOT include the word "Accept"; email subject must include "Estimate" not "Quote"; rendered email HTML must be stored on the estimate for future dispute defence. **Requires legal-review pass on default copy before Phase E/F.**
2. **Redis AOF persistence (Pitfall 4)** — production Redis (Railway) MUST have `appendonly yes` / `appendfsync everysec` before the first follow-up ships. Without it, a single Redis restart silently wipes 21 days of scheduled jobs with no alert. Previous milestones got away with ephemeral Redis because webhook jobs processed within seconds; delayed jobs live in Redis for weeks. Production smoke test: schedule a 60s delayed job, restart Redis, verify it still fires.
3. **Deterministic jobIds for follow-up cancellation (Pitfall 2 + 15)** — the `jobId` MUST be `estimate-followup:${estimateId}:${revisionNumber}:${step}` so that (a) re-adding is a silent no-op (BullMQ dedup), (b) `queue.remove(jobId)` is O(1), (c) revision bump gets a fresh jobId namespace. Every exit transition (Responded/Converted/Declined/Expired/Lost/soft-deleted) MUST call `cancelAllFollowups()`; revising MUST cancel the previous revision's jobs and schedule the new revision's. Worker processor adds defence-in-depth by re-reading estimate state and skipping if no longer in a follow-up-worthy status.
4. **Public token resolves to latest revision (Pitfall 3)** — customer's email link is token-bound, not revision-bound. Public endpoint `GET /v1/public/estimate/:token` MUST query by the token's root/parent reference and return the `isCurrent: true` revision. Revising MUST reuse the same token, not issue a new one. Without this, customers see stale numbers after "edit and resend" and dispute the price. Compound with Pitfall 1 and the legal exposure worsens.
5. **Idempotent estimate-to-quote conversion with snapshot semantics (Pitfall 6)** — conversion copies line items (with literal tax rate percentages) into the new quote; NO live references to the estimate or source collections post-conversion. API accepts an `Idempotency-Key` header (or server-computes a key from `estimateId + revisionNumber + "convert"`), stores the resulting quote ID for 24h, and returns the same quote on retry. UI disables the Convert button as soon as clicked. Conversion always pulls from the **latest revision**, not whatever is loaded in the UI. After conversion, source estimate transitions to `Converted` and further revisions are blocked.

Additional critical constraint from Pitfall 5 (rounding): **contingency range is computed on the API only**, using `dinero.js@1.9.1` `.percentage()`; the UI displays whatever the API returns. Never multiply by 1.10 anywhere. Never denormalize `lowAmount`/`highAmount` onto the entity — recompute on every read (the v1.2 quote-totals decision).

## Implications for Roadmap

Research strongly recommends the build order from ARCHITECTURE.md §10. The eight-phase sequence below is directly lifted from architecture research and annotated with the relevant pitfalls, features, and open questions for each phase.

### Phase A — Foundations (prerequisite for everything)

**Rationale:** Every subsequent phase depends on the renamed token module and widened line items. Doing the renames first means every new file in Phases B–H uses the correct names immediately — no rewrite later. This is the "do the refactors before the additions" principle.

**Delivers:**
- Rename `quote-token` → `document-token` (entity widened with `documentType` + `documentId`; migration script renames collection and fields)
- Rename `quote-settings` → `document-settings` (adds `estimateEmailTemplate` field alongside existing quote template) — **OPEN QUESTION: is this rename in v1.8 scope?**
- Widen `quote_line_items` with `parentType: "quote" | "estimate"` + optional `estimateId` field; backfill existing rows via `updateMany({}, { $set: { parentType: "quote" } })`
- Update `PublicQuoteController` to read `request.documentToken` instead of `request.quoteToken` (breaking change in controller request typing — audit all callers)

**Avoids:** Pitfalls 3 (stale token), 7 (document type confusion), 20 (N+1 on line items)

**Research flag:** NO additional research needed — all patterns verified against existing source.

### Phase B — Estimate Module CRUD (backend only)

**Rationale:** Before worrying about customer communication, the API must be able to create, read, update, and delete estimates with correct numbering, policy, contingency math, and status transitions. This unblocks frontend work and establishes the happy path for all subsequent phases.

**Delivers:**
- `EstimateEntity` (with contingency, display mode, uncertainty notes, revision fields, structured decline reason enum), `IEstimateDto`, `EstimateStatus` enum, transition table
- `EstimateRepository`, `EstimatePolicy`, `EstimateNumberGenerator` (E-YYYY-NNN via new `estimate_counters` collection — mirror the v1.2 `quote_counters` atomic pattern verbatim)
- `EstimateCreator`, `EstimateRetriever`, `EstimateUpdater`, `EstimateDeleter`
- `EstimateTotalsCalculator` with contingency math (API side, dinero v1.9.1 `.percentage()`)
- `EstimateTransitionService` (eight-state graph)
- `EstimateController` (CRUD, no send yet)
- MongoDB indexes from ARCHITECTURE.md §3: `(businessId, createdAt)`, `(jobId, createdAt)`, unique `(businessId, number)` with partial filter on `deletedAt`, plus the revision indexes (defensive even if Phase C ships later)
- Backend unit tests

**Addresses (features):** Document type toggle, E-YYYY-NNN numbering, contingency slider data model, structured decline reason taxonomy, status lifecycle, non-binding legal framing at the type level

**Avoids (pitfalls):** Pitfall 5 (rounding — API owns the math), Pitfall 7 (type confusion at discriminated-union boundary), Pitfall 8 (counter race — copy quote_counters exactly), Pitfall 12 (concurrent edit 409 via compound unique index on `(parentEstimateId, revisionNumber)`)

**Research flag:** NO research needed — explicitly follows the v1.2 quote module pattern.

### Phase C — Revisions (depends on B)

**Rationale:** Revisions are a second-order concept on top of base estimates. Getting the base data model right first avoids reworking revision logic. Can slip in favour of Phase D if needed for schedule pressure.

**Delivers:**
- `isCurrent` / `parentEstimateId` / `revisionNumber` logic in `EstimateCreator`
- `EstimateReviser` service (loads current, copies fields + line items, flips `isCurrent`, resets follow-up queue)
- `GET /v1/estimate/:id/revisions` endpoint
- Partial unique index on `(parentEstimateId, isCurrent: true)` to prevent duplicate currents

**Addresses (features):** Invisible versioned revisions; "edit and resend" UX

**Avoids (pitfalls):** Pitfall 3 (latest revision resolution), Pitfall 12 (concurrent edit), Pitfall 15 (soft-delete follow-ups on old revisions)

**Research flag:** NO — MongoDB partial unique indexes are standard; the two-write `isCurrent` flip needs a short design note on failure handling (open question #4 below).

### Phase D — Frontend Estimate CRUD (parallel with Phase C)

**Rationale:** Can be built against Phase B alone. Delivers visible progress to the user while backend revisions work proceeds in parallel.

**Delivers:**
- `src/features/estimates/` scaffolding (api slice, components, hooks)
- `EstimatesPage`, `EstimateDetailPage`
- `EstimateFormDialog` with `ContingencySlider` (Radix Slider from `radix-ui` meta package, `min=0 max=30 step=5 defaultValue=10`)
- `src/features/documents/CreateDocumentDialog` with Quote/Estimate type toggle
- Wire into job detail page "Estimates" section alongside "Quotes"
- Range / "From £X" display component powered by API-computed values

**Addresses (features):** Contingency slider, range vs "From £X" display, uncertainty notes quick-tap chips, document type toggle

**Avoids (pitfalls):** Pitfall 5 (UI displays only; math on API), Pitfall 7 (discriminated props on shared components), Pitfall 12 (UI handles 409 with "refresh to see latest changes" banner)

**Research flag:** NO — Radix Slider from the meta package is documented in STACK.md with a worked example.

### Phase E — Email + Send Flow (depends on B, D)

**Rationale:** Sending requires an estimate to exist (Phase B) and a settings module to host the template (Phase A rename). Nothing earlier depends on email.

**Delivers:**
- Estimate Maizzle email template with mandatory non-binding disclaimer
- `EstimateEmailRenderer` service (sibling to `QuoteEmailRenderer`)
- `EstimateEmailSender` service
- `POST /v1/estimate/:id/send` endpoint (transitions Draft → Sent, persists `sentAt`, stores rendered HTML on estimate for audit)
- `SendEstimateDialog` UI (message editor)
- Configurable estimate email template in Business settings tab

**Addresses (features):** Estimate email template with Maizzle, configurable template, non-binding legal language in default template

**Avoids (pitfalls):** **Pitfall 1 (binding offer — CRITICAL, legal review required)**, Pitfall 13 (spam filter triggers), Pitfall 17 (GB date format via Luxon), Pitfall 18 (no tracking pixel)

**Research flag:** **YES** — needs a targeted research pass on UK-consumer-law-appropriate template copy before the default template ships. Candidate for `/gsd:research` during phase planning.

### Phase F — Public Customer Page (depends on A, E)

**Rationale:** Customer-facing is the most-scrutinised surface. Ship it after the happy path (create → send) works so testers can see what customers see.

**Delivers:**
- Reuse `DocumentSessionAuthGuard` from Phase A
- `PublicEstimateController` at `/v1/public/estimate/:token`
- `PublicEstimateRetriever` (returns latest revision — Pitfall 3)
- `EstimateResponseHandler` service handling the four response paths
- `publicEstimateApi` RTK Query slice (no Firebase JWT, mirror `publicQuoteApi`)
- `CustomerEstimatePage.tsx` at `/estimate/:token`
- Four response buttons component: Book a site visit / Send me a quote / I have a question / Not right now
- Structured decline reason form on "Not right now"
- Site visit request form (preferred windows, availability notes)
- View tracking via `firstViewedAt` on token (reuse v1.3 pattern)

**Addresses (features):** Customer-facing page via token, four-button soft response flow, structured decline reasons, view tracking parity, site visit request (message-based, no calendar sync)

**Avoids (pitfalls):** Pitfall 1 (no "Accept" button, non-binding copy on page), Pitfall 3 (latest revision resolution), Pitfall 14 (state check before accepting customer actions — terminal states show friendly message), Pitfall 16 (cryptographic token, no ID leakage), Pitfall 21 (zero non-essential cookies on public page)

**Research flag:** POSSIBLY — if Phase E's legal-copy research has not covered the customer page, include it here. Otherwise NO.

### Phase G — Follow-up Queue (depends on E, worker)

**Rationale:** No reason to schedule follow-ups before basic send works (Phase E). Must ship before Phase H (convert) because convert calls `cancelAllFollowups`.

**Delivers:**
- `QUEUE_NAMES.ESTIMATE_FOLLOWUPS` registration in `queue.constant.ts` and `queue.module.ts`
- `EstimateFollowupScheduler` service (producer side) with deterministic `jobId` scheme including `revisionNumber`
- `EstimateFollowupProcessor` in WorkerModule with defence-in-depth state check
- Wire scheduler into `EstimateEmailSender.send` (on success → schedule 3/10/21)
- Wire `cancelAllFollowups` into `EstimateReviser`, `EstimateResponseHandler`, `EstimateDeleter`, `EstimateUpdater.markLost`
- Three follow-up Maizzle templates (3d, 10d, 21d — non-binding copy maintained)
- **Production Redis AOF persistence enabled (CRITICAL — infra work before first deploy)**
- Production smoke test: schedule 60s delayed job, restart Redis, verify fires

**Addresses (features):** Automated follow-up sequence at 3/10/21 days, reset on revision, cancel on response

**Avoids (pitfalls):** Pitfall 2 (orphaned follow-ups — deterministic jobIds + unified cancellation helper + worker state check), **Pitfall 4 (Redis AOF — CRITICAL)**, Pitfall 10 (post-expiry firing — worker checks expiresAt), Pitfall 11 (timezone — relative delays in UTC ms are DST-safe), Pitfall 15 (soft-delete follow-ups)

**Research flag:** NO — BullMQ delayed-job pattern is verified in STACK.md and ARCHITECTURE.md against official docs; just needs disciplined execution.

### Phase H — Convert to Quote (depends on B, existing QuoteCreator, G)

**Rationale:** Depends on follow-up cancellation (G), estimate persistence (B), and the existing QuoteCreator. Nothing downstream depends on it, so it can slip last without blocking anything.

**Delivers:**
- `EstimateToQuoteConverter` service (in `src/estimate/`, not `src/quote/` — preserves one-way DI)
- `POST /v1/estimate/:id/convert` endpoint with `Idempotency-Key` header support
- `EstimateUpdater.markConverted` method (transitions status, stores `convertedToQuoteId` back-link)
- Convert button in `EstimateActionStrip` UI (disabled on click)
- Back-link display on quote detail ("Converted from E-2026-0042")
- E2E test: create → send → view → convert → verify follow-ups cancelled and quote independent of source

**Addresses (features):** Convert to Quote with back-link, "drops contingency" (explicit pricing basis decision needed — see open questions), Revise Estimate action (Phase C component wired into action strip here)

**Avoids (pitfalls):** Pitfall 6 (snapshot semantics + idempotency key + convert from latest revision), Pitfall 7 (separate services, no shared mutation path)

**Research flag:** NO.

### Phase I — Polish (parallel / ongoing)

**Delivers:** Mark as Lost endpoint + structured reason; revision history panel UI (if Phase C shipped); `openapi.yaml` updates; E2E test coverage; optional estimate-expiry background sweep (may defer to future).

### Phase Ordering Rationale

- **Refactors before additions.** Phase A lands the token and line-item widening before any estimate code is written, so every new file uses the canonical names and nothing needs renaming later.
- **Backend-before-frontend at the data layer, parallel at the feature layer.** Phase B (backend CRUD) unblocks Phase D (frontend CRUD). Phase C (revisions) and Phase D can run in parallel.
- **Ship the happy path before customer-facing.** Create and send (B, D, E) must be testable internally before the customer page (F) is exposed, because F is the highest-legal-risk surface.
- **Convert-to-Quote is last backend phase.** It is the payoff feature but strictly downstream of everything else, and it's the only phase that can slip a few days without blocking the rest of the milestone.
- **Infra gates block phase completion.** Phase G cannot ship without Redis AOF persistence enabled on production. Phase E cannot ship without a legal-review pass on default template copy. These are not soft constraints.
- **Critical path for "estimates work end to end":** A → B → E → F → G. Phase C (revisions) and Phase H (convert) are value-add and can land slightly later without breaking the core estimate→send→response loop.

### Research Flags

**Phases likely needing deeper research during planning:**
- **Phase E (Email + Send flow):** Default estimate template copy needs a targeted research pass on UK-consumer-law-appropriate non-binding language. Candidate for `/gsd:research-phase`. Also verify the ICO's January 2026 storage-and-access-technologies guidance before Phase E/F to confirm cookie/tracking assumptions haven't shifted.
- **Phase F (Public customer page):** Legal-copy review for the customer-facing page (buttons, disclaimers, state-transition messaging). Can share the research pass with Phase E.

**Phases with standard patterns (skip research-phase):**
- **Phase A:** Existing source code directly demonstrates the refactor shape; no external research needed.
- **Phase B:** Mirrors the v1.2 quote module pattern verbatim.
- **Phase C:** MongoDB partial unique indexes are standard; only the transaction-semantics open question needs product sign-off.
- **Phase D:** Radix Slider + RTK Query patterns documented in STACK.md with worked examples.
- **Phase G:** BullMQ delayed-job pattern verified against official docs; just needs disciplined execution.
- **Phase H:** Straightforward one-way DI against the existing `QuoteCreator`.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Zero new dependencies; all versions confirmed against `node_modules` and v1.4/v1.7 phase research. Only medium-confidence note is the `dinero.js` v2 alpha pin (unchanged since v1.2). |
| Features | MEDIUM-HIGH | Competitor analysis and UK legal framing HIGH (verified against Tradify docs, Citizens Advice, CRA 2015, Ombudsman guidance). Contingency percentages (10–25%) and follow-up cadence (3/10/21d) MEDIUM — drawn from US construction and B2B SaaS sales research adapted to UK trade context. Exact days and percentages are defensible defaults; adjust post-launch based on usage. |
| Architecture | HIGH | Grounded in direct reads of `trade-flow-api/src/quote/*`, `src/quote-token/*`, `src/queue/*`, `src/worker/*`. Line-item `parentType` widening is MEDIUM (refactor shape is clear but production row count isn't verified). |
| Pitfalls | HIGH | UK legal pitfalls cite CRA 2015 statute, ICO guidance, and solicitor commentary. BullMQ pitfalls verified against official docs. Dinero.js rounding pitfalls verified against both v1 and v2 API docs. Conversion/revision pitfalls derived from existing Trade Flow conventions + MongoDB community patterns (MEDIUM). |

**Overall confidence: HIGH.** The research leans heavily on verified source code reads and official documentation for the load-bearing decisions (module structure, BullMQ, Mongoose indexes, Radix Slider). The MEDIUM-confidence areas (contingency defaults, cadence days) are parameters, not architecture — they can be tuned after launch without refactoring.

### Gaps to Address

These open questions should be resolved before the roadmap is finalised, or explicitly deferred into the roadmap's phase plans with owners.

1. **Follow-up delivery time semantics.** Is "3 days from send" interpreted as "exactly 72 hours later" (simple, DST-safe, what the research assumes) or "9am the morning 3 days later in the tradesperson's local time" (better engagement, more complex)? Product decision for Phase G. Research recommends relative delay for v1.8.
2. **Estimate expiry window default.** Status lifecycle includes `Expired` but the milestone spec doesn't define the default window or whether expiry is auto-transitioned via a background sweep in v1.8. Recommend: fixed default (e.g. 30 days from Sent), auto-expiry background job deferred to Phase I or later.
3. **"Revise Estimate" semantics for "From £X" displays.** When revising a "From £X" estimate, does the trader re-enter the base price and contingency fresh, or is the new base computed from the old low bound? Product decision for Phase C.
4. **Convert-to-Quote pricing basis.** Milestone spec says conversion "drops contingency." Concretely: does the new quote's line items use the estimate's base price (low end), the mid-point, or something else? And does the trader get a chance to edit before the quote is saved? Recommend: copy base price with a mandatory review step in the UI. Phase H decision.
5. **`quote-settings` → `document-settings` rename scope.** Is the rename refactor in v1.8 (cleaner long-term shape) or deferred to a later cleanup milestone (tolerate a parallel `estimate-settings` module for now)? Architecture research recommends doing the rename in Phase A.
6. **Revision `isCurrent` flip transaction semantics.** The existing codebase does not use MongoDB transactions. Is it acceptable to rely on the partial unique index as the correctness guard (with a downgrade-then-insert-else-rollback path), or should revisions be the first feature to introduce transactions? Phase C decision.
7. **Worker module import strategy.** Does `WorkerModule` import the full `EstimateModule` (heavy but simple) or a minimal `EstimateWorkerModule` exporting only `EstimateRetriever` + `EstimateEmailSender` (cleaner but extra file)? Phase G decision.
8. **GDPR right-to-be-forgotten policy for estimates.** Not a v1.8 code task but v1.8 materially expands the PII audit trail. Flag for data-protection policy review — recommend soft-redact (replace PII with `[redacted]`, retain amounts/dates/status history) as the documented stance.
9. **Line items `parentType` backfill risk.** Safe to backfill `parentType: "quote"` during deploy, or stage through a feature flag? Depends on production `quote_line_items` row count. Phase A decision.
10. **Estimate PDF export.** Out of scope for v1.8 per milestone spec, consistent with v1.3 quote PDF deferral. Flag for the future quote-PDF milestone to solve once for both document types.

## Sources

### Primary (HIGH confidence)
- **Trade Flow source code** (read directly by architecture researcher): `trade-flow-api/src/quote/*`, `src/quote-token/*`, `src/quote-settings/*`, `src/queue/*`, `src/worker/*`, `trade-flow-api/CLAUDE.md`
- **Trade Flow planning artifacts:** `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/CONVENTIONS.md`, `.planning/codebase/STACK.md` (v1.7 snapshot), `.planning/PROJECT.md` (v1.8 milestone spec and key decisions), v1.2/v1.3/v1.4/v1.6/v1.7 phase research
- **BullMQ official docs:** [Delayed jobs](https://docs.bullmq.io/guide/jobs/delayed), [Job Ids](https://docs.bullmq.io/guide/jobs/job-ids), [Deduplication](https://docs.bullmq.io/guide/jobs/deduplication), [Removing Jobs](https://docs.bullmq.io/guide/queues/removing-jobs), [Auto-removal](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs), [Going to Production](https://docs.bullmq.io/guide/going-to-production), [QueueScheduler deprecation](https://docs.bullmq.io/guide/queuescheduler)
- **UK statute and regulator:** [Consumer Rights Act 2015, Part 1, Chapter 4 (Services)](https://www.legislation.gov.uk/ukpga/2015/15/part/1/chapter/4), [ICO direct marketing guidance](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-direct-marketing-using-electronic-mail/), [ICO storage and access technologies guidance](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/)
- **Radix / shadcn official:** [Radix Primitives Slider](https://www.radix-ui.com/primitives/docs/components/slider), [shadcn/ui February 2026 unified Radix package changelog](https://ui.shadcn.com/docs/changelog/2026-02-radix-ui)
- **Dinero.js official:** [v2 is out (Sarah Dayan)](https://www.sarahdayan.com/blog/dinerojs-v2-is-out), [v2 Amount concept](https://v2.dinerojs.com/docs/core-concepts/amount), [v1.9.1 on npm](https://www.npmjs.com/package/dinero.js/v/1.9.1)
- **MongoDB official:** [Document Versioning design pattern](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/data-versioning/document-versioning/), [Mongoose Populate (self-refs)](https://mongoosejs.com/docs/populate.html)

### Secondary (MEDIUM confidence)
- **Competitor behaviour:** [Tradify — Sending an Estimate instead of a Quote](https://help.tradifyhq.com/hc/en-us/articles/22034309269785-Sending-an-Estimate-instead-of-a-Quote), [Tradify — Create an Estimate](https://help.tradifyhq.com/hc/en-us/articles/15070931267609-Create-an-Estimate), [Jobber — Converting a Request to a Quote or Job](https://help.getjobber.com/hc/en-us/articles/360056871013-Converting-a-Request-to-a-Quote-or-Job), [ServiceM8 — Quoting for Work](https://www.servicem8.com/us/articles/quoting-for-work-write-winning-job-estimates), [ServiceM8 vs Tradify vs Jobber comparison — tpsTech](https://tpstech.au/blog/service-scheduling-software-showdown-servicem8-vs-tradify-vs-jobber/)
- **UK legal practitioner commentary:** [Hegarty Solicitors — Contractor Exceeds Original Quote](https://hegarty.co.uk/news/contractor-exceeds-original-quote-what-are-your-rights), [BTO Solicitors — Estimates: Beware!](https://www.bto.co.uk/blog/estimates-%E2%80%93-beware!.aspx), [Ralli Solicitors — Estimates and Quotations](https://ralli.co.uk/estimates-and-quotations/), [Go Legal AI — Quote vs Estimate UK Legal Guide](https://go-legal.ai/difference-between-a-quote-and-estimate-uk-legal-guide/), [Dispute Resolution Ombudsman](https://www.disputeresolutionombudsman.org/blogs/q-what-is-the-difference-between-and-estimate-and-a-quotation-and-why-is-it-important)
- **Contingency percentages (US-adapted):** [Simpro Plumbing Estimate Template](https://www.simprogroup.com/blog/plumbing-estimate-template), [Housecall Pro 2026 Plumbing Price Guide](https://www.housecallpro.com/resources/marketing/how-to/how-to-price-plumbing-jobs/), [FreshBooks Plumbing Estimate](https://www.freshbooks.com/hub/estimates/plumbing-estimate), [Ultimate Calculators Plumbing Cost Calculator](https://ultimatecalculators.com/calculator/plumbing-cost-calculator/)
- **Follow-up cadence:** [Belkins Sales Follow-Up Statistics 2025](https://belkins.io/blog/sales-follow-up-statistics), [Yesware Sales Follow-Up Statistics](https://www.yesware.com/blog/sales-follow-up-statistics/), [Apollo Sales Follow-Up Email Guide](https://www.apollo.io/insights/sales-follow-up-email), [Outreach Email Cadence Best Practices](https://www.outreach.ai/resources/blog/email-cadence)
- **CRM lost-deal taxonomy:** [DealHub — What is Closed Lost](https://dealhub.io/glossary/closed-lost/), [Zendesk deal loss reasons](https://support.zendesk.com/hc/en-us/articles/4408828162330-Creating-and-using-deal-loss-reasons), [HubSpot Closed-Lost pipeline community thread](https://community.hubspot.com/t5/Tips-Tricks-Best-Practices/Closed-lost-stage-in-sales-pipeline/m-p/949918)
- **UK PECR / GDPR on tracking pixels:** [DMA Email Council on Tracking Pixels](https://dma.org.uk/article/dma-email-council-understanding-email-tracking-pixels), [Transactional vs Marketing Emails under PECR — WDPS](https://wdps.co.uk/transactional-vs-marketing-emails-pecr/)

### Tertiary (LOW confidence — informational)
- Online booking products (competitive anti-feature research): BUILT Booking, Tradease, Nabooki, SimplyBook, Acuity — not implemented, referenced to justify the message-based site-visit approach
- Forum commentary on UK estimate disputes (Just Answer): used only to reinforce solicitor sources, not as primary legal reference

---

*Research synthesized: 2026-04-10*
*Ready for roadmap: yes*
