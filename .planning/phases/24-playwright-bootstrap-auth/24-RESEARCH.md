# Phase 24: Playwright Bootstrap & Auth - Research

**Researched:** 2026-03-27
**Domain:** Playwright E2E test infrastructure setup with Firebase programmatic auth for a React 19/Vite app
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Firebase Auth**
- **D-01:** Global setup uses the **programmatic approach** — call Firebase `signInWithPassword` REST endpoint to get an ID token, inject it into the browser via `signInWithCustomToken` (or equivalent auth state injection), then save `storageState` to disk. Do not click through the real login UI in the setup script.
- **D-02:** Auth is **pure infrastructure** in the global setup project. Individual tests load saved `storageState` and land on the dashboard without re-authenticating.
- **D-03:** The Phase 24 smoke test **does exercise the /login UI** — an unauthenticated visit to `/login` verifies the login form renders. This is a lightweight sanity check that the login page is reachable, separate from the programmatic auth setup.

**Selector Strategy**
- **D-04:** Tests use **ARIA/role-based selectors by default** (`getByRole`, `getByLabel`, `getByText`). No React component changes required for the majority of tests.
- **D-05:** `data-testid` attributes are added **only where ARIA selectors are ambiguous** (e.g., multiple identical buttons on a page). Minimal component changes — add `data-testid` only when needed.

**webServer Config**
- **D-06:** `playwright.config.ts` includes `webServer` config that **auto-starts the Vite dev server** (`reuseExistingServer: true`). Developer runs `npx playwright test` and Vite starts automatically if not already running.
- **D-07:** NestJS API + Docker must be started manually before running tests. A **global setup health check** pings the API before any test runs and aborts with a clear message if unreachable: `"API not reachable at http://localhost:3000 — start docker-compose and the NestJS server first."`

**Smoke Test**
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

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | Playwright is installed and configured in trade-flow-ui with `playwright.config.ts`, `webServer` config targeting the Vite dev server, Chromium-only, and an `e2e/` directory structure (`fixtures/`, `helpers/`, `tests/`) | Standard Stack + Architecture Patterns sections cover full config shape, directory structure, and webServer config |
| FOUND-02 | A global auth setup project authenticates via the real Firebase login UI once and saves `storageState` for reuse across all tests (no per-test login) | **Note:** Per D-01, the phrase "real Firebase login UI" in the requirement is superseded by the programmatic approach. Global auth setup project authenticates via Firebase REST API (not UI), saves storageState with `indexedDB: true`. Firebase Auth section covers the full implementation. |

</phase_requirements>

## Summary

Phase 24 creates the Playwright foundation for all subsequent E2E test phases. The deliverables are: (1) Playwright installed and configured in `trade-flow-ui`, (2) a setup project that authenticates programmatically via the Firebase REST API and saves `storageState` to disk, (3) individual tests that load the saved state without re-authenticating, and (4) a smoke test validating both the unauthenticated login page render and the authenticated dashboard landing.

The most important technical finding for this phase is that Playwright v1.51+ includes native IndexedDB support in `storageState`. Firebase v12 stores auth tokens in IndexedDB, not localStorage. Using `page.context().storageState({ path: authFile, indexedDB: true })` is the correct call — without `indexedDB: true`, the Firebase auth state is not captured and every test would re-authenticate. This resolves the MEDIUM confidence gap from the v1.5 milestone research (SUMMARY.md "Gaps to Address" section).

The programmatic auth approach (D-01) avoids Firebase rate limiting entirely. The setup script calls the Firebase Identity Toolkit REST API directly (`https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={apiKey}`), receives an `idToken`, then injects the Firebase auth session into the browser via `page.evaluate()` using `signInWithCustomToken`. This produces a real Firebase session that `AuthProvider`'s `onAuthStateChanged` recognises, ensuring RTK Query's `prepareHeaders` receives a valid token.

