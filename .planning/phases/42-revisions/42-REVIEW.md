---
phase: 42-revisions
reviewed: 2026-04-12T10:00:00Z
depth: standard
files_reviewed: 22
files_reviewed_list:
  - trade-flow-api/src/core/errors/conflict.error.ts
  - trade-flow-api/src/core/errors/error-codes.enum.ts
  - trade-flow-api/src/core/errors/errors-map.constant.ts
  - trade-flow-api/src/core/errors/handle-error.utility.ts
  - trade-flow-api/src/core/test/errors/conflict.error.spec.ts
  - trade-flow-api/src/core/test/errors/handle-error.utility.spec.ts
  - trade-flow-api/src/estimate/controllers/estimate.controller.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
  - trade-flow-api/src/estimate/repositories/estimate.repository.ts
  - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts
  - trade-flow-api/src/estimate/services/estimate-reviser.service.ts
  - trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts
  - trade-flow-api/src/estimate/test/controllers/estimate.controller.spec.ts
  - trade-flow-api/src/estimate/test/mocks/estimate-revision-mock-generator.ts
  - trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
  - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
  - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
  - trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts
  - trade-flow-api/openapi.yaml
  - trade-flow-api/docs/smoke/phase-42-concurrent-revise.md
  - trade-flow-api/tsconfig.json
  - trade-flow-api/package.json
findings:
  critical: 0
  warning: 2
  info: 2
  total: 4
status: issues_found
---

# Phase 42: Code Review Report

**Reviewed:** 2026-04-12T10:00:00Z
**Depth:** standard
**Files Reviewed:** 22
**Status:** issues_found

## Summary

Phase 42 introduces estimate revision support: a `ConflictError` class, an `EstimateReviser` service with compensating rollback, new repository methods (`downgradeCurrent`, `insertRevision`, `restoreCurrent`, `softDeleteRow`, `findRevisionsByRootId`, `findCurrentInChainByRootId`), new controller endpoints (POST and GET revisions), partial unique indexes for concurrency safety, and a `NoopEstimateFollowupCanceller` hook.

The implementation is well-structured with a clear concurrency strategy (atomic `downgradeCurrent` filter + partial unique index) and a compensating rollback mechanism. The test suite is thorough with good coverage of happy paths, error paths, rollback ordering, and bundle line-item cloning. The OpenAPI spec is properly updated.

Two warnings and two informational items were identified -- no critical issues.

## Warnings

### WR-01: Missing ERRORS_MAP entries for multiple ErrorCodes values

**File:** `trade-flow-api/src/core/errors/errors-map.constant.ts`
**Issue:** Several `ErrorCodes` enum values have no corresponding entry in `ERRORS_MAP`: `ERROR_CODE_UNKNOWN` (CORE_1), `QUOTE_JOB_NOT_FOUND` (QUOTE_0), `QUOTE_INVALID_TRANSITION` (QUOTE_1), `QUOTE_EXPIRED` (QUOTE_5), `DOCUMENT_TOKEN_NOT_FOUND`, `DOCUMENT_TOKEN_EXPIRED`, `DOCUMENT_TOKEN_REVOKED`, `DOCUMENT_TOKEN_TYPE_MISMATCH`, and `EMAIL_DELIVERY_FAILED` (EMAIL_0). While most of these are pre-existing gaps, Phase 42 introduced `ConflictError` which uses `ERRORS_MAP.get(code)` with a fallback that references `ErrorCodes.ERROR_CODE_UNKNOWN` (line 11 of `conflict.error.ts`) -- a code that itself has no map entry. If `ConflictError` is ever constructed with an unmapped code, the fallback message "Error code unknown" is used correctly, but the `code` field in the fallback object would be `CORE_1` which has no human-readable map entry. This is a latent inconsistency.

**Fix:** Add the missing entries to `ERRORS_MAP`. At minimum, add `ERROR_CODE_UNKNOWN`:
```typescript
[ErrorCodes.ERROR_CODE_UNKNOWN, { code: ErrorCodes.ERROR_CODE_UNKNOWN, message: "Error code unknown" }],
```

### WR-02: `bulkInsertForRevision` uses sequential inserts instead of bulk write

**File:** `trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts:133-139`
**Issue:** The `bulkInsertForRevision` method loops through line items calling `this.create()` one at a time in a `for...of` loop. If the second insert fails, the first insert has already been persisted with no rollback at the repository level. The service-level compensating rollback in `EstimateReviser` only soft-deletes the estimate row and restores `isCurrent` -- it does not clean up partially-inserted line items from the new revision. After rollback, orphaned line items may remain in the `estimatelineitems` collection pointing to a soft-deleted estimate. These orphans are harmless at query time (they will not appear in any `findByEstimateId` call for a non-deleted estimate), but they accumulate as dead data.

**Fix:** Consider using MongoDB's `insertMany` for atomicity, or add line-item cleanup to the compensating rollback in `EstimateReviser.compensatingRollback`. A pragmatic short-term fix is to document the orphan behavior as acceptable:
```typescript
// Known: partial inserts may leave orphaned line items pointing to the soft-deleted
// revision row. These are invisible to queries (the parent estimate is deleted) and
// will be cleaned up by a future data hygiene job.
```

## Info

### IN-01: `followupCanceller` is injected but never invoked

**File:** `trade-flow-api/src/estimate/services/estimate-reviser.service.ts:38-39`
**Issue:** The `IEstimateFollowupCanceller` is injected via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` but is never called anywhere in the `revise` method. The test suite explicitly documents this as intentional (D-HOOK-03) and the `NoopEstimateFollowupCanceller` is a placeholder. This is noted for awareness -- the hook should be wired in when follow-up email cancellation is implemented.

**Fix:** No action needed now. When follow-up cancellation is implemented, call `this.followupCanceller.cancelAllFollowups(source.id, source.revisionNumber)` after a successful revision (before the final `findByIdOrFail` return).

### IN-02: `ESTIMATE_NOT_REVISABLE` error code defined but not referenced

**File:** `trade-flow-api/src/core/errors/error-codes.enum.ts:76`
**Issue:** `ESTIMATE_NOT_REVISABLE` is defined in the enum and has a map entry, but is not thrown by any code in the reviewed files. The reviser throws `ConflictError` with `ESTIMATE_REVISION_CONFLICT` for non-revisable states, and `InvalidRequestError` / `ResourceNotFoundError` for other cases. `ESTIMATE_NOT_REVISABLE` appears to be a forward declaration for a future validation path (e.g., a pre-check endpoint or a different error granularity).

**Fix:** No action needed. If it remains unused after follow-up phases, consider removing it to avoid dead code.

---

_Reviewed: 2026-04-12T10:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
