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
  - trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256
autonomous: true
requirements:
  - CONT-03

must_haves:
  truths:
    - "formatRange(low, high, mode, currencyCode) exists in src/lib/currency.ts and returns the exact strings shown in the golden-file test"
    - "Vitest golden-file test src/lib/__tests__/currency.formatRange.test.ts passes, asserting API and UI agree to the penny"
    - "Test covers both GBP and EUR smoke tuples per D-FMT-04"
    - "Committed SHA-256 hash file guards the fixture against silent drift; the test re-hashes the fixture at runtime and fails on mismatch"
    - "@radix-ui/react-slider is installed and src/components/ui/slider.tsx wraps it"
  artifacts:
    - path: "trade-flow-ui/src/components/ui/slider.tsx"
      provides: "shadcn New York style Slider wrapper around @radix-ui/react-slider"
      contains: "@radix-ui/react-slider"
    - path: "trade-flow-ui/src/lib/currency.ts"
      provides: "formatRange helper + existing formatAmount + EUR added to SUPPORTED_CURRENCIES"
      contains: "export function formatRange"
    - path: "trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts"
      provides: "Golden-file test with GBP + EUR tuples, fixture-driven API agreement, and committed-hash drift guard"
      contains: "describe(\"formatRange\""
    - path: "trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json"
      provides: "Fixture mirroring Phase 41 IEstimateDto shape with expectedFormattedRange field"
      contains: "priceRange"
    - path: "trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256"
      provides: "Committed SHA-256 hash of the canonical fixture so CI catches silent drift"
      contains: ""
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
    - from: "trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts"
      to: "trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256"
      via: "runtime SHA-256 re-hash compared to committed hash"
      pattern: "createHash.*sha256"
---

<objective>
Install the Radix slider dependency, build the shadcn New York Slider wrapper, and ship the `formatRange` helper plus its golden-file test that pins UI formatting against a real API response. The fixture is drift-guarded with a committed SHA-256 hash and the smoke-tuple suite covers both GBP and EUR per D-FMT-04.

Purpose: These are the independent foundation primitives that ContingencySlider (Plan 04) and every estimate list/detail surface (Plans 05/06) consume. The golden-file test is the contract that enforces Success Criterion #4 ("UI never multiplies base × contingency on the client; formatRange renders only what the API returns"). The committed hash file is the CI drift guard required by D-FMT-04 bullet (3).