**Primary recommendation:** Use the Playwright setup project pattern (not the legacy `globalSetup` file), call `storageState({ indexedDB: true })` after programmatic Firebase sign-in, and store the state in `e2e/.auth/user.json`. This is the current Playwright-recommended approach with full trace and HTML report support.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @playwright/test | 1.58.2 | E2E test runner, browser automation, assertions | Ships its own runner — no Jest needed. Auto-waiting, trace viewer, TypeScript-first. Current verified version. |
| playwright | 1.58.2 | Browser binaries (peer dep, auto-installed) | Provides Chromium binaries. Installed alongside @playwright/test. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @faker-js/faker | 10.4.0 | Realistic test data generation | Phase 25+ for seeded data. Add as devDep now even though smoke test doesn't use it, so directory is set up. |
| dotenv | 17.3.1 | Load `.env.e2e` into `playwright.config.ts` | Keeps Firebase test credentials out of committed config. Required immediately. |
| firebase (existing) | 12.8.0 | Firebase client SDK in setup project | Already installed in trade-flow-ui. Used for `signInWithCustomToken` in page.evaluate. No new package. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Setup project pattern | Legacy `globalSetup` file | Setup project shows in HTML reports, supports traces, uses fixtures — strictly better. globalSetup is legacy. |
| Firebase REST API auth | Firebase Client SDK auth in Node | REST API has no browser dependency; simpler to call from setup script. Client SDK requires browser context tricks. |
| `storageState({ indexedDB: true })` | `signInWithCustomToken` injection only | IndexedDB capture is cleaner and preserves the full refresh token lifecycle. The injection approach is the fallback. |

**Installation (from trade-flow-ui root):**
```bash
npm install -D @playwright/test@1.58.2 @faker-js/faker@10.4.0 dotenv@17.3.1
npx playwright install chromium
```

**Version verification:** All three versions confirmed via `npm view` against the live registry on 2026-03-27.

## Architecture Patterns

### Recommended Project Structure
```
trade-flow-ui/
  e2e/
    .auth/                  # gitignored — contains storageState with credentials
      user.json             # saved by setup project, consumed by all tests
    fixtures/               # Phase 25+ — test.extend() fixtures for API seeding
    helpers/                # Shared utilities
      auth-helper.ts        # Firebase REST API call + token injection
    tests/
      smoke.spec.ts         # Phase 24 smoke test (login form + dashboard landing)
  playwright.config.ts      # At trade-flow-ui root
  .env.e2e                  # gitignored — Firebase test credentials
  .env.e2e.example          # Committed — shows required env vars with placeholder values
```

### Pattern 1: Playwright Setup Project for Auth
**What:** A dedicated Playwright project (`name: 'setup'`) that runs `e2e/helpers/auth-helper.ts` before all other projects. It authenticates once and writes `storageState` to `e2e/.auth/user.json`. All other projects declare `dependencies: ['setup']` and `use: { storageState: 'e2e/.auth/user.json' }`.

**When to use:** Always — every authenticated test depends on this. Never authenticate per-test.

**Example:**
```typescript
// playwright.config.ts
// Source: https://playwright.dev/docs/auth (setup project pattern, verified 2026-03-27)
import { defineConfig, devices } from "@playwright/test";
import dotenv from "dotenv";

dotenv.config({ path: ".env.e2e" });

export default defineConfig({
  testDir: "./e2e",
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  workers: 1,
  fullyParallel: false,
  use: {
    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:5173",
    trace: "on-first-retry",
  },
  webServer: {
    command: "npm run dev",
    url: "http://localhost:5173",
    reuseExistingServer: true,
    timeout: 60000,
  },
  projects: [
    {
      name: "setup",
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: "chromium",
      use: {
        ...devices["Desktop Chrome"],
        storageState: "e2e/.auth/user.json",
      },
      dependencies: ["setup"],
    },
  ],
});
```

### Pattern 2: Programmatic Firebase Auth in Setup File
**What:** The setup project runs a `*.setup.ts` file that calls the Firebase Identity Toolkit REST API directly to get an `idToken`, then injects Firebase auth state into the browser via `signInWithCustomToken`, and saves `storageState` with IndexedDB.

