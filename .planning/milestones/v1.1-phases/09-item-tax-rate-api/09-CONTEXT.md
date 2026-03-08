# Phase 9: Item Tax Rate API - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Items store and validate a tax rate ID reference instead of a numeric tax rate value. All downstream consumers (quotes, onboarding defaults) work correctly with the new field. UI changes are Phase 10.

</domain>

<decisions>
## Implementation Decisions

### Tax rate requirement
- Tax rate ID (`taxRateId`) is **required** for non-bundle items (material, labour, fee)
- Bundles keep `taxRateId: null` -- tax is computed from component items at quote time
- Validation must check that the referenced tax rate exists, belongs to the same business, and is active
- Wrong-business tax rate IDs are treated as an **authorization failure** (not a validation error) -- consistent with how the app handles other cross-business access
- Only active tax rates can be assigned to items; archived rates are blocked for new assignments
- If a tax rate is archived after an item references it, the item **keeps its reference** with no impact -- existing links are preserved (aligns with v2 requirement TAXM-02)

### Default item linkage (onboarding)
- Default items reference the business's default tax rate (the one marked `isDefault: true`)
- **Reorder onboarding calls:** move `defaultTaxRatesCreator` before `defaultItemsCreator` in `BusinessCreator` so tax rates exist when items are created
- Default tax rate ID is **passed as a parameter** to `DefaultBusinessItemsCreatorService` -- explicit dependency, easy to test
- `DefaultTaxRatesCreatorService` returns the created tax rates (or at minimum the default tax rate ID) so `BusinessCreator` can pass it downstream

### Quote resolution
- Quote factories **look up the tax rate by ID** at quote creation time to resolve the percentage
- Quote line items store a **snapshot of the resolved percentage** (e.g., `taxRate: 20`) -- matches current `taxRate: number` field on quote line items
- Archived tax rates can still be resolved for quotes -- the data still exists, just can't be assigned to new items
- Quotes stay accurate even if tax rates are later modified or archived

### Field naming
- Entity field: `taxRateId` (ObjectId) -- replaces `defaultTaxRate` (number)
- DTO field: `taxRateId` (string) -- replaces `defaultTaxRate` (number)
- API request field: `taxRateId` -- same name in create and update requests
- API response field: `taxRateId` -- flat field, not nested (UI resolves details client-side)
- Consistent naming across entity, DTO, request, and response layers

### Claude's Discretion
- Whether quote factories inject `TaxRateRetrieverService` directly or use a resolution helper service
- Internal implementation details of the tax rate validation in item creator/updater
- Test structure and mocking approach

</decisions>

<specifics>
## Specific Ideas

- Authorization policy pattern for cross-business tax rate access (not generic "not found" error)
- Follow the schedule-to-visit-type reference pattern established in v1.0

</specifics>

<code_context>
## Existing Code Insights

### Reusable Assets
- `TaxRateRetrieverService`: Already exists for fetching tax rates by ID -- can be used by quote factories
- `ItemPolicy`: Existing authorization policy for items -- pattern to follow for tax rate cross-business checks
- `AuthorizedCreatorFactory`: Existing pattern for authorized creation with policy checks
- `TaxRateEntity.isDefault`: Field already exists to identify the default tax rate

### Established Patterns
- Controller -> Service -> Repository layering (strict NestJS pattern)
- `InvalidRequestError` / `ForbiddenError` for validation vs authorization failures
- `IBaseResourceDto` pattern for DTOs with id and businessId
- Entity uses `ObjectId`, DTO uses `string` for IDs (repository handles conversion)
- `findByIdOrFail` pattern on retrievers for required lookups

### Integration Points
- `BusinessCreator.create()` (line 47-48): Reorder `defaultTaxRatesCreator` before `defaultItemsCreator`
- `DefaultBusinessItemsCreatorService`: Needs `taxRateId` parameter, replace hardcoded `defaultTaxRate: 0`
- `QuoteStandardLineItemFactory.create()` (line 27): `item.defaultTaxRate ?? 0` -> resolve from tax rate ID
- `QuoteBundleLineItemFactory.buildComponentLineItem()` (line 102): Same `defaultTaxRate` -> tax rate ID resolution
- `ItemCreatorService.validateCommonFields()`: Replace numeric tax rate validation with tax rate ID validation
- `ItemRepository`: Entity mapping needs `defaultTaxRate` -> `taxRateId` field change
- `item.module.ts`: May need to import `TaxRateModule` for cross-module validation

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 09-item-tax-rate-api*
*Context gathered: 2026-03-07*
