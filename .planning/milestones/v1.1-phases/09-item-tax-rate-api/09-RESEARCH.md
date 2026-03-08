# Phase 9: Item Tax Rate API - Research

**Researched:** 2026-03-08
**Domain:** NestJS API -- replacing numeric tax rate with tax rate ID reference on items
**Confidence:** HIGH

## Summary

This phase replaces the numeric `defaultTaxRate` field on items with a `taxRateId` ObjectId reference to the tax rates collection. The change touches six areas: the item entity/DTO/repository layer, the item creator and updater services (validation), the request/response mapping layer, the onboarding flow (reordering default creators and passing the tax rate ID), and the quote line item factories (resolving percentage from tax rate ID at quote creation time).

The codebase already has all the building blocks needed. The `TaxRateRetrieverService` exists with `findByIdOrFail` and authorization checks. The schedule-to-visit-type pattern provides the exact reference validation pattern to follow. The `DefaultTaxRatesCreatorService` already creates tax rates with `isDefault: true` but currently returns `void` -- it needs to return the default tax rate ID. The quote factories currently read `item.defaultTaxRate ?? 0` and need to resolve from the tax rate ID instead.

**Primary recommendation:** Follow the schedule-to-visit-type reference pattern exactly -- inject `TaxRateRetrieverService` into item creator/updater, use `findByIdOrFail(authUser, taxRateId)` for validation (which handles both existence and cross-business authorization via `TaxRatePolicy`/`AccessController`), and add active-status checking. For quotes, inject `TaxRateRetrieverService` into the quote factories to resolve the percentage at creation time.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Tax rate ID (`taxRateId`) is **required** for non-bundle items (material, labour, fee)
- Bundles keep `taxRateId: null` -- tax is computed from component items at quote time
- Validation must check that the referenced tax rate exists, belongs to the same business, and is active
- Wrong-business tax rate IDs are treated as an **authorization failure** (not a validation error) -- consistent with how the app handles other cross-business access
- Only active tax rates can be assigned to items; archived rates are blocked for new assignments
- If a tax rate is archived after an item references it, the item keeps its reference with no impact
- Default items reference the business's default tax rate (the one marked `isDefault: true`)
- Reorder onboarding calls: move `defaultTaxRatesCreator` before `defaultItemsCreator` in `BusinessCreator`
- Default tax rate ID is passed as a parameter to `DefaultBusinessItemsCreatorService` -- explicit dependency
- `DefaultTaxRatesCreatorService` returns the created tax rates (or at minimum the default tax rate ID)
- Quote factories look up the tax rate by ID at quote creation time to resolve the percentage
- Quote line items store a snapshot of the resolved percentage (e.g., `taxRate: 20`)
- Archived tax rates can still be resolved for quotes
- Entity field: `taxRateId` (ObjectId) -- replaces `defaultTaxRate` (number)
- DTO field: `taxRateId` (string) -- replaces `defaultTaxRate` (number)
- API request field: `taxRateId` -- replaces `defaultTaxRate`
- API response field: `taxRateId` -- flat field, not nested

