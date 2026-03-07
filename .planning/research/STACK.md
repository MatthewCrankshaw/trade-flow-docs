# Technology Stack: Item Tax Rate Linkage

**Project:** Trade Flow v1.1 - Item Tax Rate Linkage
**Researched:** 2026-03-07

## Recommended Stack

### No New Libraries Required

**Confidence:** HIGH

This milestone is a data model refactor within the existing stack. Every tool needed is already in the codebase. No npm installs, no new dependencies, no new patterns to learn.

The change: replace `defaultTaxRate: number | null` with `taxRateId: string | null` (DTO layer) / `taxRateId: ObjectId | null` (entity layer) on items.

## Changes by Layer

### Entity Layer (MongoDB Document)

| Field | Current | New | Notes |
|-------|---------|-----|-------|
| `defaultTaxRate` | `number \| null` | REMOVE | Drop the numeric rate value |
| `taxRateId` | n/a | `ObjectId \| null` | ADD - reference to taxrates collection |

**Why ObjectId, not string:** The entity layer already uses `ObjectId` for all cross-document references (`businessId`, `jobId`, `visitTypeId`, bundle component `itemId`). This is the established codebase pattern. The repository `toDto()` converts to string; `toEntity()` converts back to `ObjectId`. No change to the pattern.

**Why nullable:** Bundle items derive tax from their components and must not have a direct tax rate. The existing `null` pattern for bundles carries over. Additionally, items could theoretically have no tax rate assigned yet (though business logic may require one for non-bundle items).

### DTO Layer

| Field | Current | New | Notes |
|-------|---------|-----|-------|
| `defaultTaxRate` | `number \| null` | REMOVE | |
| `taxRateId` | n/a | `string \| null` | ADD - string ID matching all other DTO references |

**Pattern precedent:** `IScheduleDto.visitTypeId: string | null` follows the exact same pattern -- an optional reference to another business-scoped entity, validated at the service layer via its retriever service.

### Request/Response Layer (API Contract)

| Layer | Current Field | New Field | Validation |
|-------|---------------|-----------|------------|
| `CreateItemRequest` | `defaultTaxRate?: number` | `taxRateId?: string` | `@IsString() @IsOptional() @IsMongoId()` |
| `UpdateItemRequest` | `defaultTaxRate?: number` | `taxRateId?: string` | `@IsString() @IsOptional() @IsMongoId()` |
| `IItemResponse` | `defaultTaxRate: number \| null` | `taxRateId: string \| null` | n/a (output only) |

**`@IsMongoId()` from class-validator:** Already available in the project. Validates that the string is a valid 24-character hex MongoDB ObjectId. This is used for structural validation only -- existence validation happens in the service layer.

### Service Layer (Cross-Module Validation)

**Confidence:** HIGH

The exact pattern to follow already exists: `ScheduleCreatorService` validates `visitTypeId` by calling `VisitTypeRetrieverService.findByIdOrFail()`. Apply the same approach:

```typescript
// In ItemCreatorService and ItemUpdaterService
constructor(
  // ... existing deps
  private readonly taxRateRetriever: TaxRateRetrieverService,
) {}

// Validation (same try/catch pattern as schedule-creator.service.ts lines 39-49)
if (item.taxRateId) {
  try {
    await this.taxRateRetriever.findByIdOrFail(authUser, item.taxRateId);
  } catch (_error) {
    throw new InvalidRequestError(
      ErrorCodes.ITEM_TAX_RATE_NOT_FOUND,
      `Tax rate with id ${item.taxRateId} not found`,
    );
  }
}
```

**Why validate at service layer, not repository:**
- Service layer has access to `authUser` for policy checks (ensures the tax rate belongs to the same business)
- `TaxRateRetrieverService.findByIdOrFail()` already enforces access control via `TaxRatePolicy`
- Repository layer only handles entity/DTO conversion and DB operations (strict layering)

### Module Wiring

```typescript
// item.module.ts - add TaxRateModule import
@Module({
  imports: [CoreModule, forwardRef(() => UserModule), forwardRef(() => TaxRateModule)],
  // ... rest unchanged
})
```

**Why `forwardRef`:** Follows the existing pattern for all cross-module imports. `TaxRateModule` exports `TaxRateRetrieverService` already (confirmed in `tax-rate.module.ts` line 16).

### Frontend Types

| Type | Current Field | New Field |
|------|---------------|-----------|
| `Item` | `defaultTaxRate: number \| null` | `taxRateId: string \| null` |
| `CreateItemRequest` | `defaultTaxRate?: number` | `taxRateId?: string` |
| `UpdateItemRequest` | `defaultTaxRate?: number` | `taxRateId?: string` |

### Frontend Form Schema (Valibot)

Current `itemFormSchema` has `defaultTaxRate` as a string-validated number field. Replace with:

```typescript
taxRateId: v.pipe(
  v.string(),
  v.minLength(1, "Tax rate is required"),
),
```

The form will use a dropdown select (populated from `useGetTaxRatesQuery`) instead of a free-text number input.

### Frontend Resolution Strategy

**Do NOT use MongoDB `$lookup` or server-side populate.** Resolve tax rate details on the frontend.