Output: New slider.tsx, new formatRange export, EUR added to SUPPORTED_CURRENCIES, new golden-file test, new JSON fixture, new `.sha256` hash file, new npm dependency.
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
export const SUPPORTED_CURRENCIES = { GBP } as const; // ← needs EUR added for D-FMT-04
export type SupportedCurrencyCode = keyof typeof SUPPORTED_CURRENCIES;
export function formatAmount(amount: number | null | undefined, currencyCode: SupportedCurrencyCode, locale?: string): string;
// formatCurrencyByCode uses a localeMap: Record<SupportedCurrencyCode, string> — EUR needs a locale entry too
// GBP returns formatted via en-GB locale, e.g. "£120.00"
// EUR (once added) will return formatted via en-IE locale, e.g. "€120.00"
```

Phase 41 D-CONT-05/06 locked DTO shape for priceRange: `{ low: { subTotal, taxTotal, total }, high: { subTotal, taxTotal, total } }` where the numbers are minor-units integers (pence for GBP, cents for EUR).
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
  <name>Task 2: Extend SUPPORTED_CURRENCIES with EUR, add formatRange, create golden-file test + fixture + committed hash</name>
  <files>trade-flow-ui/src/lib/currency.ts, trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts, trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json, trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256</files>
  <read_first>
    - trade-flow-ui/src/lib/currency.ts (for existing formatAmount / createMoney / formatCurrencyByCode and how locale mapping works — note SUPPORTED_CURRENCIES currently only has GBP)
    - trade-flow-ui/src/hooks/useCurrency.ts (for how formatAmount is consumed — but DO NOT depend on the hook; formatRange is a pure function)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-FMT-01 through D-FMT-04 and the example outputs (EUR is explicitly listed as a smoke case)
    - .planning/phases/41-estimate-module-crud-backend/41-07-estimate-crud-services-PLAN.md (for sample IEstimatePriceRangeDto numbers if present)
  </read_first>
  <behavior>
    - `SUPPORTED_CURRENCIES` map gains an `EUR` entry imported from `@dinero.js/currencies`
    - `localeMap` in `formatCurrencyByCode` gains an `EUR: "en-IE"` entry so EUR outputs `"€120.00"` (matching the en-GB-style dot decimal separator)
    - Test "matches explicit tuples" — asserts GBP tuples exactly: `formatRange({subTotal:10000,taxTotal:2000,total:12000}, {subTotal:11000,taxTotal:2200,total:13200}, "range", "GBP")` returns exactly `"£120.00 - £132.00"`
    - Test "matches EUR smoke tuple" — asserts `formatRange({subTotal:10000,taxTotal:2000,total:12000}, {subTotal:11000,taxTotal:2200,total:13200}, "range", "EUR")` returns exactly `"€120.00 - €132.00"` (verified via Node's Intl.NumberFormat("en-IE", { style: "currency", currency: "EUR" }))
    - Test "matches from mode" — same low/high, mode `"from"`, returns `"From £120.00"`
    - Test "handles zero contingency in range mode" — low.total === high.total returns `"£X.XX - £X.XX"` (do not collapse)
    - Test "handles zero tax" — subTotal === total
    - Test "agrees with golden-file fixture" — reads estimate-response.sample.json and asserts `formatRange(priceRange.low, priceRange.high, displayMode, currencyCode) === expectedFormattedRange`
    - Test "fixture hash matches committed SHA-256" — reads the fixture bytes, computes SHA-256 via Node's `crypto.createHash("sha256")`, reads the `.sha256` sidecar file, and asserts equality. Fails loudly on any drift.
    - Fixture: JSON file with shape `{ priceRange: { low: {subTotal, taxTotal, total}, high: {...}}, displayMode, currencyCode, expectedFormattedRange }`
    - Sidecar: `estimate-response.sample.json.sha256` contains the hex SHA-256 of the fixture bytes (computed and committed at plan-execution time)
  </behavior>
  <action>
Step 1: EDIT `trade-flow-ui/src/lib/currency.ts` to extend `SUPPORTED_CURRENCIES` with EUR and update the locale map. This is required because D-FMT-04 names EUR as a smoke test case and the existing `SupportedCurrencyCode` type is `"GBP"` only — without this extension the EUR tuple will not typecheck.

Replace the existing imports line:
```typescript
import { GBP } from "@dinero.js/currencies";
```
with:
```typescript
import { GBP, EUR } from "@dinero.js/currencies";
```

Replace the existing SUPPORTED_CURRENCIES block:
```typescript
export const SUPPORTED_CURRENCIES = {
  GBP,
} as const;
```
with:
```typescript
export const SUPPORTED_CURRENCIES = {
  GBP,
  EUR,
} as const;
```

Replace the existing localeMap inside `formatCurrencyByCode`:
```typescript
  const localeMap: Record<SupportedCurrencyCode, string> = {
    GBP: "en-GB",
  };
```
with:
```typescript
  const localeMap: Record<SupportedCurrencyCode, string> = {
    GBP: "en-GB",
    EUR: "en-IE",
  };
```

The `en-IE` locale is chosen because it renders EUR with the same dot-decimal convention as `en-GB` renders GBP (`€120.00`, not `120,00 €`), keeping the smoke tuple stable across CI environments. Do NOT change any other function in currency.ts.

Step 2: APPEND to `trade-flow-ui/src/lib/currency.ts` a new exported function `formatRange`. Do NOT modify the other existing exports. Add at the bottom of the file:

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

Step 3: Create NEW fixture file `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` with the exact content below. This fixture mirrors the Phase 41 `IEstimateDto` shape — an executor copying this must not introduce a drifting synthetic shape. The numbers are minor-unit integers (pence, GBP). The `expectedFormattedRange` is the literal string `formatRange` must produce.

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

Step 4: Compute the SHA-256 of the fixture file and commit it to a sidecar `.sha256` file. From the `trade-flow-ui` directory, run:

```bash
node -e "const fs=require('fs');const c=require('crypto');const p='src/lib/__tests__/fixtures/estimate-response.sample.json';const h=c.createHash('sha256').update(fs.readFileSync(p)).digest('hex');fs.writeFileSync(p+'.sha256', h+'\n');console.log(h);"
```

This writes the hex SHA-256 + trailing newline to `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256`. The test in Step 5 re-reads the fixture and the sidecar and asserts equality — this is the CI drift guard required by D-FMT-04 bullet (3).

Step 5: Create NEW test file `trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts`. Note the explicit EUR tuple (required by D-FMT-04) and the fixture-hash assertion (required by D-FMT-04 bullet 3):

```typescript
import { describe, it, expect } from "vitest";
import { readFileSync } from "node:fs";
import { createHash } from "node:crypto";
import { fileURLToPath } from "node:url";
import { dirname, resolve } from "node:path";
import { formatRange, type MoneyBreakdown, type PriceRangeDisplayMode } from "@/lib/currency";
import estimateFixture from "./fixtures/estimate-response.sample.json";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
const FIXTURE_PATH = resolve(__dirname, "fixtures/estimate-response.sample.json");
const FIXTURE_HASH_PATH = resolve(__dirname, "fixtures/estimate-response.sample.json.sha256");

