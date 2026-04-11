---
phase: 41-estimate-module-crud-backend
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-api/tsconfig.json
  - trade-flow-api/src/core/errors/error-codes.enum.ts
autonomous: false
requirements: [EST-02, EST-04, EST-05, EST-08, EST-09]
must_haves:
  truths:
    - Production MongoDB `quote_tokens` collection is verified empty before the token rename proceeds (or the operator has explicitly acknowledged data loss risk)
    - `tsconfig.json` contains path aliases `@estimate/*`, `@estimate-test/*`, `@document-token/*`, `@document-token-test/*`
    - `tsconfig.json` no longer contains `@quote-token/*` or `@quote-token-test/*`
    - `ErrorCodes` enum contains all new estimate/document-token error codes needed by downstream plans
    - `cd trade-flow-api && npm run ci` exits 0 after the foundation commit
  artifacts:
    - path: trade-flow-api/tsconfig.json
      provides: Path aliases for new modules
      contains: "@estimate/*"
    - path: trade-flow-api/src/core/errors/error-codes.enum.ts
      provides: New ESTIMATE_* and DOCUMENT_TOKEN_* error codes
      contains: ESTIMATE_INVALID_TRANSITION
  key_links:
    - from: trade-flow-api/tsconfig.json
      to: "@estimate/* source tree"
      via: paths compilerOption
      pattern: "@estimate/\\*"
    - from: trade-flow-api/src/core/errors/error-codes.enum.ts
      to: "services throwing InvalidRequestError / ResourceNotFoundError with estimate codes"
      via: enum member reference
      pattern: "ESTIMATE_(INVALID_TRANSITION|NOT_EDITABLE|NOT_FOUND|CUSTOMER_NOT_FOUND|JOB_NOT_FOUND)"
---

<objective>
Establish the Phase 41 foundation in a single atomic commit: (1) verify the BLOCKING prod-data assumption A1 from RESEARCH.md (the `quote_tokens` collection must be empty before the pure code rename can proceed), (2) update `tsconfig.json` path aliases to add `@estimate/*` / `@document-token/*` and remove `@quote-token/*`, and (3) extend the central `ErrorCodes` enum with every new `ESTIMATE_*` and `DOCUMENT_TOKEN_*` code that downstream plans in this phase need to reference.

Purpose: Downstream plans (02-08) depend on stable path aliases and defined error codes. Doing this first in one commit lets every later wave compile cleanly against the new alias namespace and throw typed domain errors without adding placeholder strings. The prod-data check is a hard blocker — if `quote_tokens` has documents, the rest of the phase must absorb a data migration before the token rename can ship.

Output: Updated `tsconfig.json`, extended `error-codes.enum.ts`, and a verified-or-acknowledged prod-data status.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md

<interfaces>
<!-- Exact tsconfig path-alias block to ADD (from RESEARCH.md §Architecture Patterns). -->
<!-- Existing `@quote/*`, `@quote-test/*` aliases MUST be preserved. -->
<!-- Only `@quote-token/*` and `@quote-token-test/*` entries are removed. -->

New aliases (add to tsconfig.json compilerOptions.paths):
- "@estimate/*": ["src/estimate/*"]
- "@estimate-test/*": ["src/estimate/test/*"]
- "@document-token/*": ["src/document-token/*"]
- "@document-token-test/*": ["src/document-token/test/*"]

Removed aliases:
- "@quote-token/*": ["src/quote-token/*"]
- "@quote-token-test/*": ["src/quote-token/test/*"]

Existing ErrorCodes prefix convention (from `src/core/errors/error-codes.enum.ts`):
- Each domain has its own prefix: QUOTE_*, SCHEDULE_*, ITEM_*, CUSTOMER_*, etc.
- Values are UPPER_SNAKE_CASE string literals matching the key.
</interfaces>
</context>

<tasks>

<task type="checkpoint:human-action" gate="blocking">
  <name>Task 1: Verify production quote_tokens collection is empty (BLOCKING PRE-CHECK)</name>
  <files>none (read-only verification against production MongoDB)</files>
  <read_first>
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Runtime State Inventory and §Assumptions Log A1, §Open Questions #1)
    - .planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md (D-TKN-01 states "Application has no production data to preserve")
  </read_first>
  <what-built>Pre-execution verification that the BLOCKING assumption A1 from RESEARCH.md holds. No files are modified in this task.</what-built>
  <action>
