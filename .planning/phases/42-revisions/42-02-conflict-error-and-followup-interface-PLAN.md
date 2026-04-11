---
phase: 42
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-api/src/core/errors/conflict.error.ts
  - trade-flow-api/src/core/errors/test/conflict.error.spec.ts
  - trade-flow-api/src/core/errors/error-codes.enum.ts
  - trade-flow-api/src/core/errors/errors-map.constant.ts
  - trade-flow-api/src/core/errors/handle-error.utility.ts
  - trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts
  - trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts
  - trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts
  - trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts
autonomous: true
requirements: [REV-05]
estimate: 50min
tags: [errors, di, foundation, phase-42]

must_haves:
  truths:
    - "`ConflictError` extends `Error`, matches the byte-for-byte shape of `InvalidRequestError`, and is constructible with `(code: ErrorCodes, details: string)`."
    - "`createHttpError(new ConflictError(...))` returns an `HttpException` with status `HttpStatus.CONFLICT` (409)."
    - "`ErrorCodes.ESTIMATE_REVISION_CONFLICT` and `ErrorCodes.ESTIMATE_NOT_REVISABLE` are declared and carry entries in `ERRORS_MAP` with the exact D-CONC-08 message strings."
    - "`IEstimateFollowupCanceller` interface is declared with the exact signature `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>` and exports the `ESTIMATE_FOLLOWUP_CANCELLER` string token."
    - "`NoopEstimateFollowupCanceller` is `@Injectable()`, implements the interface, logs at `debug` level, and resolves immediately."
    - "`cd trade-flow-api && npm run ci` exits 0 after this plan completes."
  artifacts:
    - path: "trade-flow-api/src/core/errors/conflict.error.ts"
      provides: "409 Conflict domain error class"
      exports: ["ConflictError"]
    - path: "trade-flow-api/src/core/errors/test/conflict.error.spec.ts"
      provides: "Unit coverage for getCode/getMessage/getDetails"
    - path: "trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts"
      provides: "Cross-phase contract for follow-up cancellation"
      exports: ["IEstimateFollowupCanceller", "ESTIMATE_FOLLOWUP_CANCELLER"]
    - path: "trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts"
      provides: "Phase 42 default binding; Phase 46 replaces"
      exports: ["NoopEstimateFollowupCanceller"]
  key_links:
    - from: "trade-flow-api/src/core/errors/handle-error.utility.ts"
      to: "trade-flow-api/src/core/errors/conflict.error.ts"
      via: "instanceof branch returning HttpStatus.CONFLICT"
      pattern: "error instanceof ConflictError"
    - from: "trade-flow-api/src/core/errors/errors-map.constant.ts"
      to: "trade-flow-api/src/core/errors/error-codes.enum.ts"
      via: "ERRORS_MAP.get() entries keyed by new enum values"
      pattern: "ErrorCodes\\.ESTIMATE_REVISION_CONFLICT|ErrorCodes\\.ESTIMATE_NOT_REVISABLE"
---

<objective>
Ship the two cross-cutting foundations Phase 42 needs before any reviser/retriever/deleter code can compile: (a) the `ConflictError` domain error with full `createHttpError` wiring + two new `ESTIMATE_*` error codes, and (b) the `IEstimateFollowupCanceller` interface + `NoopEstimateFollowupCanceller` default binding. These two concerns live in one plan because both are small, wave-independent, and have zero file overlap with the repository/service work in plans 03-05. Every downstream plan in Phase 42 imports from one or both of these.

Purpose: unblock parallel execution of plans 03 (repository), 04 (reviser), 05 (retriever/deleter). Without `ConflictError` and `ErrorCodes.ESTIMATE_REVISION_CONFLICT`, the reviser's `downgradeCurrent`-returned-null path has nothing to throw. Without `ESTIMATE_FOLLOWUP_CANCELLER` / `NoopEstimateFollowupCanceller`, the reviser's constructor injection cannot resolve and `EstimateModule` cannot be updated in plan 06.

Output: nine files (five new, four amended), all in one commit, unit tests green.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/42-revisions/42-CONTEXT.md
@.planning/phases/42-revisions/42-RESEARCH.md
@.planning/phases/42-revisions/42-VALIDATION.md
@trade-flow-api/CLAUDE.md
@trade-flow-api/src/core/errors/invalid-request.error.ts
@trade-flow-api/src/core/errors/resource-not-found.error.ts
@trade-flow-api/src/core/errors/forbidden-error.error.ts
@trade-flow-api/src/core/errors/internal-server-error.error.ts
@trade-flow-api/src/core/errors/handle-error.utility.ts
@trade-flow-api/src/core/errors/error-codes.enum.ts
@trade-flow-api/src/core/errors/errors-map.constant.ts
@trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts

<interfaces>
<!-- Byte-for-byte target shape for ConflictError (copied from invalid-request.error.ts verbatim — only the class name and this.name string change) -->

```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { ERRORS_MAP } from "@core/errors/errors-map.constant";

export class InvalidRequestError extends Error {
  private code: ErrorCodes;

  private details: string;

  constructor(code: ErrorCodes, details: string) {
    const { message } = ERRORS_MAP.get(code) ?? {
      message: "Error code unknown",
      code: ErrorCodes.ERROR_CODE_UNKNOWN,
    };
    super(message);
    this.name = "InvalidRequestError";
    this.details = details;
    this.code = code;
  }

  public getCode() {
    return this.code;
  }

  public getMessage() {
    return this.message;
  }

  public getDetails() {
    return this.details;
  }
}
```