describe("formatRange", () => {
  const cases: Array<{
    name: string;
    low: MoneyBreakdown;
    high: MoneyBreakdown;
    mode: PriceRangeDisplayMode;
    currency: "GBP" | "EUR";
    expected: string;
  }> = [
    {
      name: "GBP range mode with tax",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £132.00",
    },
    {
      name: "GBP from mode with tax",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "from",
      currency: "GBP",
      expected: "From £120.00",
    },
    {
      name: "GBP zero contingency collapses range to identical totals",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £120.00",
    },
    {
      name: "GBP zero tax — total equals subTotal",
      low: { subTotal: 10000, taxTotal: 0, total: 10000 },
      high: { subTotal: 11000, taxTotal: 0, total: 11000 },
      mode: "range",
      currency: "GBP",
      expected: "£100.00 - £110.00",
    },
    {
      name: "GBP maximum contingency (30%)",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 13000, taxTotal: 2600, total: 15600 },
      mode: "range",
      currency: "GBP",
      expected: "£120.00 - £156.00",
    },
    {
      name: "GBP from mode renders only low.total regardless of high",
      low: { subTotal: 5000, taxTotal: 1000, total: 6000 },
      high: { subTotal: 999999, taxTotal: 999999, total: 999999 },
      mode: "from",
      currency: "GBP",
      expected: "From £60.00",
    },
    {
      name: "EUR range mode smoke tuple (D-FMT-04)",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "range",
      currency: "EUR",
      expected: "€120.00 - €132.00",
    },
    {
      name: "EUR from mode smoke tuple (D-FMT-04)",
      low: { subTotal: 10000, taxTotal: 2000, total: 12000 },
      high: { subTotal: 11000, taxTotal: 2200, total: 13200 },
      mode: "from",
      currency: "EUR",
      expected: "From €120.00",
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

  it("fixture SHA-256 matches the committed hash sidecar (drift guard for D-FMT-04)", () => {
    const fixtureBytes = readFileSync(FIXTURE_PATH);
    const actualHash = createHash("sha256").update(fixtureBytes).digest("hex");
    const committedHash = readFileSync(FIXTURE_HASH_PATH, "utf8").trim();
    expect(actualHash).toBe(committedHash);
  });
});
```

Step 6: Run `cd trade-flow-ui && npm run test -- --run src/lib/__tests__/currency.formatRange.test.ts`. All tests must pass. If `tsconfig` does not have `resolveJsonModule` enabled, enable it in `tsconfig.app.json` — this is a legitimate fix, not a suppression.

Step 7: Run `cd trade-flow-ui && npm run ci` — full quality gate must pass (tests + lint + format:check + typecheck per CLAUDE.md CI Gate Policy).

IMPORTANT: if any of the above steps requires regenerating the fixture (e.g. because an executor reformats the JSON and whitespace changes the bytes), you MUST rerun Step 4 to regenerate the committed `.sha256` file. The hash commit is the contract; do not hand-edit it.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run test -- --run src/lib/__tests__/currency.formatRange.test.ts && npm run ci</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "export function formatRange" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "formatAmount(low.total" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "formatAmount(high.total" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "GBP, EUR" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - `grep -c "EUR: \"en-IE\"" trade-flow-ui/src/lib/currency.ts` returns at least 1
    - File `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` exists
    - File `trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256` exists
    - File `trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` exists
    - `grep -c "\"expectedFormattedRange\": \"£120.00 - £132.00\"" trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` returns 1
    - `grep -c "\"priceRange\":" trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json` returns at least 1
    - `grep -c "describe(\"formatRange\"" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "agrees with the Phase 41 golden-file fixture" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "currency: \"EUR\"" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "€120.00 - €132.00" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "fixture SHA-256 matches the committed hash sidecar" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - `grep -c "createHash(\"sha256\")" trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns at least 1
    - The sidecar file contains a 64-character hex string: `[[ $(wc -c < trade-flow-ui/src/lib/__tests__/fixtures/estimate-response.sample.json.sha256) -ge 64 ]]` is true
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/lib/currency.ts trade-flow-ui/src/lib/__tests__/currency.formatRange.test.ts` returns 0
    - `cd trade-flow-ui && npm run test -- --run src/lib/__tests__/currency.formatRange.test.ts` exits 0
    - `cd trade-flow-ui && npm run ci` exits 0
  </acceptance_criteria>
  <done>formatRange exists, EUR is in SUPPORTED_CURRENCIES, test asserts 8 tuples (GBP + EUR) plus golden-file agreement plus committed-hash drift guard, `npm run ci` passes.</done>
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
| T-43-02-02 | Tampering | Future contributor hand-edits fixture totals without updating expectedFormattedRange | mitigate | Committed SHA-256 sidecar file + runtime hash re-computation test fails immediately on any byte-level drift. Any fixture edit forces a matching `.sha256` update, which is human-reviewable in the PR diff. |
| T-43-02-03 | Supply chain | New `@radix-ui/react-slider` dependency introduces vulnerability | accept | Radix UI maintained by active security-conscious team; existing codebase already relies on 15 Radix packages; `npm audit` is part of existing CI via `npm run ci` |
| T-43-02-04 | Denial of service | Slider thumb mapping over `value` may crash if not defensively null-checked | mitigate | Slider component defaults `values` array using `React.useMemo` with explicit fallback chain — no unchecked indexing |
| T-43-02-05 | Tampering | EUR smoke tuple drifts if ICU / Node Intl data changes locale output | mitigate | `en-IE` locale is stable across Node LTS versions; if a Node major version bumps ICU and breaks the smoke tuple, CI fails loudly and the tuple is updated in a dedicated commit alongside the Node bump |
</threat_model>

<verification>
CI gate for this plan is `cd trade-flow-ui && npm run ci` — must exit 0. The golden-file test is the load-bearing verification: it asserts `formatRange` produces `"£120.00 - £132.00"` from the same priceRange shape Phase 41 puts on the wire, asserts the EUR smoke tuple `"€120.00 - €132.00"` per D-FMT-04, and asserts the committed SHA-256 of the fixture matches the runtime re-hash (the D-FMT-04 drift guard).

Manual inspection (non-blocking): verify the new slider.tsx styles look correct in isolation by importing into a throwaway Storybook or a scratch component. Not required to ship.
</verification>

<success_criteria>
- `@radix-ui/react-slider` installed and in package.json
- `src/components/ui/slider.tsx` exports shadcn-style Slider wrapping Radix primitives
- `EUR` added to `SUPPORTED_CURRENCIES` in `src/lib/currency.ts` with `en-IE` locale mapping
- `formatRange` exported from `src/lib/currency.ts` with the exact signature `(low: MoneyBreakdown, high: MoneyBreakdown, mode: "range" | "from", currencyCode: SupportedCurrencyCode) => string`
- Golden-file test exists and passes, covering both GBP and EUR tuples (D-FMT-04 compliant)
- Committed SHA-256 sidecar file exists next to the fixture; test re-hashes at runtime and asserts equality (D-FMT-04 bullet 3 compliant)
- Fixture JSON carries priceRange shape that matches Phase 41 D-CONT-05 and an `expectedFormattedRange` field
- `npm run ci` passes in trade-flow-ui
- No `as` type assertions used except within the test file's `priceRange.low as MoneyBreakdown` boundary where JSON module types do not carry the stricter shape (this is the type-guard-style narrowing of external data, not an unsafe cast). NO `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, or `eslint-disable` anywhere.
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-02-SUMMARY.md`
</output>
</content>
</invoke>