**Why this approach:** D-01 mandates no UI click-through. Firebase v12 uses IndexedDB — must use `storageState({ indexedDB: true })` (added in Playwright v1.51). The `signInWithCustomToken` call in the browser ensures `AuthProvider`'s `onAuthStateChanged` fires and the app recognises the session.

**Example:**
```typescript
// e2e/helpers/auth.setup.ts
// Source: Firebase REST API (verified signInWithPassword endpoint), Playwright auth docs
import { test as setup, expect } from "@playwright/test";
import { initializeApp } from "firebase/app";
import { getAuth, signInWithCustomToken } from "firebase/auth";

const AUTH_FILE = "e2e/.auth/user.json";

setup("authenticate", async ({ page }) => {
  // Step 1: Verify API is reachable (D-07 health check)
  const apiUrl = process.env.E2E_API_URL ?? "http://localhost:3000";
  try {
    const ping = await fetch(`${apiUrl}/v1/ping`);
    if (!ping.ok) throw new Error(`API returned ${ping.status}`);
  } catch {
    throw new Error(
      `API not reachable at ${apiUrl} — start docker-compose and the NestJS server first.`,
    );
  }

  // Step 2: Get Firebase ID token via REST API (no browser needed)
  const firebaseApiKey = process.env.E2E_FIREBASE_API_KEY;
  const signInResponse = await fetch(
    `https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=${firebaseApiKey}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        email: process.env.E2E_TEST_USER_EMAIL,
        password: process.env.E2E_TEST_USER_PASSWORD,
        returnSecureToken: true,
      }),
    },
  );
  if (!signInResponse.ok) {
    throw new Error(`Firebase sign-in failed: ${signInResponse.status}`);
  }
  const { idToken } = (await signInResponse.json()) as { idToken: string };

  // Step 3: Navigate to app and inject Firebase auth state into the browser
  await page.goto("/");
  await page.evaluate(
    async ({ idToken, firebaseConfig }) => {
      const { initializeApp, getApps } = await import("firebase/app");
      const { getAuth, signInWithCustomToken } = await import("firebase/auth");
      const app =
        getApps().length > 0
          ? getApps()[0]
          : initializeApp(firebaseConfig);
      const auth = getAuth(app);
      // signInWithCustomToken accepts an ID token directly in Firebase client SDK
      await signInWithCustomToken(auth, idToken);
    },
    {
      idToken,
      firebaseConfig: {
        apiKey: process.env.E2E_FIREBASE_API_KEY,
        authDomain: process.env.E2E_FIREBASE_AUTH_DOMAIN,
        projectId: process.env.E2E_FIREBASE_PROJECT_ID,
      },
    },
  );

  // Step 4: Wait for app to recognise auth state and redirect
  await page.waitForURL("**/dashboard", { timeout: 15000 });

  // Step 5: Save storageState including IndexedDB (where Firebase v12 stores tokens)
  await page.context().storageState({ path: AUTH_FILE, indexedDB: true });
});
```

**Critical note on `signInWithCustomToken` vs ID token:** Firebase's `signInWithCustomToken` expects a *custom token* (minted by Firebase Admin SDK), not an `idToken` from the REST API. The correct approach for injecting a REST API `idToken` is to use `signInWithEmailAndPassword` inside `page.evaluate`, or to redirect through a custom URL that calls `signInWithEmailAndPassword` with the credentials. The simplest correct approach is calling `signInWithEmailAndPassword` directly inside `page.evaluate` — see Code Examples section.

### Pattern 3: API Health Check Guard (D-07)
**What:** Before auth setup runs, ping `GET /v1/ping`. Throw a descriptive error if not reachable. This happens inside the setup file (not as a separate globalSetup) so it appears in the HTML report.

**When to use:** Every test run — the setup file always executes first via the `dependencies: ['setup']` project chain.

### Pattern 4: Smoke Test Structure
**What:** A single spec file with two tests — one unauthenticated (no storageState override needed; just don't use the chromium project's default), one authenticated.

**The unauthenticated test approach:** Configure a separate Playwright project `{ name: 'chromium-unauthenticated', use: { ...devices['Desktop Chrome'] } }` (no `storageState`, no `dependencies`), and mark the smoke test unauthenticated test to only run in that project. Alternatively, run it as part of the setup project itself before auth state is established.

**Simpler approach:** Put both smoke tests in the `setup` project — the unauthenticated test runs before auth, then auth runs, then the authenticated test is a separate spec in the `chromium` project.

**Recommended:** Two separate spec files: `e2e/tests/smoke-unauth.spec.ts` (runs in setup or standalone unauthenticated project) and `e2e/tests/smoke-auth.spec.ts` (runs in chromium project with storageState).

### Anti-Patterns to Avoid
- **`signInWithCustomToken` with an `idToken`:** Custom tokens and ID tokens are different Firebase token types. Custom tokens are minted by Admin SDK; ID tokens come from sign-in flows. Using them interchangeably causes silent auth failures.
- **`storageState()` without `indexedDB: true`:** Firebase v12 stores all auth in IndexedDB. Without this flag, the saved state is incomplete and tests will find themselves unauthenticated.
- **Hardcoding Firebase credentials in `playwright.config.ts`:** Always load from `.env.e2e` via `dotenv.config()`.
- **Legacy `globalSetup` file:** Use setup projects instead. The legacy approach doesn't appear in HTML reports and doesn't support Playwright fixtures.
- **`page.waitForTimeout()`:** Never use. Use `page.waitForURL()`, `expect(locator).toBeVisible()`, or `page.waitForResponse()` instead.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Auth state persistence | Custom token storage/session management | `storageState({ indexedDB: true })` | Playwright natively captures all browser storage including IndexedDB since v1.51 |
| Browser lifecycle management | Manual `browserType.launch()` in setup | Setup project with `test` fixture | Setup projects get full Playwright fixture support, traces, and HTML report visibility |
| URL waiting | `page.waitForTimeout(3000)` | `page.waitForURL()` / `expect(locator).toBeVisible()` | Auto-retrying assertions never introduce arbitrary delays |
| Multi-browser orchestration | Custom scripts | `projects` array in `playwright.config.ts` | Native Playwright project configuration handles browser matrix cleanly |
| Dev server startup | Shell scripts or manual | `webServer` config in `playwright.config.ts` | Built-in Playwright feature with `reuseExistingServer: true` |

**Key insight:** Playwright has native solutions for every infrastructure concern in this phase. The only custom code needed is the Firebase REST API call and the `page.evaluate` auth injection — everything else is configuration.

## Common Pitfalls

### Pitfall 1: `storageState` Without `indexedDB: true`
**What goes wrong:** The saved auth file appears complete but is missing the IndexedDB entries. Firebase v12 stores its auth tokens in IndexedDB (not localStorage). Tests load the storageState, navigate to the app, but `onAuthStateChanged` fires with `null` — the app sees no authenticated user and redirects to `/login`.

**Why it happens:** The `indexedDB` parameter was added in Playwright v1.51. Before that, IndexedDB capture was not supported. Documentation and blog posts written before v1.51 show `storageState()` without this parameter, leading developers to use the incomplete form.

**How to avoid:** Always use `page.context().storageState({ path: AUTH_FILE, indexedDB: true })` in the auth setup. Verify by checking the saved JSON file — it should have an `indexedDB` key with Firebase entries.

**Warning signs:** Authenticated tests navigate to `/login` despite loading storageState; `auth.currentUser` is `null` in browser console.

### Pitfall 2: Confusing `signInWithCustomToken` and `signInWithEmailAndPassword`
**What goes wrong:** The setup script calls the Firebase Identity Toolkit REST API and gets an `idToken`. Developer passes this `idToken` to `signInWithCustomToken()` inside `page.evaluate`. Firebase rejects it silently or throws `auth/invalid-custom-token`, because custom tokens have a different format (JWTs signed by a service account, not the identity toolkit).

**Why it happens:** Both are called "tokens" but serve different purposes. `signInWithCustomToken` is for Admin SDK-minted tokens; `idToken` from the REST API is the user's identity token, not a custom token.

**How to avoid:** Inside `page.evaluate`, use `signInWithEmailAndPassword(auth, email, password)` directly — or make the entire auth flow happen in the browser context. The REST API approach is useful for getting the `idToken` for API seeding (Phase 25), not for browser-side Firebase auth injection.

**Correct approach:**
```typescript
// Inside page.evaluate — use signInWithEmailAndPassword, not signInWithCustomToken
await signInWithEmailAndPassword(auth, email, password);
```

### Pitfall 3: Vite Dev Server Takes Too Long to Start
**What goes wrong:** The `webServer` config starts Vite but the timeout is too short. Playwright times out waiting for `http://localhost:5173` to respond, even though Vite is still initialising TypeScript compilation and HMR.