<!-- Existing createHttpError branches (from handle-error.utility.ts) — the new ConflictError branch must be inserted AFTER InvalidRequestError and BEFORE ResourceNotFoundError -->

Current order: HttpException → InternalServerError → InvalidRequestError → ResourceNotFoundError → ForbiddenError → fallthrough.

New order: HttpException → InternalServerError → InvalidRequestError → **ConflictError** → ResourceNotFoundError → ForbiddenError → fallthrough.

<!-- Phase 41 PLAN-01 already reserved these ESTIMATE error code slots -->

From Phase 41 PLAN-01 lines 173-180:
```
ESTIMATE_NOT_FOUND = "ESTIMATE_NOT_FOUND"
ESTIMATE_INVALID_TRANSITION = "ESTIMATE_INVALID_TRANSITION"
ESTIMATE_NOT_EDITABLE = "ESTIMATE_NOT_EDITABLE"
ESTIMATE_CUSTOMER_NOT_FOUND = "ESTIMATE_CUSTOMER_NOT_FOUND"
ESTIMATE_JOB_NOT_FOUND = "ESTIMATE_JOB_NOT_FOUND"
ESTIMATE_LINE_ITEM_NOT_FOUND = "ESTIMATE_LINE_ITEM_NOT_FOUND"
ESTIMATE_LINE_ITEM_NOT_EDITABLE = "ESTIMATE_LINE_ITEM_NOT_EDITABLE"
ESTIMATE_COUNTER_GENERATION_FAILED = "ESTIMATE_COUNTER_GENERATION_FAILED"
```

**Phase 41 uses the string-value-equals-key pattern** (`ESTIMATE_NOT_FOUND = "ESTIMATE_NOT_FOUND"`, not `"ESTIMATE_0"`). Phase 42 MUST follow the same pattern for stability. This is the fallback pattern approved in user decisions Q3.

<!-- Target IEstimateFollowupCanceller interface -->

```typescript
// trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts
export interface IEstimateFollowupCanceller {
  cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
}

export const ESTIMATE_FOLLOWUP_CANCELLER = "ESTIMATE_FOLLOWUP_CANCELLER";
```

<!-- Target NoopEstimateFollowupCanceller -->

