---
phase: 45-public-customer-page-response-handling
fixed_at: 2026-04-14T10:30:00Z
review_path: .planning/phases/45-public-customer-page-response-handling/45-REVIEW.md
iteration: 1
findings_in_scope: 6
fixed: 6
skipped: 0
status: all_fixed
---

# Phase 45: Code Review Fix Report

**Fixed at:** 2026-04-14T10:30:00Z
**Source review:** .planning/phases/45-public-customer-page-response-handling/45-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 6
- Fixed: 6
- Skipped: 0

## Fixed Issues

### CR-01: viewUrl Not HTML-Escaped in Email Template Renderer

**Files modified:** `trade-flow-api/src/email/services/estimate-notification-email-renderer.service.ts`
**Commit:** 9503817 (trade-flow-api)
**Applied fix:** Added `this.escapeHtml()` wrapper around `data.viewUrl` in the template replacement chain, consistent with all other template variables.

### WR-01: Race Condition Between pushResponse and publicTransition

**Files modified:** `trade-flow-api/src/estimate/services/estimate-response-handler.service.ts`
**Commit:** 01fbead (trade-flow-api)
**Applied fix:** Reordered operations so `publicTransition` (status change) executes before `pushResponse` (appending response). This is the fail-safe order: if the process crashes after transition, the estimate is in the correct terminal status but missing a response detail (less harmful than having a response recorded but the estimate remaining in a respondable status).

### WR-02: RESPONDED Status Without Responses Treated as Respondable

**Files modified:** `trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts`
**Commit:** 9625411 (trade-flow-api)
**Applied fix:** Simplified the `computeTerminalState` condition to check `DECLINED || RESPONDED` status without requiring `responses.length > 0`. An estimate in RESPONDED status is now always treated as non-respondable regardless of the responses array contents.

### WR-03: Duplicate Transition Logic Between transition and publicTransition

**Files modified:** `trade-flow-api/src/estimate/services/estimate-transition.service.ts`
**Commit:** fb50914 (trade-flow-api)
**Applied fix:** Extracted the duplicated timestamp-setting switch statement into a private `applyTimestampFields(current, updated, targetStatus)` method. Both `transition` and `publicTransition` now call this shared method, eliminating the risk of divergence when new statuses or timestamp mappings are added.

### WR-04: Duplicate firstViewedAt Update in Guard and Service

**Files modified:** `trade-flow-api/src/estimate/services/public-estimate-retriever.service.ts`
**Commit:** 19c773f (trade-flow-api)
**Applied fix:** Removed the dead-code `firstViewedAt` token update from `PublicEstimateRetriever.trackFirstView`, keeping it only in `DocumentSessionAuthGuard` where it semantically belongs. Also removed the now-unused `DocumentTokenRepository` import and constructor injection from the service.

### WR-05: Type Assertion on Error in PublicEstimatePage

**Files modified:** `trade-flow-ui/src/pages/PublicEstimatePage.tsx`
**Commit:** 4bd01a0 (trade-flow-ui)
**Applied fix:** Replaced the `as` type assertion with a `getErrorStatus` type guard function that safely narrows the unknown error type using runtime checks (`typeof`, `in` operator), per project conventions against type assertions.

---

_Fixed: 2026-04-14T10:30:00Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
