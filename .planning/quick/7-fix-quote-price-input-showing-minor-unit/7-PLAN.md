---
phase: quick-7
plan: 1
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
autonomous: true
requirements: [QUICK-7]
must_haves:
  truths:
    - "Price input field shows dollar/pound value (e.g., 2.00) not minor units (e.g., 200)"
    - "Editing price and pressing Enter/blurring sends correct minor units to API"
    - "Pressing Escape resets input to correct decimal display value"
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx"
      provides: "Mobile card list with corrected price input conversion"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      provides: "Desktop table with corrected price input conversion"
  key_links:
    - from: "QuoteLineItemsCardList.tsx"
      to: "useCurrencyConversion"
      via: "toInput/fromInput for price display and submission"
      pattern: "useCurrencyConversion|toInput|fromInput"
    - from: "QuoteLineItemsTable.tsx"
      to: "useCurrencyConversion"
      via: "toInput/fromInput for price display and submission"
      pattern: "useCurrencyConversion|toInput|fromInput"
---

<objective>
Fix quote line item price inputs to display decimal values (e.g., "2.00") instead of raw minor units (e.g., "200").

Purpose: Users editing quote line item prices see confusing minor unit values. A price of 200 pence shows as "200" in the input instead of "2.00".
Output: Both QuoteLineItemsCardList.tsx and QuoteLineItemsTable.tsx correctly convert between minor units (API) and decimal display (UI).
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
@trade-flow-ui/src/hooks/useCurrencyConversion.ts
@trade-flow-ui/src/lib/forms/currency-inputs.ts

<interfaces>
From src/hooks/useCurrencyConversion.ts:
```typescript
export function useCurrencyConversion(): {
  currency: SupportedCurrencyCode;
  toInput: (cents: number | null | undefined) => string;   // minor units -> "10.50"
  fromInput: (value: string) => number | undefined;         // "10.50" -> 1050
}
```

From src/lib/forms/currency-inputs.ts:
```typescript
export function currencyToInput(cents: number | null | undefined, currencyCode: SupportedCurrencyCode): string;
export function inputToCurrency(value: string, currencyCode: SupportedCurrencyCode): number | undefined;
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Fix price input conversion in QuoteLineItemsCardList and QuoteLineItemsTable</name>
  <files>
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx,
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
  </files>
  <action>
In BOTH files, apply the same three fixes:

1. **Import useCurrencyConversion:** Add `import { useCurrencyConversion } from "@/hooks/useCurrencyConversion";` and call `const { toInput, fromInput } = useCurrencyConversion();` inside the component (alongside the existing `useBusinessCurrency` call).

2. **Fix defaultValue on price Input:** Change `defaultValue={lineItem.unitPrice}` to `defaultValue={toInput(lineItem.unitPrice)}`. This converts minor units (e.g., 200) to decimal string (e.g., "2.00"). Also update the `key` prop from `key={\`price-${lineItem.id}-${lineItem.unitPrice}\`}` to `key={\`price-${lineItem.id}-${toInput(lineItem.unitPrice)}\`}` so React re-renders when the converted value changes.

3. **Fix handlePriceBlur:** Replace the current logic that does `parseFloat(value)` and sends raw value as `unitPrice`. Instead:
   ```typescript
   const handlePriceBlur = (lineItem: QuoteLineItem, value: string) => {
     const minorUnits = fromInput(value);
     if (minorUnits !== undefined && minorUnits >= 0 && minorUnits !== lineItem.unitPrice) {
       updateLineItem({
         businessId,
         quoteId,
         lineItemId: lineItem.id,
         unitPrice: minorUnits,
       });
     }
   };
   ```

4. **Fix handleKeyDown Escape reset:** Change the Escape branch from `String(lineItem.unitPrice)` to `toInput(lineItem.unitPrice)` for the unitPrice field so it resets to the decimal display value, not minor units.

Do NOT change any quantity-related logic or any formatAmount display calls (those already correctly accept minor units).
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npm run typecheck && npm run lint</automated>
  </verify>
  <done>
    - Price input defaultValue shows decimal value (e.g., "2.00" for 200 minor units)
    - Editing price submits correct minor units to API (typing "5.50" sends 550)
    - Escape key resets to decimal display value
    - Both CardList and Table components updated consistently
    - No TypeScript or lint errors
  </done>
</task>

</tasks>

<verification>
- `npm run typecheck` passes in trade-flow-ui
- `npm run lint` passes in trade-flow-ui
- Grep confirms no remaining `defaultValue={lineItem.unitPrice}` in price input contexts
- Grep confirms `toInput` and `fromInput` are used in both files
</verification>

<success_criteria>
Price input fields in quote line items display decimal currency values instead of minor units. User types decimal values (e.g., "10.50") and the API receives correct minor unit values (e.g., 1050).
</success_criteria>

<output>
After completion, create `.planning/quick/7-fix-quote-price-input-showing-minor-unit/7-SUMMARY.md`
</output>
