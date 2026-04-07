---
status: awaiting_human_verify
trigger: "UI build fails with ~50+ TS2339 errors - jest-dom matchers not found on Vitest Assertion type"
created: 2026-04-07T00:00:00Z
updated: 2026-04-07T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED - tsconfig.app.json included test files in production build
test: npm run build and npm run test both pass
expecting: N/A - fix verified locally
next_action: Await human verification

## Symptoms

expected: `npm run build` (`tsc -b && vite build`) succeeds
actual: TypeScript compilation fails with ~50+ TS2339 errors across 7 test files
errors: Property 'toBeInTheDocument' does not exist on type 'Assertion<HTMLElement>'. Same for 'toBeDisabled' and 'toHaveAttribute'. Also one TS6133 unused import.
reproduction: Run `tsc -b` in trade-flow-ui directory
started: Recently added test files (phases 37-40 features)

## Eliminated

## Evidence

- timestamp: 2026-04-07
  checked: tsconfig.app.json include/exclude patterns
  found: `"include": ["src"]` with no exclude pattern - all files under src/ including __tests__/ directories are compiled by tsc -b
  implication: Test files are type-checked by the build tsconfig which has `"types": ["vitest/globals"]` but NOT the jest-dom type augmentation

- timestamp: 2026-04-07
  checked: vitest.setup.ts
  found: Imports `@testing-library/jest-dom/vitest` which augments Vitest Assertion type with jest-dom matchers
  implication: This augmentation only works at Vitest runtime, not during tsc -b compilation

- timestamp: 2026-04-07
  checked: Post-fix build and test results
  found: `tsc -b` produces zero errors, `npm run build` succeeds, all 9 test files (69 tests) pass
  implication: Fix is correct - excluding test files from build tsconfig resolves the issue without breaking tests

## Resolution

root_cause: tsconfig.app.json had `"include": ["src"]` with no exclude pattern, causing `tsc -b` to type-check test files under `src/**/__tests__/`. These test files use `@testing-library/jest-dom` matchers (toBeInTheDocument, toBeDisabled, toHaveAttribute) which augment the Vitest Assertion type. The augmentation is loaded via vitest.setup.ts at test runtime, but tsc -b does not process setup files, so TypeScript only sees the base Vitest Assertion type without jest-dom matchers.
fix: Added exclude pattern to tsconfig.app.json to skip test files during build: `"exclude": ["src/**/__tests__/**", "src/**/*.test.ts", "src/**/*.test.tsx", "src/**/*.spec.ts", "src/**/*.spec.tsx"]`
verification: `npm run build` succeeds (tsc -b + vite build), all 69 tests pass across 9 test files
files_changed: [tsconfig.app.json]