### Claude's Discretion
- Whether quote factories inject `TaxRateRetrieverService` directly or use a resolution helper service
- Internal implementation details of the tax rate validation in item creator/updater
- Test structure and mocking approach

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| ITAX-01 | Item entity stores a tax rate ID reference instead of a numeric tax rate value | Entity/DTO/Repository field change from `defaultTaxRate: number` to `taxRateId: ObjectId/string` |
| ITAX-02 | Creating an item validates that the referenced tax rate exists and belongs to the business | Follow schedule-to-visit-type pattern: inject `TaxRateRetrieverService`, use `findByIdOrFail` which enforces `TaxRatePolicy` (cross-business = ForbiddenError), add active-status check |
| ITAX-03 | Updating an item validates the new tax rate ID if changed | Add validation to `ItemUpdaterService.update()` using same `TaxRateRetrieverService` pattern |
| ITAX-04 | Item API response includes the tax rate ID (UI resolves details client-side) | Update `IItemResponse`, `mapItemToResponse`, request classes, and request-to-DTO mappers |
| ITAX-05 | Default items created during onboarding reference the correct default tax rate | Reorder `BusinessCreator`, change `DefaultTaxRatesCreatorService` return type, pass tax rate ID to `DefaultBusinessItemsCreatorService` |
| QUOT-01 | Quote line item factory resolves tax rate percentage from the referenced tax rate ID | Inject `TaxRateRetrieverService` into `QuoteStandardLineItemFactory` and `QuoteBundleLineItemFactory`, resolve rate at creation time |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | Application framework | Project standard |
| TypeScript | 5.1.x | Language | Project standard |
| MongoDB (native driver) | 7.0 | Database | Project standard (not Mongoose) |
| Jest | latest | Testing | Project standard |
| class-validator | latest | Request validation | Project standard for DTO validation |
| class-transformer | latest | Request transformation | Project standard |

### Supporting
No new libraries needed. This phase uses only existing project infrastructure.

## Architecture Patterns

### Recommended Change Structure
```
src/
  item/
    entities/item.entity.ts          # defaultTaxRate -> taxRateId (ObjectId)
    data-transfer-objects/item.dto.ts # defaultTaxRate -> taxRateId (string)
    repositories/item.repository.ts  # Update toDto/toEntity mapping, update $set
    services/item-creator.service.ts # Add TaxRateRetrieverService, validate taxRateId
    services/item-updater.service.ts # Add TaxRateRetrieverService, validate taxRateId
    requests/create-item.request.ts  # defaultTaxRate -> taxRateId (string)
    requests/update-item.request.ts  # defaultTaxRate -> taxRateId (string)
    responses/item.response.ts       # defaultTaxRate -> taxRateId (string)
    controllers/mappers/             # Update all three mappers
    item.module.ts                   # Import TaxRateModule
  business/
    services/business-creator.service.ts              # Reorder calls
    services/default-tax-rates-creator.service.ts     # Return default tax rate ID
    services/default-business-items-creator.service.ts # Accept and use taxRateId param
  quote/
    services/quote-standard-line-item-factory.service.ts # Resolve tax rate from ID
    services/quote-bundle-line-item-factory.service.ts   # Resolve tax rate from ID
    quote.module.ts                                      # Import TaxRateModule
```

### Pattern 1: Foreign Key Validation via Retriever Service (Schedule-to-VisitType Pattern)
**What:** Use the existing retriever service with `findByIdOrFail(authUser, id)` to validate that a referenced entity exists and the user has access.
**When to use:** Any time one entity references another by ID.
**Why it works:** The retriever already fetches the entity and runs it through `AccessControllerFactory` + policy, which throws `ForbiddenError` for cross-business access -- exactly what the CONTEXT.md requires.

**Example from ScheduleCreatorService (the established pattern):**
```typescript
// Source: trade-flow-api/src/schedule/services/schedule-creator.service.ts
if (schedule.visitTypeId) {
  try {
    await this.visitTypeRetriever.findByIdOrFail(authUser, schedule.visitTypeId);
  } catch (_error) {
    throw new InvalidRequestError(
      ErrorCodes.SCHEDULE_VISIT_TYPE_NOT_FOUND,
      `Visit type with id ${schedule.visitTypeId} not found`,
    );
  }
}
```

**For item-to-tax-rate, the pattern differs slightly:**
- Tax rate ID is **required** for non-bundle items (not optional like visitTypeId on schedules)
- Must also check that the tax rate status is `enabled` (active) -- `findByIdOrFail` only validates existence + business ownership
- Cross-business access naturally throws `ForbiddenError` from `TaxRatePolicy` via `AccessController`, which propagates up as an authorization failure (no need to catch and rethrow)