**Why it happens:** Cold Vite starts take 5-15 seconds when there are many dependencies. The default `webServer.timeout` may not be sufficient.

**How to avoid:** Set `timeout: 60000` in `webServer` config. With `reuseExistingServer: true`, local dev is unaffected (existing server is reused instantly).

**Warning signs:** CI fails with "webServer timed out" even though Vite works fine locally.

### Pitfall 4: `.env.e2e` File Not Found When Running Tests
**What goes wrong:** `dotenv.config({ path: ".env.e2e" })` in `playwright.config.ts` loads relative to the current working directory. If tests are run from a directory other than `trade-flow-ui/`, the env file is not found and all `process.env.E2E_*` variables are `undefined`.

**Why it happens:** `dotenv` resolves relative paths from `process.cwd()`, not from the config file's location.

**How to avoid:** Use `path.resolve(__dirname, ".env.e2e")` in the `dotenv.config` call, or document that tests must be run from the `trade-flow-ui/` root.

**Warning signs:** Firebase sign-in fails with "INVALID_API_KEY" or similar; `process.env.E2E_FIREBASE_API_KEY` is `undefined`.

### Pitfall 5: Unauthenticated Smoke Test Accidentally Gets StorageState
**What goes wrong:** The smoke test checking `/login` form is placed in the `chromium` project which has `storageState` configured. The test navigates to `/login` but is immediately redirected to `/dashboard` because the auth state is already loaded. The test fails because the login form is never visible.

