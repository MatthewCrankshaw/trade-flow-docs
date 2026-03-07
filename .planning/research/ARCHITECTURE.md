# Architecture: Item-Tax Rate Linkage

**Research Date:** 2026-03-07
**Domain:** Cross-module reference integration (item -> tax-rate)
**Confidence:** HIGH -- all patterns derived from existing codebase analysis

## Recommended Architecture

Replace the `defaultTaxRate: number` field on items with a `taxRateId: string` reference to the tax-rate collection. The API validates the reference exists on create/update, stores only the ID, and returns only the ID. The UI resolves tax rate details client-side using RTK Query cache (same pattern as schedule -> visit-type).

### Why This Approach

The codebase already has a proven cross-module reference pattern: **schedule stores `visitTypeId`, API validates it via `VisitTypeRetrieverService.findByIdOrFail()`, returns only the ID, and UI resolves display details client-side via `useGetVisitTypesQuery`**. Item-tax rate linkage should follow this exact pattern for consistency.

Embedding tax rate details in the item response was considered and rejected because:
1. It breaks the established response pattern (schedule does not embed visit-type details)
2. It creates a coupling where item responses change when tax rates are renamed
3. RTK Query already caches tax rates from the existing `useGetTaxRatesQuery` -- no extra request needed

## Component Boundaries

### Backend -- Modules Affected

```
┌─────────────────────────────────────────────┐
│ Item Module (MODIFIED)                       │
│                                              │
│  Entity:     defaultTaxRate: number | null   │
│              → taxRateId: ObjectId | null     │
│                                              │
│  DTO:        defaultTaxRate: number | null    │
│              → taxRateId: string | null       │
│                                              │
│  Request:    defaultTaxRate?: number          │
│              → taxRateId?: string             │
│                                              │
│  Response:   defaultTaxRate: number | null    │
│              → taxRateId: string | null       │
│                                              │
│  Service:    Validates taxRateId exists       │
│              via TaxRateRetrieverService      │
│                                              │
│  Module:     imports TaxRateModule            │
│                                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Tax Rate Module (UNCHANGED)                  │
│                                              │
│  Already exports TaxRateRetrieverService     │
│  Already has findByIdOrFail(authUser, id)    │
│  No changes needed.                          │
│                                              │
└─────────────────────────────────────────────┘
```

### Frontend -- Features Affected

```
┌─────────────────────────────────────────────┐
│ features/items/ (MODIFIED)                   │
│                                              │
│  api/itemApi.ts        -- No change needed   │
│                                              │
│  forms/shared/         -- MODIFIED           │
│    useItemForm.ts      -- taxRateId field    │
│                                              │
│  forms/                -- MODIFIED           │
│    MaterialItemForm    -- dropdown replaces  │
│    LabourItemForm      --   number input     │
│    FeeItemForm         --   for tax rate     │
│                                              │
│  components/           -- MODIFIED           │
│    ItemsTable.tsx      -- show tax rate name  │
│    ItemsCardList.tsx   -- show tax rate name  │
│                                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ features/tax-rates/ (UNCHANGED)              │
│                                              │
│  useGetTaxRatesQuery already exists          │
│  UI will reuse cached data                   │
│                                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ types/api.types.ts (MODIFIED)                │
│                                              │
│  Item interface:                             │
│    defaultTaxRate: number | null             │
│    → taxRateId: string | null               │
│                                              │
│  CreateItemRequest:                          │
│    defaultTaxRate?: number                   │
│    → taxRateId?: string                     │
│                                              │
│  UpdateItemRequest:                          │
│    defaultTaxRate?: number                   │
│    → taxRateId?: string                     │
│                                              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ lib/forms/schemas/item.schema.ts (MODIFIED)  │
│                                              │
│  defaultTaxRate string validation            │
│  → taxRateId picklist/string validation     │
│                                              │
└─────────────────────────────────────────────┘
```

## Data Flow

### Create Item (with tax rate)

```
UI: MaterialItemForm (or Fee/Labour)
  → User selects tax rate from dropdown
    (dropdown populated via useGetTaxRatesQuery -- already cached)
  → Form submits { ..., taxRateId: "abc123" }
  → RTK Query mutation POST /v1/business/:businessId/item
    → ItemController.create()
      → mapCreateItemRequestToDto() maps taxRateId to DTO
      → ItemCreatorService.create()
        → validateItem() calls TaxRateRetrieverService.findByIdOrFail()
          → If not found: throw InvalidRequestError(ITEM_TAX_RATE_NOT_FOUND)
          → If found: proceed
        → AuthorizedCreator.create() persists to MongoDB
      → mapItemToResponse() returns { taxRateId: "abc123" }
    → UI receives response
  → Item list shows tax rate name by looking up taxRateId
    in the tax rates RTK Query cache
```

### Edit Item (change tax rate)

```
UI: MaterialItemForm (edit mode)
  → Form pre-populates dropdown with item.taxRateId
  → User selects different tax rate
  → Form submits PATCH { taxRateId: "def456" }
    → ItemController.update()
      → mergeExistingItemWithChanges() merges taxRateId
      → ItemUpdaterService.update() (or ItemCreatorService revalidation)
        → Validates new taxRateId via TaxRateRetrieverService
      → Returns updated item with new taxRateId
```

