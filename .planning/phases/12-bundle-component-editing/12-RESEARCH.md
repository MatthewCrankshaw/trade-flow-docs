# Phase 12: Bundle Component Editing - Research

**Researched:** 2026-03-08
**Domain:** Bundle component CRUD (API update endpoint + UI edit form + read-only display enhancements)
**Confidence:** HIGH

## Summary

Phase 12 requires two complementary changes: (1) extending the API update path to accept a `components` array inside `bundleConfig`, and (2) enhancing the UI's `BundleComponentsList` display format and ensuring the `BundleItemForm` edit flow sends components through to the API. The expandable component display in ItemsTable/ItemsCardList already exists from Phase 11 but needs visual refinement per user decisions (two-line format with type badges).

The API gap is precise and well-scoped: `UpdateBundleConfigRequest` currently accepts only `priceStrategy` and `bundlePrice` -- it must also accept a `components` array. The `mergeBundleConfig` utility must merge the components into the existing config. The `ItemUpdaterService` must validate component changes using the same rules as `ItemCreatorService` (active, non-bundle, 1-100 count, quantity > 0).

The UI gap is equally clear: `UpdateItemRequest` (frontend type) excludes `components` from `bundleConfig`. The `ItemFormDialog.handleSubmit` already passes `bundleConfig` through for both create and update. Once the types are aligned, the existing `BundleItemForm` component management (add/update/remove via `useWatch` + `setValue`) will work for edits because it already reads `item?.bundleConfig?.components` as defaultValues.

**Primary recommendation:** Start with the API changes (UpdateBundleConfigRequest + merge utility + updater validation), then update the frontend types, and finally enhance the component display format.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Two-line format per component: item name + colour-coded type badge on first line, quantity x unit on second line
- Type badges use same colour coding as items list page (Material/Labour/Fee each get their own colour)
- Header shows component count: "Components (3)" -- no cost/price shown
- No cost shown per row or in header -- cost is a quote concern, not bundle definition
- Bundle rows on items list page (table and card) are expandable -- collapsed by default
- Expanded view uses same two-line format: name + type badge, qty x unit
- On mobile card view, expandable section shows "Components (3)" that reveals the list
- No confirmation on component removal -- nothing saved until form submitted
- Quantity editing via direct input field (same as create form)
- Block save with validation error when zero components: "At least one component is required"
- No change tracking or visual diff -- standard form behavior
- Empty state: "No components added yet" with SearchableItemPicker add button below

### Claude's Discretion
- Exact chevron/expand icon styling for collapsible rows
- Indentation and spacing for nested component rows in table
- How the card view expandable section animates
- API update endpoint changes needed to support component array updates

### Deferred Ideas (OUT OF SCOPE)
None
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| BNDL-02 | User can edit bundle components on an existing bundle (add/remove items, change quantities) | API: extend UpdateBundleConfigRequest with components array, update mergeBundleConfig, add validation in ItemUpdaterService. UI: extend UpdateItemRequest type to include components in bundleConfig. BundleItemForm already supports add/remove/update via React Hook Form setValue. |
| BNDL-04 | User can view bundle components in a structured list showing item name, quantity, and unit | ItemsTable and ItemsCardList already have expandable bundle rows. Enhance display to two-line format with type badges per user decisions. BundleComponentsList in edit form needs same two-line format enhancement. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Project standard |
| class-validator | latest | Request validation decorators | Project standard for API DTOs |
| class-transformer | latest | Request body transformation | Project standard for nested validation |
| React Hook Form | latest | Form state management | Project standard, already used in BundleItemForm |
| Valibot | latest | Schema validation | Project standard for frontend form schemas |
| RTK Query | latest | API data fetching | Project standard, itemApi already has updateItem mutation |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| shadcn/ui Badge | - | Type badges for components | Display component type with colour coding |
| Lucide React | - | Chevron icons for expand/collapse | Already imported in ItemsTable/ItemsCardList |

