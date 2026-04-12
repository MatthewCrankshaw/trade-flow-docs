---
phase: 41-estimate-module-crud-backend
reviewed: 2026-04-12T14:30:00Z
depth: standard
files_reviewed: 53
files_reviewed_list:
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/core/errors/error-codes.enum.ts
  - trade-flow-api/src/core/errors/errors-map.constant.ts
  - trade-flow-api/src/document-token/controllers/public-quote.controller.ts
  - trade-flow-api/src/document-token/data-transfer-objects/document-token.dto.ts
  - trade-flow-api/src/document-token/document-token.module.ts
  - trade-flow-api/src/document-token/entities/document-token.entity.ts
  - trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
  - trade-flow-api/src/document-token/repositories/document-token.repository.ts
  - trade-flow-api/src/document-token/services/document-token-creator.service.ts
  - trade-flow-api/src/document-token/services/document-token-retriever.service.ts
  - trade-flow-api/src/document-token/services/document-token-revoker.service.ts
  - trade-flow-api/src/document-token/services/public-quote-retriever.service.ts
  - trade-flow-api/src/document-token/services/quote-response-handler.service.ts
  - trade-flow-api/src/estimate/controllers/estimate.controller.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-line-item.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-totals.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-price-range.dto.ts
  - trade-flow-api/src/estimate/entities/estimate.entity.ts
  - trade-flow-api/src/estimate/entities/estimate-line-item.entity.ts
  - trade-flow-api/src/estimate/enums/estimate-status.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-transitions.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/estimate/policies/estimate.policy.ts
  - trade-flow-api/src/estimate/policies/estimate-line-item.policy.ts
  - trade-flow-api/src/estimate/repositories/estimate.repository.ts
  - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
  - trade-flow-api/src/estimate/requests/create-estimate.request.ts
  - trade-flow-api/src/estimate/requests/update-estimate.request.ts
  - trade-flow-api/src/estimate/requests/list-estimates.request.ts
  - trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts
  - trade-flow-api/src/estimate/requests/update-estimate-line-item.request.ts
  - trade-flow-api/src/estimate/responses/estimate.responses.ts
  - trade-flow-api/src/estimate/services/estimate-creator.service.ts
  - trade-flow-api/src/estimate/services/estimate-retriever.service.ts
  - trade-flow-api/src/estimate/services/estimate-updater.service.ts
  - trade-flow-api/src/estimate/services/estimate-deleter.service.ts
  - trade-flow-api/src/estimate/services/estimate-number-generator.service.ts
  - trade-flow-api/src/estimate/services/estimate-totals-calculator.service.ts
  - trade-flow-api/src/estimate/services/estimate-transition.service.ts
  - trade-flow-api/src/estimate/services/estimate-standard-line-item-factory.service.ts
  - trade-flow-api/src/estimate/services/estimate-bundle-line-item-factory.service.ts
  - trade-flow-api/src/estimate/services/estimate-line-item-creator.service.ts
  - trade-flow-api/src/estimate/services/estimate-line-item-retriever.service.ts
  - trade-flow-api/src/item/interfaces/line-item-tax-input.interface.ts
  - trade-flow-api/src/item/item.module.ts
  - trade-flow-api/src/item/services/bundle-config-validator.service.ts
  - trade-flow-api/src/item/services/bundle-pricing-planner.service.ts
  - trade-flow-api/src/item/services/bundle-tax-rate-calculator.service.ts
  - trade-flow-api/src/quote/controllers/quote.controller.ts
  - trade-flow-api/src/quote/quote.module.ts
  - trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts
  - trade-flow-api/src/quote/services/quote-email-sender.service.ts
  - trade-flow-api/src/quote/services/quote-transition.service.ts
  - trade-flow-api/tsconfig.json
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 41: Code Review Report

**Reviewed:** 2026-04-12T14:30:00Z
**Depth:** standard
**Files Reviewed:** 53
**Status:** issues_found

## Summary

Phase 41 implements the Estimate Module CRUD backend, lifts bundle helpers to the shared `@item` module, and renames the quote-token module to document-token with a type discriminator. The overall architecture follows established NestJS patterns well: layered controller/service/repository separation, policy-based authorization, and consistent DTO usage.

Key concerns: (1) the `DocumentSessionAuthGuard` hardcodes `QuoteRepository` lookups for business name resolution, making it unable to handle estimate tokens when they are eventually served through public endpoints; (2) the `CreateEstimateLineItemRequest.itemId` uses `@IsString()` instead of `@IsMongoId()`, weakening input validation; (3) missing authorization check in the `EstimateDeleter`; (4) an unchecked `businessIds[0]` access pattern that could crash if the array is empty.

