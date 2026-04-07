---
phase: quick-260407-orl
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/package.json
  - trade-flow-ui/vitest.config.ts
  - trade-flow-ui/vitest.setup.ts
  - trade-flow-ui/tsconfig.json
  - trade-flow-ui/src/lib/__tests__/utils.test.ts
autonomous: true
requirements: [quick-task]
must_haves:
  truths:
    - "npm run test executes vitest and exits cleanly"
    - "vitest resolves the existing @ path alias so imports work in tests"
    - "A sample test file proves the setup works end-to-end"
  artifacts:
    - path: "trade-flow-ui/vitest.config.ts"
      provides: "Vitest configuration with jsdom and path aliases"
    - path: "trade-flow-ui/vitest.setup.ts"
      provides: "Global test setup (jsdom cleanup, testing-library extensions)"
    - path: "trade-flow-ui/src/lib/__tests__/utils.test.ts"
      provides: "Proof-of-concept test exercising cn() utility"
  key_links:
    - from: "trade-flow-ui/vitest.config.ts"
      to: "trade-flow-ui/vite.config.ts"
      via: "mergeConfig or shared resolve.alias"
      pattern: "resolve\\.alias.*@"
---

<objective>
Set up Vitest as the unit test runner for trade-flow-ui so developers and Claude can write and run component/utility tests.

Purpose: The frontend currently has zero test infrastructure (confirmed in TESTING.md: "The frontend does not have tests configured"). This enables TDD and regression testing for UI code.
Output: Working vitest setup with path aliases, jsdom environment, testing-library integration, and a passing proof-of-concept test.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/codebase/TESTING.md
@.planning/codebase/STACK.md
@.planning/codebase/CONVENTIONS.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Install vitest dependencies and create configuration</name>
  <files>trade-flow-ui/package.json, trade-flow-ui/vitest.config.ts, trade-flow-ui/vitest.setup.ts, trade-flow-ui/tsconfig.json</files>
  <action>
In the trade-flow-ui repo:

1. Install dev dependencies:
   ```
   npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
   ```

2. Create `vitest.config.ts` at project root. Use `mergeConfig` from vitest to extend the existing `vite.config.ts` so path aliases (the `@` alias pointing to `./src`) are inherited automatically:
   ```ts
   import { defineConfig, mergeConfig } from "vitest/config";
   import viteConfig from "./vite.config";

   export default mergeConfig(
     viteConfig,
     defineConfig({
       test: {
         globals: true,
         environment: "jsdom",
         setupFiles: ["./vitest.setup.ts"],
         css: false,
         include: ["src/**/*.{test,spec}.{ts,tsx}"],
       },
     }),
   );
   ```

3. Create `vitest.setup.ts` at project root:
   ```ts
   import "@testing-library/jest-dom/vitest";
   ```
   This registers custom matchers like `toBeInTheDocument()`, `toHaveTextContent()`, etc.

4. Add test scripts to `package.json` scripts section:
   - `"test": "vitest run"` (single run, CI-friendly)
   - `"test:watch": "vitest"` (watch mode for development)
   - `"test:coverage": "vitest run --coverage"` (coverage report)

5. Update `tsconfig.json` (or `tsconfig.app.json` if that is where compilerOptions live) to include vitest global types. Add `"vitest/globals"` to `compilerOptions.types` array so TypeScript recognizes `describe`, `it`, `expect` without imports. If no `types` array exists, create one: `"types": ["vitest/globals"]`.

IMPORTANT: Do NOT modify the existing vite.config.ts -- extend it via mergeConfig in vitest.config.ts. This keeps build config and test config cleanly separated.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npx vitest run --passWithNoTests 2>&1 | tail -5</automated>
  </verify>
  <done>vitest runs successfully with no errors, jsdom environment loads, @testing-library/jest-dom matchers are available globally, and the @ path alias resolves correctly in the test environment.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Create proof-of-concept test for cn() utility</name>
  <files>trade-flow-ui/src/lib/__tests__/utils.test.ts</files>
  <behavior>
    - cn("px-2", "py-1") returns "px-2 py-1" (merges classes)
    - cn("px-2", "px-4") returns "px-4" (tailwind-merge deduplicates conflicting classes)
    - cn("px-2", undefined, "py-1") returns "px-2 py-1" (handles falsy values via clsx)
    - cn("px-2", false && "hidden") returns "px-2" (conditional class)
  </behavior>
  <action>
Create `src/lib/__tests__/utils.test.ts` in trade-flow-ui. This test exercises the existing `cn()` utility from `@/lib/utils` which combines clsx + tailwind-merge. The test file should:

1. Import `cn` from `@/lib/utils` (this also validates that the @ path alias works in tests).
2. Write a `describe("cn")` block with the behavior cases listed above.
3. Run the test to confirm RED-then-GREEN (the tests should pass immediately since cn() already exists -- the purpose is proving the test infrastructure works end-to-end).

Use `describe`, `it`, and `expect` as globals (no import needed thanks to vitest globals config from Task 1).
  </action>
  <verify>
    <automated>cd trade-flow-ui && npx vitest run src/lib/__tests__/utils.test.ts 2>&1 | tail -10</automated>
  </verify>
  <done>All 4 test cases pass. The test proves: vitest runs, @ path alias resolves, globals (describe/it/expect) work without imports, and the test infrastructure is fully operational.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

No trust boundaries affected -- this is dev tooling configuration only.

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-quick-01 | N/A | Dev tooling | accept | No production code changed; test deps are devDependencies only |
</threat_model>

<verification>
1. `cd trade-flow-ui && npm run test` exits 0 with passing tests
2. `cd trade-flow-ui && npx vitest run src/lib/__tests__/utils.test.ts` shows 4 passing assertions
3. No changes to vite.config.ts (build pipeline unaffected)
</verification>

<success_criteria>
- `npm run test` command exists and runs vitest successfully
- jsdom test environment is configured
- @testing-library/jest-dom matchers available (toBeInTheDocument, etc.)
- @ path alias works in test files (proven by importing from @/lib/utils)
- At least one test file with passing tests exists as proof-of-concept
</success_criteria>

<output>
After completion, create `.planning/quick/260407-orl-implement-the-capabilities-to-be-able-to/260407-orl-SUMMARY.md`
</output>