## Architecture Patterns

### API Update Flow (Existing)
```
Controller.update()
  -> itemRetriever.findByIdOrFail()        // fetch existing
  -> mergeExistingItemWithChanges()         // merge request into existing DTO
  -> itemUpdater.update()                   // validate + persist
     -> itemRepository.update()            // MongoDB $set with full bundleConfig
```

The repository's `update()` method already `$set`s the entire `bundleConfig` object including `components`. No repository changes needed -- the merge utility and updater service are the only API changes.

### Frontend Edit Flow (Existing)
```
ItemFormDialog (edit mode)
  -> BundleItemForm (receives item with bundleConfig.components as defaultValues)
     -> useForm({ defaultValues: { components: item.bundleConfig.components } })
     -> useWatch({ name: "components" })
     -> handleAddComponent / handleUpdateComponent / handleRemoveComponent
        -> form.setValue("components", [...], { shouldValidate: true })
     -> handleSubmit
        -> builds itemData with bundleConfig.components
        -> calls onSubmit(itemData)
  -> ItemFormDialog.handleSubmit
     -> builds UpdateItemRequest with bundleConfig
     -> calls updateItem mutation (PATCH)
```

The form already populates components from `item?.bundleConfig?.components` and builds the full `bundleConfig` in `handleSubmit`. The gap is only in the `UpdateItemRequest` type and API endpoint.

### Pattern: Two-Line Component Display
```
[Item Name]  [Type Badge]
[quantity] x [unit]
```
This pattern applies to both the edit form (BundleComponentsList) and the read-only expanded view (ItemsTable/ItemsCardList). Extract a shared component or replicate the pattern in both locations.

### Type Badge Colour Mapping
Already established in ItemsTable/ItemsCardList:
```typescript
const typeColors: Record<ItemType, "default" | "secondary" | "destructive" | "outline"> = {
  material: "default",
  labour: "secondary",
  fee: "outline",
  bundle: "default",
};
```
These badge variants map to shadcn/ui Badge component variants.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Component validation (API) | Custom validation logic | Reuse ItemCreatorService validation patterns (active check, non-bundle check, count limits, quantity > 0) | Same business rules apply; extract to shared utility or duplicate carefully |
| Component form state | Custom state management | React Hook Form `useWatch` + `setValue` (already implemented) | Form state is already managed correctly in BundleItemForm |
| Expandable rows | Custom expand/collapse | Existing `useState<Set<string>>` pattern (already in ItemsTable/ItemsCardList) | Pattern already works, just enhance the expanded content |
| Request validation (API) | Manual field checks | class-validator decorators with `@ValidateNested({ each: true })` | CreateBundleConfigRequest already demonstrates this pattern |

## Common Pitfalls

### Pitfall 1: UpdateBundleConfigRequest Missing Components Validation
**What goes wrong:** Adding `components` array to `UpdateBundleConfigRequest` without proper `@ValidateNested({ each: true })` and `@Type(() => ...)` decorators means nested component objects won't be validated.
**Why it happens:** class-transformer requires explicit `@Type()` for nested class arrays.
**How to avoid:** Copy the pattern from `CreateBundleConfigRequest` exactly: `@IsArray() @ValidateNested({ each: true }) @Type(() => CreateBundleComponentRequest)`.
**Warning signs:** API accepts invalid component data without errors.

### Pitfall 2: mergeBundleConfig Drops Components
**What goes wrong:** The current `mergeBundleConfig` function only merges `priceStrategy` and `bundlePrice` -- it does NOT merge components. If components are sent in the update request but the merge function ignores them, they'll be silently dropped.
**Why it happens:** Components were intentionally excluded from the original update endpoint.
**How to avoid:** Explicitly add `components` to the merged config: `mergedConfig.components = newConfig.components ?? existingConfig.components`.
**Warning signs:** Components appear saved on the frontend but revert after page refresh.