**Recommended implementation:**
```typescript
// In ItemCreatorService -- inject TaxRateRetrieverService
private async validateTaxRateReference(authUser: IUserDto, item: IItemDto): Promise<void> {
  if (item.type === ItemType.BUNDLE) {
    if (item.taxRateId !== null) {
      throw new InvalidRequestError(
        ErrorCodes.DEFAULT_TAX_RATE_NOT_ALLOWED,
        `Tax rate ID is not allowed for bundle items`,
      );
    }
    return;
  }

  if (item.taxRateId === null) {
    throw new InvalidRequestError(
      ErrorCodes.INVALID_ITEM_DEFAULT_TAX_RATE,
      `Tax rate ID is required for ${item.type} items`,
    );
  }

  // findByIdOrFail throws ResourceNotFoundError if not found,
  // ForbiddenError if wrong business (via TaxRatePolicy)
  const taxRate = await this.taxRateRetriever.findByIdOrFail(authUser, item.taxRateId);

  if (taxRate.status !== TaxRateStatus.ENABLED) {
    throw new InvalidRequestError(
      ErrorCodes.INVALID_ITEM_DEFAULT_TAX_RATE,
      `Tax rate must be active to assign to an item`,
    );
  }
}
```

### Pattern 2: Quote Tax Rate Resolution
**What:** At quote creation time, look up the tax rate by ID to get the percentage, then snapshot it on the line item.
**When to use:** When creating quote line items from items that reference tax rate IDs.

**Current code (to be replaced):**
```typescript
// Source: trade-flow-api/src/quote/services/quote-standard-line-item-factory.service.ts line 27
taxRate: item.defaultTaxRate ?? 0,
```

**Recommendation:** Inject `TaxRateRetrieverService` directly into both `QuoteStandardLineItemFactory` and `QuoteBundleLineItemFactory`. Both factories need to make their `create`/`buildComponentBlueprints` methods async (the bundle factory already is async). The resolution call uses `findByIdOrFail` on the repository directly (not the retriever service with auth), because the item's tax rate reference was validated at item creation time, and archived rates should still resolve.

