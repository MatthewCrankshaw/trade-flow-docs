# Phase 41: Estimate Module CRUD (Backend) - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in `41-CONTEXT.md` — this log preserves the alternatives considered.

**Date:** 2026-04-11
**Phase:** 41 — Estimate Module CRUD (Backend)
**Areas discussed:** Token rename rollout, Contingency totals & response shape, EstimateLineItem mirror fidelity, RESP-08 response summary shape, Uncertainty field reservation, Status transition scope, Draft-only enforcement, Entity & DTO shape review (terminology + display-mode rationale)

---

## Token Rename Rollout

### Q1: How should the quote_tokens → document_tokens migration be executed?

| Option | Description | Selected |
|--------|-------------|----------|
| Single migration file, backfill then rename collection | New IMigration with `updateMany` + `renameCollection`, runnable via POST /v1/migrations/run | Initially selected, then revised |
| Two migration files — add fields first, rename collection second | Split into transitional fields + rename | |
| Dual-read fallback period | New repo reads document_tokens then falls back to quote_tokens | |

**User's choice:** Pure code rename, no migration. Application has no production data to preserve, so backwards compatibility is not a concern.
**Notes:** User clarified mid-discussion that the app is not yet in use and data preservation is unnecessary. The entire migration conversation was softened to "code rename only" as a result. User added the constraint that IF a migration were ever needed, it must run via `POST /v1/migrations/run` on the existing migration endpoints — never on app startup.

### Q2: Post-rename, what happens to the old QuoteSessionAuthGuard name?

| Option | Description | Selected |
|--------|-------------|----------|
| Rename guard and request field in place | QuoteSessionAuthGuard → DocumentSessionAuthGuard; req.quoteToken → req.documentToken; PublicQuoteController asserts documentType === "quote" | ✓ |
| Keep deprecated alias re-export | QuoteSessionAuthGuard aliased to DocumentSessionAuthGuard | |

**User's choice:** Rename in place, no back-compat alias.

### Q3: How do we guarantee the existing `/v1/public/quote/:token` flow still resolves legacy tokens?

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated integration test asserting legacy-shape token resolution | Seed pre-migration fixture, run migration, hit endpoint, assert | |
| Unit tests on the guard + repository only | Cover DocumentTokenRepository.findByTokenOrFail and DocumentSessionAuthGuard.canActivate | ✓ |
| Smoke test run manually once before merge | Local seed + manual hit | |

**User's choice:** Unit tests only. Extended with "drop the legacy-shape test entirely" once the no-data-migration decision was made.

### Q4: Rename scope given no data migration is needed

| Option | Description | Selected |
|--------|-------------|----------|
| Pure code rename, no migration file | Delete src/quote-token/, create src/document-token/ from scratch | ✓ |
| Code rename + empty migration file for audit trail | Purely for migration history | |
| Code rename + collection drop migration | Drops quote_tokens if it exists (dev-only safety net) | |

**User's choice:** Pure code rename, no migration file.

### Q5: Should we drop the "existing quote tokens continue to validate" test requirement?

| Option | Description | Selected |
|--------|-------------|----------|
| Drop it entirely | Only unit tests on DocumentTokenRepository and DocumentSessionAuthGuard covering the new shape | ✓ |
| Keep a single documentType === "quote" dispatch test | Forward-facing type-dispatch test as a Pitfall 7 regression guard | |

**User's choice:** Drop entirely. Roadmap success criterion #5 is rewritten in CONTEXT.md to reflect the "code rename only, no data migration" posture.

---

## Contingency Totals & Response Shape

This area was explained in a single paragraph after the user asked for the three questions to be combined, and then answered in detail.

### Q1: Range formula

| Option | Description | Selected |
|--------|-------------|----------|
| low = base, high = base × (1 + c%) | Base is the floor; contingency is pure upside | ✓ |
| low = base × (1 - c%), high = base × (1 + c%) | Symmetric band around base | |
| low = base, high = base × (1 + c% × 2) | Half-band interpretation | |

**User's reasoning (captured inline):** The tradesperson's line-item subtotal is their honest assessment of what the job costs if everything goes to plan. Contingency represents additional cost from unknowns — it's upside risk only. A symmetric range would imply the tradesperson thinks the job might cost less than what they've priced, which contradicts how they built the estimate. Customer expectations match: "roughly £1,000 to £1,100 depending on what we find" communicates £1,000 as the starting point, not the midpoint.

### Q2: Tax handling

