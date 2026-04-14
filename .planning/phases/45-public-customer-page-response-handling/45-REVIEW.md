---
phase: 45-public-customer-page-response-handling
reviewed: 2026-04-13T14:30:00Z
depth: standard
files_reviewed: 33
files_reviewed_list:
  - trade-flow-api/src/document-token/guards/document-session-auth.guard.ts
  - trade-flow-api/src/email/services/estimate-notification-email-renderer.service.ts
  - trade-flow-api/src/email/templates/estimate-response.html
  - trade-flow-api/src/estimate/controllers/public-estimate.controller.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate-response-summary.dto.ts
  - trade-flow-api/src/estimate/data-transfer-objects/estimate.dto.ts
  - trade-flow-api/src/estimate/entities/estimate.entity.ts
  - trade-flow-api/src/estimate/enums/estimate-decline-reason.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-response-type.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-status.enum.ts
  - trade-flow-api/src/estimate/enums/estimate-transitions.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/document-token/document-token.module.ts
  - trade-flow-api/src/estimate/repositories/estimate.repository.ts
  - trade-flow-api/src/estimate/requests/respond-to-estimate.request.ts
  - trade-flow-api/src/estimate/responses/public-estimate.response.ts
  - trade-flow-api/src/estimate/services/estimate-response-handler.service.ts
  - trade-flow-api/src/estimate/services/estimate-transition.service.ts
  - trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts
  - trade-flow-api/src/estimate/services/estimate-retriever.service.ts
  - trade-flow-ui/src/features/public-estimate/api/publicEstimateApi.ts
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateCard.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateDeclineForm.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateDisclaimer.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateError.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateMessageForm.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimatePriceRange.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateResponseButtons.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateSkeleton.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateTerminalState.tsx
  - trade-flow-ui/src/features/public-estimate/components/PublicEstimateUncertainty.tsx
  - trade-flow-ui/src/features/public-estimate/types/public-estimate.types.ts
  - trade-flow-ui/src/pages/PublicEstimatePage.tsx
findings:
  critical: 1
  warning: 5
  info: 3
  total: 9
status: issues_found
---

# Phase 45: Code Review Report

**Reviewed:** 2026-04-13T14:30:00Z
**Depth:** standard
**Files Reviewed:** 33
**Status:** issues_found

## Summary

Phase 45 implements public customer-facing estimate pages with response handling. The implementation follows the project's layered architecture well: public controller with throttle and session auth guards, service layer for business logic, repository for persistence. The React UI is clean with good component decomposition.

Key concerns: one security issue with unescaped URL injection in email templates, a race condition in the response handler between pushing a response and transitioning status, and a respondability logic gap where `RESPONDED` status with no responses (edge case) is incorrectly treated as respondable.

## Critical Issues

### CR-01: viewUrl Not HTML-Escaped in Email Template Renderer

**File:** `trade-flow-api/src/email/services/estimate-notification-email-renderer.service.ts:30`
**Issue:** The `viewUrl` value is injected into the HTML template without `escapeHtml()` on line 30. All other template variables (`businessName`, `customerName`, `estimateNumber`) are properly escaped. The `viewUrl` is constructed from `APP_URL` or `CORS_ORIGIN` config values concatenated with an estimate ID, but if either config value were to contain user-controllable content or special characters (`"`, `<`, `>`), it would result in HTML injection in the rendered email. The URL is placed in both an `href` attribute and a VML `href` attribute in the template.

While the current construction (`configValue + "/estimates/" + estimateId`) uses server-controlled values, this is a defense-in-depth concern -- the escapeHtml method exists and is used for every other field but was missed here.

**Fix:**
```typescript
.replace(/\{\{\s*viewUrl\s*\}\}/g, this.escapeHtml(data.viewUrl))
```

Note: Standard HTML escaping is appropriate for URLs in `href` attributes. The `&amp;` encoding of `&` is valid in HTML attribute values. If the URL needs query parameters, they will still work correctly after HTML entity encoding.

## Warnings

### WR-01: Race Condition Between pushResponse and publicTransition

**File:** `trade-flow-api/src/estimate/services/estimate-response-handler.service.ts:54-57`
**Issue:** The `handleResponse` method performs two sequential mutations that are not atomic: first `pushResponse` (line 54) appends the response to the array, then `publicTransition` (line 57) changes the estimate status. If the process crashes or errors between these two calls, the estimate will have a response recorded but remain in `VIEWED`/`SENT` status -- leaving it in an inconsistent state where it appears respondable but has a response. Additionally, a concurrent request could slip through the `validateRespondableState` check before the status transition completes.

**Fix:** Consider combining both mutations into a single MongoDB `findOneAndUpdate` operation, or at minimum add a status guard to the `pushResponse` call:
```typescript
// Option 1: Transition first, then push response (fail-safe order)
await this.estimateTransitionService.publicTransition(estimate.id, targetStatus);
await this.estimateRepository.pushResponse(estimate.id, responseEntity);

// Option 2: Use MongoDB findOneAndUpdate with status precondition
await this.estimateRepository.pushResponseAndTransition(
  estimate.id,
  responseEntity,
  targetStatus,
  [EstimateStatus.VIEWED, EstimateStatus.SENT], // precondition
);
```

### WR-02: RESPONDED Status Without Responses Treated as Respondable

