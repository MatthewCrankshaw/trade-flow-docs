---
phase: quick
plan: 5
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/lib/currency.ts
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
autonomous: true
requirements: [QUICK-5]
must_haves:
  truths:
    - "createMoney() never crashes with non-integer amount values"
    - "lineTotalIncTax produces integer minor-unit values"
    - "Quote pages render without Dinero.js 'Amount is invalid' errors"
  artifacts:
    - path: "trade-flow-ui/src/lib/currency.ts"
      provides: "Defensive Math.round in createMoney"
      contains: "Math.round(amount)"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      provides: "Rounded lineTotalIncTax helper"
      contains: "Math.round"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx"
      provides: "Rounded lineTotalIncTax helper"
      contains: "Math.round"
  key_links:
    - from: "QuoteLineItemsTable.tsx"
      to: "currency.ts createMoney"
      via: "formatAmount -> createMoney"
      pattern: "formatAmount"
---

<objective>
Fix Dinero.js "Amount is invalid" crash by rounding non-integer values before passing to dinero().

Purpose: Prevent runtime crash on quote pages when tax calculations produce fractional minor-unit values.
Output: Defensive rounding in createMoney() and lineTotalIncTax helpers in both Table and CardList components.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/src/lib/currency.ts
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add defensive rounding to createMoney and lineTotalIncTax helpers</name>
  <files>
    trade-flow-ui/src/lib/currency.ts
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
  </files>
  <action>
    1. In `trade-flow-ui/src/lib/currency.ts` line 36, wrap the amount parameter with `Math.round()`:
       ```typescript
       return dinero({ amount: Math.round(amount), currency: getCurrency(currencyCode) });
       ```
       This is the primary defensive fix — all callers of createMoney (formatAmount, formatCurrencyByCode, etc.) are protected.

    2. In `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` lines 129-130, wrap the lineTotalIncTax result:
       ```typescript
       const lineTotalIncTax = (item: QuoteLineItem) =>
           Math.round(item.lineTotal * (1 + item.taxRate / 100));
       ```

    3. In `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` lines 121-122, apply the same fix:
       ```typescript
       const lineTotalIncTax = (item: QuoteLineItem) =>
           Math.round(item.lineTotal * (1 + item.taxRate / 100));
       ```

    The lineTotalIncTax fixes are belt-and-suspenders — even though createMoney now rounds, callers should produce clean integers to avoid masking logic bugs elsewhere.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint</automated>
  </verify>
  <done>
    - createMoney() rounds amount before passing to dinero()
    - Both lineTotalIncTax helpers produce integers via Math.round()
    - TypeScript compiles cleanly, lint passes
  </done>
</task>

</tasks>

<verification>
- `npm run typecheck` passes in trade-flow-ui
- `npm run lint` passes in trade-flow-ui
- `grep -n "Math.round" trade-flow-ui/src/lib/currency.ts` shows rounding on the createMoney line
- `grep -n "Math.round" trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` shows rounding in lineTotalIncTax
- `grep -n "Math.round" trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` shows rounding in lineTotalIncTax
</verification>

<success_criteria>
- Dinero.js no longer throws "Amount is invalid" for non-integer money values
- All three files contain Math.round at the identified locations
- typecheck and lint pass
</success_criteria>

<output>
After completion, create `.planning/quick/5-fix-dinero-js-invalid-amount-round-non-i/5-SUMMARY.md`
</output>
