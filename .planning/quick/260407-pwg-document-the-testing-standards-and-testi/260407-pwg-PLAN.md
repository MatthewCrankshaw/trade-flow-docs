---
phase: quick
plan: 260407-pwg
type: execute
wave: 1
depends_on: []
files_modified:
  - CLAUDE.md
autonomous: true
requirements: []
must_haves:
  truths:
    - "CLAUDE.md documents both backend Jest and frontend Vitest testing setups"
    - "Frontend testing libraries (Vitest, Testing Library) listed under Frontend libraries"
    - "Line 177 no longer says 'Static analysis only' for frontend testing"
    - "A dedicated Testing section covers standards, patterns, and commands for both repos"
  artifacts:
    - path: "CLAUDE.md"
      provides: "Updated testing documentation"
      contains: "Vitest"
  key_links: []
---

<objective>
Add comprehensive testing documentation to CLAUDE.md covering both backend (Jest) and frontend (Vitest) testing setups, standards, and commands.

Purpose: The frontend recently gained Vitest testing capabilities (quick task 260407-orl) but CLAUDE.md still says "Frontend: Static analysis only". This update corrects that and adds a dedicated Testing section so Claude and developers know the testing tools, patterns, and conventions for both repos.

Output: Updated CLAUDE.md with testing documentation.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@CLAUDE.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Document testing standards and tools in CLAUDE.md</name>
  <files>CLAUDE.md</files>
  <action>
Make three edits to CLAUDE.md:

**Edit 1 — Add frontend Testing subsection under Frontend libraries (after line 67, the Authentication section):**

Insert a new `#### Testing` subsection under `### Frontend (trade-flow-ui)` between `#### Authentication` and `#### Development Tools`:

```
#### Testing
- **vitest 4.1.3** - Fast unit test runner (Vite-native)
- **@testing-library/react 16.3.2** - React component testing utilities
- **@testing-library/jest-dom 6.9.1** - Custom DOM matchers for assertions
- **@testing-library/user-event 14.6.1** - Simulates realistic user interactions
```

**Edit 2 — Fix line 177 (Key Technical Characteristics):**

Replace:
```
- Frontend: Static analysis only (ESLint, TypeScript)
```
With:
```
- Frontend: Vitest unit tests with Testing Library for component testing
```

**Edit 3 — Add a dedicated `## Testing` section AFTER the `## Key Technical Characteristics` section (before the `<!-- GSD:conventions-start -->` marker on line 187):**

Insert this new section:

```markdown
## Testing
### Backend (trade-flow-api) — Jest
- **Runner:** Jest 30.2.0 with ts-jest transformer
- **Environment:** Node
- **Libraries:** @nestjs/testing (test module), supertest (HTTP assertions)
- **File pattern:** `*.spec.ts`
- **Location:** `src/{feature}/test/` subdirectories mirroring source structure
  - `test/controllers/` — controller specs
  - `test/services/` — service specs (creator, retriever, updater, deleter)
  - `test/repositories/` — repository specs
  - `test/policies/` — policy specs
  - `test/mocks/` — shared mock factories
- **Path aliases:** Configured in jest moduleNameMapper (matches tsconfig paths)
- **Commands:**
  - `npm run test` — run all tests with coverage
  - `npm run test:watch` — watch mode
  - `npm run test:debug` — debug mode
- **Coverage output:** `../coverage/`

### Frontend (trade-flow-ui) — Vitest
- **Runner:** Vitest 4.1.3 (Vite-native, extends vite.config via mergeConfig)
- **Environment:** jsdom
- **Libraries:** @testing-library/react, @testing-library/jest-dom, @testing-library/user-event
- **Setup file:** `vitest.setup.ts` imports `@testing-library/jest-dom/vitest` for matchers
- **Globals:** Enabled (describe, it, expect available without imports)
- **File pattern:** `src/**/*.{test,spec}.{ts,tsx}`
- **Location:** `__tests__/` directories alongside source files
- **CSS:** Disabled in tests
- **Path aliases:** `@` alias inherited from vite config
- **Commands:**
  - `npm run test` — run all tests (vitest run)
  - `npm run test:watch` — watch mode
  - `npm run test:coverage` — run with coverage
```

IMPORTANT: Do NOT alter any content outside these three specific edit locations. The GSD markers (`<!-- GSD:stack-start -->`, `<!-- GSD:stack-end -->`) must remain intact and the new Testing section must go between `<!-- GSD:stack-end -->` content and `<!-- GSD:conventions-start -->`.
  </action>
  <verify>
    <automated>grep -n "vitest 4.1.3" CLAUDE.md && grep -n "Vitest unit tests with Testing Library" CLAUDE.md && grep -c "Static analysis only" CLAUDE.md | grep -q "^0$" && echo "PASS: All edits verified"</automated>
  </verify>
  <done>
    - Frontend Testing subsection lists Vitest 4.1.3, @testing-library/react, @testing-library/jest-dom, @testing-library/user-event
    - Line 177 reads "Frontend: Vitest unit tests with Testing Library for component testing" (not "Static analysis only")
    - Dedicated ## Testing section exists with both backend Jest and frontend Vitest subsections including runners, libraries, file patterns, locations, and commands
    - No other content in CLAUDE.md is altered
  </done>
</task>

</tasks>

<verification>
- `grep "Static analysis only" CLAUDE.md` returns nothing (old line removed)
- `grep "Vitest unit tests" CLAUDE.md` returns the updated Key Technical Characteristics line
- `grep -A 5 "#### Testing" CLAUDE.md` shows both backend and frontend testing library subsections
- `grep -A 30 "## Testing" CLAUDE.md` shows the dedicated testing standards section
</verification>

<success_criteria>
CLAUDE.md accurately documents the testing setup for both trade-flow-api (Jest) and trade-flow-ui (Vitest) with tools, patterns, file locations, and commands.
</success_criteria>

<output>
After completion, create `.planning/quick/260407-pwg-document-the-testing-standards-and-testi/260407-pwg-SUMMARY.md`
</output>
