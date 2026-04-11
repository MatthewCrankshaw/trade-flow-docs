---
phase: 42
plan: 02
subsystem: trade-flow-api/core/errors, trade-flow-api/estimate
tags: [errors, di, foundation, phase-42, conflict-error, followup-canceller]
requires:
  - Phase 42 D-CONC-07 (ErrorCodes.ESTIMATE_REVISION_CONFLICT, ESTIMATE_NOT_REVISABLE reserved by Phase 42)
  - Phase 42 D-CONC-08 (intentionally generic 409 message — no race-condition leakage)
  - Phase 42 D-HOOK-01 (IEstimateFollowupCanceller signature)
  - Phase 42 D-HOOK-02 (ESTIMATE_FOLLOWUP_CANCELLER DI token)
  - Phase 42 D-HOOK-03 (Phase 42 ships no-op default; Phase 46 replaces with real implementation)
provides:
  - ConflictError domain class with createHttpError → 409 wiring
  - ErrorCodes.ESTIMATE_REVISION_CONFLICT and ErrorCodes.ESTIMATE_NOT_REVISABLE enum members and ERRORS_MAP entries
  - IEstimateFollowupCanceller interface and ESTIMATE_FOLLOWUP_CANCELLER string token
  - NoopEstimateFollowupCanceller @Injectable default implementation
  - @estimate/* and @estimate-test/* path aliases (tsconfig + Jest moduleNameMapper)
affects:
  - Phase 42 plans 03 (repository), 04 (reviser), 05 (retriever/deleter) can now compile against ConflictError + ErrorCodes.ESTIMATE_REVISION_CONFLICT
  - Phase 42 plan 06 (controller/module/openapi) can wire NoopEstimateFollowupCanceller into EstimateModule via the ESTIMATE_FOLLOWUP_CANCELLER token
  - Phase 46 (follow-up queue) will replace NoopEstimateFollowupCanceller binding with the BullMQ-backed implementation, no signature changes
tech-stack:
  added: []
  patterns:
    - "Domain error class mirrors InvalidRequestError shape verbatim — no shared base class, byte-for-byte duplication is the established convention (separation over DRY at error-class boundary)"
    - "createHttpError instanceof branches: 4xx errors at warn log level, 5xx at error level"
    - "DI token + interface in single file for cross-phase contracts (matches AccessControllerFactory pattern)"
    - "Type-guard helpers (isErrorEntry / extractFirstError / hasStringField) instead of as casts in spec narrowing"
key-files:
  created:
    - trade-flow-api/src/core/errors/conflict.error.ts
    - trade-flow-api/src/core/test/errors/conflict.error.spec.ts
    - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts
    - trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts
    - trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts
    - .planning/phases/42-revisions/42-02-SUMMARY.md
  modified:
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/core/errors/errors-map.constant.ts
    - trade-flow-api/src/core/errors/handle-error.utility.ts
    - trade-flow-api/src/core/test/errors/handle-error.utility.spec.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json
decisions:
  - "Spec file location follows existing convention src/core/test/errors/ (not src/core/errors/test/ as plan suggested) — matches the precedent set by handle-error.utility.spec.ts"
  - "Added @estimate/* + @estimate-test/* path aliases ahead of Phase 41 estimate scaffold (Rule 3 — blocking issue: imports otherwise won't compile)"
  - "Type-guard helpers in spec use Reflect.get() for unknown-key access; no `as` casts introduced. Pre-existing `as HttpException` casts in the unmodified parts of handle-error.utility.spec.ts left intact (out of scope per SCOPE BOUNDARY rule)"
  - "Followed plan request to keep ConflictError byte-for-byte identical to InvalidRequestError except for class name and this.name string"
metrics:
  duration: "~25min"
  completed: 2026-04-11
  tasks: 4
  files: 11
---

# Phase 42 Plan 02: ConflictError + IEstimateFollowupCanceller Foundation Summary

**One-liner:** Ship the ConflictError → HTTP 409 wiring (with two new ESTIMATE_* error codes) and the IEstimateFollowupCanceller interface + Noop default — both cross-cutting foundations every later Phase 42 plan imports from.

## What Changed

### 1. ConflictError domain class
A new error class at `trade-flow-api/src/core/errors/conflict.error.ts` mirroring `InvalidRequestError` byte-for-byte. Only differences: class name and the `this.name = "ConflictError"` literal. No shared base class — Phase 42 follows the established convention of duplicating error class shape rather than refactoring all four existing error classes.

### 2. Two new ESTIMATE_* error codes
Added to `error-codes.enum.ts` between the `Quotes` and `Email` blocks, using the **string-value-equals-key** pattern (per Phase 41 PLAN-01 lines 173-180):

```typescript
// Estimates — Phase 42 (Phase 41 reserves ESTIMATE_NOT_FOUND, ESTIMATE_INVALID_TRANSITION, etc.)
ESTIMATE_REVISION_CONFLICT = "ESTIMATE_REVISION_CONFLICT",
ESTIMATE_NOT_REVISABLE = "ESTIMATE_NOT_REVISABLE",
```

`ERRORS_MAP` entries carry the **exact D-CONC-08 messages**:
- `ESTIMATE_REVISION_CONFLICT` → `"Estimate has already been revised or is no longer revisable"` (intentionally generic — does not leak which of the three atomic-filter-miss conditions tripped)
- `ESTIMATE_NOT_REVISABLE` → `"Estimate cannot be revised in its current state"`

### 3. createHttpError ConflictError branch
Inserted into `handle-error.utility.ts` at **line 45**, after the InvalidRequestError block (lines 30-43) and before the ResourceNotFoundError block (line 60). Uses `Logger.warn` to match the precedent of the other 4xx branches. Returns `HttpException` with `HttpStatus.CONFLICT` (409).

Branch order verified:
```
Line 30: if (error instanceof InvalidRequestError)
Line 45: if (error instanceof ConflictError)
Line 60: if (error instanceof ResourceNotFoundError)
```

### 4. IEstimateFollowupCanceller interface + Noop default
- **Interface file** (`estimate-followup-canceller.interface.ts`): exactly two exports — the interface with the D-HOOK-01 signature `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>` and the `ESTIMATE_FOLLOWUP_CANCELLER` const string DI token.
- **Noop service** (`noop-estimate-followup-canceller.service.ts`): `@Injectable()`, implements the interface, single `Logger.debug()` call, returns immediately. Phase 46 will replace the binding with the BullMQ-backed implementation — signature must not drift.
- **Spec** (`noop-estimate-followup-canceller.service.spec.ts`): uses `Test.createTestingModule` to prove DI-resolvability + asserts the method resolves to `undefined`.

### 5. Path-alias scaffolding (Rule 3 — blocking issue)
The plan assumes `@estimate/*` resolves, but Phase 41 has not yet executed and the alias was missing. Without it, the new files would not compile or pass Jest's module resolution. Added entries to:
- `tsconfig.json` `compilerOptions.paths` (for `tsc`/`tsc --noEmit`)
- `package.json` `jest.moduleNameMapper` (for the test runner)

Both `@estimate/*` and `@estimate-test/*` aliases were added.

## Task Execution

| Task | Name                                                                              | Commit    | Files                                                                                                                                     |
| ---- | --------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 1+2  | ConflictError class + spec; ESTIMATE_* error codes + ERRORS_MAP entries           | `63d2bf5` | conflict.error.ts, conflict.error.spec.ts, error-codes.enum.ts, errors-map.constant.ts                                                    |
| 3    | Wire ConflictError → HttpStatus.CONFLICT in createHttpError                       | `eb79f87` | handle-error.utility.ts, handle-error.utility.spec.ts                                                                                     |
| 4    | IEstimateFollowupCanceller interface + Noop default + path aliases                | `bedf0c7` | estimate-followup-canceller.interface.ts, noop-estimate-followup-canceller.service.ts, ...spec.ts, tsconfig.json, package.json            |

Note: Tasks 1 and 2 were committed together (`63d2bf5`) because Task 1's spec depends on Task 2's enum entry to assert the `getMessage` mapped value. Splitting would have required an intermediate failing-spec commit.

All commits land in the `trade-flow-api` git repo (sub-repo, not the docs repo).

## Verification

```bash
$ cd trade-flow-api && npm run ci
# tests
Test Suites: 99 passed, 99 total
Tests:       (all passing — including 6 new ConflictError tests, 2 new createHttpError ConflictError-branch tests, 2 new NoopEstimateFollowupCanceller DI tests)
# lint
✖ 24 problems (0 errors, 24 warnings)   ← all pre-existing, none in files touched by this plan
# format
All matched files use Prettier code style!
# typecheck
(clean exit)
```

`npm run ci` exits 0. The 24 lint warnings are pre-existing and out of scope per SCOPE BOUNDARY rule.

## Deviations from Plan

### Rule 3 — Blocking issue (path aliases)

**Found during:** Task 4 (interface + noop service) — the new files import from `@estimate/services/...` but neither `tsconfig.json` nor Jest's `moduleNameMapper` had any `@estimate/*` entry.

**Fix:** Added `@estimate/*` and `@estimate-test/*` to both. Mirrors the precedent set by every other module alias in the project (e.g., `@quote/*`, `@subscription/*`, `@business-test/*`).

**Files modified:** `trade-flow-api/tsconfig.json`, `trade-flow-api/package.json`

**Commit:** `bedf0c7` (rolled into Task 4 commit since Task 4 introduced the dependency)

### Rule 3 — Spec file location convention

**Found during:** Task 1 read-first phase — the plan specified spec files at `src/core/errors/test/...` but the existing handle-error spec lives at `src/core/test/errors/handle-error.utility.spec.ts`. There is no `src/core/errors/test/` directory in the repository.

**Fix:** Placed `conflict.error.spec.ts` at `src/core/test/errors/conflict.error.spec.ts` to match the established convention. Extended the existing `handle-error.utility.spec.ts` in place rather than creating a new file.

**Files modified:** Spec file paths only — no behavioral change.

### Rule 1 — Type-guard helpers without `as` casts

**Found during:** Task 3 — initial draft of `extractFirstError` used `as { errors: unknown }` and `as Record<string, unknown>` casts, which violate CLAUDE.md's `Avoid as type assertions` rule and the user's `feedback_no_type_assertions.md` memory.

**Fix:** Refactored to use `Reflect.get(value, field)` plus `typeof` checks, with two small predicate helpers (`hasStringField`, `hasField`) to keep the type guard readable without any `as` casts. The pre-existing `(result as HttpException).getStatus()` casts elsewhere in the file were left untouched per the SCOPE BOUNDARY rule.

**Files modified:** `trade-flow-api/src/core/test/errors/handle-error.utility.spec.ts` (helpers only)

## Threat Mitigations Confirmed

| Threat ID    | Mitigation Verification |
| ------------ | ----------------------- |
| T-42-02-01 (info disclosure) | Generic 409 message asserted in `errors-map.constant.ts` and `conflict.error.spec.ts` — single message string for all three atomic-filter miss conditions |
| T-42-02-02 (privilege misroute) | `handle-error.utility.spec.ts` ConflictError branch tests assert `HttpStatus.CONFLICT` (409); regression tests for other 4xx branches still passing |
| T-42-02-03 (enum collision) | Used string-value-equals-key per Phase 41 PLAN-01 convention; new codes are namespaced strings (`"ESTIMATE_REVISION_CONFLICT"`) — collision impossible by construction |
| T-42-02-04 (signature drift) | Signature `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>` is verbatim from D-HOOK-01; Phase 46 will fail to compile if it drifts |
| T-42-02-05 (DoS via hang) | Spec asserts `await ... resolves.toBeUndefined()`; no awaits/branches in implementation |
| T-42-02-06 (PII in debug log) | Disposition: `accept`. Estimate IDs are opaque internal identifiers, not PII; debug level not shipped to production logs |

## Self-Check

**Created files:**
- `FOUND: trade-flow-api/src/core/errors/conflict.error.ts`
- `FOUND: trade-flow-api/src/core/test/errors/conflict.error.spec.ts`
- `FOUND: trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts`
- `FOUND: trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts`
- `FOUND: trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts`

**Modified files:**
- `FOUND (modified): trade-flow-api/src/core/errors/error-codes.enum.ts (ESTIMATE_REVISION_CONFLICT, ESTIMATE_NOT_REVISABLE)`
- `FOUND (modified): trade-flow-api/src/core/errors/errors-map.constant.ts (two new map entries with D-CONC-08 wording)`
- `FOUND (modified): trade-flow-api/src/core/errors/handle-error.utility.ts (ConflictError branch at line 45)`
- `FOUND (modified): trade-flow-api/src/core/test/errors/handle-error.utility.spec.ts (ConflictError branch describe block + type-guard helpers)`
- `FOUND (modified): trade-flow-api/tsconfig.json (@estimate/* + @estimate-test/* aliases)`
- `FOUND (modified): trade-flow-api/package.json (Jest moduleNameMapper @estimate/* + @estimate-test/*)`

**Commits (in trade-flow-api repo):**
- `FOUND: 63d2bf5 feat(42-02): add ConflictError domain class and ESTIMATE_* error codes`
- `FOUND: eb79f87 feat(42-02): wire ConflictError to HttpStatus.CONFLICT in createHttpError`
- `FOUND: bedf0c7 feat(42-02): add IEstimateFollowupCanceller interface + Noop default`

## Self-Check: PASSED
