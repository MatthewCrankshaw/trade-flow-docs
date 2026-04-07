---
phase: 40-subscription-guard-onboarding-bypass
reviewed: 2026-04-07T18:44:34Z
depth: standard
files_reviewed: 3
files_reviewed_list:
  - trade-flow-api/src/user/controllers/user.controller.ts
  - trade-flow-api/src/business/controllers/business.controller.ts
  - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 40: Code Review Report

**Reviewed:** 2026-04-07T18:44:34Z
**Depth:** standard
**Files Reviewed:** 3
**Status:** issues_found

## Summary

Reviewed three files related to the SubscriptionGuard onboarding bypass feature. The controllers apply `@SkipSubscriptionCheck()` on the correct create/update endpoints, and the guard spec covers the bypass scenarios. However, `business.controller.ts` contains a runtime crash bug on the `PATCH /v1/business/:id` route — `.findById()` is not a valid array method. Two additional warning-level issues exist around undefined field propagation in the update payload and a dead-code null check. The guard spec has a duplicate test misplaced in the wrong describe block.

## Warnings

### WR-01: `.findById()` called on a plain array — will crash at runtime

**File:** `trade-flow-api/src/business/controllers/business.controller.ts:102`
**Issue:** `businesses.findById(request.params.id)` calls a method that does not exist on JavaScript arrays. The `Array` prototype has `find()` and `findIndex()` but not `findById()`. At runtime this throws `TypeError: businesses.findById is not a function` for every `PATCH /v1/business/:id` request. No existing business can ever be updated.
**Fix:**
```typescript
const existingBusiness = businesses.find((b) => b.id === request.params.id);
```

### WR-02: Undefined fields in update payload may overwrite existing data

**File:** `trade-flow-api/src/business/controllers/business.controller.ts:110-120`
**Issue:** The partial update DTO is constructed by directly assigning `business.name`, `business.country`, and `business.currency` from the request body. If these fields are absent from the `PATCH` request (valid for a partial update), they will be set to `undefined` in the DTO object. If `BusinessUpdater.update()` passes all DTO fields to the repository without filtering out `undefined` values, it will overwrite existing stored values with `undefined`/null, causing silent data loss.
**Fix:** Conditionally include fields only when they are present in the request body:
```typescript
const updatedBusiness: Partial<IBusinessDto> = {
  ...(business.name !== undefined && { name: business.name }),
  ...(business.country !== undefined && { country: business.country }),
  ...(business.currency !== undefined && { currency: business.currency }),
  ...(business.trade && {
    trade: {
      primaryTrade: business.trade.primaryTrade ?? existingBusiness.trade.primaryTrade,
      customTradeName: business.trade.customTradeName,
    },
  }),
};
```
Alternatively, verify that the `BusinessUpdater` and repository layer strip `undefined` values before writing to MongoDB.

### WR-03: Misplaced duplicate test inflates `onboarding bypass routes` describe block

**File:** `trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts:166-175`
**Issue:** The test "should throw ForbiddenError for POST without subscription when skipSubscriptionCheck is false" is nested under the `onboarding bypass routes` describe block but tests the opposite of a bypass — it tests the enforcement path (no subscription, no skip, guard rejects). This test body is also identical to the test at line 242-251 in the `subscription status checks` block. The duplication is harmless but the misplacement makes the `onboarding bypass routes` block misleading: a reader expects only bypass-allowing cases there.
**Fix:** Remove the duplicate test at line 166-175 from `onboarding bypass routes`. The identical test at line 242-251 in `subscription status checks` already provides this coverage in the correct location.

## Info

### IN-01: Dead-code null check after `findByIdOrFail`

**File:** `trade-flow-api/src/business/controllers/business.controller.ts:34-36`
**Issue:** After calling `this.businessRetriever.findByIdOrFail(...)`, the controller checks `if (!business)` and returns an empty response. By contract, `findByIdOrFail` throws `ResourceNotFoundError` when the resource is absent — it never returns `null` or `undefined`. This null-check is dead code and could mislead future readers into thinking the method can return falsy values.
**Fix:** Remove the dead null-check:
```typescript
const business = await this.businessRetriever.findByIdOrFail(request.user, businessId);
const businessResponseData = this.mapToResponse(business);
return createResponse([businessResponseData]);
```

### IN-02: `GET /v1/user/me/onboarding-progress` lacks `@SkipSubscriptionCheck()` — note for reviewers

**File:** `trade-flow-api/src/user/controllers/user.controller.ts:52-70`
**Issue:** The `getOnboardingProgress` endpoint used during onboarding does not carry `@SkipSubscriptionCheck()`. This is currently safe because the guard spec (lines 88-114) confirms `GET` requests are unconditionally allowed without a subscription check. However, if the guard's read-method bypass logic ever changes (e.g., to require a subscription even for `GET`), this endpoint would silently start blocking new users mid-onboarding. The `PATCH /v1/user/me` endpoint (line 35) correctly uses the decorator as explicit documentation of intent.
**Fix:** Add `@SkipSubscriptionCheck()` to `getOnboardingProgress` as explicit documentation that this endpoint must remain accessible during onboarding, independent of future guard evolution:
```typescript
@UseGuards(JwtAuthGuard)
@SkipSubscriptionCheck()
@Get("onboarding-progress")
public async getOnboardingProgress(...)
```

---

_Reviewed: 2026-04-07T18:44:34Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