### Pitfall 3: ItemUpdaterService Lacks Bundle Validation
**What goes wrong:** The `ItemUpdaterService.update()` currently only validates `taxRateId`. It does NOT validate bundle components (active status, non-bundle type, count limits). A user could add invalid components.
**Why it happens:** Components weren't editable when the updater was written.
**How to avoid:** Add bundle component validation to `ItemUpdaterService.update()` -- check if the item is a bundle and if components changed, then run the same validation as `ItemCreatorService.validateBundleItem()`.
**Warning signs:** Bundles with inactive or bundle-type components can be saved via edit.

### Pitfall 4: Frontend UpdateItemRequest Type Missing Components
**What goes wrong:** The frontend `UpdateItemRequest` type defines `bundleConfig` without `components`:
```typescript
bundleConfig?: { priceStrategy: BundlePriceStrategy; bundlePrice?: number | null; } | null;
```
TypeScript will prevent sending components even if the API accepts them.
**How to avoid:** Add `components` to the `bundleConfig` type in `UpdateItemRequest`.
**Warning signs:** TypeScript compilation errors when trying to include components in update payload.

### Pitfall 5: Components Array Must Be Complete Replacement
**What goes wrong:** Sending a partial components array thinking it will merge with existing components.
**Why it happens:** Misunderstanding the merge semantics.
**How to avoid:** The merge utility should treat components as a complete replacement when provided (not a merge). The frontend already sends the full components array from form state. Document this clearly: "components in update replaces all existing components."
**Warning signs:** Components disappear or duplicate after editing.

## Code Examples

### API: Extended UpdateBundleConfigRequest
```typescript
// Source: Modeled after CreateBundleConfigRequest in create-item.request.ts
export class UpdateBundleConfigRequest {
  @IsEnum(PriceStrategy)
  priceStrategy: PriceStrategy;

  @ValidateIf((o) => o.priceStrategy === PriceStrategy.FIXED)
  @IsNumber()
  @Min(0)
  bundlePrice: number | null;

  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => CreateBundleComponentRequest)
  components?: CreateBundleComponentRequest[];
}
```

### API: Updated mergeBundleConfig
```typescript
// Source: Extending merge-existing-item-with-changes.utility.ts
const mergeBundleConfig = (
  existingConfig: IBundleConfigDto | null,
  newConfig?: UpdateBundleConfigRequest,
): IBundleConfigDto | null => {
  if (newConfig) {
    if (existingConfig === null) {
      throw new InvalidRequestError(
        ErrorCodes.BUNDLE_CONFIG_NOT_ALLOWED,
        "Cannot update bundle configuration on a non-bundle item.",
      );
    }

    const mergedConfig: IBundleConfigDto = {
      ...existingConfig,
      priceStrategy: newConfig.priceStrategy,
      bundlePrice: mergeBundlePrice(existingConfig.bundlePrice, newConfig.bundlePrice),
      components: newConfig.components
        ? newConfig.components.map((c) => ({
            itemId: c.itemId,
            quantity: c.quantity,
            isOptional: c.isOptional,
          }))
        : existingConfig.components,
    };
    return mergedConfig;
  }

  return existingConfig;
};
```

### Frontend: Updated UpdateItemRequest Type
```typescript
// Source: Extending api.types.ts
export interface UpdateItemRequest {
  name?: string;
  description?: string | null;
  status?: ItemStatus;
  unit?: string;
  defaultPrice?: number;
  taxRateId?: string;
  bundleConfig?: {
    priceStrategy: BundlePriceStrategy;
    bundlePrice?: number | null;
    components?: BundleComponent[];
  } | null;
}
```