**Alternative approach (Claude's discretion):** Rather than injecting the retriever into both factories, consider using `TaxRateRepository.findByIdOrFail()` directly in the factories since auth was already validated when the item was created/updated. This avoids redundant auth checks during quote creation. However, using `TaxRateRetrieverService` with the auth user is more consistent with existing patterns.

**Recommended approach:** Use `TaxRateRetrieverService` with authUser in both factories. The auth user is already available in the bundle factory (via `ICreateBundleLineItemsParams.authUser`). For the standard factory, add `authUser` as a parameter. This is consistent with the codebase pattern where all service-level lookups go through retriever services with auth.

### Pattern 3: Onboarding Call Reordering
**What:** Move `defaultTaxRatesCreator.createDefaultTaxRates()` before `defaultItemsCreator.createDefaultItems()` in `BusinessCreator.create()`.
**Current order (line 47-48 of business-creator.service.ts):**
```typescript
await this.defaultItemsCreator.createDefaultItems(userWithRole, createdBusiness);
await this.defaultTaxRatesCreator.createDefaultTaxRates(userWithRole, createdBusiness);
```
**New order:**
```typescript
const defaultTaxRateId = await this.defaultTaxRatesCreator.createDefaultTaxRates(userWithRole, createdBusiness);
await this.defaultItemsCreator.createDefaultItems(userWithRole, createdBusiness, defaultTaxRateId);
```

### Anti-Patterns to Avoid
- **Returning the full tax rate in item responses:** The decision is to return `taxRateId` only; UI resolves details client-side from RTK Query cache.
- **Catching ForbiddenError and converting to InvalidRequestError:** Wrong-business access should remain a `ForbiddenError` (authorization failure), not be masked as a validation error.
- **Skipping active-status check:** `findByIdOrFail` only checks existence and business ownership. The active/enabled status must be checked separately after retrieval.
- **Hardcoding default tax rate in items:** The `defaultTaxRate: 0` values in `DefaultBusinessItemsCreatorService` must be replaced with the actual default tax rate ID from the created tax rates.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Cross-business authorization | Custom businessId comparison | `TaxRateRetrieverService.findByIdOrFail` + `TaxRatePolicy`/`AccessController` | Already handles auth consistently; throws `ForbiddenError` |
| Entity existence check | Manual repository query + null check | `findByIdOrFail` pattern | Standard pattern, throws `ResourceNotFoundError` |
| Request validation | Manual field checking | `class-validator` decorators (`@IsString`, `@IsMongoId`) | Consistent with all other request classes |

## Common Pitfalls

### Pitfall 1: Forgetting to update the repository $set fields
**What goes wrong:** The `ItemRepository.update()` method has an explicit `$set` with `defaultTaxRate` on line 73. If you rename the entity field but forget the `$set`, updates silently fail to persist.
**How to avoid:** When renaming `defaultTaxRate` to `taxRateId` in the entity and `toEntity`/`toDto`, also update the `$set` in the `update` method.

### Pitfall 2: Bundle items sending taxRateId in requests
**What goes wrong:** `CreateItemRequest` uses `@ValidateIf((o) => o.type !== ItemType.BUNDLE)` for defaultTaxRate. The same pattern must be replicated for `taxRateId`. If omitted, bundles will require a tax rate ID.
**How to avoid:** Keep the same `@ValidateIf` conditional on the new `taxRateId` field.

### Pitfall 3: QuoteStandardLineItemFactory is currently synchronous
**What goes wrong:** `QuoteStandardLineItemFactory.create()` is currently a synchronous method. Adding a `TaxRateRetrieverService.findByIdOrFail()` call makes it async. Any caller must be updated to `await` the result.
**How to avoid:** Change the return type to `Promise<IQuoteLineItemDto>` and update `QuoteLineItemCreator` (the caller) to await it.

### Pitfall 4: Merge utility for updates
**What goes wrong:** `mergeExistingItemWithChanges` has a `mergeDefaultTaxRate` function that handles the `number | undefined` merge. This must be rewritten for `string | undefined` (the tax rate ID). The update request also needs to change from `@IsNumber` to `@IsString`.
**How to avoid:** Update `merge-existing-item-with-changes.utility.ts` to handle `taxRateId` as a string merge.

### Pitfall 5: DefaultTaxRatesCreatorService currently returns void
**What goes wrong:** `createDefaultTaxRates()` returns `Promise<void>`, but we need the default tax rate ID. The method creates four tax rates but discards the returned DTOs (each `taxRateCreator.create()` returns an `ITaxRateDto`).
**How to avoid:** Capture the return value of the Standard VAT creation (the one with `isDefault: true`) and return its ID.

### Pitfall 6: TaxRateStatus enum uses "enabled"/"disabled" not "active"/"archived"
**What goes wrong:** The CONTEXT.md refers to "active" and "archived" tax rates, but the actual enum is `TaxRateStatus.ENABLED` and `TaxRateStatus.DISABLED`. Using wrong enum values causes silent bugs.
**How to avoid:** Use `TaxRateStatus.ENABLED` for the active check: `taxRate.status !== TaxRateStatus.ENABLED`.

### Pitfall 7: Error codes may need new entries
**What goes wrong:** The existing error codes `DEFAULT_TAX_RATE_NOT_ALLOWED` (ITEM_9) and `INVALID_ITEM_DEFAULT_TAX_RATE` (ITEM_10) can be reused but may need updated messages. A new error code might be needed for "tax rate not found for item" vs the generic validation error.
**How to avoid:** Review whether existing error codes suffice or if a new one (e.g., `ITEM_TAX_RATE_NOT_FOUND`) should be added to `ErrorCodes` and `ERRORS_MAP`.

## Code Examples

### Entity Field Change
```typescript
// Source: trade-flow-api/src/item/entities/item.entity.ts
// BEFORE:
export interface IItemEntity extends IBaseEntity {
  // ...
  defaultTaxRate: number | null;
}

// AFTER:
export interface IItemEntity extends IBaseEntity {
  // ...
  taxRateId: ObjectId | null;
}
```

### DTO Field Change
```typescript
// Source: trade-flow-api/src/item/data-transfer-objects/item.dto.ts
// BEFORE:
export interface IItemDto extends IBaseResourceDto {
  // ...
  defaultTaxRate: number | null;
}

// AFTER:
export interface IItemDto extends IBaseResourceDto {
  // ...
  taxRateId: string | null;
}
```

### Repository Mapping Change
```typescript
// Source: trade-flow-api/src/item/repositories/item.repository.ts
// toDto: defaultTaxRate: item.defaultTaxRate -> taxRateId: item.taxRateId?.toString() ?? null
// toEntity: defaultTaxRate: dto.defaultTaxRate -> taxRateId: dto.taxRateId ? new ObjectId(dto.taxRateId) : null
// update $set: defaultTaxRate: entityToUpdate.defaultTaxRate -> taxRateId: entityToUpdate.taxRateId
```

### Request Class Change
```typescript
// Source: trade-flow-api/src/item/requests/create-item.request.ts
// BEFORE:
@ValidateIf((o) => o.type !== ItemType.BUNDLE)
@IsNumber()
defaultTaxRate?: number;

// AFTER:
@ValidateIf((o) => o.type !== ItemType.BUNDLE)
@IsString()
@IsNotEmpty()
taxRateId?: string;
```

### DefaultTaxRatesCreatorService Return Change
```typescript
// Source: trade-flow-api/src/business/services/default-tax-rates-creator.service.ts
// BEFORE: public async createDefaultTaxRates(...): Promise<void>
// AFTER:  public async createDefaultTaxRates(...): Promise<string>
// Capture the standard VAT creation result and return its ID:
const createdStandardVat = await this.taxRateCreator.create(authUser, standardVat);
// ...create others...
return createdStandardVat.id;
```

### BusinessCreator Reordering
```typescript
// Source: trade-flow-api/src/business/services/business-creator.service.ts
// BEFORE (lines 47-48):
await this.defaultItemsCreator.createDefaultItems(userWithRole, createdBusiness);
await this.defaultTaxRatesCreator.createDefaultTaxRates(userWithRole, createdBusiness);

// AFTER:
const defaultTaxRateId = await this.defaultTaxRatesCreator.createDefaultTaxRates(userWithRole, createdBusiness);
await this.defaultItemsCreator.createDefaultItems(userWithRole, createdBusiness, defaultTaxRateId);
```

### Quote Factory Resolution
```typescript
// Source: trade-flow-api/src/quote/services/quote-standard-line-item-factory.service.ts
// BEFORE: taxRate: item.defaultTaxRate ?? 0,
// AFTER (factory becomes async, injects TaxRateRetrieverService):
const taxRate = item.taxRateId
  ? await this.taxRateRetriever.findByIdOrFail(authUser, item.taxRateId)
  : null;
// ...
taxRate: taxRate?.rate ?? 0,
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `defaultTaxRate: number` on items | `taxRateId: ObjectId` reference to tax rates collection | Phase 9 (this phase) | Enables tax rate management without updating every item |

## Open Questions

1. **Error code reuse vs new codes**
   - What we know: `DEFAULT_TAX_RATE_NOT_ALLOWED` (ITEM_9) and `INVALID_ITEM_DEFAULT_TAX_RATE` (ITEM_10) exist.
   - What's unclear: Whether to reuse ITEM_10 for "tax rate not found" or add a new code like `ITEM_TAX_RATE_NOT_FOUND`.
   - Recommendation: Reuse existing codes with updated messages. Add a new code only if the distinction matters for the UI. For wrong-business access, `ForbiddenError` already has its own error handling path.

2. **QuoteStandardLineItemFactory async conversion impact**
   - What we know: The factory is called from `QuoteLineItemCreator`. The bundle factory is already async.
   - What's unclear: Whether there are additional callers that need updating.
   - Recommendation: Check `QuoteLineItemCreator` service for how it calls the standard factory and update to async.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (via NestJS Test utilities) |
| Config file | `jest` config in `package.json` |
| Quick run command | `npm run test -- --testPathPattern="item\|business\|quote" --no-coverage` |
| Full suite command | `npm run test` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ITAX-01 | Item entity stores taxRateId (ObjectId) instead of defaultTaxRate (number) | unit | `npm run test -- --testPathPattern="item.repository" --no-coverage` | No - Wave 0 |
| ITAX-02 | Creating item validates taxRateId exists, same business, active | unit | `npm run test -- --testPathPattern="item-creator" --no-coverage` | No - Wave 0 |
| ITAX-03 | Updating item validates new taxRateId if changed | unit | `npm run test -- --testPathPattern="item-updater" --no-coverage` | No - Wave 0 |
| ITAX-04 | Item API response includes taxRateId field | unit | `npm run test -- --testPathPattern="map-item-to-response\|map-create-item-request" --no-coverage` | No - Wave 0 |
| ITAX-05 | Default onboarding items reference correct default tax rate | unit | `npm run test -- --testPathPattern="business-creator\|default-business-items\|default-tax-rates" --no-coverage` | Partial - business-creator.service.spec.ts exists |
| QUOT-01 | Quote factory resolves tax rate percentage from taxRateId | unit | `npm run test -- --testPathPattern="quote-standard-line-item\|quote-bundle-line-item" --no-coverage` | No - Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern="item\|business\|quote" --no-coverage`
- **Per wave merge:** `npm run test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/item/test/services/item-creator.service.spec.ts` -- covers ITAX-02
- [ ] `src/item/test/services/item-updater.service.spec.ts` -- covers ITAX-03
- [ ] `src/item/test/controllers/mappers/map-create-item-request-to-dto.utility.spec.ts` -- covers ITAX-04
- [ ] `src/item/test/controllers/mappers/map-item-to-response.utility.spec.ts` -- covers ITAX-04
- [ ] `src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts` -- covers ITAX-03/ITAX-04
- [ ] `src/item/test/repositories/item.repository.spec.ts` -- covers ITAX-01
- [ ] `src/item/test/mocks/item-mock-generator.ts` -- shared mock fixtures
- [ ] `src/quote/test/services/quote-standard-line-item-factory.service.spec.ts` -- covers QUOT-01
- [ ] `src/quote/test/services/quote-bundle-line-item-factory.service.spec.ts` -- covers QUOT-01
- [ ] Update `src/business/test/services/business-creator.service.spec.ts` -- covers ITAX-05 (reordering, return value)

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection of all files listed in Architecture Patterns section
- `trade-flow-api/src/schedule/services/schedule-creator.service.ts` -- established reference validation pattern
- `trade-flow-api/src/schedule/test/services/schedule-creator.service.spec.ts` -- established test pattern for reference validation
- `trade-flow-api/src/tax-rate/services/tax-rate-retriever.service.ts` -- existing retriever with auth
- `trade-flow-api/src/tax-rate/enums/tax-rate-status.enum.ts` -- ENABLED/DISABLED (not active/archived)
- `trade-flow-api/.github/copilot-instructions.md` -- project coding standards

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - no new libraries, all existing project infrastructure
- Architecture: HIGH - established patterns exist in codebase (schedule-to-visit-type), all integration points inspected
- Pitfalls: HIGH - identified from direct code inspection of actual files that need changing

**Research date:** 2026-03-08
**Valid until:** 2026-04-08 (stable, internal project patterns)
