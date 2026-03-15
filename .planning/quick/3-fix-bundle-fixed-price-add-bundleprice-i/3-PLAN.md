---
phase: quick
plan: 3
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/lib/forms/schemas/item.schema.ts
  - trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
  - trade-flow-api/src/item/requests/create-item.request.ts
  - trade-flow-api/src/item/requests/update-item.request.ts
autonomous: true
requirements: [QUICK-3]

must_haves:
  truths:
    - "When priceStrategy is 'fixed', user sees a bundle price input field"
    - "When priceStrategy is 'component_based', no bundle price field is shown"
    - "Form cannot be submitted with fixed strategy and empty/missing bundle price"
    - "Backend rejects null bundlePrice when priceStrategy is FIXED"
    - "Editing an existing fixed-price bundle pre-populates the bundlePrice field"
  artifacts:
    - path: "trade-flow-ui/src/lib/forms/schemas/item.schema.ts"
      provides: "bundlePrice field with conditional validation in bundleItemFormSchema"
      contains: "bundlePrice"
    - path: "trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx"
      provides: "Conditional bundlePrice input field and currency conversion"
      contains: "bundlePrice"
    - path: "trade-flow-api/src/item/requests/create-item.request.ts"
      provides: "Non-null enforcement for bundlePrice when FIXED"
      contains: "IsDefined"
    - path: "trade-flow-api/src/item/requests/update-item.request.ts"
      provides: "Non-null enforcement for bundlePrice when FIXED"
      contains: "IsDefined"
  key_links:
    - from: "BundleItemForm.tsx"
      to: "item.schema.ts"
      via: "bundlePrice field in schema drives form validation"
      pattern: "bundlePrice"
    - from: "BundleItemForm.tsx"
      to: "bundleConfig.bundlePrice"
      via: "fromInput converts dollars to cents on submit"
      pattern: "fromInput.*bundlePrice"
---

<objective>
Fix bundle fixed-price bug: add bundlePrice input field to the bundle item form, add conditional validation in the Valibot schema, and enforce non-null bundlePrice in the backend when priceStrategy is FIXED.

Purpose: Without this fix, users cannot set a fixed price for bundles, causing a 422 error at quote creation time.
Output: Working bundle price input that appears conditionally and validates correctly end-to-end.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/src/lib/forms/schemas/item.schema.ts
@trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx
@trade-flow-ui/src/features/items/components/forms/shared/useItemForm.ts
@trade-flow-ui/src/features/items/components/forms/MaterialItemForm.tsx
@trade-flow-ui/src/hooks/useCurrencyConversion.ts
@trade-flow-api/src/item/requests/create-item.request.ts
@trade-flow-api/src/item/requests/update-item.request.ts

<interfaces>
<!-- Existing patterns the executor needs -->

From useCurrencyConversion.ts:
```typescript
// toInput converts cents (number | null | undefined) to form string
toInput: (cents: number | null | undefined): string
// fromInput converts form string to cents (number | undefined)
fromInput: (value: string): number | undefined
```

From item.schema.ts (existing priceCheck pattern):
```typescript
const priceCheck = (value: string): boolean => {
  if (!value) return false;
  const parsed = Number.parseFloat(value);
  return !Number.isNaN(parsed) && parsed >= 0;
};
```