## Critical Issues

### CR-01: EstimateDeleter bypasses authorization policy check

**File:** `trade-flow-api/src/estimate/services/estimate-deleter.service.ts:17-25`
**Issue:** The `softDelete` method fetches the estimate and checks its status, but never verifies that the authenticated user has permission to delete it. Every other mutating service method (`EstimateUpdater.update`, `EstimateUpdater.addLineItem`, etc.) calls `accessController.canUpdate(authUser, current)` before proceeding. The deleter skips this, allowing any authenticated user to delete any estimate by ID (regardless of business ownership).
**Fix:**
```typescript
public async softDelete(authUser: IUserDto, id: string): Promise<IEstimateDto> {
  const current = await this.estimateRepository.findByIdOrFail(id);

  const accessController = this.accessControllerFactory.create<IEstimateDto>(this.estimatePolicy);
  accessController.canUpdate(authUser, current);

  if (current.status !== EstimateStatus.DRAFT) {
    throw new InvalidRequestError(ErrorCodes.ESTIMATE_NOT_EDITABLE, "Estimate can only be deleted in Draft status");
  }

  return this.estimateTransitionService.transition(authUser, id, EstimateStatus.DELETED);
}
```
Note: `EstimateTransitionService.transition` does check `canUpdate`, but the defense-in-depth principle requires the deleter itself to gate access before any further logic. A caller reading `softDelete` expects authorization to be handled at entry.

## Warnings

### WR-01: CreateEstimateLineItemRequest.itemId uses @IsString instead of @IsMongoId

**File:** `trade-flow-api/src/estimate/requests/create-estimate-line-item.request.ts:5-6`
**Issue:** The `itemId` field is validated with `@IsString()` rather than `@IsMongoId()`. This allows arbitrary strings (including malicious input) to reach the service layer where they will be passed to `new ObjectId(itemId)`. While `ObjectId` constructor throws on invalid hex strings, that exception is caught generically and returns a 500 error rather than a clean 400 validation error.
**Fix:**
```typescript
import { IsMongoId, IsNumber, IsOptional, IsString } from "class-validator";

export class CreateEstimateLineItemRequest {
  @IsMongoId()
  public itemId!: string;

  @IsNumber()
  public quantity!: number;

  @IsOptional()
  @IsString()
  public priceStrategy?: string;
}
```

### WR-02: Unchecked businessIds[0] access could crash on empty array

**File:** `trade-flow-api/src/estimate/controllers/estimate.controller.ts:120,143,165,177`
**Issue:** Multiple controller methods access `request.user.businessIds[0]` without verifying the array is non-empty. If a user exists without a business association (e.g., during onboarding), `businessIds[0]` returns `undefined`, which would propagate through the service layer and eventually cause a `new ObjectId(undefined)` crash in the repository.
**Fix:** Extract a helper or add a guard clause at the controller level:
```typescript
private getBusinessId(authUser: IUserDto): string {
  if (!authUser.businessIds || authUser.businessIds.length === 0) {
    throw createHttpError(new InvalidRequestError(
      ErrorCodes.BUSINESS_ACTION_NOT_ALLOWED,
      "User must belong to a business to manage estimates",
    ));
  }
  return authUser.businessIds[0];
}
```

### WR-03: DocumentSessionAuthGuard hardcodes QuoteRepository for business name resolution

**File:** `trade-flow-api/src/document-token/guards/document-session-auth.guard.ts:14,34`
**Issue:** When a token is expired or revoked, the guard fetches the document via `this.quoteRepository.findByIdOrFail(tokenDto.documentId)` to retrieve the business name for the error message. Since document tokens now support `"estimate"` type, this will fail (or return wrong data) for estimate tokens -- the documentId is an estimate ID, not a quote ID. The catch block swallows the error silently, so it degrades gracefully (empty business name), but this is a latent bug that should be addressed before estimate public endpoints are built.
**Fix:** Add document-type-aware resolution:
```typescript
let businessName = "";
try {
  if (tokenDto.documentType === "quote") {
    const quote = await this.quoteRepository.findByIdOrFail(tokenDto.documentId);
    const business = await this.businessRepository.findByIdOrFail(quote.businessId);
    businessName = business.name;
  }
  // Future: add estimate branch when public estimate endpoints exist
} catch {
  // Fall through with empty name if lookup fails
}
```

### WR-04: EstimateRepository.toEntity always generates new ObjectId, ignoring dto.id

