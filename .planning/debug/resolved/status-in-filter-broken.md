---
status: diagnosed
trigger: "Investigate why the structured filter ?filter:status:in=Scheduled,Confirmed does not work on the schedule list endpoint"
created: 2026-03-01T00:00:00Z
updated: 2026-03-01T00:00:00Z
---

## Current Focus

hypothesis: Code is correct -- UAT failure was likely caused by external factors (stale build, test data, or HTTP client behavior), not a code bug
test: Exhaustive code tracing, runtime simulation, Express query parsing verification
expecting: Identify any code-level defect in the filter chain
next_action: Return diagnosis

## Symptoms

expected: `?filter:status:in=Scheduled,Confirmed` returns schedules with status "scheduled" or "confirmed"
actual: User reported "false" during UAT-02 Test 1 (exact observed behavior not detailed)
errors: None reported
reproduction: GET /v1/business/:businessId/schedule?filter:status:in=Scheduled,Confirmed
started: After 04-03 gap closure added structured filter parser

## Eliminated

- hypothesis: Express/qs splits comma-separated value into array, parser discards all but first element
  evidence: Express 5 uses "simple" query parser (Node querystring module). Verified via runtime check that `querystring.parse('filter:status:in=Scheduled,Confirmed')` produces `{"filter:status:in":"Scheduled,Confirmed"}` -- a single string, NOT an array. The `qs` library also preserves commas as strings by default (comma:false). The array-handling branch in parseStructuredFilters (line 34) is never triggered for comma-separated values.
  timestamp: 2026-03-01

- hypothesis: applyStatusFilter does not handle `in` operator or fails to split comma values
  evidence: Code at lines 36-43 explicitly handles `operator === "in"`, splits on comma, normalizes each value via toLowerCase(), and filters against VALID_SCHEDULE_STATUSES. Full runtime simulation confirmed `applyStatusFilter` produces `["scheduled","confirmed"]` from input `"Scheduled,Confirmed"`.
  timestamp: 2026-03-01

- hypothesis: Case normalization missing for `in` operator (works for `eq` but not `in`)
  evidence: Both `eq` and `in` branches use the same `normalize()` function which calls `.trim().toLowerCase()`. The normalize function is defined once (lines 31-33) and used by both branches.
  timestamp: 2026-03-01

- hypothesis: Repository buildFilter generates incorrect MongoDB query for status array
  evidence: buildFilter lines 131-133 produce `{ status: { $in: filters.status } }` which is correct MongoDB $in syntax. Verified the filter DTO status field is `ScheduleStatus[]` and the repository correctly wraps it in `$in`.
  timestamp: 2026-03-01

- hypothesis: ValidationPipe or middleware strips/transforms query parameters
  evidence: Controller uses `@Req()` to access raw request object, not `@Query()` with DTO class. ValidationPipe only applies to decorated DTOs. No middleware or interceptors found that modify query params. paginationQueryToBaseQueryOptions only reads `query.page` and `query.limit`, does not mutate the query object.
  timestamp: 2026-03-01

- hypothesis: Compiled dist/ code does not match source
  evidence: Compared dist/schedule/controllers/schedule.controller.js and dist/core/filters/parse-structured-filters.utility.js against their TypeScript sources. Both match exactly -- same logic, same flow, same array handling.
  timestamp: 2026-03-01

## Evidence