**Why it happens:** `playwright.config.ts` sets `storageState` as the default for the `chromium` project, affecting all tests in that project including the unauthenticated smoke test.

**How to avoid:** Create a separate unauthenticated project (no `storageState`, no `dependencies`) for the login form smoke test. Or use `test.use({ storageState: undefined })` to override the project-level storageState in a specific test.

**Warning signs:** Unauthenticated test immediately lands on `/dashboard` instead of `/login`.

## Code Examples

### env vars file template
```bash
# .env.e2e.example — commit this file
# .env.e2e — gitignore this file, fill in locally
E2E_BASE_URL=http://localhost:5173
E2E_API_URL=http://localhost:3000
E2E_FIREBASE_API_KEY=your-firebase-api-key
E2E_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
E2E_FIREBASE_PROJECT_ID=your-project-id
E2E_TEST_USER_EMAIL=e2e-test@your-project.com
E2E_TEST_USER_PASSWORD=your-test-user-password
```

### Correct auth injection using `signInWithEmailAndPassword`
```typescript
// e2e/helpers/auth.setup.ts — correct approach (avoids custom token confusion)
// Source: Playwright auth docs (setup project pattern) + Firebase SDK docs
import { test as setup } from "@playwright/test";
import * as path from "path";

const AUTH_FILE = path.join(__dirname, "../.auth/user.json");

setup("authenticate", async ({ page }) => {
  // 1. API health check (D-07)
  const apiUrl = process.env.E2E_API_URL ?? "http://localhost:3000";
  const pingResponse = await fetch(`${apiUrl}/v1/ping`).catch(() => null);
  if (!pingResponse?.ok) {
    throw new Error(
      `API not reachable at ${apiUrl} — start docker-compose and the NestJS server first.`,
    );
  }

  // 2. Navigate to app and authenticate via Firebase SDK in browser context
  await page.goto("/");
  await page.evaluate(
    async ({ email, password, firebaseConfig }) => {
      const { initializeApp, getApps } = await import("firebase/app");
      const { getAuth, signInWithEmailAndPassword } = await import(
        "firebase/auth"
      );
      const app =
        getApps().length > 0
          ? getApps()[0]
          : initializeApp(firebaseConfig);
      const auth = getAuth(app);
      await signInWithEmailAndPassword(auth, email, password);
    },
    {
      email: process.env.E2E_TEST_USER_EMAIL!,
      password: process.env.E2E_TEST_USER_PASSWORD!,
      firebaseConfig: {
        apiKey: process.env.E2E_FIREBASE_API_KEY,
        authDomain: process.env.E2E_FIREBASE_AUTH_DOMAIN,
        projectId: process.env.E2E_FIREBASE_PROJECT_ID,
      },
    },
  );

  // 3. Wait for authenticated redirect
  await page.waitForURL("**/dashboard", { timeout: 15000 });

  // 4. Save storageState WITH IndexedDB (required for Firebase v12)
  await page.context().storageState({ path: AUTH_FILE, indexedDB: true });
});
```