From MaterialItemForm.tsx (existing price field pattern):
```tsx
<FormField<ItemFormValues>
  name="defaultPrice"
  label="Price"
  required
  type="number"
  min="0"
  step="0.01"
  placeholder="0.00"
/>
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add bundlePrice to schema and form UI</name>
  <files>trade-flow-ui/src/lib/forms/schemas/item.schema.ts, trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx</files>
  <action>
**Schema (item.schema.ts):**
Add `bundlePrice` field to the `bundleItemFormSchema` object. Use `v.optional(v.string())` as the base type. Then add a `v.forward` check at the end of the pipe (after the existing component checks) that validates: when `priceStrategy === "fixed"`, `bundlePrice` must pass `priceCheck` (reuse the existing `priceCheck` function). Forward the error to `["bundlePrice"]`. Error message: "Bundle price is required for fixed pricing".

**Form (BundleItemForm.tsx):**
1. Import `useCurrencyConversion` from `@/hooks/useCurrencyConversion` and `useWatch` is already imported.
2. In the component, call `const { toInput, fromInput } = useCurrencyConversion();`
3. Add `bundlePrice: toInput(item?.bundleConfig?.bundlePrice)` to `defaultValues`.
4. Add a `useWatch` for `priceStrategy`: `const priceStrategy = useWatch({ control: form.control, name: "priceStrategy" });`
5. After the pricing strategy RadioGroup div (line 149), add a conditional block:
   ```tsx
   {priceStrategy === "fixed" && (
     <FormField<BundleItemFormValues>
       name="bundlePrice"
       label="Bundle Price"
       required
       type="number"
       min="0"
       step="0.01"
       placeholder="0.00"
     />
   )}
   ```
6. In `handleSubmit`, update the bundleConfig assignment to use the form value with currency conversion:
   ```typescript
   itemData.bundleConfig = {
     priceStrategy: values.priceStrategy,
     bundlePrice: values.priceStrategy === "fixed"
       ? (fromInput(values.bundlePrice ?? "") ?? null)
       : null,
     components: values.components,
   };
   ```
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npm run typecheck && npm run lint</automated>
  </verify>
  <done>
    - bundleItemFormSchema includes bundlePrice with conditional validation
    - BundleItemForm shows price input when priceStrategy is "fixed"
    - BundleItemForm hides price input when priceStrategy is "component_based"
    - Form converts between display dollars and API cents using toInput/fromInput
    - Editing existing fixed-price bundle pre-populates the price field
    - TypeScript and lint pass
  </done>
</task>

<task type="auto">
  <name>Task 2: Enforce non-null bundlePrice in backend validation</name>
  <files>trade-flow-api/src/item/requests/create-item.request.ts, trade-flow-api/src/item/requests/update-item.request.ts</files>
  <action>
In both `CreateBundleConfigRequest` and `UpdateBundleConfigRequest`:

1. Import `IsDefined` from `class-validator` (add to existing import).
2. On the `bundlePrice` property, add `@IsDefined()` decorator AFTER the `@ValidateIf` decorator and BEFORE `@IsNumber()`. This ensures that when priceStrategy is FIXED, bundlePrice cannot be null/undefined.
3. Change the type from `number | null` to `number` since the `@ValidateIf` means validation only runs when FIXED, and when FIXED it must be a defined number.

The decorator stack for bundlePrice should be:
```typescript
@ValidateIf((o) => o.priceStrategy === PriceStrategy.FIXED)
@IsDefined()
@IsNumber()
@Min(0)
bundlePrice: number | null;
```

Keep the type as `number | null` to allow null when component_based (ValidateIf skips validation in that case).
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-api && npm run validate</automated>
  </verify>
  <done>
    - CreateBundleConfigRequest rejects null bundlePrice when priceStrategy is FIXED
    - UpdateBundleConfigRequest rejects null bundlePrice when priceStrategy is FIXED
    - Both still allow null bundlePrice when priceStrategy is COMPONENT_BASED
    - TypeScript and lint pass
  </done>
</task>

</tasks>

<verification>
1. `cd trade-flow-ui && npm run typecheck && npm run lint` -- passes
2. `cd trade-flow-api && npm run validate` -- passes
3. Manual: Create a bundle item, select "Fixed price" strategy, confirm price input appears
4. Manual: Enter a price, submit, confirm no 422 error when creating a quote with the bundle
</verification>

<success_criteria>
- Bundle item form shows a price input field when "Fixed price" strategy is selected
- Price input is hidden when "Component-based" strategy is selected
- Form validation prevents submission without a price when strategy is fixed
- Backend rejects null bundlePrice for FIXED strategy
- Existing fixed-price bundles show their price when editing
- Both UI and API pass typecheck and lint
</success_criteria>

<output>
After completion, create `.planning/quick/3-fix-bundle-fixed-price-add-bundleprice-i/3-SUMMARY.md`
</output>