### UI: Two-Line Component Display Pattern
```typescript
// Shared display pattern for components (edit form + read-only views)
<div className="space-y-1">
  <div className="flex items-center gap-2">
    <span className="font-medium truncate">{componentName}</span>
    <Badge variant={typeColors[componentItem.type]} className="text-xs">
      {typeLabels[componentItem.type]}
    </Badge>
  </div>
  <div className="text-sm text-muted-foreground">
    {component.quantity} x {componentUnit}
  </div>
</div>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Bundle update without components | Bundle update WITH components | Phase 12 | Enables full bundle editing |
| Single-line component display | Two-line display with type badges | Phase 12 | Consistent with planned Phase 14 quote bundle lines |

## Open Questions

1. **Shared validation between Creator and Updater services**
   - What we know: Both need identical component validation (active, non-bundle, 1-100 count, quantity > 0)
   - What's unclear: Whether to extract to a shared utility or duplicate in ItemUpdaterService
   - Recommendation: Extract a `validateBundleComponents` shared utility from ItemCreatorService, or simply call the same validation methods. The ItemCreatorService has private methods that could be extracted. Planner should decide on approach -- extraction is cleaner but both work.

2. **Reuse of CreateBundleComponentRequest for update**
   - What we know: The component shape is identical for create and update (itemId, quantity, isOptional)
   - What's unclear: Whether to create a separate `UpdateBundleComponentRequest` class
   - Recommendation: Reuse `CreateBundleComponentRequest` directly. The validation rules are the same. Import it in the update request file.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (API), Vitest not configured (UI) |
| Config file | `trade-flow-api/jest.config.ts` |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern="item" --no-coverage` |
| Full suite command | `cd trade-flow-api && npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BNDL-02 | mergeBundleConfig handles components array | unit | `cd trade-flow-api && npx jest --testPathPattern="merge-existing-item" --no-coverage` | Exists but needs new test cases |
| BNDL-02 | UpdateBundleConfigRequest validates components | unit | `cd trade-flow-api && npx jest --testPathPattern="update-item.request" --no-coverage` | Needs new file |
| BNDL-02 | ItemUpdaterService validates bundle components on update | unit | `cd trade-flow-api && npx jest --testPathPattern="item-updater" --no-coverage` | Exists but needs new test cases |
| BNDL-04 | Component display shows name, type badge, qty x unit | manual-only | Visual verification in browser | N/A -- UI component, no test framework |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npm run test -- --testPathPattern="item" --no-coverage`
- **Per wave merge:** `cd trade-flow-api && npm run test`
- **Phase gate:** Full API test suite green + visual verification of component display

### Wave 0 Gaps
- [ ] `trade-flow-api/src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts` -- needs new test cases for component merging
- [ ] `trade-flow-api/src/item/test/services/item-updater.service.spec.ts` -- needs new test cases for bundle component validation

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection of all files listed in Code Examples section
- `trade-flow-api/src/item/requests/update-item.request.ts` -- current UpdateBundleConfigRequest (missing components)
- `trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts` -- current merge logic (drops components)
- `trade-flow-api/src/item/services/item-creator.service.ts` -- bundle validation logic to replicate
- `trade-flow-api/src/item/services/item-updater.service.ts` -- current update validation (only taxRateId)
- `trade-flow-api/src/item/repositories/item.repository.ts` -- confirms bundleConfig is fully $set including components
- `trade-flow-ui/src/types/api.types.ts` -- current UpdateItemRequest type (missing components)
- `trade-flow-ui/src/features/items/components/forms/BundleItemForm.tsx` -- form already manages components
- `trade-flow-ui/src/features/items/components/ItemsTable.tsx` -- existing expandable bundle display
- `trade-flow-ui/src/features/items/components/ItemsCardList.tsx` -- existing expandable card display

### Secondary (MEDIUM confidence)
- None needed -- all findings are from direct codebase inspection

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - direct codebase inspection, all libraries already in use
- Architecture: HIGH - traced full update flow from controller to repository
- Pitfalls: HIGH - identified from reading actual code gaps (merge utility, updater service, frontend types)

**Research date:** 2026-03-08
**Valid until:** 2026-04-08 (stable -- internal project, no external dependency changes)
