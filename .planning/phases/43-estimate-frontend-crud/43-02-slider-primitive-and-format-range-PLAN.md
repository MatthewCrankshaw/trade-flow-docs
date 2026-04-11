---
phase: 43-estimate-frontend-crud
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/package.json
  - trade-flow-ui/package-lock.json
  - trade-flow-ui/src/components/ui/slider.tsx
  - trade-flow-ui/src/lib/currency.ts
  - trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts
  - trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json
autonomous: true
requirements:
  - CONT-03

must_haves:
  truths:
    - "formatRange(low, high, mode, currencyCode) exists in src/lib/currency.ts and returns the exact strings shown in the golden-file test"
    - "Vitest golden-file test src/lib/__tests__/currency.formatRange.test.ts passes, asserting API and UI agree to the penny"
    - "@radix-ui/react-slider is installed and src/components/ui/slider.tsx wraps it"
  artifacts:
    - path: "trade-flow-ui/src/components/ui/slider.tsx"
      provides: "shadcn New York style Slider wrapper around @radix-ui/react-slider"
      contains: "@radix-ui/react-slider"
    - path: "trade-flow-ui/src/lib/currency.ts"
      provides: "formatRange helper + existing formatAmount"
      contains: "export function formatRange"
    - path: "trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts"
      provides: "Golden-file test with explicit tuples + fixture-driven API agreement"
      contains: "describe(\"formatRange\""
    - path: "trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json"
      provides: "Fixture mirroring Phase 41 IEstimateDto shape with expectedFormattedRange field"
      contains: "priceRange"
    - path: "trade-flow-ui/package.json"
      provides: "@radix-ui/react-slider dependency"
      contains: "@radix-ui/react-slider"
  key_links:
    - from: "trade-flow-ui/src/lib/currency.ts"
      to: "formatRange return string"
      via: "delegates to formatAmount for each of low.total and high.total"
      pattern: "formatRange.*formatAmount"
    - from: "trade-flow-ui/src/components/ui/slider.tsx"
      to: "@radix-ui/react-slider"
      via: "SliderPrimitive wrapper"
      pattern: "import \\* as SliderPrimitive from \"@radix-ui/react-slider\""
---

<objective>
Install the Radix slider dependency, build the shadcn New York Slider wrapper, and ship the `formatRange` helper plus its golden-file test that pins UI formatting against a real API response.

Purpose: These are the independent foundation primitives that ContingencySlider (Plan 04) and every estimate list/detail surface (Plans 05/06) consume. The golden-file test is the contract that enforces Success Criterion #4 ("UI never multiplies base × contingency on the client; formatRange renders only what the API returns").

Output: New slider.tsx, new formatRange export, new golden-file test, new JSON fixture, new npm dependency.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@trade-flow-ui/src/components/ui/radio-group.tsx
@trade-flow-ui/src/lib/currency.ts
@trade-flow-ui/src/hooks/useCurrency.ts
@trade-flow-ui/package.json
@trade-flow-ui/CLAUDE.md

<interfaces>
<!-- Reference shape for shadcn Radix wrappers - new slider.tsx MUST mirror this pattern -->

From trade-flow-ui/src/components/ui/radio-group.tsx:
```typescript
import * as React from "react";
import { RadioGroup as RadioGroupPrimitive } from "radix-ui";
import { cn } from "@/lib/utils";

function RadioGroup({ className, ...props }: React.ComponentProps<typeof RadioGroupPrimitive.Root>) {
  return <RadioGroupPrimitive.Root data-slot="radio-group" className={cn("grid gap-3", className)} {...props} />;
}
// ... RadioGroupItem uses forwardRef pattern, className merging, disabled styles, focus-visible rings
export { RadioGroup, RadioGroupItem };
```

