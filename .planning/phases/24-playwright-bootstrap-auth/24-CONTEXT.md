# Phase 24: Playwright Bootstrap & Auth - Context

**Gathered:** 2026-03-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Install Playwright in `trade-flow-ui`, configure the test infrastructure, implement programmatic auth storageState reuse so any authenticated test can skip per-test login, and verify the foundation works with a smoke test.

Scope is infrastructure only — no test data seeding (Phase 25), no feature flow tests (Phases 25–28).

</domain>

<decisions>
## Implementation Decisions

### Firebase Auth

- **D-01:** Global setup uses the **programmatic approach** — call Firebase `signInWithPassword` REST endpoint to get an ID token, inject it into the browser via `signInWithCustomToken` (or equivalent auth state injection), then save `storageState` to disk. Do not click through the real login UI in the setup script.
- **D-02:** Auth is **pure infrastructure** in the global setup project. Individual tests load saved `storageState` and land on the dashboard without re-authenticating.
- **D-03:** The Phase 24 smoke test **does exercise the /login UI** — an unauthenticated visit to `/login` verifies the login form renders. This is a lightweight sanity check that the login page is reachable, separate from the programmatic auth setup.

### Selector Strategy

- **D-04:** Tests use **ARIA/role-based selectors by default** (`getByRole`, `getByLabel`, `getByText`). No React component changes required for the majority of tests.
- **D-05:** `data-testid` attributes are added **only where ARIA selectors are ambiguous** (e.g., multiple identical buttons on a page). Minimal component changes — add `data-testid` only when needed.

### webServer Config

- **D-06:** `playwright.config.ts` includes `webServer` config that **auto-starts the Vite dev server** (`reuseExistingServer: true`). Developer runs `npx playwright test` and Vite starts automatically if not already running.
- **D-07:** NestJS API + Docker must be started manually before running tests. A **global setup health check** pings the API before any test runs and aborts with a clear message if unreachable: `"API not reachable at http://localhost:3000 — start docker-compose and the NestJS server first."`

### Smoke Test

- **D-08:** Phase 24 smoke test has two assertions:
  1. Unauthenticated visit to `/login` shows the login form (verifies login page renders)
  2. Authenticated test loads saved `storageState` and lands on `/dashboard` (verifies auth storageState reuse works end-to-end)
- **D-09:** No API data assertions in the smoke test (e.g., business name appearing). That's Phase 25 territory once seeding infrastructure exists.

### Claude's Discretion

- Exact `storageState` file path (community standard is `e2e/.auth/user.json` — Claude can choose)
- Whether `e2e/.auth/` is gitignored (yes — contains credentials)
- How the Firebase test user credentials are passed to the setup script (env vars, `.env.test` file)
- Specific Playwright version to install
- Whether to use `test.extend()` for the auth fixture in Phase 24 or keep it simple and use `storageState` directly in `playwright.config.ts`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements

- `.planning/REQUIREMENTS.md` §FOUND-01, §FOUND-02 — Exact acceptance criteria for this phase (Playwright config shape, storageState behavior, directory structure, smoke test requirements)

### Research

- `.planning/research/SUMMARY.md` — Firebase v12 auth strategy analysis, fixture vs POM decision, API seeding approach, known pitfalls (Firebase rate limiting, IndexedDB persistence, email bypass). Read the "Gaps to Address" section specifically for Firebase auth implementation caveats.

### Codebase

- `.planning/codebase/TESTING.md` — Existing backend test patterns (Jest). Frontend has no tests yet — this phase creates the first test infrastructure.
- `.planning/codebase/CONVENTIONS.md` — Project conventions for naming and structure.

No external specs — requirements fully captured in decisions and references above.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- No existing Playwright or frontend test infrastructure — this phase creates it from scratch.
- `trade-flow-ui` has a `.env` / `.env.example` pattern for environment variables — follow the same convention for test-specific env vars (e.g. Firebase test credentials, API URL).

### Established Patterns

- Backend uses Jest with `spec.ts` test files in `test/` subdirectories. Frontend E2E tests follow a different convention: `e2e/` at the project root with `fixtures/`, `helpers/`, `tests/` subdirectories.
- Auth flow in production uses Firebase `onAuthStateChanged()` in `AuthProvider` — the programmatic auth approach must set auth state in a way that `AuthProvider` recognises (i.e., the Firebase SDK in the browser sees a valid auth session, not just a cookie).

### Integration Points

- `trade-flow-ui` package.json — `@playwright/test` and `@faker-js/faker` added as devDependencies
- `playwright.config.ts` at `trade-flow-ui` root — new file, not yet in codebase
- Firebase test user — must be pre-created in Firebase Console before this phase can run

</code_context>

<specifics>
## Specific Ideas

- The smoke test covers two distinct verification paths: unauthenticated (login form renders) and authenticated (storageState → dashboard). Both in a single spec file for Phase 24.
- Programmatic auth preferred specifically to avoid Firebase v12 IndexedDB/storageState uncertainty that the research flagged as MEDIUM confidence.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 24-playwright-bootstrap-auth*
*Context gathered: 2026-03-27*