**Why:**
- Tax rates are already fetched via `useGetTaxRatesQuery(businessId)` and cached by RTK Query
- The item list/detail just needs to display tax rate name/percentage alongside the item
- Client-side join: `taxRates.find(tr => tr.id === item.taxRateId)` -- trivial
- No new API endpoints needed
- Follows existing pattern: bundle component items are resolved client-side the same way

### Error Codes

Add one new error code to `ErrorCodes` enum:

```typescript
ITEM_TAX_RATE_NOT_FOUND = "ITEM_11",  // follows existing ITEM_0 through ITEM_10
```

### Migration

A data migration is needed to convert existing items from `defaultTaxRate: number` to `taxRateId: ObjectId`. The migration should:

1. For each item with a non-null `defaultTaxRate`, find the matching tax rate in the same business by rate value
2. Set `taxRateId` to the matched tax rate's `_id`
3. Unset the `defaultTaxRate` field
4. Items where no matching tax rate is found should have `taxRateId` set to `null` (log a warning)

Follow the existing migration pattern (`IMigration` interface with `up`/`down`/`isBootstrap`).

## What NOT to Add

| Library/Approach | Why Not |
|-----------------|---------|
| Mongoose populate / `$lookup` | No server-side joins needed. Frontend resolves from cached tax rates |
| New API endpoint for "items with tax rate details" | Over-engineering. Client-side join is trivial with RTK Query cache |
| Denormalized tax rate fields on item (name, rate) | Creates stale data problem when tax rate is updated |
| `class-validator` custom decorator for tax rate existence | Decorators can't do async DB lookups cleanly. Service-layer validation is the established pattern |
| Any new npm packages | Everything needed is already installed |

## Alternatives Considered

| Decision | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Reference type | `ObjectId` in entity, `string` in DTO | `string` everywhere | Breaks entity layer convention; all other refs use ObjectId |
| Validation location | Service layer (try/catch on retriever) | Custom class-validator decorator | Service layer has auth context; matches schedule pattern exactly |
| Tax rate resolution | Frontend join from RTK Query cache | `$lookup` aggregation | Adds backend complexity for no benefit; tax rates already cached |
| Nullable taxRateId | `null` for bundles, required for others | Always required | Bundles derive tax from components; forcing a reference is incorrect |
| Field naming | `taxRateId` | `defaultTaxRateId` | Shorter is clearer; the "default" prefix was about the value being a default price-level rate, but now it's a reference not a value |

## Touched Files Summary

### Backend (trade-flow-api)

| File | Change |
|------|--------|
| `src/item/entities/item.entity.ts` | Replace `defaultTaxRate: number \| null` with `taxRateId: ObjectId \| null` |
| `src/item/data-transfer-objects/item.dto.ts` | Replace `defaultTaxRate: number \| null` with `taxRateId: string \| null` |
| `src/item/requests/create-item.request.ts` | Replace `defaultTaxRate` with `taxRateId`, use `@IsMongoId()` |
| `src/item/requests/update-item.request.ts` | Replace `defaultTaxRate` with `taxRateId`, use `@IsMongoId()` |
| `src/item/responses/item.response.ts` | Replace `defaultTaxRate` with `taxRateId` |
| `src/item/repositories/item.repository.ts` | Update `toDto()` and `toEntity()` conversions, update `$set` in `update()` |
| `src/item/services/item-creator.service.ts` | Add `TaxRateRetrieverService` dep, add tax rate existence validation |
| `src/item/services/item-updater.service.ts` | Add `TaxRateRetrieverService` dep, add tax rate existence validation |
| `src/item/controllers/mappers/map-create-item-request-to-dto.utility.ts` | Map `taxRateId` instead of `defaultTaxRate` |
| `src/item/controllers/mappers/map-item-to-response.utility.ts` | Map `taxRateId` instead of `defaultTaxRate` |
| `src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts` | Merge `taxRateId` instead of `defaultTaxRate` |
| `src/item/item.module.ts` | Add `TaxRateModule` import |
| `src/core/errors/error-codes.enum.ts` | Add `ITEM_TAX_RATE_NOT_FOUND = "ITEM_11"` |
| `src/migration/migrations/` | New migration to convert existing data |

### Frontend (trade-flow-ui)

| File | Change |
|------|--------|
| `src/types/api.types.ts` | Update `Item`, `CreateItemRequest`, `UpdateItemRequest` types |
| `src/lib/forms/schemas/item.schema.ts` | Replace `defaultTaxRate` field with `taxRateId` dropdown select |
| `src/features/items/api/itemApi.ts` | No changes needed (endpoints unchanged) |
| Item form components | Replace tax rate number input with tax rate select dropdown |
| Item display components | Resolve and display tax rate name/percentage from cached tax rates |

## Sources

- Codebase analysis: direct file inspection (HIGH confidence -- verified against actual source)
- Pattern precedent: `ScheduleCreatorService` cross-module validation of `visitTypeId` via `VisitTypeRetrieverService`
- Pattern precedent: `IScheduleEntity.visitTypeId: ObjectId | null` reference pattern
- Pattern precedent: `TaxRateModule` already exports `TaxRateRetrieverService`

---
*Research completed: 2026-03-07*