Coordinate with the operator to run the production MongoDB count query against `quote_tokens`. This task is a blocking manual gate: the operator runs the query locally and enters the resume signal. The executor's role is to pause execution, present the instructions from <how-to-verify>, wait for the signal, record the result in the plan summary, and only then proceed. Do NOT attempt to run the production query from the build environment — the MongoDB connection string is operator-only.
  </action>

  <how-to-verify>
    1. Obtain the production MongoDB connection string from Railway environment (variable `MONGO_URL` on the trade-flow-api service).
    2. Run exactly this command against production:
       `mongosh "$MONGO_URL" --quiet --eval 'db.quote_tokens.countDocuments({})'`
       (if mongosh is not available, use `mongo` or the Railway CLI `railway run` with the MongoDB CLI).
    3. Expected result: the number `0`.
    4. If the result is `0`: type "approved — quote_tokens is empty, proceed with pure code rename" to resume.
    5. If the result is `> 0`: STOP. Do not proceed. The token rename in Plan 03 is a pure code rename per D-TKN-01 and will break existing customer links. The phase must be re-planned to absorb a one-shot data migration (currently deferred per CONTEXT.md). Report the count back to the planner for re-planning. The operator may explicitly override and accept data loss by typing "override — accept data loss" — but this is a last-resort escape hatch, not a default.
    6. Record the verification result (count, timestamp, operator name) in a comment on the commit for Plan 03.
  </how-to-verify>
  <resume-signal>Type exactly "approved — quote_tokens is empty, proceed with pure code rename" OR "override — accept data loss, acknowledged"</resume-signal>
  <acceptance_criteria>
    - Operator provided the resume signal with one of the two exact strings above
    - The verification result is captured in the plan summary for audit
    - If the count was > 0 and no override was given, execution halts and the planner is notified
  </acceptance_criteria>
  <verify>
    <automated>MISSING — this is a manual pre-check gate per D-TKN-01 and RESEARCH.md Open Question #1; automation would require connecting to production MongoDB which is operator-only</automated>
  </verify>
  <done>Operator has verified production `quote_tokens` count is 0 (or explicitly accepted data loss) and provided the resume signal</done>
</task>

<task type="auto">
  <name>Task 2: Add estimate and document-token path aliases to tsconfig.json; remove quote-token aliases</name>
  <files>trade-flow-api/tsconfig.json</files>
  <read_first>
    - trade-flow-api/tsconfig.json (current path alias block — executor must see the existing shape before editing)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Architecture Patterns → Recommended Project Structure block that lists required alias changes; §Assumptions Log A6 confirms no `@estimate/*` entry present today)
  </read_first>
  <action>
Edit `trade-flow-api/tsconfig.json`. Locate `compilerOptions.paths`. Apply these mutations verbatim:

**ADD these four entries** (preserve alphabetical ordering if the existing block is ordered; otherwise append after `@quote-test/*`):
```
"@document-token/*": ["src/document-token/*"],
"@document-token-test/*": ["src/document-token/test/*"],
"@estimate/*": ["src/estimate/*"],
"@estimate-test/*": ["src/estimate/test/*"],
```

**REMOVE these two entries** (they will no longer resolve after Plan 03 deletes `src/quote-token/`):
```
"@quote-token/*": ["src/quote-token/*"],
"@quote-token-test/*": ["src/quote-token/test/*"],
```

Do NOT touch any other aliases (`@core/*`, `@quote/*`, `@item/*`, `@customer/*`, `@business/*`, `@job/*`, `@tax-rate/*`, `@user/*`, `@auth/*`, `@email/*`, `@ping/*`, `@migration/*`, `@core-test/*`, `@business-test/*`, `@quote-test/*` — all untouched).

After the edit, run `cd trade-flow-api && npm run typecheck` and confirm it still passes. (It WILL still pass because `src/quote-token/` still exists at this point; Plan 03 handles the actual rename.) If typecheck fails, the executor added a typo — fix before committing.

No narrative comments in the JSON. Commit message: `refactor(41): add estimate and document-token tsconfig aliases, remove quote-token aliases`.
  </action>
  <acceptance_criteria>
    - `grep -n "@estimate/\\*" trade-flow-api/tsconfig.json` returns exactly one match
    - `grep -n "@estimate-test/\\*" trade-flow-api/tsconfig.json` returns exactly one match
    - `grep -n "@document-token/\\*" trade-flow-api/tsconfig.json` returns exactly one match
    - `grep -n "@document-token-test/\\*" trade-flow-api/tsconfig.json` returns exactly one match
    - `grep -n "@quote-token/\\*" trade-flow-api/tsconfig.json` returns zero matches
    - `grep -n "@quote-token-test/\\*" trade-flow-api/tsconfig.json` returns zero matches
    - `grep -c "@quote/\\*" trade-flow-api/tsconfig.json` returns 1 (preserved)
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `cd trade-flow-api && npm run format:check -- tsconfig.json` exits 0 (or format:check is not configured for JSON — in which case `npm run ci` verifies this in Task 4)
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck</automated>
  </verify>
  <done>tsconfig.json contains the four new aliases, removes the two quote-token aliases, and typecheck passes</done>
