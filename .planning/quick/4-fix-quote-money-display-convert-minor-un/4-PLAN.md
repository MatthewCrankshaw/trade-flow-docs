---
phase: quick-4
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/features/quotes/components/QuotesTable.tsx
  - trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
  - trade-flow-ui/src/pages/QuoteDetailPage.tsx
  - trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx
autonomous: true
requirements: [QUICK-4]
must_haves:
  truths:
    - "Quote list displays correct dollar amounts (e.g. 1000 cents shows as $10.00, not $1,000.00)"
    - "Quote detail page shows correct subtotal, tax, and total amounts"
    - "Quote line items show correct unit prices and line totals"
    - "Job detail quote tab shows correct quote totals"
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuotesTable.tsx"
      provides: "Quote list table with correct currency formatting"
      contains: "formatAmount"
    - path: "trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx"
      provides: "Quote list cards with correct currency formatting"
      contains: "formatAmount"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      provides: "Line items table with correct currency formatting"
      contains: "formatAmount"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx"
      provides: "Line items cards with correct currency formatting"
      contains: "formatAmount"
    - path: "trade-flow-ui/src/pages/QuoteDetailPage.tsx"
      provides: "Quote detail totals with correct currency formatting"
      contains: "formatAmount"
    - path: "trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx"
      provides: "Job detail quote totals with correct currency formatting"
      contains: "formatAmount"
  key_links:
    - from: "all quote UI components"
      to: "useBusinessCurrency().formatAmount"
      via: "minor-unit to major-unit conversion"
      pattern: "formatAmount"
---

<objective>
Fix quote money display: replace all `formatDecimal()` calls with `formatAmount()` in quote UI components.

Purpose: The API returns money values in minor units (cents). `formatDecimal` treats these as dollar amounts, showing 100x the correct value. `formatAmount` correctly converts from minor units.
Output: All quote-related components display correct dollar amounts.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/src/hooks/useCurrency.ts
</context>

<tasks>

<task type="auto">
  <name>Task 1: Replace formatDecimal with formatAmount in quote list and detail components</name>
  <files>
    trade-flow-ui/src/features/quotes/components/QuotesTable.tsx
    trade-flow-ui/src/features/quotes/components/QuotesCardList.tsx
    trade-flow-ui/src/pages/QuoteDetailPage.tsx
    trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx
  </files>
  <action>
In these files, the hook is accessed as `currency = useBusinessCurrency()` and called as `currency.formatDecimal(...)`. Replace each call with `currency.formatAmount(...)`:

**QuotesTable.tsx** (line 72):
- `currency.formatDecimal(quote.totals.total)` -> `currency.formatAmount(quote.totals.total)`

**QuotesCardList.tsx** (line 72):
- `currency.formatDecimal(quote.totals.total)` -> `currency.formatAmount(quote.totals.total)`

**QuoteDetailPage.tsx** (lines 155, 159, 164):
- `currency.formatDecimal(quote.totals.subTotal)` -> `currency.formatAmount(quote.totals.subTotal)`
- `currency.formatDecimal(quote.totals.taxTotal)` -> `currency.formatAmount(quote.totals.taxTotal)`
- `currency.formatDecimal(quote.totals.total)` -> `currency.formatAmount(quote.totals.total)`

**JobDetailTabs.tsx** (line 319):
- `currency.formatDecimal(q.totals.total)` -> `currency.formatAmount(q.totals.total)`

After changes, verify no remaining `formatDecimal` usage in these files. If `formatDecimal` is no longer imported/destructured anywhere in the file, remove it from the destructuring or import.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && grep -rn "formatDecimal" src/features/quotes/components/QuotesTable.tsx src/features/quotes/components/QuotesCardList.tsx src/pages/QuoteDetailPage.tsx src/features/jobs/components/JobDetailTabs.tsx; echo "EXIT:$?"</automated>
  </verify>
  <done>No formatDecimal calls remain in these 4 files. All money values use formatAmount.</done>
</task>

<task type="auto">
  <name>Task 2: Replace formatDecimal with formatAmount in quote line item components</name>
  <files>
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
    trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
  </files>
  <action>
In these files, the hook is destructured as `const { formatDecimal } = useBusinessCurrency()`. Change to destructure `formatAmount` instead:

**QuoteLineItemsTable.tsx** (line 61):
- `const { formatDecimal } = useBusinessCurrency()` -> `const { formatAmount } = useBusinessCurrency()`
- Replace all 4 calls (lines 240, 251, 305, 309):
  - `formatDecimal(lineItem.unitPrice)` -> `formatAmount(lineItem.unitPrice)`
  - `formatDecimal(lineTotalIncTax(lineItem))` -> `formatAmount(lineTotalIncTax(lineItem))`
  - `formatDecimal(component.unitPrice)` -> `formatAmount(component.unitPrice)`
  - `formatDecimal(lineTotalIncTax(component))` -> `formatAmount(lineTotalIncTax(component))`

**QuoteLineItemsCardList.tsx** (line 54):
- `const { formatDecimal } = useBusinessCurrency()` -> `const { formatAmount } = useBusinessCurrency()`
- Replace all 4 calls (lines 240, 244, 292, 295):
  - `formatDecimal(lineItem.unitPrice)` -> `formatAmount(lineItem.unitPrice)`
  - `formatDecimal(lineTotalIncTax(lineItem))` -> `formatAmount(lineTotalIncTax(lineItem))`
  - `formatDecimal(component.unitPrice)` -> `formatAmount(component.unitPrice)`
  - `formatDecimal(lineTotalIncTax(component))` -> `formatAmount(lineTotalIncTax(component))`
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && grep -rn "formatDecimal" src/features/quotes/components/QuoteLineItemsTable.tsx src/features/quotes/components/QuoteLineItemsCardList.tsx; echo "EXIT:$?"</automated>
  </verify>
  <done>No formatDecimal calls remain in line item components. All money values use formatAmount.</done>
</task>

</tasks>

<verification>
Run full verification that no quote component uses formatDecimal:
```bash
cd trade-flow-ui && grep -rn "formatDecimal" src/features/quotes/ src/pages/QuoteDetailPage.tsx src/features/jobs/components/JobDetailTabs.tsx
```
Expected: no output (no matches).

Build check:
```bash
cd trade-flow-ui && npx tsc --noEmit
```
Expected: no type errors.
</verification>

<success_criteria>
- Zero `formatDecimal` calls remain in any quote-related component
- All quote money displays use `formatAmount` (minor-unit conversion)
- TypeScript compiles without errors
</success_criteria>

<output>
After completion, create `.planning/quick/4-fix-quote-money-display-convert-minor-un/4-SUMMARY.md`
</output>
