---
phase: 42
plan: 03
subsystem: estimate-repository
tags: [repository, mongodb, indexes, phase-42, revision]
dependency_graph:
  requires: [42-01, 42-02]
  provides: [downgradeCurrent, insertRevision, restoreCurrent, findRevisionsByRootId, findCurrentInChainByRootId, softDeleteRow, findNonDeletedByEstimateId, bulkInsertForRevision]
  affects: [42-04, 42-05]
tech_stack:
  added: []
  patterns: [findOneAndUpdate-atomic-filter, sequential-bulk-insert, compensating-rollback-primitive]
key_files:
  created: []
  modified:
    - trade-flow-api/src/estimate/repositories/estimate.repository.ts
    - trade-flow-api/src/estimate/repositories/estimate-line-item.repository.ts
    - trade-flow-api/src/estimate/test/repositories/estimate.repository.spec.ts
    - trade-flow-api/src/estimate/test/repositories/estimate-line-item.repository.spec.ts
decisions:
  - D-DET-02 list filter ownership consolidated in plan 42-03 (not 42-05)
  - findRevisionsByRootId and findCurrentInChainByRootId use direct db.collection() cursor access (matching findPaginatedByBusinessId pattern)
  - bulkInsertForRevision delegates to this.create() rather than raw writer.insertOne for consistency
metrics:
  duration: 7m
  completed: "2026-04-12T18:25:23Z"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 4
---

# Phase 42 Plan 03: Estimate Repository Revision Methods Summary

Six new repository methods on EstimateRepository and two on EstimateLineItemRepository for Phase 42 revision support, with D-DET-02 list filter and verified five-index topology.

## Task Results

### Task 1: EstimateRepository revision methods + index verification

**Commit (RED):** `88b7b26` - Failing tests for all six methods + D-DET-02 + index assertions
**Commit (GREEN):** `bf0d103` - Implementation of all six methods + D-DET-02 filter

**Methods added to `estimate.repository.ts`:**

| Method | Line | Signature | Purpose |
|--------|------|-----------|---------|
| `downgradeCurrent` | ~158 | `(id: string, allowedSourceStatuses: EstimateStatus[]): Promise<IEstimateDto \| null>` | Atomic findOneAndUpdate with `{_id, isCurrent: true, status: {$in}}` filter; returns null on miss |
| `insertRevision` | ~173 | `(dto: IEstimateDto): Promise<IEstimateDto>` | Inserts fully-formed revision entity via insertOne |
| `restoreCurrent` | ~179 | `(id: string): Promise<void>` | Idempotent findOneAndUpdate setting isCurrent: true; no-op on miss |
| `findRevisionsByRootId` | ~188 | `(rootEstimateId: string): Promise<IEstimateDto[]>` | Cursor-based query with `{rootEstimateId, deletedAt: null}` sorted by revisionNumber ascending |
| `findCurrentInChainByRootId` | ~196 | `(rootEstimateId: string): Promise<IEstimateDto \| null>` | findOne with `{rootEstimateId, isCurrent: true, deletedAt: null}`; returns null during zero-current window |
| `softDeleteRow` | ~207 | `(id: string): Promise<void>` | Compensating rollback primitive; filter `{_id, isCurrent: false}` sets status DELETED + deletedAt + updatedAt |

**D-DET-02 list filter:** `findPaginatedByBusinessId` filter now includes `isCurrent: true` alongside `deletedAt: null`.

**Index topology verified (5 createIndex calls):**
1. `{ businessId: 1, createdAt: -1 }`
2. `{ jobId: 1, createdAt: -1 }`
3. `{ businessId: 1, number: 1 }` - unique, partialFilterExpression: `{ deletedAt: null, isCurrent: true }`
4. `{ rootEstimateId: 1, isCurrent: 1 }` - unique, partialFilterExpression: `{ isCurrent: true }`
5. `{ rootEstimateId: 1, revisionNumber: 1 }`

**Spec coverage:** 18 new tests in "Phase 42 revision methods" describe block (38 total).

### Task 2: EstimateLineItemRepository revision support methods

**Commit (RED):** `15c6f85` - Failing tests for both methods
**Commit (GREEN):** `710ed4e` - Implementation of both methods

**Methods added to `estimate-line-item.repository.ts`:**

| Method | Signature | Purpose |
|--------|-----------|---------|
| `findNonDeletedByEstimateId` | `(estimateId: string): Promise<IEstimateLineItemDto[]>` | Returns plain DTO array filtered by `{estimateId, status: {$ne: DELETED}}` |
| `bulkInsertForRevision` | `(lineItems: IEstimateLineItemDto[]): Promise<IEstimateLineItemDto[]>` | Sequential `for...of` loop calling `this.create()` per item; no Promise.all |

**Spec coverage:** 6 new tests in "Phase 42 revision support methods" describe block (17 total).

### Formatting fix

**Commit:** `bd4259e` - Prettier formatting corrections across all 4 files.

## Verification

- `npm run ci` exits 0 (0 errors, 27 pre-existing warnings)
- 55 total tests across both spec files (38 estimate + 17 line-item), all passing
- No `any`, no `as`, no suppression comments in any modified file

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Prettier formatting corrections**
- **Found during:** Task 1 and Task 2 implementation
- **Issue:** Multi-line method signatures and chained mock calls exceeded prettier's single-line threshold
- **Fix:** Collapsed to single-line format per 125-char print width
- **Files modified:** All 4 files
- **Commit:** `bd4259e`

## Threat Mitigations Applied

- **T-42-03-01:** `downgradeCurrent` filter shape verified via spec assertion (`status: { $in: [...] }`, `isCurrent: true`)
- **T-42-03-06:** All five createIndex calls verified present with exact partial filter expressions via spec + grep
- **T-42-03-07:** `restoreCurrent` filter `{_id, isCurrent: false}` verified idempotent via null-return test

## Self-Check: PASSED

All 4 files confirmed present. All 5 commits confirmed in git log.