NOTE: the existing repo imports Radix components from the `radix-ui` meta package (e.g. `import { RadioGroup as RadioGroupPrimitive } from "radix-ui"`) rather than `@radix-ui/react-radio-group`. Inspect `radio-group.tsx` at read time and determine whether the slider should follow the same pattern (`import { Slider as SliderPrimitive } from "radix-ui"`) OR install and import from `@radix-ui/react-slider` directly. Decision rule: if `radix-ui` meta package is installed in `package.json`, use `import { Slider as SliderPrimitive } from "radix-ui"` for consistency with the existing wrappers. Otherwise install `@radix-ui/react-slider` directly. Either way, the resulting wrapper file must import a primitive and expose a named `Slider` component.

From trade-flow-ui/src/lib/currency.ts:
```typescript
// existing exports include:
export function formatAmount(amount: number | null | undefined, currencyCode: SupportedCurrencyCode, locale?: string): string;
// GBP returns formatted via en-GB locale, e.g. "£120.00"
```

Phase 41 D-CONT-05/06 locked DTO shape for priceRange: `{ low: { subTotal, taxTotal, total }, high: { subTotal, taxTotal, total } }` where the numbers are minor-units integers (pence for GBP).
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Install Radix slider + create src/components/ui/slider.tsx wrapper</name>
  <files>trade-flow-ui/package.json, trade-flow-ui/package-lock.json, trade-flow-ui/src/components/ui/slider.tsx</files>
  <read_first>
    - trade-flow-ui/package.json (check if `radix-ui` meta package is listed — this determines the import path)
    - trade-flow-ui/src/components/ui/radio-group.tsx (pattern for Radix wrapper with `cn()` + className merging + data-slot attribute)
    - trade-flow-ui/components.json (shadcn config — confirms "new-york" style)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-SLD-01 through D-SLD-05
  </read_first>
  <action>
Step 1: From `trade-flow-ui/` directory, install the Radix slider primitive:

```bash
cd trade-flow-ui && npm install @radix-ui/react-slider
```

(If `radix-ui` meta package is already in dependencies, this install is still required — the meta package re-exports all Radix primitives but shadcn New York style components directly import `@radix-ui/react-slider` in some templates. Install and then choose the import path that matches `radio-group.tsx`. If `radio-group.tsx` imports from `"radix-ui"`, use `import { Slider as SliderPrimitive } from "radix-ui"` in the new file. If it imports from `@radix-ui/react-radio-group`, use `@radix-ui/react-slider`.)

Step 2: Create NEW file `trade-flow-ui/src/components/ui/slider.tsx` with shadcn New York style. The component must:
- Export a named `Slider` component
- Wrap `SliderPrimitive.Root` + `SliderPrimitive.Track` + `SliderPrimitive.Range` + `SliderPrimitive.Thumb`
- Accept all standard Radix Slider props via `React.ComponentProps<typeof SliderPrimitive.Root>`
- Use `cn()` from `@/lib/utils` for className merging
- Include `data-slot="slider"` on the root for style consistency
- Apply Tailwind classes for: track (relative h-2 rounded-full bg-muted), range (absolute h-full rounded-full bg-primary), thumb (block size-5 rounded-full border-2 border-primary bg-background shadow-sm focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50)
- Render one thumb per value (use `props.value ?? props.defaultValue ?? []` and map over it)
- Do NOT implement any contingency logic — that lives in `ContingencySlider` in plan 04

Reference shape (adapt to match `radio-group.tsx` styling conventions exactly — matching `aria-invalid` ring colours, `disabled:cursor-not-allowed disabled:opacity-50` patterns):