- timestamp: 2026-03-01
  checked: Express version and query parser
  found: Express 5.2.1 is installed. Express 5 defaults to "simple" query parser (Node's querystring module). NestJS runtime confirmed to use "simple" parser via `app.get('query parser')`.
  implication: Query parsing uses Node's built-in querystring, not the qs library. Both parsers handle `filter:status:in=Scheduled,Confirmed` identically (single string value).

- timestamp: 2026-03-01
  checked: Node querystring.parse behavior
  found: `querystring.parse('filter:status:in=Scheduled,Confirmed')` produces `{"filter:status:in":"Scheduled,Confirmed"}` (single string). Commas are NOT treated as delimiters by querystring.
  implication: The raw query value reaching parseStructuredFilters is always a single string with embedded commas for the `in` operator.

- timestamp: 2026-03-01
  checked: Full chain simulation (compiled dist code)
  found: Loaded actual compiled modules and ran the complete chain: parseStructuredFilters -> applyStatusFilter -> filter DTO output. Input `{"filter:status:in":"Scheduled,Confirmed"}` produces `{status:["scheduled","confirmed"]}`. All 4 UAT test scenarios simulated -- ALL PASS including Test 1 (the failing one).
  implication: The code is correct. The filter chain produces the expected output for ALL inputs matching the UAT test descriptions.

- timestamp: 2026-03-01
  checked: All 14 existing unit tests
  found: `npx jest --testPathPatterns="parse-structured-filters"` -- 14/14 tests pass. The parser correctly preserves comma-separated values as single strings.
  implication: Parser behavior is verified and correct.

- timestamp: 2026-03-01
  checked: UAT-02 test results comparison
  found: Test 1 (in with multiple values: "Scheduled,Confirmed") FAILED. Test 4 (combined: in with single value "Scheduled" + bt) PASSED. The key difference is Test 1 uses comma-separated multiple values while Test 4 uses a single value with the `in` operator.
  implication: The failure is specific to the multi-value `in` case, but code analysis shows this case is handled correctly.

- timestamp: 2026-03-01
  checked: UAT-02 Test 1 user report detail
  found: User simply reported "false" with no additional detail about what they actually observed (no error messages, no response body, no details about their HTTP client or test method).
  implication: Cannot determine the exact runtime behavior the user experienced. The failure report is binary with no diagnostic information.

- timestamp: 2026-03-01
  checked: dist/ compiled output vs source code
  found: dist/schedule/controllers/schedule.controller.js matches source exactly. dist/core/filters/parse-structured-filters.utility.js matches source exactly.
  implication: No stale build artifacts. The compiled code matches the source.

- timestamp: 2026-03-01
  checked: UI codebase for filter usage
  found: No references to `filter:status:in` or structured filter query params anywhere in trade-flow-ui. The schedule filtering UI has not been implemented yet.
  implication: User tested via direct HTTP request tool (Postman, curl, browser). The exact request format they used is unknown.

## Resolution

root_cause: |
  NO CODE DEFECT FOUND. After exhaustive investigation:

  1. The generic parser (parseStructuredFilters) correctly preserves comma-separated values as a single string
  2. The controller's applyStatusFilter correctly splits on comma, normalizes to lowercase, and validates against the enum
  3. The repository's buildFilter correctly produces a MongoDB $in query
  4. Express 5's "simple" query parser correctly delivers comma-separated values as single strings
  5. The compiled dist/ output matches the TypeScript source
  6. Full runtime simulation of the compiled code confirms correct output for all UAT test inputs

  The UAT failure (user reported "false") cannot be reproduced via code analysis or runtime simulation. The most likely explanations for the user's experience are:

  A. **Test data issue** -- The user's database may not have had schedules with matching statuses, making it appear the filter didn't work (returned empty results or all results if no filter match)
  B. **Application not restarted** -- The user may have tested against a running instance that was started before the 04-03 commits, running stale compiled code
  C. **HTTP client encoding** -- The user's testing tool may have encoded the URL in an unexpected way (e.g., URL-encoding the comma as %2C which querystring does handle correctly, OR using repeated params like `?filter:status:in=Scheduled&filter:status:in=Confirmed` which WOULD trigger the array-first-element behavior)
  D. **Wrong endpoint or query format** -- The user may have used the old ad-hoc format or a different URL pattern

  If the issue is scenario C (repeated params), there IS a latent design weakness: the parser's `Array.isArray(rawValue) ? rawValue[0] : rawValue` on line 34 silently discards repeated query param values. For the `in` operator, this could cause data loss if a client sends `?filter:status:in=Scheduled&filter:status:in=Confirmed` instead of `?filter:status:in=Scheduled,Confirmed`. However, this is the documented and tested behavior, not a bug per se.

fix: (not applied - diagnosis only)
verification: (not applied - diagnosis only)
files_changed: []