### Smoke test structure
```typescript
// e2e/tests/smoke-auth.spec.ts — runs in chromium project (storageState loaded)
// Source: Playwright test structure conventions
import { test, expect } from "@playwright/test";

test("authenticated user lands on dashboard", async ({ page }) => {
  await page.goto("/dashboard");
  // Verify we are on the dashboard, not redirected to login
  await expect(page).toHaveURL(/\/dashboard/);
  // Minimal UI presence check — no API data assertions (D-09)
  await expect(page.getByRole("main")).toBeVisible();
});
```

```typescript
// e2e/tests/smoke-unauth.spec.ts — runs in unauthenticated project (no storageState)
// Source: Playwright test structure conventions
import { test, expect } from "@playwright/test";

test("login page renders for unauthenticated user", async ({ page }) => {
  await page.goto("/login");
  // Verify login form is visible (D-03, D-04 — ARIA selectors)
  await expect(page.getByRole("heading", { name: /sign in|log in/i })).toBeVisible();
  await expect(page.getByRole("textbox", { name: /email/i })).toBeVisible();
  await expect(page.getByRole("textbox", { name: /password/i })).toBeVisible();
});
```

### Full `playwright.config.ts`
```typescript
// Source: https://playwright.dev/docs/auth (verified 2026-03-27)
import { defineConfig, devices } from "@playwright/test";
import dotenv from "dotenv";
import * as path from "path";

dotenv.config({ path: path.resolve(__dirname, ".env.e2e") });

export default defineConfig({
  testDir: "./e2e",
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,
  workers: 1,
  fullyParallel: false,
  reporter: [["html", { open: "never" }]],
  use: {
    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:5173",
    trace: "on-first-retry",
  },
  webServer: {
    command: "npm run dev",
    url: "http://localhost:5173",
    reuseExistingServer: true,
    timeout: 60000,
  },
  projects: [
    // Setup project — runs first, writes auth state
    {
      name: "setup",
      testMatch: /.*\.setup\.ts/,
    },
    // Authenticated tests — depend on setup
    {
      name: "chromium",
      use: {
        ...devices["Desktop Chrome"],
        storageState: "e2e/.auth/user.json",
      },
      dependencies: ["setup"],
    },
    // Unauthenticated tests — no storageState, no setup dependency
    {
      name: "chromium-unauth",
      use: { ...devices["Desktop Chrome"] },
      testMatch: /.*unauth\.spec\.ts/,
    },
  ],
});
```

