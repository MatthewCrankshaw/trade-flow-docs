---
phase: 41-estimate-module-crud-backend
plan: 02
type: execute
wave: 2
depends_on: [01]
files_modified:
  - trade-flow-api/src/item/services/bundle-config-validator.service.ts
  - trade-flow-api/src/item/services/bundle-pricing-planner.service.ts
  - trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts
  - trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts
  - trade-flow-api/src/item/item.module.ts
  - trade-flow-api/src/item/test/services/bundle-config-validator.service.spec.ts
  - trade-flow-api/src/item/test/services/bundle-pricing-planner.service.spec.ts
  - trade-flow-api/src/item/test/services/bundle-tax-rate-calculator.service.spec.ts
  - trade-flow-api/src/quote/quote.module.ts
  - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
autonomous: true
requirements: [EST-03]
must_haves:
  truths:
    - BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator live under `@item/services/` and are exported from `ItemModule`
    - BundleTaxRateCalculator accepts the structural `ILineItemTaxInput` interface (not `IQuoteLineItemDto[]`) so the estimate module can reuse it without importing quote types
    - QuoteBundleLineItemFactory imports the three helpers from `@item/services/` and still passes its existing spec
    - Quote module providers no longer list the three helpers
    - `cd trade-flow-api && npm run ci` exits 0 (full quote test suite stays green — Pitfall 7)
  artifacts:
    - path: trade-flow-api/src/item/services/bundle-config-validator.service.ts
      provides: Shared bundle config validation
      min_lines: 20
    - path: trade-flow-api/src/item/services/bundle-pricing-planner.service.ts
      provides: Shared bundle pricing plan
      min_lines: 20
    - path: trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts
      provides: Shared bundle tax calculator generalised to ILineItemTaxInput
      min_lines: 20
    - path: trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts
      provides: Structural interface { unitPrice: Money; quantity: number; taxRate: number }
      exports: ["ILineItemTaxInput"]
  key_links:
    - from: trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
      to: trade-flow-api/src/item/services/bundle-config-validator.service.ts
      via: import statement
      pattern: "from \"@item/services/bundle-config-validator.service\""
    - from: trade-flow-api/src/item/item.module.ts
      to: "BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator"
      via: providers and exports arrays
      pattern: "Bundle(Config|Pricing|Tax).*Service"
---

<objective>
Lift the three bundle helper services — `BundleConfigValidator`, `BundlePricingPlanner`, and `BundleTaxRateCalculator` — from `@quote/services/` into `@item/services/` so both the quote module and the new estimate module (Plan 06) can consume them without cross-module coupling. Generalise `BundleTaxRateCalculator` to accept a new structural `ILineItemTaxInput` interface from `@item/interfaces/` instead of `IQuoteLineItemDto[]`. Update `QuoteBundleLineItemFactory` imports, update `QuoteModule` and `ItemModule` provider arrays, and verify the full quote test suite still passes (Pitfall 7).

Purpose: Per D-LI-01 in CONTEXT.md and the separation-over-DRY project memory, any helper shared between quote and estimate modules must live outside both — in `@item/services/`. The estimate module cannot import from `@quote/services/` (rejected in <deferred>). This plan is a pure refactor: no behavior changes, no API changes, just moved files and generalised tax-calculator input.

Output: Three relocated service files, one new interface file, one updated module file, one updated factory file, and a green CI gate proving zero regression.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Source-of-truth files the executor MUST read before editing. -->

From trade-flow-api/src/quote/services/bundle-config-validator.service.ts:
- class BundleConfigValidator (constructor injection, @Injectable())
- public methods consumed by QuoteBundleLineItemFactory