| Option | Description | Selected |
|--------|-------------|----------|
| Pre-tax: range applied to subtotal, then tax recomputed on each bound | Scales per-line-item tax logic proportionally | ✓ |
| Post-tax: compute total then multiply | Applies contingency to the already-taxed total | |
| Pre-tax only, range displayed net of tax | B2B pricing convention | |

**User's reasoning (captured inline):** Tax is a levy on the actual supply value. If work ends up costing £1,100 rather than £1,000, the tax is the applicable percentage of £1,100, not of £1,000 plus a contingency lump. Multiplying the final tax-inclusive total happens to give the same answer at a single rate (20%) by arithmetic coincidence, but breaks the moment there are mixed rates (e.g. 20% on labour, 0% on energy-saving materials). Computing tax per line item first and then applying contingency to the pre-tax sum keeps the math clean under mixed rates.

### Q3: "From" display mode DTO shape

| Option | Description | Selected |
|--------|-------------|----------|
| high is always computed; displayMode just flips presentation | API always returns both low and high; UI decides labelling | ✓ |
| high is null when displayMode is "from" | Expresses "upper bound unknown" at the type level | |
| Response omits high entirely for "from" mode | Different DTO shapes per mode | |

**User's reasoning (captured inline):** Always computing both values means the UI for "from" mode still has high internally (useful if the tradesperson flips to range mode in a revision), the customer page can include "final cost unlikely to exceed £X" as a tooltip, and convert-to-quote can reference the ceiling. The contract adds a displayMode field and the UI owns rendering.

### Q4: Terminology (raised later during entity shape review)

**User's directive:** Use country-agnostic terminology throughout. Replace `vat` / `VAT` / `exclVat` / `inclVat` / `vatRate` / `vatRegistered` with `tax` / `exclTax` / `inclTax` / `taxRate`. This aligns with existing Trade Flow convention — the quote module already uses `taxRate` and `taxTotal`.

**Resulting simplification:** Dropped `vatRegistered` and the effective `vatRate` field from IEstimatePriceRangeDto entirely. Businesses that don't collect tax just have `taxRate: 0` on all line items, which naturally produces `tax: Money.zero()` and `exclTax === inclTax`. This eliminated the open question about whether Business needs new fields — the lookup was moot.

### Q5: Why two display modes? (raised during entity shape review)

**User's question:** What is the reason for having two display modes?

**Explanation given:** Two structurally different uncertainty types. Range mode applies when the scope is known but the cost has a confidence interval (kitchen tap, boiler swap); contingency covers surprises within a known scope. "From" mode applies when the scope itself is unknown (diagnostic work like "shower has low pressure"); the tradesperson can commit to a minimum but cannot bound the ceiling until they investigate. Collapsing to one mode breaks either well-defined jobs (lose the ceiling) or diagnostic jobs (force dishonest upper bounds).

**User's choice:** Keep both modes as defined.

---

## EstimateLineItem Mirror Fidelity

### Q1: How literal should the mirror be for the three bundle helpers that look quote-agnostic?

| Option | Description | Selected |
|--------|-------------|----------|
| Lift helpers to @item, share all three | Move BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator to @item/services/. Generalise BundleTaxRateCalculator to accept a structural ILineItemTaxInput interface | ✓ |
| Full literal duplication | Copy all three into @estimate/services/ verbatim | |
| Estimate imports from @quote/services/ | Cross-module dependency | |

**User's choice:** Lift to @item, share all three. Investigation confirmed that BundleConfigValidator and BundlePricingPlanner already use only @item and @core types; BundleTaxRateCalculator has a trivial IQuoteLineItemDto coupling that uses only `{ unitPrice, quantity, taxRate }` and can be generalised to a structural interface. D-01 separation is about entity boundaries, not pure item-level math.

### Q2: For entity-scoped classes, how strictly file-for-file?

| Option | Description | Selected |
|--------|-------------|----------|
| 1:1 file mirror, estimate-prefixed names | Mechanical rename of every file; methods, signatures, tests identical | ✓ |
| Reshape to estimate-specific needs where it makes sense | Start from quote shape but restructure where estimate differs | |

**User's choice:** 1:1 file mirror. Side-by-side code review must remain possible.

---

## RESP-08 Response Summary Shape

### Q1: How much of the response summary DTO gets defined in Phase 41?

| Option | Description | Selected |
|--------|-------------|----------|
| Full DTO shape, always null until Phase 45 populates | IEstimateResponseSummaryDto defined with { type, reason?, message?, respondedAt }; IEstimateDto carries responseSummary: DTO | null | ✓ |
| Minimal field reservation only | Untyped placeholder; Phase 45 defines the shape | |
| Omit entirely from Phase 41 | Phase 45 introduces the field as a breaking addition | |