**File:** `trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts:105-108`
**Issue:** The `computeTerminalState` method marks an estimate as `customer_responded` (non-respondable) only when `status === RESPONDED` AND `responses.length > 0`. If the estimate is in `RESPONDED` status but has an empty responses array (possible via direct admin action or data inconsistency), it falls through to the default return of `{ terminalState: null, respondable: true }`. This would incorrectly show response buttons to the customer for an estimate already in RESPONDED status.

**Fix:**
```typescript
if (
  estimate.status === EstimateStatus.DECLINED ||
  estimate.status === EstimateStatus.RESPONDED
) {
  return { terminalState: "customer_responded", respondable: false };
}
```

### WR-03: Duplicate Transition Logic Between transition and publicTransition

**File:** `trade-flow-api/src/estimate/services/estimate-transition.service.ts:70-114`
**Issue:** The `publicTransition` method (lines 70-114) is a near-exact copy of the `transition` method (lines 21-68) with only the authorization check removed. The entire switch statement for setting timestamp fields is duplicated. If a new status is added or a timestamp mapping changes, both methods must be updated identically or they will diverge.

**Fix:** Extract the shared timestamp-setting logic into a private method:
```typescript
private applyTimestampFields(current: IEstimateDto, updated: IEstimateDto, targetStatus: EstimateStatus): void {
  const now = DateTime.now();
  switch (targetStatus) {
    case EstimateStatus.SENT: updated.sentAt = now; break;
    case EstimateStatus.VIEWED:
      if (!current.firstViewedAt) updated.firstViewedAt = now;
      break;
    case EstimateStatus.RESPONDED: updated.respondedAt = now; break;
    // ... remaining cases
  }
}
```

### WR-04: Duplicate firstViewedAt Update in Guard and Service

**File:** `trade-flow-api/src/document-token/guards/document-session-auth.guard.ts:59-61` and `trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts:86-89`
**Issue:** The `firstViewedAt` field on the document token is updated in two places: the `DocumentSessionAuthGuard` (line 60) and the `PublicEstimateRetriever.trackFirstView` method (line 88). Both check `!tokenDto.firstViewedAt` and call `documentTokenRepository.updateFirstViewedAt`. Since the guard runs before the service, the guard will always set `firstViewedAt` first, making the service's check dead code. This is not a bug per se, but it creates confusion about ownership of this concern and means the service check will never trigger.

**Fix:** Remove the duplicate `firstViewedAt` token update from the service, keeping it only in the guard where it semantically belongs (first access tracking):
```typescript
// In PublicEstimateRetriever.trackFirstView, remove the token update:
private async trackFirstView(_token: IDocumentTokenDto, estimate: IEstimateDto): Promise<void> {
  if (estimate.status === EstimateStatus.SENT) {
    this.logger.log("Triggering Sent -> Viewed transition", { estimateId: estimate.id });
    await this.estimateTransitionService.publicTransition(estimate.id, EstimateStatus.VIEWED);
  }
}
```

### WR-05: Type Assertion on Error in PublicEstimatePage

**File:** `trade-flow-ui/src/pages/PublicEstimatePage.tsx:34`
**Issue:** The error object is cast using `as` assertion: `(error as { status?: number } | undefined)?.status`. Per project conventions, `as` type assertions should be avoided in favor of type guards.

**Fix:** Use a type guard:
```typescript
function getErrorStatus(error: unknown): number | undefined {
  if (error && typeof error === "object" && "status" in error && typeof error.status === "number") {
    return error.status;
  }
  return undefined;
}

// Usage:
const status = getErrorStatus(error);
```

## Info

### IN-01: Dynamic Import via Function Constructor

**File:** `trade-flow-api/src/email/services/estimate-notification-email-renderer.service.ts:40`
**Issue:** `Function('return import("@maizzle/framework")')()` is used as a workaround for dynamic ESM imports in a CommonJS context. While this is functionally equivalent to `eval`, it is wrapped in a try-catch and only imports a known package name (not user input), so the security risk is negligible. The pattern is consistent with other email renderers in the codebase.

**Fix:** Add a brief comment explaining why this workaround exists, e.g.:
```typescript
// Dynamic ESM import workaround for CommonJS runtime
const maizzle = await (Function('return import("@maizzle/framework")')() as Promise<Record<string, unknown>>);
```

### IN-02: Unused Import in PublicEstimatePage

**File:** `trade-flow-ui/src/pages/PublicEstimatePage.tsx:7`
**Issue:** `PublicEstimateTerminalState` is imported but the `trader_closed` terminal state is handled by rendering the component directly in the page (line 47). This is actually used, but the component is also imported in `PublicEstimateCard.tsx` (line 11) for the `customer_responded` case. The architecture is fine -- this is just a note that the same component serves two different rendering contexts.

**Fix:** No action needed -- both imports are used in their respective rendering paths.

### IN-03: Hardcoded Currency Code in PublicEstimatePriceRange

**File:** `trade-flow-ui/src/features/public-estimate/components/PublicEstimatePriceRange.tsx:15`
**Issue:** The currency is hardcoded to `"GBP"`. This is consistent with the current single-market scope of Trade Flow but will need to be dynamic if multi-currency support is added.

**Fix:** Consider passing the currency from the API response when multi-currency is needed. No action required for current scope.

---

_Reviewed: 2026-04-13T14:30:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
