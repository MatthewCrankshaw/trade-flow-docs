---
phase: quick
plan: 6
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
  - trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
autonomous: true
requirements: [QUICK-6]
must_haves:
  truths:
    - "Fixed-price bundles show the bundle's unitPrice in the unit price column/area"
    - "Component-based bundles show the sum of (component.unitPrice * component.quantity) in the unit price column/area"
    - "Bundle unit prices are read-only (not editable) in both table and card views"
    - "Tax rate column still shows dash for bundles"
  artifacts:
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx"
      provides: "Bundle unit price display in table view"
    - path: "trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx"
      provides: "Bundle unit price display in card list view"
  key_links:
    - from: "QuoteLineItemsTable.tsx"
      to: "itemsById map"
      via: "itemsById.get(lineItem.itemId)?.bundleConfig?.priceStrategy"
      pattern: "priceStrategy.*fixed|component_based"
---

<objective>
Show bundle unit price in quote line items instead of a dash. Fixed-price bundles display their unitPrice; component-based bundles display the computed sum of component prices.

Purpose: Users need to see what a bundle costs per unit to make informed quoting decisions.
Output: Both table and card list views display bundle unit prices correctly.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx
@trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx
@trade-flow-ui/src/types/api.types.ts

<interfaces>
<!-- Both components receive itemsById as a prop -->
From QuoteLineItemsTable.tsx (line 48):
```typescript
itemsById: Map<string, Item>;
```

From api.types.ts:
```typescript
export type BundlePriceStrategy = "fixed" | "component_based";

// Item.bundleConfig has:
priceStrategy: BundlePriceStrategy;

// QuoteLineItem has:
unitPrice: number;          // in minor units
components?: QuoteLineItemComponent[];

// QuoteLineItemComponent has:
unitPrice: number;          // in minor units
quantity: number;
```

Both components already have `itemsById` destructured from props and `formatAmount()` imported.
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Show bundle unit price in QuoteLineItemsTable</name>
  <files>trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx</files>
  <action>
In QuoteLineItemsTable.tsx, replace the unit price column logic for bundles (around lines 222-241).

Current code shows a dash for all bundles. Replace with bundle-aware pricing:

1. Add a helper function inside the component (before the return statement):

```typescript
const getBundleUnitPrice = (lineItem: QuoteLineItem): number => {
  const item = itemsById.get(lineItem.itemId);
  if (item?.bundleConfig?.priceStrategy === "fixed") {
    return lineItem.unitPrice;
  }
  // component_based: sum of component unit prices * quantities
  const components = lineItem.components ?? [];
  return components.reduce((sum, c) => sum + c.unitPrice * c.quantity, 0);
};
```

2. Replace the unit price TableCell content (lines 222-241) from:
```tsx
{isBundle ? (
    <span className="text-muted-foreground">&mdash;</span>
) : isEditable ? (
```
to:
```tsx
{isBundle ? (
    formatAmount(getBundleUnitPrice(lineItem))
) : isEditable ? (
```

The rest of the ternary (Input for editable, formatAmount for read-only non-bundles) stays the same.

Do NOT change the tax rate column (lines 243-248) — it should continue showing a dash for bundles.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npm run typecheck</automated>
  </verify>
  <done>Table view shows formatted bundle unit price (fixed price or component sum) instead of a dash. Tax rate still shows dash. Bundle price is read-only.</done>
</task>

<task type="auto">
  <name>Task 2: Show bundle unit price in QuoteLineItemsCardList</name>
  <files>trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx</files>
  <action>
In QuoteLineItemsCardList.tsx, replace the unit price display logic for bundles (around lines 218-242).

1. Add the same helper function inside the component (before the return statement):

```typescript
const getBundleUnitPrice = (lineItem: QuoteLineItem): number => {
  const item = itemsById.get(lineItem.itemId);
  if (item?.bundleConfig?.priceStrategy === "fixed") {
    return lineItem.unitPrice;
  }
  const components = lineItem.components ?? [];
  return components.reduce((sum, c) => sum + c.unitPrice * c.quantity, 0);
};
```

2. Replace lines 218-242 from:
```tsx
{!isBundle && isEditable ? (
    <div className="flex items-center gap-1">
      <span className="text-xs text-muted-foreground">@</span>
      <Input ... />
    </div>
) : !isBundle ? (
    <span className="text-muted-foreground">
      @ {formatAmount(lineItem.unitPrice)}
    </span>
) : null}
```
to:
```tsx
{!isBundle && isEditable ? (
    <div className="flex items-center gap-1">
      <span className="text-xs text-muted-foreground">@</span>
      <Input ... />
    </div>
) : (
    <span className="text-muted-foreground">
      @ {isBundle ? formatAmount(getBundleUnitPrice(lineItem)) : formatAmount(lineItem.unitPrice)}
    </span>
)}
```

This changes the last two branches: instead of showing nothing for bundles (`null`), it now shows the bundle unit price formatted the same way as non-bundle items with the `@` prefix. Non-bundle read-only items continue to show their unitPrice as before.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npm run typecheck && npm run lint</automated>
  </verify>
  <done>Card list view shows bundle unit price with @ prefix (matching non-bundle format). Fixed bundles show unitPrice, component-based bundles show component sum. Both typecheck and lint pass.</done>
</task>

</tasks>

<verification>
1. `npm run typecheck` passes — no type errors from accessing bundleConfig.priceStrategy
2. `npm run lint` passes — no linting issues
3. `npm run build` passes — production build succeeds
</verification>

<success_criteria>
- Fixed-price bundles display their unitPrice in both table and card views
- Component-based bundles display the sum of (component.unitPrice * component.quantity) in both views
- Bundle unit prices are read-only (no Input field)
- Tax rate column continues to show dash for bundles
- TypeScript compiles, lint passes
</success_criteria>

<output>
After completion, create `.planning/quick/6-show-bundle-unit-price-in-quote-line-ite/6-SUMMARY.md`
</output>