**User's choice:** Full DTO shape defined now, always null on the response until Phase 45 populates it.

### Q2: Where does customer response data live on the estimate?

| Option | Description | Selected |
|--------|-------------|----------|
| Embedded on the estimate entity | Fields like lastResponseType, lastResponseAt directly on IEstimateEntity | ✓ |
| Separate estimate_responses collection, summary denormalised | History feature out of scope | |
| Separate collection, no denormalisation | Worst read performance | |

**User's choice:** Embedded on the estimate entity, matching RESP-08's "persisted on the estimate" wording.

---

## Uncertainty Fields (CONT-04)

### Q1: Where do uncertaintyReasons/uncertaintyNotes land?

| Option | Description | Selected |
|--------|-------------|----------|
| Reserve nullable fields on the entity now in Phase 41 | Entity carries optional fields; CRUD accepts them; Phase 43 UI wires into existing fields | ✓ |
| Omit from Phase 41; Phase 43 adds them | Cleaner scope boundary but forces Phase 43 to touch backend | |

**User's choice:** Reserve nullable fields now. Scope blur is cheaper than a mid-milestone schema change.

---

## Status Transition Scope

### Q1: Does Phase 41 expose any HTTP endpoints that trigger transitions?

| Option | Description | Selected |
|--------|-------------|----------|
| Only CRUD plus internal EstimateTransitionService | CRUD endpoints + transition service used internally by later phases | ✓ |
| CRUD plus generic POST /estimates/:id/transition | Exposes raw state machine to client | |
| CRUD only, no transition service | Violates phase success criterion #4 | |

**User's choice:** Only CRUD plus internal service. Transition service mirrors the existing QuoteTransitionService pattern. Phase 44/45/46/47 call it internally; Phase 41 exposes no route that triggers a non-CRUD transition.

---

## Draft-Only Enforcement

### Q1: Which layer owns the Draft-only gate for PATCH and DELETE?

| Option | Description | Selected |
|--------|-------------|----------|
| Service layer only | EstimateUpdater / EstimateDeleter throws InvalidRequestError if status !== Draft | ✓ |
| Policy layer only | EstimatePolicy.canUpdate returns false for non-Draft | |
| Both layers — defence in depth | Two places to change if the rule evolves | |

**User's choice:** Service layer only. Policy handles ownership (who); service handles business rules (whether).

---

## Entity & DTO Shape Review

After the seven areas above were captured, the user requested the concrete entity and DTO definitions before locking CONTEXT.md. The shapes in CONTEXT.md §specifics were presented for review.

**User's feedback:**
1. Replace all `vat*` / `VAT` terminology with country-agnostic `tax*` naming. This cascaded: `IEstimatePriceRangeBoundDto` fields renamed to `exclTax` / `tax` / `inclTax`; `vatRegistered` and `vatRate` dropped entirely from `IEstimatePriceRangeDto`.
2. Asked for the rationale behind two display modes. Explanation accepted; both modes retained.

No field, enum value, transition map entry, or collection name was challenged beyond the terminology point. The final shapes captured in CONTEXT.md §specifics are locked.

---

## Claude's Discretion

The following were explicitly flagged as "Claude's discretion" during discussion — the planner may override with rationale:

- class-validator decorator stack for `contingencyPercent` (0, 5, 10, 15, 20, 25, 30)
- Default values on create: `displayMode: "range"`, `contingencyPercent: 10`, `estimateDate: now`, revision fields defaulted to root-revision values
- Request and response DTO class structures
- Endpoint list and pagination pattern
- Line-item write semantics on PATCH (mirrors QuoteUpdater)
- Error code naming in `ErrorCodes` enum
- Test file names, mock generator shape
- Choice of whether `PublicQuoteController` moves into the quote module or stays in a renamed public-quote module (recommendation: move to `src/quote/controllers/public-quote.controller.ts`)

---

## Deferred Ideas

- Dual-read fallback for token rename
- Empty audit-trail migration file
- Symmetric range formula
- Single-mode simplification
- Legacy-shape document-token integration test
- Uncertainty reasons enum in Phase 41 (deferred to Phase 43)
- Cross-module reuse of @quote/services/ from estimate module
- Public transition endpoint
- Separate estimate_responses history collection
- `vatRegistered` + effective `vatRate` fields

All captured in CONTEXT.md §deferred for traceability.