### Display Item (resolve tax rate name)

```
UI: ItemsTable / ItemsCardList
  → useGetItemsQuery provides items (each has taxRateId)
  → useGetTaxRatesQuery provides tax rates (already cached)
  → Component builds taxRateMap: Map<string, TaxRate>
  → For each item: taxRateMap.get(item.taxRateId) → display name + rate
  → Fallback: if taxRateId not in map → show "Unknown" or "--"
```

This mirrors exactly how `ScheduleTable` resolves visit type names via `visitTypeMap: Map<string, VisitType>`.

## Patterns to Follow

### Pattern 1: Cross-Module Reference Validation (from ScheduleCreatorService)

The schedule module validates `visitTypeId` and `jobId` references in the creator service. Item should follow the same pattern for `taxRateId`.

```typescript
// In ItemCreatorService -- follow ScheduleCreatorService pattern
if (item.taxRateId) {
  try {
    await this.taxRateRetriever.findByIdOrFail(authUser, item.taxRateId);
  } catch (_error) {
    this.logger.warn("Tax rate not found for item creation", { taxRateId: item.taxRateId });
    throw new InvalidRequestError(
      ErrorCodes.ITEM_TAX_RATE_NOT_FOUND,
      `Tax rate with id ${item.taxRateId} not found`,
    );
  }
}
```

### Pattern 2: Module Import for Cross-Module Access (from ScheduleModule)

The schedule module imports `VisitTypeModule` and `JobModule` to access their retriever services. Item module should import `TaxRateModule`.

```typescript
// item.module.ts
@Module({
  imports: [CoreModule, forwardRef(() => UserModule), forwardRef(() => TaxRateModule)],
  // ...
})
export class ItemModule {}
```

Use `forwardRef` to be safe against circular dependency, consistent with existing pattern.

### Pattern 3: Client-Side Reference Resolution (from ScheduleTable)

Schedule components receive a `visitTypeMap` built from the visit types query. Item display components should build a `taxRateMap` the same way.

```typescript
// In parent component or hook
const { data: taxRates } = useGetTaxRatesQuery(businessId);
const taxRateMap = useMemo(() => {
  const map = new Map<string, TaxRate>();
  taxRates?.forEach((tr) => map.set(tr.id, tr));
  return map;
}, [taxRates]);
```

### Pattern 4: Form Dropdown for Reference Selection (from ScheduleFormDialog)

The schedule form uses `useGetVisitTypesQuery` to populate a visit type dropdown, filtering to active-only. Item forms should do the same for tax rates.