```typescript
import * as React from "react";
import { Slider as SliderPrimitive } from "radix-ui"; // or "@radix-ui/react-slider" — match radio-group.tsx
import { cn } from "@/lib/utils";

function Slider({ className, defaultValue, value, min = 0, max = 100, ...props }: React.ComponentProps<typeof SliderPrimitive.Root>) {
  const values = React.useMemo(
    () => (Array.isArray(value) ? value : Array.isArray(defaultValue) ? defaultValue : [min, max]),
    [value, defaultValue, min, max],
  );
  return (
    <SliderPrimitive.Root
      data-slot="slider"
      defaultValue={defaultValue}
      value={value}
      min={min}
      max={max}
      className={cn("relative flex w-full touch-none items-center select-none data-[disabled]:opacity-50", className)}
      {...props}
    >
      <SliderPrimitive.Track
        data-slot="slider-track"
        className="relative h-2 w-full grow overflow-hidden rounded-full bg-muted"
      >
        <SliderPrimitive.Range data-slot="slider-range" className="absolute h-full rounded-full bg-primary" />
      </SliderPrimitive.Track>
      {values.map((_, i) => (
        <SliderPrimitive.Thumb
          data-slot="slider-thumb"
          key={i}
          className="block size-5 shrink-0 rounded-full border-2 border-primary bg-background shadow-sm transition-[color,box-shadow] focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50"
        />
      ))}
    </SliderPrimitive.Root>
  );
}

export { Slider };
```

Step 3: Run `cd trade-flow-ui && npm run typecheck && npm run lint && npm run format:check` and fix any issues inside the new file (NEVER via @ts-ignore, eslint-disable, or format-skip).
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/components/ui/slider.tsx</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/components/ui/slider.tsx` exists
    - `grep -c "@radix-ui/react-slider\|Slider as SliderPrimitive" trade-flow-ui/src/components/ui/slider.tsx` returns at least 1
    - `grep -c "\"@radix-ui/react-slider\"" trade-flow-ui/package.json` returns at least 1
    - `grep -c "export { Slider }" trade-flow-ui/src/components/ui/slider.tsx` returns at least 1
    - `grep -c "data-slot=\"slider" trade-flow-ui/src/components/ui/slider.tsx` returns at least 1
    - `grep -c "SliderPrimitive.Thumb" trade-flow-ui/src/components/ui/slider.tsx` returns at least 1
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/components/ui/slider.tsx` returns 0
    - `cd trade-flow-ui && npm run typecheck` exits 0
    - `cd trade-flow-ui && npm run lint` exits 0 for this file
  </acceptance_criteria>
  <done>New shadcn-style Slider component exists, wraps Radix primitives, passes typecheck and lint.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Add formatRange to src/lib/currency.ts + golden-file test + fixture</name>
  <files>trade-flow-ui/src/lib/currency.ts, trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts, trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json</files>
  <read_first>
    - trade-flow-ui/src/lib/currency.ts (for existing formatAmount / createMoney / formatCurrencyByCode and how locale mapping works)
    - trade-flow-ui/src/hooks/useCurrency.ts (for how formatAmount is consumed — but DO NOT depend on the hook; formatRange is a pure function)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-FMT-01 through D-FMT-04 and the example outputs
    - .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md (for sample IEstimatePriceRangeDto numbers if present)
  </read_first>
  <behavior>
    - Test "matches explicit tuples" — asserts `formatRange({subTotal:10000,taxTotal:2000,total:12000}, {subTotal:11000,taxTotal:2200,total:13200}, "range", "GBP")` returns exactly `"£120.00 - £132.00"`
    - Test "matches from mode" — same low/high, mode `"from"`, returns `"From £120.00"`
    - Test "handles zero contingency in range mode" — low.total === high.total returns `"£X.XX - £X.XX"` (or `"£X.XX"`? Keep literal — do not collapse)
    - Test "handles zero tax" — subTotal === total
    - Test "agrees with golden-file fixture" — reads estimate-response.sample.json and asserts `formatRange(priceRange.low, priceRange.high, displayMode, currencyCode) === expectedFormattedRange`
    - Fixture: JSON file with shape `{ priceRange: { low: {subTotal, taxTotal, total}, high: {...}}, displayMode, currencyCode, expectedFormattedRange }`
  </behavior>
  <action>
Step 1: APPEND to `trade-flow-ui/src/lib/currency.ts` a new exported function `formatRange`. Do NOT modify existing exports. Add at the bottom of the file:

```typescript
export type MoneyBreakdown = {
  subTotal: number;
  taxTotal: number;
  total: number;
};

export type PriceRangeDisplayMode = "range" | "from";

/**
 * Format an API-returned price range (low + high totals) for UI display.
 *
 * Delegates all per-amount formatting to `formatAmount`, so thousand separators,
 * decimal places, currency symbol, and locale all come from the existing dinero.js path.
 *
 * `formatRange` performs NO arithmetic — it renders whatever the API returns.
 * This is the contract that enforces Success Criterion #4 of Phase 43: the UI must
 * never multiply base × contingency client-side.
 */
export function formatRange(
  low: MoneyBreakdown,
  high: MoneyBreakdown,
  mode: PriceRangeDisplayMode,
  currencyCode: SupportedCurrencyCode,
): string {
  const lowFormatted = formatAmount(low.total, currencyCode);
  if (mode === "from") {
    return `From ${lowFormatted}`;
  }
  const highFormatted = formatAmount(high.total, currencyCode);
  return `${lowFormatted} - ${highFormatted}`;
}
```

Step 2: Create NEW fixture file `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` with the exact content below. This fixture mirrors the Phase 41 `IEstimateDto` shape — an executor copying this must not introduce a drifting synthetic shape. The numbers are minor-unit integers (pence, GBP). The `expectedFormattedRange` is the literal string `formatRange` must produce.

```json
{
  "id": "est_golden_001",
  "businessId": "biz_golden_001",
  "customerId": "cust_golden_001",
  "jobId": "job_golden_001",
  "number": "E-2026-001",
  "title": "Bathroom refit estimate (golden fixture)",
  "status": "draft",
  "estimateDate": "2026-04-11",
  "updatedAt": "2026-04-11T12:00:00.000Z",
  "customerName": "Golden Fixture Customer",
  "jobTitle": "Bathroom refit (golden fixture)",
  "contingencyPercent": 10,
  "displayMode": "range",
  "currencyCode": "GBP",
  "uncertaintyReasons": ["site_inspection", "hidden_conditions"],
  "uncertaintyNotes": "Golden-file fixture — do not edit without updating the expectedFormattedRange.",
  "lineItems": [],
  "totals": { "subTotal": 10000, "taxTotal": 2000, "total": 12000 },
  "priceRange": {
    "low":  { "subTotal": 10000, "taxTotal": 2000, "total": 12000 },
    "high": { "subTotal": 11000, "taxTotal": 2200, "total": 13200 }
  },
  "responseSummary": null,
  "expectedFormattedRange": "£120.00 - £132.00"
}
```

Step 3: Create NEW test file `trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { formatRange, type MoneyBreakdown, type PriceRangeDisplayMode } from "@/lib/currency";
import estimateFixture from "./fixtures/estimate-response.sample.json";

describe("formatRange", () => {
  const cases: Array<{
    name: string;
    low: MoneyBreakdown;
    high: MoneyBreakdown;
    mode: PriceRangeDisplayMode;
    currency: "GBP";
    expected: string;
  }> = [
    {
      name: "range mode with tax",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £132.00",
    },
    {
      name: "from mode with tax",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "from",
      currency: "GBP",
      expected: "From £120.00",
    },
    {
      name: "zero contingency collapses range to identical totals",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £120.00",
    },
    {
      name: "zero tax — total equals subTotal",
      low: { subTotal: 10000, taxTotal: 0, total: 10000 },
      high: { subTotal: 11000, taxTotal: 0, total: 11000 },
      mode: "range",
      currency: "GBP",
      expected: "£100.00 - £110.00",
    },
    {
      name: "maximum contingency (30%)",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 13000, taxTotal: 2600, total: 15600 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £156.00",
    },
    {
      name: "from mode renders only low.total regardless of high",
      low: { subTotal: 5000, taxTotal: 1000, total: 6000 },
      high: { subTotal: 999999, taxTotal: 999999, total: 999999 },
      mode: "from",
      currency: "GBP",
      expected: "From £60.00",
    },
  ];

  it.each(cases)("$name", ({ low, high, mode, currency, expected }) => {
    expect(formatRange(low, high, mode, currency)).toBe(expected);
  });

  it("agrees with the Phase 41 golden-file fixture", () => {
    const { priceRange, displayMode, currencyCode, expectedFormattedRange } = estimateFixture;
    const result = formatRange(
      priceRange.low as MoneyBreakdown,
      priceRange.high as MoneyBreakdown,
      displayMode as PriceRangeDisplayMode,
      currencyCode as "GBP",
    );
    expect(result).toBe(expectedFormattedRange);
  });
});
```