**File:** `trade-flow-api/src/estimate/repositories/estimate.repository.ts:219`
**Issue:** The `toEntity` method creates `_id: new ObjectId()` regardless of whether `dto.id` is set. This means the `update` method (line 114) builds an entity with a brand new `_id`, then uses `dto.id` in the filter `{ _id: new ObjectId(dto.id) }`. While the update works because the filter uses the correct ID, the entity object passed to `$set` has a mismatched `_id`. The `$set` does not include `_id` so there is no data corruption, but the `toEntity` method is misleading -- it produces an entity whose `_id` does not match the document being updated.
**Fix:**
```typescript
public toEntity(dto: IEstimateDto): IEstimateEntity {
  const entity: IEstimateEntity = {
    _id: new ObjectId(dto.id),
    ...createAuditFields(),
    // ... rest of fields
  };
  return entity;
}
```

### WR-05: UpdateEstimateRequest allows customerId/jobId changes without re-validation

**File:** `trade-flow-api/src/estimate/services/estimate-updater.service.ts:54-65`
**Issue:** The `update` method spreads `body.customerId` and `body.jobId` into the DTO when provided, but does not validate that the new customer exists, is active, or that the new job exists and belongs to the same business. The `EstimateCreator.create` validates customer status and job existence, but the updater skips this. An attacker could set `customerId` to any valid MongoDB ObjectId (including a customer from another business).
**Fix:** Add validation for customerId and jobId changes, similar to `EstimateCreator.validateEstimate`:
```typescript
if (body.customerId !== undefined) {
  const customer = await this.customerRetriever.findByIdOrFail(authUser, body.customerId);
  if (customer.status !== CustomerStatus.ACTIVE) {
    throw new InvalidRequestError(ErrorCodes.CUSTOMER_INACTIVE, "Cannot assign inactive customer");
  }
}
if (body.jobId !== undefined) {
  await this.jobRetrieverService.findByIdOrFail(authUser, body.jobId);
}
```

## Info

### IN-01: ErrorCodes enum uses inconsistent naming convention for estimate codes

**File:** `trade-flow-api/src/core/errors/error-codes.enum.ts:67-76`
**Issue:** The estimate error codes use descriptive string values (e.g., `ESTIMATE_NOT_FOUND = "ESTIMATE_NOT_FOUND"`) while all other domain error codes use a prefix-number pattern (e.g., `QUOTE_JOB_NOT_FOUND = "QUOTE_0"`, `SCHEDULE_JOB_NOT_FOUND = "SCHEDULE_0"`). The same inconsistency exists for `DOCUMENT_TOKEN_*` codes on lines 78-81. This does not cause bugs but breaks the established pattern.
**Fix:** Consider renaming to use the `EST_N` pattern to match conventions: `ESTIMATE_NOT_FOUND = "EST_0"`, `ESTIMATE_INVALID_TRANSITION = "EST_1"`, etc.

### IN-02: EstimateLineItemStatus uses UPPER_CASE values instead of lowercase

**File:** `trade-flow-api/src/estimate/enums/estimate-line-item-status.enum.ts:1-6`
**Issue:** The enum values use uppercase strings (`"PENDING"`, `"APPROVED"`, etc.) while the project convention (documented in CLAUDE.md) specifies lowercase enum values (e.g., `EstimateStatus` uses `"draft"`, `"sent"`, etc.; `QuoteLineItemStatus` likely follows the same pattern).
**Fix:** Change to lowercase to match conventions:
```typescript
export enum EstimateLineItemStatus {
  PENDING = "pending",
  APPROVED = "approved",
  REJECTED = "rejected",
  DELETED = "deleted",
}
```

### IN-03: Estimate updateLineItem does not reload line items before recalculating totals

**File:** `trade-flow-api/src/estimate/services/estimate-updater.service.ts:136-138`
**Issue:** After updating a line item and re-fetching the estimate, `updateLineItem` calls `totalsCalculator.calculateTotals(updatedEstimate)` but the re-fetched estimate from `findByIdOrFail` returns with `lineItems: DtoCollection.empty()` (line 201 of repository). The totals calculator iterates over `estimate.lineItems` which will be empty, producing zero totals. Compare with the `reloadWithTotals` method used by `update()` which explicitly loads line items. The same pattern exists in `deleteLineItem` (line 168).
**Fix:** Use `reloadWithTotals` consistently:
```typescript
// In updateLineItem, replace lines 137-138 with:
return this.reloadWithTotals(authUser, await this.estimateRepository.findByIdOrFail(estimateId));

// In deleteLineItem, replace lines 168-169 with:
return this.reloadWithTotals(authUser, await this.estimateRepository.findByIdOrFail(estimateId));
```

---

_Reviewed: 2026-04-12T14:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