```typescript
const { data: taxRates } = useGetTaxRatesQuery(businessId);
const activeTaxRates = taxRates?.filter((tr) => tr.status === "active");
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Embedding Tax Rate Details in Item Response

Do not return `{ taxRate: { id, name, rate, rateType } }` in the item response. This breaks the established pattern where the API returns only IDs and the UI resolves details. It also means item responses would change when tax rates are renamed, breaking caching assumptions.

### Anti-Pattern 2: Populating/Joining at the Database Layer

Do not use MongoDB `$lookup` or manual population in the item repository to join tax rate data. The repository should stay focused on the item collection. Cross-module data is resolved either in the service layer (for validation) or in the UI (for display).

### Anti-Pattern 3: Validating Tax Rate in the Controller

Validation belongs in the service layer, not the controller. The controller maps requests to DTOs; the service validates business rules. This is the established pattern -- `ScheduleCreatorService` validates `visitTypeId`, not the controller.

### Anti-Pattern 4: Making taxRateId Required for All Item Types

Bundle items do not have `defaultTaxRate` today (it is null, and the service enforces this). The `taxRateId` should follow the same rule: null for bundles, required for material/labour/fee items. Do not change this constraint.

## Detailed Change Inventory

### API Changes (trade-flow-api)

| File | Change Type | What Changes |
|------|-------------|--------------|
| `item/entities/item.entity.ts` | MODIFY | `defaultTaxRate: number \| null` -> `taxRateId: ObjectId \| null` |
| `item/data-transfer-objects/item.dto.ts` | MODIFY | `defaultTaxRate: number \| null` -> `taxRateId: string \| null` |
| `item/requests/create-item.request.ts` | MODIFY | `defaultTaxRate?: number` -> `taxRateId?: string` with `@IsString` + `@IsNotEmpty` |
| `item/requests/update-item.request.ts` | MODIFY | `defaultTaxRate?: number` -> `taxRateId?: string` with `@IsOptional` + `@IsString` |
| `item/responses/item.response.ts` | MODIFY | `defaultTaxRate: number \| null` -> `taxRateId: string \| null` |
| `item/controllers/mappers/map-create-item-request-to-dto.utility.ts` | MODIFY | Map `taxRateId` instead of `defaultTaxRate` |
| `item/controllers/mappers/map-item-to-response.utility.ts` | MODIFY | Map `taxRateId` instead of `defaultTaxRate` |
| `item/controllers/mappers/merge-existing-item-with-changes.utility.ts` | MODIFY | Merge `taxRateId` instead of `defaultTaxRate`, remove `mergeDefaultTaxRate` function |
| `item/services/item-creator.service.ts` | MODIFY | Add `TaxRateRetrieverService` dependency, validate `taxRateId` exists |
| `item/services/item-updater.service.ts` | MODIFY | Add `TaxRateRetrieverService` dependency, validate `taxRateId` on update |
| `item/repositories/item.repository.ts` | MODIFY | `toDto`/`toEntity` map `taxRateId` (ObjectId <-> string) instead of `defaultTaxRate` |
| `item/item.module.ts` | MODIFY | Add `TaxRateModule` to imports |
| `core/errors/error-codes.enum.ts` | MODIFY | Add `ITEM_TAX_RATE_NOT_FOUND` error code |
| `core/errors/errors-map.constant.ts` | MODIFY | Add error mapping for `ITEM_TAX_RATE_NOT_FOUND` |

### UI Changes (trade-flow-ui)

| File | Change Type | What Changes |
|------|-------------|--------------|
| `types/api.types.ts` | MODIFY | `Item.defaultTaxRate` -> `Item.taxRateId`, same for request types |
| `lib/forms/schemas/item.schema.ts` | MODIFY | `defaultTaxRate` string validation -> `taxRateId` string validation |
| `features/items/components/forms/shared/useItemForm.ts` | MODIFY | Form handles `taxRateId` string instead of `defaultTaxRate` number |
| `features/items/components/forms/MaterialItemForm.tsx` | MODIFY | Replace tax number input with tax rate Select dropdown |
| `features/items/components/forms/LabourItemForm.tsx` | MODIFY | Replace tax number input with tax rate Select dropdown |
| `features/items/components/forms/FeeItemForm.tsx` | MODIFY | Replace tax number input with tax rate Select dropdown |
| `features/items/components/ItemsTable.tsx` | MODIFY | Show resolved tax rate name instead of number |
| `features/items/components/ItemsCardList.tsx` | MODIFY | Show resolved tax rate name instead of number |

### Database Migration

Existing items in MongoDB have `defaultTaxRate: number`. A migration script must:

1. For each item with `defaultTaxRate` not null, find or create a matching tax rate in the business's tax rates
2. Set `taxRateId` to the matched/created tax rate's `_id`
3. Unset the `defaultTaxRate` field

This can be a one-time migration script run manually or as a NestJS migration module task.

## Suggested Build Order

### Phase 1: API Changes

1. **Error codes** -- Add `ITEM_TAX_RATE_NOT_FOUND` to error codes enum and error map
2. **Entity + DTO + Response** -- Change `defaultTaxRate` to `taxRateId` across entity, DTO, and response interfaces
3. **Request classes** -- Change `defaultTaxRate` to `taxRateId` with `@IsString` / `@IsNotEmpty` validation
4. **Repository** -- Update `toDto` / `toEntity` mapping (ObjectId <-> string)
5. **Mapper utilities** -- Update `mapCreateItemRequestToDto`, `mapItemToResponse`, `mergeExistingItemWithChanges`
6. **Module** -- Import `TaxRateModule` into `ItemModule`
7. **Services** -- Inject `TaxRateRetrieverService`, add validation to `ItemCreatorService` and `ItemUpdaterService`
8. **Tests** -- Update existing tests, add cross-module validation tests
9. **Migration** -- Script to convert existing `defaultTaxRate` values to `taxRateId` references

### Phase 2: UI Changes

1. **Types** -- Update `Item`, `CreateItemRequest`, `UpdateItemRequest` in `api.types.ts`
2. **Form schema** -- Update `item.schema.ts` to validate `taxRateId` as a string
3. **Form hook** -- Update `useItemForm.ts` to handle `taxRateId`
4. **Item forms** -- Replace number input with tax rate Select dropdown in Material/Labour/Fee forms
5. **Display components** -- Update `ItemsTable` and `ItemsCardList` to resolve tax rate names from cache

### Why This Order

- API first because the UI depends on the API contract
- Error codes first within API because services reference them
- Entity/DTO/Response before services because services operate on DTOs
- Module import before services because DI requires the module import
- Migration after services because the migration needs the new schema to be in place
- UI types first within UI because all components depend on the type definitions
- Forms before display because create/edit is the primary interaction

## Scalability Considerations

| Concern | Current Scale | At Scale |
|---------|---------------|----------|
| Tax rate lookup on item create | One `findByIdOrFail` per create -- negligible | Still one lookup per create -- no issue |
| Client-side tax rate resolution | `useGetTaxRatesQuery` fetches all business tax rates (typically 2-5) | Even at 50 tax rates, this is a trivial payload |
| Orphaned references | If a tax rate is deleted, items still reference its ID | Tax rates use soft-delete (status: inactive), so this is not a concern today. If hard delete is added later, a cascade check is needed. |

---
*Research completed: 2026-03-07*