### .gitignore additions (trade-flow-ui root)
```gitignore
# Playwright
e2e/.auth/
playwright-report/
test-results/
.env.e2e
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `globalSetup` file for auth | Setup project (`testMatch: /.*\.setup\.ts/`) | Playwright v1.20+ | Setup projects show in HTML reports, support fixtures and traces |
| `storageState()` without IndexedDB | `storageState({ indexedDB: true })` | Playwright v1.51 | Required for Firebase v12 which uses IndexedDB for token persistence |
| `signInWithCustomToken` for auth injection | `signInWithEmailAndPassword` in `page.evaluate` | Firebase v9 SDK (ESM imports) | Custom token requires Admin SDK; email/password injection uses only client SDK |

**Deprecated/outdated:**
- `globalSetup` + `globalTeardown` as TypeScript files: Still works but is the legacy pattern. Playwright's own docs now lead with setup projects.
- `storageState()` without `indexedDB` for Firebase apps: Produces incomplete state. All Firebase v9+ apps should use `indexedDB: true`.

## Open Questions

1. **Firebase app already initialised in `page.evaluate`**
   - What we know: The Vite dev app initialises Firebase in `src/lib/firebase.ts` on load. When `page.evaluate` runs, `getApps()` should return the existing app instance.
   - What's unclear: Whether `getApps()` is accessible in `page.evaluate` context (which runs inside the browser) or whether ESM dynamic imports work as expected with the Vite-served app.
   - Recommendation: Use `getApps().length > 0 ? getApps()[0] : initializeApp(config)` pattern to handle both cases. If dynamic imports fail, pre-load the app module via `page.addInitScript`.

2. **Pre-created Firebase test user**
   - What we know: A dedicated Firebase test user must exist in the Firebase Console before Phase 24 can run (noted in STATE.md blockers).
   - What's unclear: Whether the existing Firebase project has a test user already, or if one needs to be created.
   - Recommendation: Make the first task in Phase 24 a documentation/prerequisite check: verify `E2E_TEST_USER_EMAIL` exists in the Firebase project. This is a one-time manual action outside the codebase.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Running Playwright | ✓ | 22.15.0 | — |
| npm | Package installation | ✓ | 11.7.0 | — |
| Docker | NestJS API + MongoDB (manual start) | ✓ | 29.2.1 | — |
| Playwright (global) | Test runner | ✓ | 1.58.2 | Install via npm |
| @playwright/test (local devDep) | Test runner (per project) | ✗ | — | `npm install -D @playwright/test@1.58.2` |
| Firebase test user | Auth setup | ✗ (unverified) | — | Must be created in Firebase Console |

**Missing dependencies with no fallback:**
- Firebase test user account: Must be pre-created in the Firebase Console (manual, one-time). Without it, the auth setup project cannot authenticate and all tests fail. Document this as a prerequisite in the plan.

**Missing dependencies with fallback:**
- `@playwright/test` as local devDep: Playwright 1.58.2 is installed globally, so `npx playwright test` works immediately. However, local devDep installation is required for TypeScript types in the config and test files.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | @playwright/test 1.58.2 |
| Config file | `trade-flow-ui/playwright.config.ts` (Wave 0 — does not yet exist) |
| Quick run command | `npx playwright test e2e/tests/smoke-auth.spec.ts --project=chromium` |
| Full suite command | `npx playwright test` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | `playwright.config.ts` exists with correct shape, `e2e/` directory structure created | smoke | `npx playwright test --list` (verifies config loads and tests are discovered) | ❌ Wave 0 |
| FOUND-02 | Auth setup project runs, writes `e2e/.auth/user.json`, authenticated test lands on `/dashboard` | smoke | `npx playwright test e2e/tests/smoke-auth.spec.ts --project=chromium` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `npx playwright test --list` (config syntax check, < 2 seconds)
- **Per wave merge:** `npx playwright test` (full smoke suite, < 60 seconds including Vite startup)
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `trade-flow-ui/playwright.config.ts` — Playwright config (covers FOUND-01)
- [ ] `trade-flow-ui/e2e/helpers/auth.setup.ts` — setup project auth file (covers FOUND-02)
- [ ] `trade-flow-ui/e2e/tests/smoke-auth.spec.ts` — authenticated smoke test (covers FOUND-02)
- [ ] `trade-flow-ui/e2e/tests/smoke-unauth.spec.ts` — unauthenticated login form test (covers D-03, D-08)
- [ ] `trade-flow-ui/.env.e2e.example` — env template (required for setup to run)
- [ ] Framework install: `npm install -D @playwright/test@1.58.2 @faker-js/faker@10.4.0 dotenv@17.3.1` (run once from trade-flow-ui/)

## Project Constraints (from CLAUDE.md)

Directives from CLAUDE.md that constrain planning and implementation:

- **TypeScript strict mode:** All new files must satisfy `strictNullChecks` and `noImplicitAny`. The `playwright.config.ts` and setup files must be valid TypeScript.
- **Two-repo structure:** `trade-flow-ui` and `trade-flow-api` are independent repos. All Phase 24 changes are in `trade-flow-ui` only — no API changes in this phase.
- **GSD workflow enforcement:** All file changes must go through a GSD command. Do not make direct edits outside a GSD workflow.
- **File naming:** E2E test helpers use kebab-case (`auth-helper.ts`); spec files use kebab-case with `.spec.ts` suffix (`smoke-auth.spec.ts`); setup files use `.setup.ts` suffix.
- **Code style:** 125-character print width, 2-space indentation, double quotes, trailing commas, LF line endings — same as the rest of trade-flow-ui.
- **No standalone calendar / Solo operator constraints:** Not applicable to test infrastructure.
- **GSD entry point for this phase:** `/gsd:execute-phase`

## Sources

### Primary (HIGH confidence)
- Playwright documentation (https://playwright.dev/docs/auth) — setup project pattern, storageState with IndexedDB parameter — verified 2026-03-27
- Playwright documentation (https://playwright.dev/docs/test-global-setup-teardown) — setup project vs globalSetup comparison — verified 2026-03-27
- Playwright release notes — IndexedDB support confirmed added in v1.51 — verified 2026-03-27
- npm registry: `@playwright/test@1.58.2`, `@faker-js/faker@10.4.0`, `dotenv@17.3.1` — verified current via `npm view` 2026-03-27
- `.planning/research/STACK.md` — existing v1.5 milestone research, stack decisions
- `.planning/research/PITFALLS.md` — 15 pitfalls, Firebase auth strategies
- `.planning/phases/24-playwright-bootstrap-auth/24-CONTEXT.md` — locked implementation decisions D-01 through D-09

### Secondary (MEDIUM confidence)
- Firebase Identity Toolkit REST API: `https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword` — endpoint and response format confirmed via WebSearch results against Google Cloud docs (2026-03-27)
- Firebase v12 IndexedDB persistence behaviour — confirmed by Playwright release notes explicitly calling out Firebase as the use case for `indexedDB: true`

### Tertiary (LOW confidence)
- `page.evaluate` dynamic import behaviour with Vite-served ESM modules — training data only; needs validation in Phase 24 execution

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — versions verified via npm registry 2026-03-27
- Architecture: HIGH — Playwright setup project pattern verified against official docs; IndexedDB support confirmed in release notes
- Firebase auth injection: MEDIUM-HIGH — `signInWithEmailAndPassword` in `page.evaluate` is well-established; dynamic import behaviour in Vite context is LOW confidence (needs runtime validation)
- Pitfalls: HIGH — sourced from official Playwright docs and codebase analysis

**Research date:** 2026-03-27
**Valid until:** 2026-04-27 (Playwright releases roughly monthly; re-verify if using a later version)