Step 4: Run `cd trade-flow-ui && npm run test -- currency.formatRange`. Test must pass. If `tsconfig` does not have `resolveJsonModule` enabled, enable it in `tsconfig.app.json` — this is a legitimate fix, not a suppression.

Step 5: Run `cd trade-flow-ui && npm run ci` — full quality gate must pass.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run test -- --run src/lib/__tests__/currency.formatRange.test.ts && npm run typecheck && npm run lint && npm run format:check</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "export function formatRange" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "formatAmount(low.total" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "formatAmount(high.total" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - File `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` exists
    - File `trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` exists
    - `grep -c "\"expectedFormattedRange\": \"£120.00 - £132.00\"" trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` returns 1
    - `grep -c "\"priceRange\":" trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` returns at least 1
    - `grep -c "describe(\"formatRange\"" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "agrees with the Phase 41 golden-file fixture" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/lib/currency.ts trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns 0
    - `cd trade-flow-ui && npm run test -- --run src/lib/__tests__/currency.formatRange.test.ts` exits 0
    - `cd trade-flow-ui && npm run ci` exits 0
  </acceptance_criteria>
  <done>formatRange exists, test asserts 6+ tuples plus golden-file agreement, npm run ci passes.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| none-runtime | This plan adds pure helper code + a dev dependency + test + fixture. No network boundary, no user input, no auth path. |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-02-01 | Information disclosure | formatRange drifts from Phase 41 API shape silently | mitigate | Golden-file fixture + agreement assertion — any drift breaks `npm run test`, any drift between fixture and backend breaks Phase 41 tests (both in same repo, visible in CI) |
| T-43-02-02 | Tampering | Future contributor hand-edits fixture totals without updating expectedFormattedRange | mitigate | Fixture carries an explicit comment field; test file asserts the exact string so any drift fails immediately; acceptance criteria grep includes the literal expected value |
| T-43-02-03 | Supply chain | New `@radix-ui/react-slider` dependency introduces vulnerability | accept | Radix UI maintained by active security-conscious team; existing codebase already relies on 15 Radix packages; `npm audit` is part of existing CI via `npm run ci` |
| T-43-02-04 | Denial of service | Slider thumb mapping over `value` may crash if not defensively null-checked | mitigate | Slider component defaults `values` array using `React.useMemo` with explicit fallback chain — no unchecked indexing |
</threat_model>

<verification>
CI gate for this plan is `cd trade-flow-ui && npm run ci` — must exit 0. The golden-file test is the load-bearing verification: it asserts `formatRange` produces `"£120.00 - £132.00"` from the same priceRange shape Phase 41 puts on the wire.

Manual inspection (non-blocking): verify the new slider.tsx styles look correct in isolation by importing into a throwaway Storybook or a scratch component. Not required to ship.
</verification>

<success_criteria>
- `@radix-ui/react-slider` installed and in package.json
- `src/components/ui/slider.tsx` exports shadcn-style Slider wrapping Radix primitives
- `formatRange` exported from `src/lib/currency.ts` with the exact signature `(low: MoneyBreakdown, high: MoneyBreakdown, mode: "range" | "from", currencyCode: SupportedCurrencyCode) => string`
- Golden-file test exists and passes
- Fixture JSON carries priceRange shape that matches Phase 41 D-CONT-05 and an `expectedFormattedRange` field
- `npm run ci` passes in trade-flow-ui
- No `as` type assertions used except within the test file's `priceRange.low as MoneyBreakdown` boundary where JSON module types do not carry the stricter shape (this is the type-guard-style narrowing of external data, not an unsafe cast). NO `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, or `eslint-disable` anywhere.
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-02-SUMMARY.md`
</output>