From trade-flow-api/src/quote/services/bundle-pricing-planner.service.ts:
- class BundlePricingPlanner (@Injectable())
- imports only from @item/* and @core/*

From trade-flow-api/src/quote/services/bundle-tax-rate-calculator.service.ts:
- class BundleTaxRateCalculator (@Injectable())
- currently typed against IQuoteLineItemDto[]; only reads { unitPrice, quantity, taxRate }
- MUST be generalised to accept the new ILineItemTaxInput interface

New interface to create:
```typescript
// trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts
import { Money } from "@core/value-objects/money.value-object";

export interface ILineItemTaxInput {
  unitPrice: Money;
  quantity: number;
  taxRate: number;
}
```

QuoteBundleLineItemFactory callsite: pass quote line items directly (they already satisfy ILineItemTaxInput structurally — TypeScript structural typing handles this with no cast, no mapper).
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create ILineItemTaxInput interface and move the three bundle helpers to @item/services/</name>
  <files>
    trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts,
    trade-flow-api/src/item/services/bundle-config-validator.service.ts,
    trade-flow-api/src/item/services/bundle-pricing-planner.service.ts,
    trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts,
    trade-flow-api/src/item/test/services/bundle-config-validator.service.spec.ts,
    trade-flow-api/src/item/test/services/bundle-pricing-planner.service.spec.ts,
    trade-flow-api/src/item/test/services/bundle-tax-rate-calculator.service.spec.ts
  </files>
  <read_first>
    - trade-flow-api/src/quote/services/bundle-config-validator.service.ts (source file to move — EXECUTOR MUST NOT guess contents)
    - trade-flow-api/src/quote/services/bundle-pricing-planner.service.ts (source file to move)
    - trade-flow-api/src/quote/services/bundle-tax-rate-calculator.service.ts (source file to move AND generalise)
    - trade-flow-api/src/quote/test/services/bundle-config-validator.service.spec.ts (existing spec to relocate — if present)
    - trade-flow-api/src/quote/test/services/bundle-pricing-planner.service.spec.ts (existing spec to relocate — if present)
    - trade-flow-api/src/quote/test/services/bundle-tax-rate-calculator.service.spec.ts (existing spec to relocate — if present)
    - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts (callsite of BundleTaxRateCalculator — executor needs to see how quote line items are passed to confirm structural compatibility)
    - trade-flow-api/src/core/value-objects/money.value-object.ts (to confirm the Money type signature used in ILineItemTaxInput)
  </read_first>
  <behavior>
    - After the move, `BundleConfigValidator`, `BundlePricingPlanner`, `BundleTaxRateCalculator` exist at `src/item/services/` with identical class names, public method signatures, and behavior as the originals (modulo the tax-calculator input-type generalisation).
    - `BundleTaxRateCalculator` accepts `ILineItemTaxInput[]` (or whatever parameter name the original method uses, but typed against the new interface) and still produces identical results for any input that previously satisfied `IQuoteLineItemDto`.
    - The relocated spec files still pass when run from their new location: `cd trade-flow-api && npm run test -- --testPathPattern=item/test/services/bundle` exits 0.
    - Zero narrative comments added; zero `eslint-disable`; zero `as` casts introduced (per CLAUDE.md and user memory).
    - No changes to the public behavior of any helper.
  </behavior>
  <action>
**Step 1: Create `trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts`** with exactly:

```typescript
import { Money } from "@core/value-objects/money.value-object";

export interface ILineItemTaxInput {
  unitPrice: Money;
  quantity: number;
  taxRate: number;
}
```

No comments. No JSDoc unless the project convention for interfaces includes it — check `trade-flow-api/src/core/interfaces/` for an existing interface and match its JSDoc style if present.

**Step 2: Move the three service files** using `git mv` to preserve history:

```
cd trade-flow-api
git mv src/quote/services/bundle-config-validator.service.ts src/item/services/bundle-config-validator.service.ts
git mv src/quote/services/bundle-pricing-planner.service.ts src/item/services/bundle-pricing-planner.service.ts
git mv src/quote/services/bundle-tax-rate-calculator.service.ts src/item/services/bundle-tax-rate-calculator.service.ts
```

**Step 3: Move the three spec files** (if they exist; skip individually if not present):

```
git mv src/quote/test/services/bundle-config-validator.service.spec.ts src/item/test/services/bundle-config-validator.service.spec.ts
git mv src/quote/test/services/bundle-pricing-planner.service.spec.ts src/item/test/services/bundle-pricing-planner.service.spec.ts
git mv src/quote/test/services/bundle-tax-rate-calculator.service.spec.ts src/item/test/services/bundle-tax-rate-calculator.service.spec.ts
```

If a spec file does not exist in the quote module for one of the helpers, CREATE a minimal new spec under `src/item/test/services/` covering at least one happy-path assertion per public method. The existing helpers currently inject via constructor — mirror the existing quote-module spec structure for TestingModule bootstrapping.

**Step 4: Update imports inside the moved files** to use relative-to-new-location path aliases:
- Any `import { ... } from "@quote/..."` inside the three moved files MUST be removed or replaced. Per D-LI-01 these helpers were already quote-agnostic, so there should be zero `@quote/*` imports to remove. If the executor finds any `@quote/*` import, STOP and report it — this indicates the helper is not as quote-agnostic as research claimed.
- `BundleTaxRateCalculator` currently imports `IQuoteLineItemDto` from `@quote/data-transfer-objects/quote-line-item.dto`. Replace that import with:
  ```typescript
  import { ILineItemTaxInput } from "@item/interfaces/line-item-tax-input.interface";
  ```
  and change every `IQuoteLineItemDto[]` parameter type annotation to `ILineItemTaxInput[]`. Field accesses (`lineItem.unitPrice`, `lineItem.quantity`, `lineItem.taxRate`) remain unchanged because `ILineItemTaxInput` exposes exactly those three fields.
- If the method parameter name is `lineItems: IQuoteLineItemDto[]`, keep the parameter name and only change the type.

**Step 5: Update imports inside the three moved spec files** to use the new source-file location (`@item/services/...` instead of `@quote/services/...`) and, for the tax-rate spec, update any fixture that constructs test data to satisfy `ILineItemTaxInput` (trivial — existing fixtures already set `unitPrice`, `quantity`, `taxRate`).

**Step 6: Run the slice** `cd trade-flow-api && npm run test -- --testPathPattern=item/test/services/bundle` and confirm all three specs pass.

Commit message: `refactor(41): lift bundle helpers from @quote/services to @item/services with ILineItemTaxInput generalisation`.
  </action>
  <acceptance_criteria>
    - `trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts` exists
    - `grep -c "export interface ILineItemTaxInput" trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts` returns 1
    - `trade-flow-api/src/item/services/bundle-config-validator.service.ts` exists
    - `trade-flow-api/src/item/services/bundle-pricing-planner.service.ts` exists
    - `trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts` exists
    - `trade-flow-api/src/quote/services/bundle-config-validator.service.ts` does NOT exist (`test ! -f trade-flow-api/src/quote/services/bundle-config-validator.service.ts`)
    - `trade-flow-api/src/quote/services/bundle-pricing-planner.service.ts` does NOT exist
    - `trade-flow-api/src/quote/services/bundle-tax-rate-calculator.service.ts` does NOT exist
    - `grep -c "IQuoteLineItemDto" trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts` returns 0
    - `grep -c "ILineItemTaxInput" trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts` returns at least 2 (import + type annotation)
    - `grep -rc "@quote/" trade-flow-api/src/item/services/bundle-*.ts` returns 0 for each file (no cross-module imports)
    - `cd trade-flow-api && npm run test -- --testPathPattern=item/test/services/bundle` exits 0
    - `grep -rc "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|as unknown as" trade-flow-api/src/item/services/bundle-*.ts trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=item/test/services/bundle</automated>
  </verify>
  <done>Three helpers relocated, ILineItemTaxInput created, BundleTaxRateCalculator generalised, item module spec slice passes</done>
</task>

<task type="auto">
  <name>Task 2: Wire helpers into ItemModule and update QuoteBundleLineItemFactory + QuoteModule imports</name>
  <files>
    trade-flow-api/src/item/item.module.ts,
    trade-flow-api/src/quote/quote.module.ts,
    trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
  </files>
  <read_first>
    - trade-flow-api/src/item/item.module.ts (current providers/exports arrays — executor must see the existing shape)
    - trade-flow-api/src/quote/quote.module.ts (current providers array listing the three bundle helpers to be removed)
    - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts (existing import block referencing `@quote/services/bundle-*`)
    - trade-flow-api/src/item/services/bundle-config-validator.service.ts (from Task 1 — confirms the class name/path to import)
  </read_first>
  <action>
**Step 1: Update `trade-flow-api/src/item/item.module.ts`:**

- Add the three bundle helpers to the `providers` array:
  ```typescript
  import { BundleConfigValidator } from "@item/services/bundle-config-validator.service";
  import { BundlePricingPlanner } from "@item/services/bundle-pricing-planner.service";
  import { BundleTaxRateCalculator } from "@item/services/bundle-tax-rate-calculator.service";
  ```
  and append `BundleConfigValidator, BundlePricingPlanner, BundleTaxRateCalculator` to both `providers: [...]` and `exports: [...]`.

- If `ItemModule` currently does not export anything (only provides), add an `exports:` field containing at minimum these three services so `QuoteModule` and the future `EstimateModule` can consume them transitively via `imports: [ItemModule]`.

- Preserve all existing providers and existing exports exactly as-is.

**Step 2: Update `trade-flow-api/src/quote/quote.module.ts`:**

- Remove the three imports:
  ```typescript
  // DELETE these three lines
  import { BundleConfigValidator } from "@quote/services/bundle-config-validator.service";
  import { BundlePricingPlanner } from "@quote/services/bundle-pricing-planner.service";
  import { BundleTaxRateCalculator } from "@quote/services/bundle-tax-rate-calculator.service";
  ```
- Remove the three entries from the `providers: [...]` array and from any `exports: [...]` entry if they were exported.
- Do NOT touch any other providers/imports/exports.
- Confirm `QuoteModule` already has `imports: [..., ItemModule, ...]` (it does, per RESEARCH.md §Code Examples). If it does not, add `ItemModule` to the imports array.

**Step 3: Update `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts`:**

- Change import paths from `@quote/services/bundle-*` to `@item/services/bundle-*`. Exact line-level replacement:
  ```typescript
  // BEFORE
  import { BundleConfigValidator } from "@quote/services/bundle-config-validator.service";
  import { BundlePricingPlanner } from "@quote/services/bundle-pricing-planner.service";
  import { BundleTaxRateCalculator } from "@quote/services/bundle-tax-rate-calculator.service";

  // AFTER
  import { BundleConfigValidator } from "@item/services/bundle-config-validator.service";
  import { BundlePricingPlanner } from "@item/services/bundle-pricing-planner.service";
  import { BundleTaxRateCalculator } from "@item/services/bundle-tax-rate-calculator.service";
  ```
- If the factory passes `IQuoteLineItemDto[]` arrays to `BundleTaxRateCalculator`, NO change is needed at the callsite — TypeScript structural typing accepts `IQuoteLineItemDto[]` as `ILineItemTaxInput[]` because `IQuoteLineItemDto` already contains `{ unitPrice: Money, quantity: number, taxRate: number }`. Verify the existing callsite still compiles without a cast.

**Step 4: Run the full quote suite to confirm zero regression (Pitfall 7):**

```
cd trade-flow-api && npm run test -- --testPathPattern=quote
```

Every existing quote test MUST still pass. If any test fails, investigate whether:
- a spec file still imports from `@quote/services/bundle-*` (update the spec import)
- DI wiring is broken because `QuoteModule` no longer provides the helper but a quote spec's TestingModule doesn't import `ItemModule`
- `BundleTaxRateCalculator` spec fixtures don't satisfy the new `ILineItemTaxInput` type

Fix at the root cause. Do not add type assertions. Do not suppress lint/typecheck.

Commit message: `refactor(41): rewire QuoteBundleLineItemFactory and QuoteModule to use @item/services bundle helpers`.
  </action>
  <acceptance_criteria>
    - `grep -c "BundleConfigValidator" trade-flow-api/src/item/item.module.ts` returns at least 3 (import + providers + exports)
    - `grep -c "BundlePricingPlanner" trade-flow-api/src/item/item.module.ts` returns at least 3
    - `grep -c "BundleTaxRateCalculator" trade-flow-api/src/item/item.module.ts` returns at least 3
    - `grep -c "BundleConfigValidator\\|BundlePricingPlanner\\|BundleTaxRateCalculator" trade-flow-api/src/quote/quote.module.ts` returns 0
    - `grep -c "@quote/services/bundle-" trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` returns 0
    - `grep -c "@item/services/bundle-config-validator" trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` returns 1
    - `grep -c "@item/services/bundle-pricing-planner" trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` returns 1
    - `grep -c "@item/services/bundle-tax-rate-calculator" trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` returns 1
    - `grep -rc "@quote/services/bundle-config-validator\\|@quote/services/bundle-pricing-planner\\|@quote/services/bundle-tax-rate-calculator" trade-flow-api/src` returns 0 (no dangling references anywhere in the tree)
    - `cd trade-flow-api && npm run test -- --testPathPattern=quote` exits 0
    - `grep -rc "as " trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` returns 0 (no new type assertions introduced at the BundleTaxRateCalculator callsite)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run test -- --testPathPattern=quote</automated>
  </verify>
  <done>ItemModule provides and exports the three helpers, QuoteModule no longer references them, quote factory imports from @item/services, full quote test suite passes</done>
</task>

<task type="auto">
  <name>Task 3: Run full CI gate to confirm refactor is non-regressive</name>
  <files>none</files>
  <read_first>
    - trade-flow-api/package.json (confirm ci script composition)
  </read_first>
  <action>
Run `cd trade-flow-api && npm run ci`. Expected exit code: 0.

This runs: test (full suite), lint:check, format:check, typecheck.

Pitfall 7 (RESEARCH.md) warns that the refactor most often breaks:
1. A stale import in a spec file that wasn't moved
2. A DI wiring miss in a TestingModule that didn't import `ItemModule`
3. A type error in `BundleTaxRateCalculator` call if an existing caller passed something that structurally satisfied `IQuoteLineItemDto` but not `ILineItemTaxInput`

If the CI gate fails, diagnose at the root cause. Do not use `@ts-ignore` or `eslint-disable` under any circumstances. If a test fails because a spec's TestingModule missed `ItemModule`, add the import to the spec. If a type error appears at a callsite, confirm structural compatibility and update the type annotation — no casts.

Commit any fix-up changes with message `chore(41): fix bundle helper refactor fallout`.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - No `@ts-ignore`, `@ts-expect-error`, `eslint-disable`, or `as unknown as` markers in the diff for this plan
    - Quote test file count unchanged (no test was silently deleted to make CI pass)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green; quote module still works end-to-end after the helper lift</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| quote module → shared helpers | Quote factory now consumes helpers from `@item/services/` via DI |
| future estimate module → shared helpers | Same DI path; no cross-module coupling |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain (no new threat surface introduced; mechanical refactor only).

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-02-01 | Tampering | Behavioral drift in BundleTaxRateCalculator after type generalisation | mitigate | Existing spec suite runs unchanged against the new signature; structural typing guarantees caller compatibility; Task 3 full CI gate catches any regression |
| T-41-02-02 | Elevation of Privilege | Incorrect module wiring lets unauthenticated code path reach bundle math | accept | Bundle helpers are pure functions with no auth surface; they never appear in a controller; DI wiring changes do not expose new endpoints |
| T-41-02-03 | Tampering | Quote test suite silently disabled to mask regression | mitigate | Task 3 acceptance criterion explicitly verifies quote test file count is unchanged; code review rule |
</threat_model>

<verification>
1. Three bundle helpers live under `trade-flow-api/src/item/services/`, exported from `ItemModule`.
2. `ILineItemTaxInput` interface exists under `trade-flow-api/src/item/interfaces/`.
3. `QuoteBundleLineItemFactory` imports from `@item/services/...`.
4. `QuoteModule` no longer lists the three helpers as providers.
5. `cd trade-flow-api && npm run ci` exits 0.
6. `grep -rc "@quote/services/bundle-" trade-flow-api/src` returns 0.
</verification>

<success_criteria>
- Plan 06 (estimate factories) can import BundleConfigValidator / BundlePricingPlanner / BundleTaxRateCalculator from `@item/services/` without touching quote code
- No quote-module regression (Pitfall 7 avoided)
- No `as` casts or type suppression added anywhere
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-02-SUMMARY.md` documenting:
- Which files were moved (before/after paths)
- Whether `ILineItemTaxInput` required changes to existing quote callsites (expected: no, structural typing)
- The full `npm run ci` output
</output>