</task>

<task type="auto">
  <name>Task 3: Extend ErrorCodes enum with estimate and document-token codes</name>
  <files>trade-flow-api/src/core/errors/error-codes.enum.ts</files>
  <read_first>
    - trade-flow-api/src/core/errors/error-codes.enum.ts (current enum — executor must see existing `QUOTE_*` entries to match naming convention)
    - trade-flow-api/src/quote/services/quote-updater.service.ts (example of how `QUOTE_NOT_EDITABLE` is thrown, to confirm the naming and error-shape convention)
    - .planning/phases/41-estimate-module-crud-backend/41-RESEARCH.md (§Assumptions Log A8 and §Common Pitfalls #9)
  </read_first>
  <action>
Open `trade-flow-api/src/core/errors/error-codes.enum.ts`. Locate the `ErrorCodes` enum. After the existing `QUOTE_*` group (and matching its ordering/casing conventions — key and value both UPPER_SNAKE_CASE strings matching each other), add these entries:

```typescript
ESTIMATE_NOT_FOUND = "ESTIMATE_NOT_FOUND",
ESTIMATE_INVALID_TRANSITION = "ESTIMATE_INVALID_TRANSITION",
ESTIMATE_NOT_EDITABLE = "ESTIMATE_NOT_EDITABLE",
ESTIMATE_CUSTOMER_NOT_FOUND = "ESTIMATE_CUSTOMER_NOT_FOUND",
ESTIMATE_JOB_NOT_FOUND = "ESTIMATE_JOB_NOT_FOUND",
ESTIMATE_LINE_ITEM_NOT_FOUND = "ESTIMATE_LINE_ITEM_NOT_FOUND",
ESTIMATE_LINE_ITEM_NOT_EDITABLE = "ESTIMATE_LINE_ITEM_NOT_EDITABLE",
ESTIMATE_COUNTER_GENERATION_FAILED = "ESTIMATE_COUNTER_GENERATION_FAILED",
```

Then, after the existing `QUOTE_TOKEN_*` entries (if any — if there are no explicit quote-token error codes today, add the block after the estimate block), add these document-token entries:

```typescript
DOCUMENT_TOKEN_NOT_FOUND = "DOCUMENT_TOKEN_NOT_FOUND",
DOCUMENT_TOKEN_EXPIRED = "DOCUMENT_TOKEN_EXPIRED",
DOCUMENT_TOKEN_REVOKED = "DOCUMENT_TOKEN_REVOKED",
DOCUMENT_TOKEN_TYPE_MISMATCH = "DOCUMENT_TOKEN_TYPE_MISMATCH",
```

If any `QUOTE_TOKEN_*` entries already exist, LEAVE THEM IN PLACE for this plan (Plan 03 will rename them). Do NOT modify existing entries in this task. Do NOT add narrative comments — the enum keys are self-documenting per the comments standard in CLAUDE.md.

Run `cd trade-flow-api && npm run typecheck` and `cd trade-flow-api && npm run lint:check` to confirm the enum compiles and lints cleanly. Commit message: `feat(41): add estimate and document-token error codes`.
  </action>
  <acceptance_criteria>
    - `grep -c "ESTIMATE_NOT_FOUND" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2 (key + string literal)
    - `grep -c "ESTIMATE_INVALID_TRANSITION" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_NOT_EDITABLE" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_CUSTOMER_NOT_FOUND" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_JOB_NOT_FOUND" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_LINE_ITEM_NOT_FOUND" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_LINE_ITEM_NOT_EDITABLE" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "ESTIMATE_COUNTER_GENERATION_FAILED" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "DOCUMENT_TOKEN_NOT_FOUND" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "DOCUMENT_TOKEN_EXPIRED" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "DOCUMENT_TOKEN_REVOKED" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `grep -c "DOCUMENT_TOKEN_TYPE_MISMATCH" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 2
    - `cd trade-flow-api && npm run typecheck` exits 0
    - `cd trade-flow-api && npm run lint:check` exits 0
    - `grep -c "@ts-ignore\\|@ts-expect-error\\|eslint-disable" trade-flow-api/src/core/errors/error-codes.enum.ts` returns 0
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run typecheck && npm run lint:check</automated>
  </verify>
  <done>ErrorCodes enum contains all 12 new entries, compiles, and lints cleanly</done>
</task>

<task type="auto">
  <name>Task 4: Run full CI gate to confirm foundation is green</name>
  <files>none (verification only)</files>
  <read_first>
    - trade-flow-api/package.json (confirm the `ci` script command is `npm run test -- --passWithNoTests && npm run lint:check && npm run format:check && npm run typecheck`)
  </read_first>
  <action>
Run the full API CI gate from the trade-flow-api repo root:

```
cd trade-flow-api && npm run ci
```

Expected result: exit code 0. Output must show all four stages passed (test, lint:check, format:check, typecheck).

If any stage fails:
- **test failure:** unlikely at this point (no tests were added) — if something fails, it is a real regression in the ErrorCodes enum breaking an existing spec. Fix the enum, do not suppress the test.
- **lint failure:** check ErrorCodes enum entries for trailing whitespace, wrong casing, or inconsistent formatting with existing entries. Fix in place.
- **format:check failure:** run `cd trade-flow-api && npm run format` to auto-fix, then re-commit with message `chore(41): prettier fixes for foundation`.
- **typecheck failure:** the executor introduced a typo in tsconfig or the enum. Fix before proceeding.

Do NOT use `@ts-ignore`, `eslint-disable`, or `--force` to bypass a failure. Fix the root cause per CLAUDE.md CI gate policy. Plans 02-08 depend on a clean baseline.
  </action>
  <acceptance_criteria>
    - `cd trade-flow-api && npm run ci` exits 0
    - Output contains lines confirming: tests pass, lint:check passes, format:check passes, typecheck passes
    - No `eslint-disable`, `@ts-ignore`, or `@ts-expect-error` markers were added to the diff
  </acceptance_criteria>
  <verify>
    <automated>cd trade-flow-api && npm run ci</automated>
  </verify>
  <done>Full CI gate green on the foundation commit</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| planner → production MongoDB | Pre-check reads production data; operator-only gate |
| developer → tsconfig.json / error-codes.enum.ts | Configuration changes consumed transitively by every module |

## STRIDE Threat Register

Reference: RESEARCH.md §Security Domain.

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-41-01-01 | Tampering | Pure code rename (Plan 03) over existing prod `quote_tokens` data | mitigate | BLOCKING pre-check in Task 1 verifies `db.quote_tokens.countDocuments({}) === 0` before the rename; operator explicit override required to proceed with data loss |
| T-41-01-02 | Information Disclosure | Production connection string handled by operator during pre-check | accept | Read-only query; operator uses Railway-provisioned `MONGO_URL`; no credentials logged to commit |
| T-41-01-03 | Tampering | Partial path-alias update leaves dangling imports | mitigate | Task 2 runs `npm run typecheck` post-edit; Task 4 runs full `npm run ci` to catch any cross-module breakage |
| T-41-01-04 | Elevation of Privilege | New ESTIMATE_* error codes with wrong HTTP mapping | mitigate | Codes are string literals only in this plan; actual HTTP status mapping happens in `createHttpError` (existing utility) and domain error classes (existing); new codes inherit existing safe defaults (422 for InvalidRequest, 404 for ResourceNotFound) |
| T-41-01-05 | Information Disclosure | New error messages could leak business ids or internal state | mitigate | Downstream plans must use the short human-readable messages listed in CONTEXT.md D-DRAFT-01 style ("Estimate can only be edited in Draft status") — enforced by existing `createErrorResponse` sanitization |
</threat_model>

<verification>
1. `cd trade-flow-api && npm run ci` exits 0 on the foundation commit.
2. All four new estimate path aliases and all 12 new error codes are present in their respective files.
3. All six `quote-token` path aliases are gone.
4. Operator has confirmed prod `quote_tokens` is empty (or explicitly accepted data loss).
</verification>

<success_criteria>
- Plan 02 and Plan 03 can begin Wave 2 against a clean baseline
- Error codes needed by Plans 05-08 exist in the central enum
- No path-alias imports in downstream plans will fail compilation
- The BLOCKING assumption A1 is resolved before the token rename proceeds
</success_criteria>

<output>
After completion, create `.planning/phases/41-estimate-module-crud-backend/41-01-SUMMARY.md` capturing:
- The pre-check result (count, operator name, timestamp)
- The exact tsconfig paths added/removed
- The list of new ErrorCodes entries
- The final `npm run ci` output
</output>