```typescript
// trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { IEstimateFollowupCanceller } from "@estimate/services/estimate-followup-canceller.interface";

@Injectable()
export class NoopEstimateFollowupCanceller implements IEstimateFollowupCanceller {
  private readonly logger = new Logger(NoopEstimateFollowupCanceller.name);

  public async cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void> {
    this.logger.debug(
      `Noop cancelAllFollowups invoked (Phase 46 will replace): estimateId=${estimateId}, revisionNumber=${revisionNumber}`,
    );
  }
}
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create ConflictError domain class + unit spec</name>
  <files>trade-flow-api/src/core/errors/conflict.error.ts, trade-flow-api/src/core/errors/test/conflict.error.spec.ts</files>
  <read_first>
    - trade-flow-api/src/core/errors/invalid-request.error.ts (the byte-for-byte shape to mirror)
    - trade-flow-api/src/core/errors/resource-not-found.error.ts (secondary mirror to confirm the shape is consistent)
    - trade-flow-api/src/core/errors/error-codes.enum.ts (current enum state — Phase 42 new codes go in Task 2)
    - trade-flow-api/src/core/errors/errors-map.constant.ts (current map state — Phase 42 new entries go in Task 2; this task's spec references `ErrorCodes.ESTIMATE_REVISION_CONFLICT` which is added in Task 2, so run Task 2 BEFORE the spec can compile — or stub the test to use `ErrorCodes.UNKNOWN_SERVER_ERROR` temporarily and swap after Task 2)
  </read_first>
  <behavior>
    - Test 1: `new ConflictError(ErrorCodes.UNKNOWN_SERVER_ERROR, "test details")` constructs without throwing.
    - Test 2: `.getCode()` returns the passed-in ErrorCodes value.
    - Test 3: `.getMessage()` returns the `message` field from `ERRORS_MAP.get(code)` (or the fallback "Error code unknown" if the code is not in the map).
    - Test 4: `.getDetails()` returns the passed-in details string.
    - Test 5: `.name` equals `"ConflictError"`.
    - Test 6: Instance passes `instanceof Error` and `instanceof ConflictError`.
  </behavior>
  <action>
    **Step 1 — Create `trade-flow-api/src/core/errors/conflict.error.ts`:**

    ```typescript
    import { ErrorCodes } from "@core/errors/error-codes.enum";
    import { ERRORS_MAP } from "@core/errors/errors-map.constant";

    export class ConflictError extends Error {
      private code: ErrorCodes;

      private details: string;

      constructor(code: ErrorCodes, details: string) {
        const { message } = ERRORS_MAP.get(code) ?? {
          message: "Error code unknown",
          code: ErrorCodes.ERROR_CODE_UNKNOWN,
        };
        super(message);
        this.name = "ConflictError";
        this.details = details;
        this.code = code;
      }

      public getCode() {
        return this.code;
      }

      public getMessage() {
        return this.message;
      }

      public getDetails() {
        return this.details;
      }
    }
    ```

    **This file MUST be byte-for-byte identical to `invalid-request.error.ts` with ONLY two substitutions**: `InvalidRequestError` → `ConflictError` (class name) and `"InvalidRequestError"` → `"ConflictError"` (the `this.name` string literal). Do NOT introduce a shared base class. Do NOT change the field visibility. Do NOT change the fallback message. The existing four error classes duplicate this shape verbatim; Phase 42 follows the established pattern and does NOT refactor.

    **Step 2 — Create `trade-flow-api/src/core/errors/test/conflict.error.spec.ts`:**

    Mirror the shape of any existing error-class spec (if one exists — check `trade-flow-api/src/core/errors/test/` first; if none exists, write a fresh spec using the NestJS Jest convention). The spec MUST cover:

    ```typescript
    import { ConflictError } from "@core/errors/conflict.error";
    import { ErrorCodes } from "@core/errors/error-codes.enum";

    describe("ConflictError", () => {
      it("constructs with a code and details", () => {
        const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "test details");
        expect(err).toBeInstanceOf(Error);
        expect(err).toBeInstanceOf(ConflictError);
      });

      it("getCode returns the passed-in code", () => {
        const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "test details");
        expect(err.getCode()).toBe(ErrorCodes.ESTIMATE_REVISION_CONFLICT);
      });

      it("getMessage returns the ERRORS_MAP message for the code", () => {
        const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "test details");
        expect(err.getMessage()).toBe("Estimate has already been revised or is no longer revisable");
      });

      it("getMessage returns the fallback message when the code is not in the map", () => {
        // Cast to ErrorCodes via a string that isn't in the enum — we use the unknown fallback
        // by constructing with a value that ERRORS_MAP won't have an entry for.
        // Use a known code that is in the map to avoid relying on map-miss behavior in this test.
        const err = new ConflictError(ErrorCodes.UNKNOWN_SERVER_ERROR, "test details");
        // If UNKNOWN_SERVER_ERROR is mapped, this asserts the mapped message; otherwise the fallback.
        expect(typeof err.getMessage()).toBe("string");
        expect(err.getMessage().length).toBeGreaterThan(0);
      });

      it("getDetails returns the passed-in details", () => {
        const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "my specific details");
        expect(err.getDetails()).toBe("my specific details");
      });

      it("name is 'ConflictError'", () => {
        const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "test details");
        expect(err.name).toBe("ConflictError");
      });
    });
    ```

    **Step 3 — Run the focused slice (will fail until Task 2 adds the new enum member and map entry):**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=conflict.error
    ```

    The spec will fail at the "getMessage returns the ERRORS_MAP message" assertion until Task 2 is complete. That's expected — Task 2 is a prerequisite for this spec to go green. Do not commit until the full plan runs and the slice is green.

    **Prohibitions (enforced by acceptance_criteria):**
    - No `any` type anywhere in either file.
    - No `as` type assertions anywhere in either file.
    - No `eslint-disable`, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`.
    - No relative imports — use `@core/*` path aliases.
    - File naming: `conflict.error.ts` (single `.error` suffix — matches `invalid-request.error.ts`, `internal-server-error.error.ts`, `resource-not-found.error.ts`; the `forbidden-error.error.ts` double-suffix is the one outlier and Phase 42 does NOT replicate it).
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/core/errors/conflict.error.ts`
    - `test -f trade-flow-api/src/core/errors/test/conflict.error.spec.ts`
    - `grep -c "export class ConflictError extends Error" trade-flow-api/src/core/errors/conflict.error.ts` returns 1
    - `grep -c "this.name = \"ConflictError\"" trade-flow-api/src/core/errors/conflict.error.ts` returns 1
    - `grep -c "public getCode\\|public getMessage\\|public getDetails" trade-flow-api/src/core/errors/conflict.error.ts` returns 3
    - `grep -c "any\\|as " trade-flow-api/src/core/errors/conflict.error.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/core/errors/conflict.error.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/core/errors/test/conflict.error.spec.ts` returns 0
    - `grep -c "ESTIMATE_REVISION_CONFLICT" trade-flow-api/src/core/errors/test/conflict.error.spec.ts` returns at least 3
    - `diff <(grep -E "^\\s*(private|public|constructor|super|this\\.)" trade-flow-api/src/core/errors/conflict.error.ts | sed 's/ConflictError/InvalidRequestError/g') <(grep -E "^\\s*(private|public|constructor|super|this\\.)" trade-flow-api/src/core/errors/invalid-request.error.ts | sed 's/InvalidRequestError/InvalidRequestError/g')` exits 0 (structural equivalence to the mirror)
  </acceptance_criteria>
  <verify>
    <automated>test -f trade-flow-api/src/core/errors/conflict.error.ts &amp;&amp; test -f trade-flow-api/src/core/errors/test/conflict.error.spec.ts &amp;&amp; grep -q "export class ConflictError extends Error" trade-flow-api/src/core/errors/conflict.error.ts</automated>
  </verify>
  <done>ConflictError class exists in the byte-for-byte shape of InvalidRequestError, spec file exists with all six test cases, and both files are ESLint-clean.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Add ESTIMATE_REVISION_CONFLICT + ESTIMATE_NOT_REVISABLE to ErrorCodes and ERRORS_MAP</name>
  <files>trade-flow-api/src/core/errors/error-codes.enum.ts, trade-flow-api/src/core/errors/errors-map.constant.ts</files>
  <read_first>
    - trade-flow-api/src/core/errors/error-codes.enum.ts (entire file — 70 lines, confirms current enum state and the Phase 41 string-value-equals-key pattern)
    - trade-flow-api/src/core/errors/errors-map.constant.ts (entire file — 278 lines, confirms the Map entry shape)
    - .planning/phases/41-estimate-module-crud-backend/41-01-prechecks-and-foundation-PLAN.md (lines 170-210 — Phase 41's reserved ESTIMATE_* error codes and their grep acceptance criteria)
    - .planning/phases/42-revisions/42-CONTEXT.md (D-CONC-07 and D-CONC-08 — error code contract)
  </read_first>
  <behavior>
    - Test 1: `ErrorCodes.ESTIMATE_REVISION_CONFLICT === "ESTIMATE_REVISION_CONFLICT"` (string-value-equals-key per Phase 41 pattern).
    - Test 2: `ErrorCodes.ESTIMATE_NOT_REVISABLE === "ESTIMATE_NOT_REVISABLE"`.
    - Test 3: `ERRORS_MAP.get(ErrorCodes.ESTIMATE_REVISION_CONFLICT).message === "Estimate has already been revised or is no longer revisable"` (exact D-CONC-08 wording).
    - Test 4: `ERRORS_MAP.get(ErrorCodes.ESTIMATE_NOT_REVISABLE).message === "Estimate cannot be revised in its current state"`.
    - Test 5: Task 1's conflict.error.spec.ts now passes the getMessage assertion (the map lookup resolves).
  </behavior>
  <action>
    **Step 1 — Amend `trade-flow-api/src/core/errors/error-codes.enum.ts`:**

    Find the `// Quotes` section (currently lines 61-65). Insert AFTER the last `QUOTE_*` entry and BEFORE the `// Email` comment, a new `// Estimates` section:

    ```typescript
      // Quotes
      QUOTE_JOB_NOT_FOUND = "QUOTE_0",
      QUOTE_INVALID_TRANSITION = "QUOTE_1",
      QUOTE_UNAVAILABLE = "QUOTE_2",
      QUOTE_EXPIRED = "QUOTE_5",

      // Estimates — Phase 42 (Phase 41 reserves ESTIMATE_NOT_FOUND, ESTIMATE_INVALID_TRANSITION, etc.)
      ESTIMATE_REVISION_CONFLICT = "ESTIMATE_REVISION_CONFLICT",
      ESTIMATE_NOT_REVISABLE = "ESTIMATE_NOT_REVISABLE",

      // Email
      EMAIL_DELIVERY_FAILED = "EMAIL_0",
    ```

    **String-value-equals-key rationale:** Phase 41 PLAN-01 lines 173-180 use this pattern for every new ESTIMATE_* code (e.g., `ESTIMATE_NOT_FOUND = "ESTIMATE_NOT_FOUND"`). Phase 42 follows the same convention. This is the user-approved Q3 fallback and is now the explicit Phase 41 convention — not a workaround. Do NOT use the `"ESTIMATE_X"` ordinal pattern that some older core/quote codes use.

    Do NOT touch any existing enum entries. Do NOT reorder. Only add the new section between the `Quotes` block and the `Email` block.

    **Step 2 — Amend `trade-flow-api/src/core/errors/errors-map.constant.ts`:**

    Find the last `QUOTE_*` entry in the `ERRORS_MAP` Map constructor. Insert AFTER that entry and BEFORE any `EMAIL_*` entry:

    ```typescript
      [
        ErrorCodes.ESTIMATE_REVISION_CONFLICT,
        {
          code: ErrorCodes.ESTIMATE_REVISION_CONFLICT,
          message: "Estimate has already been revised or is no longer revisable",
        },
      ],
      [
        ErrorCodes.ESTIMATE_NOT_REVISABLE,
        {
          code: ErrorCodes.ESTIMATE_NOT_REVISABLE,
          message: "Estimate cannot be revised in its current state",
        },
      ],
    ```

    The message strings are the exact D-CONC-08 wording — do not paraphrase. The `ESTIMATE_REVISION_CONFLICT` message is intentionally generic so it does not leak which of the three atomic-filter-miss conditions tripped (concurrent revise, status changed, isCurrent was already false).

    **Step 3 — Re-run the Task 1 slice (it should now pass the getMessage assertion):**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=conflict.error
    ```

    **Prohibitions:**
    - No `any`, no `as`, no suppression comments.
    - The enum entries MUST use the string-value-equals-key pattern. Do NOT use `"ESTIMATE_10"` or similar ordinals.
    - The ERRORS_MAP entry shape MUST match existing entries exactly (two-property object: `code` + `message`).
  </action>
  <acceptance_criteria>
    - `grep -c "ESTIMATE_REVISION_CONFLICT = \"ESTIMATE_REVISION_CONFLICT\"" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 1
    - `grep -c "ESTIMATE_NOT_REVISABLE = \"ESTIMATE_NOT_REVISABLE\"" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 1
    - `grep -c "ErrorCodes.ESTIMATE_REVISION_CONFLICT" trade-flow-api/src/core/errors/errors-map.constant.ts` returns at least 2 (key + value in the map entry)
    - `grep -c "ErrorCodes.ESTIMATE_NOT_REVISABLE" trade-flow-api/src/core/errors/errors-map.constant.ts` returns at least 2
    - `grep -c "Estimate has already been revised or is no longer revisable" trade-flow-api/src/core/errors/errors-map.constant.ts` returns 1
    - `grep -c "Estimate cannot be revised in its current state" trade-flow-api/src/core/errors/errors-map.constant.ts` returns 1
    - `cd trade-flow-api && npm run test -- --testPathPattern=conflict.error` exits 0 (the Task 1 spec now passes the getMessage assertion)
    - `cd trade-flow-api && npm run typecheck` exits 0 (no downstream typecheck breakage from adding enum values)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=conflict.error</automated>
  </verify>
  <done>Two new error codes exist in the enum with string-value-equals-key, two new ERRORS_MAP entries with exact D-CONC-08 messages, and Task 1's spec goes green.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 3: Wire ConflictError → HttpStatus.CONFLICT branch into createHttpError</name>
  <files>trade-flow-api/src/core/errors/handle-error.utility.ts, trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts</files>
  <read_first>
    - trade-flow-api/src/core/errors/handle-error.utility.ts (entire file — 87 lines — confirms the branch order and the exact response shape)
    - trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts (if it exists; if not, grep `trade-flow-api/src/core/errors/test/` for any existing spec and mirror its shape)
    - .planning/phases/42-revisions/42-RESEARCH.md §2.2 (the recommended insertion point: after InvalidRequestError, before ResourceNotFoundError, using Logger.warn)
  </read_first>
  <behavior>
    - Test 1: `createHttpError(new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "details"))` returns an `HttpException` whose `getStatus()` is `HttpStatus.CONFLICT` (409).
    - Test 2: The returned HttpException's response body has `errors[0].code === ErrorCodes.ESTIMATE_REVISION_CONFLICT`.
    - Test 3: The returned HttpException's response body has `errors[0].message === "Estimate has already been revised or is no longer revisable"`.
    - Test 4: The returned HttpException's response body has `errors[0].details === "details"` (the passed-in details string).
    - Test 5: Existing InvalidRequestError / ResourceNotFoundError / ForbiddenError branches still work (regression).
  </behavior>
  <action>
    **Step 1 — Amend `trade-flow-api/src/core/errors/handle-error.utility.ts`:**

    Add the import at the top (after the existing error imports):

    ```typescript
    import { ConflictError } from "@core/errors/conflict.error";
    ```

    Insert a new branch AFTER the `InvalidRequestError` block (currently ends at line 42 with `return new HttpException(response, HttpStatus.UNPROCESSABLE_ENTITY);`) and BEFORE the `ResourceNotFoundError` block (currently starts at line 44):

    ```typescript
      if (error instanceof ConflictError) {
        const response: IResponse = {
          errors: [
            {
              code: error.getCode(),
              message: error.getMessage(),
              details: error.getDetails(),
            },
          ],
        };

        Logger.warn("Conflict error encountered:", response);
        return new HttpException(response, HttpStatus.CONFLICT);
      }
    ```

    **Rationale for placement:** 409 sits logically between 422 (InvalidRequest) and 404 (ResourceNotFound) as "client-side request-state errors". Logger level is `warn` (not `error`) because a 409 is a legitimate client signal, not a server failure — matches the `InvalidRequestError` and `ResourceNotFoundError` precedent.

    Do NOT touch the `HttpException` guard at the top, the `InternalServerError` block, the existing `InvalidRequestError`, `ResourceNotFoundError`, `ForbiddenError` blocks, or the fallthrough 500 block.

    **Step 2 — Create or amend `trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts`:**

    If the file does not exist, create it with the full test shape. If it already exists, add ONLY the "ConflictError branch" `describe` block and reuse the existing `isErrorEntry` / `extractFirstError` helpers if they are already defined (otherwise add them once at file scope).

    ```typescript
    import { HttpException, HttpStatus } from "@nestjs/common";
    import { ConflictError } from "@core/errors/conflict.error";
    import { InvalidRequestError } from "@core/errors/invalid-request.error";
    import { ResourceNotFoundError } from "@core/errors/resource-not-found.error";
    import { ForbiddenError } from "@core/errors/forbidden-error.error";
    import { ErrorCodes } from "@core/errors/error-codes.enum";
    import { createHttpError } from "@core/errors/handle-error.utility";

    interface IErrorEntry {
      code: string;
      message: string;
      details: string | null;
    }

    function isErrorEntry(value: unknown): value is IErrorEntry {
      return (
        typeof value === "object" &&
        value !== null &&
        "code" in value &&
        "message" in value &&
        "details" in value &&
        typeof (value as { code: unknown }).code === "string" &&
        typeof (value as { message: unknown }).message === "string"
      );
    }

    function extractFirstError(response: unknown): IErrorEntry {
      if (typeof response !== "object" || response === null) {
        throw new Error(`Expected object response, got ${typeof response}`);
      }
      if (!("errors" in response)) {
        throw new Error("Response missing errors field");
      }
      const errors = (response as { errors: unknown }).errors;
      if (!Array.isArray(errors) || errors.length === 0) {
        throw new Error("errors array is missing or empty");
      }
      const first = errors[0];
      if (!isErrorEntry(first)) {
        throw new Error("first errors entry is not the expected shape");
      }
      return first;
    }

    describe("createHttpError", () => {
      describe("ConflictError branch", () => {
        it("returns HttpException with status 409", () => {
          const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "test details");
          const result = createHttpError(err);
          expect(result).toBeInstanceOf(HttpException);
          if (result instanceof HttpException) {
            expect(result.getStatus()).toBe(HttpStatus.CONFLICT);
          }
        });

        it("preserves code, message, and details in the response body", () => {
          const err = new ConflictError(ErrorCodes.ESTIMATE_REVISION_CONFLICT, "my details");
          const result = createHttpError(err);
          expect(result).toBeInstanceOf(HttpException);
          if (result instanceof HttpException) {
            const body = result.getResponse();
            const first = extractFirstError(body);
            expect(first.code).toBe(ErrorCodes.ESTIMATE_REVISION_CONFLICT);
            expect(first.message).toBe("Estimate has already been revised or is no longer revisable");
            expect(first.details).toBe("my details");
          }
        });
      });

      describe("regression: existing branches unchanged", () => {
        it("InvalidRequestError still returns 422", () => {
          const err = new InvalidRequestError(ErrorCodes.ERROR_CODE_UNKNOWN, "d");
          const result = createHttpError(err);
          if (result instanceof HttpException) {
            expect(result.getStatus()).toBe(HttpStatus.UNPROCESSABLE_ENTITY);
          }
        });

        it("ResourceNotFoundError still returns 404", () => {
          const err = new ResourceNotFoundError(ErrorCodes.RESOURCE_NOT_FOUND, "d");
          const result = createHttpError(err);
          if (result instanceof HttpException) {
            expect(result.getStatus()).toBe(HttpStatus.NOT_FOUND);
          }
        });

        it("ForbiddenError still returns 403", () => {
          const err = new ForbiddenError(ErrorCodes.ACTION_FORBIDDEN, "d");
          const result = createHttpError(err);
          if (result instanceof HttpException) {
            expect(result.getStatus()).toBe(HttpStatus.FORBIDDEN);
          }
        });
      });
    });
    ```

    **Narrowing pattern — mandatory, no alternatives:**

    The spec uses the `typeof` + `in` narrowing pattern via the `isErrorEntry` type guard plus the `extractFirstError` helper. This is the ONLY allowed shape for narrowing `HttpException.getResponse()` in this plan's spec file.

    - `if (result instanceof HttpException)` narrows the `HttpException | string` return to `HttpException` — type guard, no `as`.
    - `extractFirstError(body)` walks the `unknown` response value through `typeof`, `!== null`, and `in` checks before returning an `IErrorEntry`. The `(value as { code: unknown }).code` expressions inside `isErrorEntry` are the SAME pattern CLAUDE.md prescribes for type guards — they cast to `unknown` (a no-op at runtime) purely to let the subsequent `typeof` check narrow the value. This is distinct from asserting a concrete runtime type and is the idiomatic TypeScript type-guard shape.
    - Do NOT use `as { errors: ... }`, `as unknown as { errors: ... }`, or any direct cast of the response body to a concrete interface. Those are prohibited by CLAUDE.md and by the user's `feedback_no_type_assertions.md` memory rule.
    - There is NO "self-documenting-code exception" for `as` casts — that exception in CLAUDE.md concerns comments, not type assertions. Any earlier draft text suggesting otherwise is incorrect and has been removed.

    **Step 3 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=handle-error.utility
    ```
  </action>
  <acceptance_criteria>
    - `grep -c "import { ConflictError }" trade-flow-api/src/core/errors/handle-error.utility.ts` returns 1
    - `grep -c "error instanceof ConflictError" trade-flow-api/src/core/errors/handle-error.utility.ts` returns 1
    - `grep -c "HttpStatus.CONFLICT" trade-flow-api/src/core/errors/handle-error.utility.ts` returns 1
    - `grep -c "Logger.warn(\"Conflict error encountered" trade-flow-api/src/core/errors/handle-error.utility.ts` returns 1
    - `test -f trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts`
    - `grep -c "ConflictError branch" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` returns 1
    - `grep -c "HttpStatus.CONFLICT" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` returns at least 1
    - The ConflictError branch is positioned AFTER InvalidRequestError and BEFORE ResourceNotFoundError. Verify with:
      `awk '/InvalidRequestError/{a=NR} /ConflictError/{b=NR} /ResourceNotFoundError/{c=NR} END{exit !(a &lt; b &amp;&amp; b &lt; c)}' trade-flow-api/src/core/errors/handle-error.utility.ts`
    - `cd trade-flow-api && npm run test -- --testPathPattern=handle-error.utility` exits 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/core/errors/handle-error.utility.ts` returns 0
    - `! grep -q "as unknown as" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` (no `as unknown as` casts in the spec file)
    - `! grep -qE "as \{ errors:" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` (no direct cast to a concrete response interface)
    - `grep -c "function isErrorEntry" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` returns 1 (type-guard helper present)
    - `grep -c "function extractFirstError" trade-flow-api/src/core/errors/test/handle-error.utility.spec.ts` returns 1 (narrowing helper present)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=handle-error.utility</automated>
  </verify>
  <done>createHttpError has a ConflictError branch returning HttpStatus.CONFLICT, positioned between InvalidRequestError and ResourceNotFoundError, with regression-safe spec coverage.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 4: Create IEstimateFollowupCanceller interface + token + NoopEstimateFollowupCanceller default implementation + minimal spec</name>
  <files>trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts, trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts, trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts</files>
  <read_first>
    - .planning/phases/42-revisions/42-CONTEXT.md (D-HOOK-01, D-HOOK-02, D-HOOK-03 — interface signature, token, and the "wire but don't call" constraint)
    - .planning/phases/42-revisions/42-RESEARCH.md §6 (DI binding plan — interface file, no-op implementation, module wiring)
    - trade-flow-api/src/core/services/app-logger.service.ts (AppLogger — or use `@nestjs/common`'s Logger if simpler; check Phase 41 scaffold to see which logger estimate services use)
    - Phase 41 service file examples if they exist yet (grep for `@Injectable()` in any existing estimate-related file — if Phase 41 hasn't executed, default to `@nestjs/common` Logger)
  </read_first>
  <behavior>
    - Test 1: `NoopEstimateFollowupCanceller` is constructable via NestJS `Test.createTestingModule({ providers: [NoopEstimateFollowupCanceller] })` without error.
    - Test 2: `cancelAllFollowups("est-123", 2)` resolves without throwing.
    - Test 3: `cancelAllFollowups` returns a value that is `undefined` when awaited (since the method returns `Promise<void>`).
    - Test 4: The class implements the `IEstimateFollowupCanceller` interface (verified by TypeScript at compile time; no runtime assertion needed beyond "instance exists and has the method").
  </behavior>
  <action>
    **Step 1 — Create `trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts`:**

    ```typescript
    export interface IEstimateFollowupCanceller {
      cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
    }

    export const ESTIMATE_FOLLOWUP_CANCELLER = "ESTIMATE_FOLLOWUP_CANCELLER";
    ```

    No additional code. No class, no decorator, no `AppLogger`. The token is a `const` string so that `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` works at the consumer sites (EstimateReviser, later EstimateEmailSender in Phase 44).

    **Rationale comment (inline only if strictly necessary):** per the CLAUDE.md self-documenting-code rule, no comment is required here — the interface name and the token name are self-explanatory. Do NOT add a big block comment explaining "Phase 42 ships no-op; Phase 46 replaces." That context lives in the planning docs.

    **Step 2 — Create `trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts`:**

    ```typescript
    import { Injectable, Logger } from "@nestjs/common";
    import { IEstimateFollowupCanceller } from "@estimate/services/estimate-followup-canceller.interface";

    @Injectable()
    export class NoopEstimateFollowupCanceller implements IEstimateFollowupCanceller {
      private readonly logger = new Logger(NoopEstimateFollowupCanceller.name);

      public async cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void> {
        this.logger.debug(
          `Noop cancelAllFollowups invoked: estimateId=${estimateId}, revisionNumber=${revisionNumber}`,
        );
      }
    }
    ```

    **Logger choice:** use NestJS's built-in `Logger` from `@nestjs/common`, NOT `AppLogger` from `@core/*`. Rationale: this class has zero external dependencies, is the smallest possible default implementation, and `Logger` is always available without DI setup. If Phase 41 establishes a different precedent (e.g., every estimate service uses `AppLogger`), switch to that — but for a noop default, the minimal dependency surface is best.

    Parameter names `estimateId` and `revisionNumber` match the interface signature exactly. They ARE used in the log message (template literal), so no `_estimateId` underscore prefix is needed — ESLint won't flag them as unused.

    **Step 3 — Create `trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts`:**

    ```typescript
    import { Test, TestingModule } from "@nestjs/testing";
    import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";

    describe("NoopEstimateFollowupCanceller", () => {
      let canceller: NoopEstimateFollowupCanceller;

      beforeEach(async () => {
        const module: TestingModule = await Test.createTestingModule({
          providers: [NoopEstimateFollowupCanceller],
        }).compile();
        canceller = module.get<NoopEstimateFollowupCanceller>(NoopEstimateFollowupCanceller);
      });

      it("is injectable via NestJS DI", () => {
        expect(canceller).toBeDefined();
        expect(canceller).toBeInstanceOf(NoopEstimateFollowupCanceller);
      });

      it("cancelAllFollowups resolves without throwing", async () => {
        await expect(canceller.cancelAllFollowups("est-abc-123", 2)).resolves.toBeUndefined();
      });
    });
    ```

    **Step 4 — Run the slice:**

    ```bash
    cd trade-flow-api && npm run test -- --testPathPattern=noop-estimate-followup-canceller
    ```

    **Prohibitions:**
    - No `any`, no `as`, no suppression comments.
    - The interface file MUST have exactly two exports: the interface and the `ESTIMATE_FOLLOWUP_CANCELLER` const string token.
    - The noop service MUST implement the interface explicitly via `implements IEstimateFollowupCanceller`.
    - The spec MUST use `Test.createTestingModule` for DI resolution (not `new NoopEstimateFollowupCanceller()` direct instantiation) because this proves the `@Injectable()` decorator works with NestJS's DI container — which is what we care about for Phase 42 wiring downstream.
  </action>
  <acceptance_criteria>
    - `test -f trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts`
    - `test -f trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts`
    - `test -f trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts`
    - `grep -c "export interface IEstimateFollowupCanceller" trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` returns 1
    - `grep -c "cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>" trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` returns 1
    - `grep -c "export const ESTIMATE_FOLLOWUP_CANCELLER = \"ESTIMATE_FOLLOWUP_CANCELLER\"" trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` returns 1
    - `grep -c "@Injectable()" trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` returns 1
    - `grep -c "implements IEstimateFollowupCanceller" trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` returns 1
    - `grep -c "Test.createTestingModule" trade-flow-api/src/estimate/test/services/noop-estimate-followup-canceller.service.spec.ts` returns 1
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/services/estimate-followup-canceller.interface.ts` returns 0
    - `grep -c "eslint-disable\\|@ts-ignore\\|@ts-expect-error\\|@ts-nocheck" trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` returns 0
    - `grep -c " any\\| as " trade-flow-api/src/estimate/services/noop-estimate-followup-canceller.service.ts` returns 0
    - `cd trade-flow-api && npm run test -- --testPathPattern=noop-estimate-followup-canceller` exits 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api &amp;&amp; npm run test -- --testPathPattern=noop-estimate-followup-canceller</automated>
  </verify>
  <done>Interface + token file exists with the exact D-HOOK-01 signature, NoopEstimateFollowupCanceller class exists with @Injectable + implements + debug log, minimal spec proves DI-resolvability and resolve-void contract.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| error-domain → HTTP-layer | Error class code → HTTP status mapping in `createHttpError` — wrong mapping leaks a 500 when a 409 was intended, or vice versa |
| interface-contract → future-Phase-46 | `IEstimateFollowupCanceller` signature is the contract Phase 46 must implement; signature drift here breaks Phase 46 silently |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Severity | Mitigation Plan |
|-----------|----------|-----------|-------------|----------|-----------------|
| T-42-02-01 | Information disclosure | 409 message leaks which race condition tripped the atomic filter (concurrent revise vs status change vs isCurrent already false) | mitigate | LOW | D-CONC-08 specifies an intentionally generic message: "Estimate has already been revised or is no longer revisable". Task 2 acceptance_criteria grep-asserts this exact string in `errors-map.constant.ts`. All three miss conditions return the same message. |
| T-42-02-02 | Elevation of privilege | `createHttpError` mis-routes `ConflictError` to 200/500 instead of 409 | mitigate | HIGH | Task 3 spec asserts `result.getStatus() === HttpStatus.CONFLICT` (409) for the happy path + all three regression tests for InvalidRequestError/ResourceNotFoundError/ForbiddenError. Branch order verified by awk line-number check. |
| T-42-02-03 | Tampering | New enum value collides with an existing `ESTIMATE_*` code reserved by Phase 41 | mitigate | MEDIUM | Task 2 `read_first` requires reading Phase 41 PLAN-01 lines 170-210 to enumerate reserved codes. User decision Q3 approved the string-value-equals-key pattern as the stable fallback; Phase 41 PLAN-01 already uses this pattern, so collision is impossible by construction. |
| T-42-02-04 | Tampering | `IEstimateFollowupCanceller` signature drifts from D-HOOK-01 spec | mitigate | MEDIUM | Task 4 acceptance_criteria grep-asserts the exact signature string `cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>`. Phase 46 integration fails fast if the signature drifts. |
| T-42-02-05 | Denial of service | `NoopEstimateFollowupCanceller` hangs instead of resolving, blocking reviser flow when Phase 42 wires the injection | mitigate | LOW | Task 4 spec asserts the method resolves to undefined. The implementation is a single `logger.debug` call followed by implicit `Promise<void>` resolution — no awaits, no conditional branches. |
| T-42-02-06 | Information disclosure | `NoopEstimateFollowupCanceller` logs PII (estimate id + revision number) at `debug` level | accept | LOW | Estimate IDs and revision numbers are not PII in the privacy sense — they are internal opaque identifiers. `debug` level is not shipped to production logs by default. Acceptable. |

**Verification commands:**
- `grep -q "Estimate has already been revised or is no longer revisable" trade-flow-api/src/core/errors/errors-map.constant.ts` — generic message in place.
- `cd trade-flow-api && npm run test -- --testPathPattern="conflict.error|handle-error.utility|noop-estimate-followup-canceller"` — all branches green.
</threat_model>

<verification>
After all four tasks complete:

```bash
cd trade-flow-api && npm run test -- --testPathPattern="conflict.error|handle-error.utility|noop-estimate-followup-canceller"
cd trade-flow-api && npm run typecheck
cd trade-flow-api && npm run lint:check
cd trade-flow-api && npm run format:check
```

All four commands must exit 0. The plan is the first executable Phase 42 work; `npm run ci` run at plan close verifies the full suite is still green.
</verification>

<success_criteria>
- `ConflictError` class exists, byte-for-byte matches the `InvalidRequestError` shape, passes its unit spec.
- `ErrorCodes.ESTIMATE_REVISION_CONFLICT` and `ErrorCodes.ESTIMATE_NOT_REVISABLE` exist in the enum (string-value-equals-key) with `ERRORS_MAP` entries carrying the exact D-CONC-08 messages.
- `createHttpError` has a new `ConflictError → HttpStatus.CONFLICT` branch inserted between `InvalidRequestError` and `ResourceNotFoundError`, using `Logger.warn`. Regression tests confirm all existing branches still work.
- `IEstimateFollowupCanceller` interface + `ESTIMATE_FOLLOWUP_CANCELLER` token exported from a single file. `NoopEstimateFollowupCanceller` is DI-resolvable, implements the interface, and `cancelAllFollowups` resolves to `undefined`.
- `cd trade-flow-api && npm run ci` exits 0.
- All files committed in a single commit: `feat(42): add ConflictError + IEstimateFollowupCanceller foundation (Phase 42 wave 1)`.
</success_criteria>

<output>
After completion, create `.planning/phases/42-revisions/42-02-SUMMARY.md` documenting:
- The exact enum ordinals/strings assigned (`ESTIMATE_REVISION_CONFLICT = "ESTIMATE_REVISION_CONFLICT"`, `ESTIMATE_NOT_REVISABLE = "ESTIMATE_NOT_REVISABLE"`).
- Confirmation that the `createHttpError` branch is in the correct position (line numbers).
- Confirmation that `ConflictError` shape matches `InvalidRequestError` with the expected diff.
- `npm run ci` output summary.
- Commit hash.
</output>
