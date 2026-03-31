---
status: fixing
trigger: "Two Playwright auth setup tests fail: (1) API reachability check fails for localhost:3000, (2) page.evaluate throws TypeError: Failed to resolve module specifier 'firebase/app'"
created: 2026-03-28T00:00:00Z
updated: 2026-03-28T00:00:00Z
---

## Current Focus

hypothesis: Two independent bugs -- (1) fetch() in setup runs in Node.js but API may not be running OR env missing, (2) page.evaluate cannot use bare module specifiers like 'firebase/app' because injected code bypasses Vite's module resolution
test: Fix error 2 by using Firebase REST API instead of SDK inside page.evaluate; verify error 1 is env/runtime only
expecting: Both auth setup tests pass when API is running and Firebase REST auth is used
next_action: Rewrite auth.setup.ts to use Firebase REST API for authentication instead of page.evaluate with SDK imports

## Symptoms

expected: Playwright auth setup completes -- API health check passes, Firebase auth token obtained, storageState saved
actual: (1) API not reachable at http://localhost:3000; (2) page.evaluate TypeError Failed to resolve module specifier 'firebase/app'
errors: Error 1 - fetch to /v1/ping returns null/not-ok; Error 2 - bare module specifier cannot resolve in browser context outside Vite
reproduction: cd trade-flow-ui && npx playwright test
started: First time running phase 25 e2e tests

## Eliminated

- hypothesis: Wrong API URL in .env.e2e
  evidence: .env.e2e has E2E_API_URL=http://localhost:3000 which matches expected API port; playwright.config.ts loads .env.e2e via dotenv
  timestamp: 2026-03-28T00:00:00Z

## Evidence

- timestamp: 2026-03-28T00:01:00Z
  checked: auth.setup.ts lines 25-42 -- page.evaluate uses dynamic import('firebase/app')
  found: page.evaluate injects code directly into browser JS context. Bare module specifiers (firebase/app) only resolve through Vite's dev server module transform. Code injected via page.evaluate bypasses Vite entirely -- the browser's native ESM resolver cannot handle bare specifiers.
  implication: This is a fundamental approach error. Cannot use firebase SDK via page.evaluate. Must use alternative auth method.

- timestamp: 2026-03-28T00:02:00Z
  checked: .env.e2e contents vs .env.e2e.example
  found: .env.e2e is MISSING E2E_TEST_USER_EMAIL and E2E_TEST_USER_PASSWORD (required by first setup block line 34-35). Only has onboarding user credentials.
  implication: Even if firebase auth worked, first setup block would fail with undefined email/password.

- timestamp: 2026-03-28T00:03:00Z
  checked: Firebase config in src/config/firebase.ts
  found: App already initializes Firebase at module load. After page.goto("/"), the app's Firebase instance is alive in the browser. But page.evaluate still can't import firebase/app because it doesn't go through Vite.
  implication: Best fix is to use Firebase REST API (identitytoolkit.googleapis.com) from Node.js context, then inject the token into the page via the app's existing auth flow, OR navigate to login page and fill the form programmatically.

## Resolution

root_cause: Two bugs -- (1) .env.e2e is missing E2E_TEST_USER_EMAIL and E2E_TEST_USER_PASSWORD; (2) auth.setup.ts uses dynamic import('firebase/app') inside page.evaluate which runs in raw browser context that cannot resolve bare module specifiers (Vite's module transform is bypassed)
fix: Rewrite auth.setup.ts to authenticate via Firebase REST API (signInWithPassword endpoint) from Node.js context, then use the app's login page or inject credentials. Also document required .env.e2e vars.
verification: []
files_changed: []
