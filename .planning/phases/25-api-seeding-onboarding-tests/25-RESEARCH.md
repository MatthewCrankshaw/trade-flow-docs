# Phase 25: API Seeding Infrastructure + Onboarding Tests - Research

**Researched:** 2026-03-28
**Domain:** Playwright E2E test data seeding via HTTP API + MongoDB cleanup + onboarding wizard testing
**Confidence:** HIGH

## Summary

Phase 25 builds the test data infrastructure that all subsequent E2E phases (26-28) depend on, then validates it with auth redirect tests and the full onboarding wizard flow. The work divides into three distinct concerns: (1) a typed HTTP seeding client that creates entities through the existing NestJS REST API, (2) a MongoDB cleanup helper that removes test data by businessId after each test, and (3) four E2E test specs covering auth redirects and onboarding.

The seeding client is straightforward -- it wraps `fetch()` calls to existing API endpoints (`POST /v1/customers`, `POST /v1/jobs`, `POST /v1/quotes`) with the Firebase JWT from Phase 24's storageState. Factory functions use `@faker-js/faker` (already installed in Phase 24) to generate unique test data. The MongoDB cleanup helper connects directly using the `mongodb` native driver (v7.0.0, already used by the API) and deletes by `businessId` filter across known collections.

The onboarding wizard test is the most complex deliverable. It requires a **second Firebase test user** with no existing business, a separate storageState file, and a modified global setup that authenticates both users. The test walks through the multi-step wizard UI and verifies that default entities (job types, tax rates, items, visit types) are created by making GET API calls after wizard completion.

**Primary recommendation:** Build the seeding client and MongoDB cleanup as Playwright fixtures via `test.extend()`, expose them to specs through a `test-base.ts` barrel, then write the four test specs (auth-login, auth-redirect, onboarding-wizard) that prove the infrastructure works.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Seeding client calls existing NestJS REST API endpoints (e.g. `POST /v1/customers`) rather than inserting directly into MongoDB. The API's creator services, access controllers, and policies handle data creation consistently -- tests don't need to track schema changes.
- **D-02:** Authentication via the Firebase JWT extracted from Phase 24's saved storageState. Bearer token passed in fetch headers. No separate service account.
- **D-03:** Factory functions with auto-chaining dependencies. e.g. `seedJob()` auto-creates a customer if no `customerId` provided. Each function returns the created entity with its ID. Tests only specify what they care about.
- **D-04:** Single shared business per test user. Users cannot have multiple businesses, so "fresh business per test" is not viable.
- **D-05:** Cleanup via direct MongoDB connection. Test helper connects to the same MongoDB instance (via `MONGO_URL`) and deletes documents by `businessId` across all relevant collections after each test. No API delete endpoints needed.
- **D-06:** Two Firebase test users with distinct roles:
  - **Pre-onboarded user** -- has a business already created. Used by most tests (seeding, CRUD, auth redirect). storageState from Phase 24.
  - **Fresh onboarding user** -- no business. Used exclusively by the onboarding wizard test. Separate storageState.
- **D-07:** Unique naming via Faker + timestamp (e.g. "Test Plumbing 1711532400123") for any test-created entities. Human-readable for debugging.
- **D-08:** Full multi-step wizard walkthrough via UI. The test fills in each step of the onboarding wizard (business details, trade selection, etc.) using the fresh onboarding user.
- **D-09:** Four assertions after wizard completion:
  1. Business created with correct fields (name, trade match what was entered)
  2. Default entities exist (job types, tax rates, items, visit types) -- verified via API calls (GET endpoints), not UI navigation
  3. User redirected to the dashboard after completing onboarding
  4. Subsequent login skips the onboarding wizard (log out, log back in, lands on dashboard)
- **D-10:** Auth tests use the pre-onboarded user's storageState. Valid login lands on dashboard; unauthenticated access to protected route redirects to login. Implementation details at Claude's discretion.

### Claude's Discretion
- MongoDB cleanup helper implementation details (connection management, collection list)
- Seeding client internal structure (file organization, TypeScript types)
- Faker library choice and data generation patterns
- Auth redirect test specifics (which protected route to test, exact assertions)
- How the fresh onboarding user's Firebase credentials are injected (env vars, fixture config)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-03 | A typed API seeding client (fetch-based helper) can create and delete test entities via the NestJS API, authenticated with a Firebase test user token | Seeding client architecture, factory pattern, MongoDB cleanup helper, API endpoint mapping |
| AUTH-01 | Test verifies user can log in with valid credentials and land on the dashboard | Auth redirect test patterns using pre-onboarded user storageState |
| AUTH-02 | Test verifies unauthenticated users are redirected to login when accessing protected routes | Chromium-unauth project pattern from Phase 24 |
| AUTH-03 | Test verifies the full onboarding wizard creates a business with default job types, tax rates, items, and visit types | Second Firebase user setup, onboarding wizard UI flow, API verification of defaults |
</phase_requirements>

## Standard Stack

### Core (no new packages required beyond Phase 24)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @playwright/test | 1.58.2 | Test runner, fixtures, assertions | Already installed in Phase 24 |
| @faker-js/faker | 10.4.0 | Generate unique test entity names | Already installed in Phase 24 |
| mongodb | 7.0.0 | Direct MongoDB connection for cleanup | Same version used by trade-flow-api; install as devDep in trade-flow-ui |

### New devDependency

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| mongodb | 7.0.0 | MongoDB native driver for test cleanup | Cleanup helper connects directly to delete test documents by businessId |

**Installation:**
```bash
# Run from trade-flow-ui root
npm install -D mongodb@7.0.0
```

**Version verification:** `mongodb@7.0.0` matches the API's driver version exactly. Using the same major version avoids BSON serialization mismatches.

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Direct MongoDB cleanup | API DELETE endpoints | Would require creating delete endpoints for every entity type -- user explicitly rejected this (D-05) |
| mongodb native driver | mongoose | API uses native driver (not mongoose despite it being in package.json) -- match the same approach |
| Faker for test data | Hardcoded strings | Faker prevents test coupling to specific strings; timestamp suffix ensures uniqueness across parallel runs |

## Architecture Patterns

### Recommended Project Structure

```
trade-flow-ui/e2e/
├── fixtures/
│   ├── test-base.ts             # Extended test with apiClient + dbCleanup fixtures
│   └── .gitkeep                 # (remove once files exist)
├── helpers/
│   ├── auth.setup.ts            # Phase 24 -- authenticates BOTH users now
│   ├── api-client.ts            # Typed HTTP seeding client
│   ├── db-cleanup.ts            # MongoDB cleanup helper
│   └── seed-factories.ts        # Faker-based factory functions
├── tests/
│   ├── auth/
│   │   ├── login.spec.ts        # AUTH-01: valid login lands on dashboard
│   │   └── redirect-unauth.spec.ts  # AUTH-02: protected route redirects
│   └── onboarding/
│       └── onboarding-wizard.spec.ts  # AUTH-03: full wizard flow
└── .auth/
    ├── user.json                # Pre-onboarded user storageState (Phase 24)
    └── onboarding-user.json     # Fresh user storageState (new in Phase 25)
```

### Pattern 1: Typed API Seeding Client

**What:** A class wrapping `fetch()` calls to the NestJS REST API with typed request/response interfaces. Each method maps to one API endpoint.

**When to use:** Every test that needs pre-existing data (customers, jobs, quotes).

**Example:**
```typescript
// e2e/helpers/api-client.ts
interface SeedCustomerResponse {
  data: Array<{ id: string; name: string; email: string; businessId: string }>;
}

export class ApiClient {
  constructor(
    private readonly baseUrl: string,
    private readonly token: string,
    private readonly businessId: string,
  ) {}

  async createCustomer(overrides?: Partial<CustomerPayload>): Promise<SeedCustomerResponse["data"][0]> {
    const payload = { ...defaultCustomerPayload(this.businessId), ...overrides };
    const res = await fetch(`${this.baseUrl}/v1/customer`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${this.token}`,
      },
      body: JSON.stringify(payload),
    });
    if (!res.ok) throw new Error(`Seed customer failed: ${res.status} ${await res.text()}`);
    const json: SeedCustomerResponse = await res.json();
    return json.data[0];
  }

  // Similar methods for createJob, createQuote, etc.
}
```

### Pattern 2: Factory Functions with Auto-Chaining (D-03)

**What:** Seed functions that automatically create prerequisite entities when not provided.

**When to use:** Tests that need a job but don't care about the specific customer.

**Example:**
```typescript
// e2e/helpers/seed-factories.ts
import { faker } from "@faker-js/faker";

export function customerPayload(businessId: string) {
  const ts = Date.now();
  return {
    name: `${faker.person.fullName()} ${ts}`,
    email: faker.internet.email(),
    phone: faker.phone.number(),
    businessId,
  };
}

export function jobPayload(businessId: string, customerId: string) {
  const ts = Date.now();
  return {
    title: `${faker.commerce.productName()} ${ts}`,
    customerId,
    businessId,
  };
}
```

### Pattern 3: MongoDB Cleanup Helper (D-05)

**What:** Connects to MongoDB and deletes all documents matching a businessId across known collections.

**When to use:** afterEach/afterAll in tests that create data via the seeding client.

**Example:**
```typescript
// e2e/helpers/db-cleanup.ts
import { MongoClient, Db } from "mongodb";

const CLEANUP_COLLECTIONS = [
  "customers",
  "jobs",
  "quotes",
  "quote_line_items",
  "items",
  "tax_rates",
  "job_types",
  // Note: do NOT delete from "users", "business", "business_users" -- these are test user fixtures
];

export class DbCleanup {
  private client: MongoClient;
  private db: Db;

  constructor(mongoUrl: string) {
    this.client = new MongoClient(mongoUrl);
    this.db = this.client.db("trade-flow-db");
  }

  async connect() {
    await this.client.connect();
  }

  async cleanByBusinessId(businessId: string) {
    for (const collection of CLEANUP_COLLECTIONS) {
      await this.db.collection(collection).deleteMany({ businessId });
    }
  }

  async close() {
    await this.client.close();
  }
}
```

### Pattern 4: Extended Test Fixture (test-base.ts)

**What:** Playwright `test.extend()` that provides `apiClient` and `dbCleanup` fixtures to all spec files.

**When to use:** Every spec file imports `test` and `expect` from here instead of `@playwright/test`.

**Example:**
```typescript
// e2e/fixtures/test-base.ts
import { test as base } from "@playwright/test";
import { ApiClient } from "../helpers/api-client";
import { DbCleanup } from "../helpers/db-cleanup";

type TestFixtures = {
  apiClient: ApiClient;
  dbCleanup: DbCleanup;
};

export const test = base.extend<TestFixtures>({
  apiClient: async ({ page }, use) => {
    // Extract token from storageState -- Firebase stores auth in IndexedDB
    // The token is available from the API response after page loads
    const apiUrl = process.env.E2E_API_URL ?? "http://localhost:3000";
    const businessId = process.env.E2E_TEST_BUSINESS_ID!;
    const token = await extractTokenFromPage(page);
    const client = new ApiClient(apiUrl, token, businessId);
    await use(client);
  },

  dbCleanup: async ({}, use) => {
    const mongoUrl = process.env.E2E_MONGO_URL ?? "mongodb://localhost:27017";
    const cleanup = new DbCleanup(mongoUrl);
    await cleanup.connect();
    await use(cleanup);
    await cleanup.close();
  },
});

export { expect } from "@playwright/test";
```

### Pattern 5: Second Firebase User for Onboarding (D-06)

**What:** Global setup authenticates two users; onboarding test uses a fresh user with no business.

**When to use:** Exclusively for the onboarding wizard test (AUTH-03).

**Example modification to auth.setup.ts:**
```typescript
// Extend Phase 24's auth.setup.ts to handle both users
const AUTH_FILE = path.join(__dirname, "../.auth/user.json");
const ONBOARDING_AUTH_FILE = path.join(__dirname, "../.auth/onboarding-user.json");

setup("authenticate pre-onboarded user", async ({ page }) => {
  // ... existing Phase 24 auth logic using E2E_TEST_USER_EMAIL ...
  await page.context().storageState({ path: AUTH_FILE, indexedDB: true });
});

setup("authenticate onboarding user", async ({ page }) => {
  // Same pattern but with E2E_ONBOARDING_USER_EMAIL / E2E_ONBOARDING_USER_PASSWORD
  // This user has NO business -- used only by the onboarding wizard test
  await page.context().storageState({ path: ONBOARDING_AUTH_FILE, indexedDB: true });
});
```

### Anti-Patterns to Avoid

- **Inserting directly into MongoDB for seeding:** Bypasses validation, policies, and access control. When the API schema changes, tests silently create invalid data. Decision D-01 explicitly prohibits this.
- **Using the pre-onboarded user for onboarding tests:** The pre-onboarded user already has a business. The onboarding wizard won't appear for them. Decision D-06 requires a separate fresh user.
- **Asserting on UI for default entity verification:** After onboarding, verifying defaults via UI navigation is fragile (depends on list page rendering, pagination, etc.). Decision D-09 specifies API GET calls for verification.
- **Forgetting to clean up after onboarding test:** The onboarding test creates a business + defaults for the fresh user. If not cleaned up, subsequent test runs will see "user already has a business" and skip onboarding. The cleanup must delete the business, business_users entry, and all defaults.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Unique test data | Manual timestamp concatenation | `@faker-js/faker` + `Date.now()` suffix | Faker provides realistic field values; timestamp ensures uniqueness |
| MongoDB connection management | Raw MongoClient with no lifecycle | Playwright fixture with connect/close in setup/teardown | Ensures connections are properly closed even on test failure |
| Token extraction from storageState | Parsing the JSON file manually for Firebase tokens | `page.evaluate(() => firebase.auth().currentUser.getIdToken())` | Firebase SDK handles token refresh; manual parsing risks expired tokens |
| HTTP error handling in seeding | Ignoring non-2xx responses | Throw with status code + response body | Silent seeding failures cause confusing test failures downstream |

**Key insight:** The seeding client is intentionally thin -- it delegates all business logic to the API. Hand-rolling data creation logic in tests would duplicate the API's validation and create a second source of truth that diverges over time.

## Common Pitfalls

### Pitfall 1: Firebase Token Expiration in Long Test Runs

**What goes wrong:** The Firebase ID token saved in storageState has a 1-hour expiry. If the test suite takes longer than 1 hour, later tests fail with 401 from the API.
**Why it happens:** storageState captures the token at setup time. Firebase's `onAuthStateChanged` refreshes it in a live browser session, but the raw token in storageState is not auto-refreshed.
**How to avoid:** Extract the token from the live page context (`page.evaluate(() => ...)`) rather than parsing it from the storageState JSON file. The Firebase SDK in the running app handles refresh automatically.
**Warning signs:** 401 errors in tests that run near the end of the suite; tests pass individually but fail when the full suite runs.

### Pitfall 2: MongoDB Cleanup Missing Collections

**What goes wrong:** A new collection is added to the API (e.g., `schedules`) but not added to the cleanup helper's collection list. Tests leave orphaned data that pollutes subsequent runs.
**Why it happens:** The cleanup collection list is maintained manually and is disconnected from the API's repository definitions.
**How to avoid:** Document the collection list explicitly and reference it in the cleanup helper. When a new module is added to the API, the cleanup list must be updated. Consider adding a comment in the cleanup helper pointing to `INTEGRATIONS.md` as the source of truth.
**Warning signs:** Test data accumulates in MongoDB over multiple runs; `mongosh` shows unexpected document counts.

### Pitfall 3: Onboarding User State Not Cleaned Between Runs

**What goes wrong:** The onboarding wizard test creates a business for the fresh user. On the next test run, the fresh user already has a business, so the onboarding wizard doesn't appear. The test fails.
**Why it happens:** The onboarding test's afterAll cleanup must delete: (1) the business document, (2) the business_users link, (3) all default entities (job_types, tax_rates, items, visit types). Missing any one of these leaves the user in a partially-onboarded state.
**How to avoid:** The onboarding test's cleanup must be comprehensive -- delete from `business`, `business_users`, `items`, `tax_rates`, `job_types` where `businessId` matches, AND update the `users` document to clear businessIds. Use a dedicated cleanup function that is tested separately.
**Warning signs:** Onboarding test passes on first run but fails on second; `e2e/.auth/onboarding-user.json` contains a user that already has a businessId.

### Pitfall 4: Onboarding Wizard Step Selectors Break on UI Changes

**What goes wrong:** The onboarding wizard uses Radix UI dialog components. Selector strategies that depend on DOM structure (`.dialog > .content > form`) break when the wizard layout changes.
**Why it happens:** Onboarding is a dialog-based multi-step flow, not a page-based flow. Standard page-level selectors don't work well.
**How to avoid:** Use ARIA role selectors (`getByRole("dialog")`, `getByLabel("Business name")`, `getByRole("combobox", { name: /trade/i })`) per D-04. If ambiguous, add minimal `data-testid` attributes per D-05.
**Warning signs:** Tests fail after UI refactoring despite no functional changes.

### Pitfall 5: Race Condition Between Onboarding Completion and Default Entity Creation

**What goes wrong:** The test completes the wizard and immediately makes API GET calls to verify defaults. The API returns empty lists because the default entity creation (job types, tax rates, items) is still in progress.
**Why it happens:** Business creation triggers `DefaultJobTypesCreator`, `DefaultTaxRatesCreator`, and `DefaultBusinessItemsCreator` services. These run synchronously in the API's create flow, but the UI redirect to dashboard may happen before all defaults are persisted if there's any async behavior.
**How to avoid:** After the wizard completes and the user is redirected to the dashboard, wait for the dashboard to fully load before making verification API calls. Alternatively, poll the API with a retry loop (e.g., retry 3 times with 500ms delay) for default entities to appear.
**Warning signs:** Onboarding test is flaky -- passes sometimes, fails with "expected at least 1 job type, got 0" intermittently.

## Code Examples

### Extracting Firebase Token from Live Page Context

```typescript
// Use this in fixtures to get a fresh (non-expired) token
async function extractTokenFromPage(page: Page): Promise<string> {
  const token = await page.evaluate(async () => {
    const { getAuth } = await import("firebase/auth");
    const auth = getAuth();
    const user = auth.currentUser;
    if (!user) throw new Error("No authenticated user in browser context");
    return user.getIdToken();
  });
  return token;
}
```

### Onboarding Wizard Test Structure

```typescript
// e2e/tests/onboarding/onboarding-wizard.spec.ts
import { test, expect } from "../../fixtures/test-base";

test.describe("Onboarding wizard", () => {
  test.use({ storageState: "e2e/.auth/onboarding-user.json" });

  test.afterAll(async () => {
    // Comprehensive cleanup: delete business + all defaults for onboarding user
    // Must also clear businessIds from the users collection
  });

  test("completes wizard and creates business with defaults", async ({ page }) => {
    await page.goto("/");
    // Wizard should appear automatically for user with no business

    // Step 1: Enter business name
    await page.getByLabel(/business name/i).fill("Test Plumbing 1711532400123");

    // Step 2: Select trade
    await page.getByRole("combobox", { name: /trade/i }).click();
    await page.getByRole("option", { name: /plumbing/i }).click();

    // Step 3: Complete wizard (click through any remaining steps)
    await page.getByRole("button", { name: /complete|finish|continue/i }).click();

    // Assert: redirected to dashboard
    await expect(page).toHaveURL(/\/dashboard/);

    // Assert: verify defaults via API
    const apiUrl = process.env.E2E_API_URL ?? "http://localhost:3000";
    const token = await extractTokenFromPage(page);
    // ... GET /v1/job-type, /v1/tax-rate, /v1/item to verify defaults exist
  });
});
```

### Auth Redirect Tests

```typescript
// e2e/tests/auth/login.spec.ts (AUTH-01)
// Runs in "chromium" project with pre-onboarded user storageState
import { test, expect } from "@playwright/test";

test("authenticated user lands on dashboard", async ({ page }) => {
  await page.goto("/dashboard");
  await expect(page).toHaveURL(/\/dashboard/);
  await expect(page.getByRole("main")).toBeVisible();
});
```

```typescript
// e2e/tests/auth/redirect-unauth.spec.ts (AUTH-02)
// Runs in "chromium-unauth" project -- no storageState
import { test, expect } from "@playwright/test";

test("unauthenticated user is redirected to login", async ({ page }) => {
  await page.goto("/dashboard");
  await expect(page).toHaveURL(/\/login/);
});
```

## API Endpoints for Seeding Client

The seeding client wraps these existing API endpoints:

| Entity | Create Endpoint | List/Get Endpoint | Notes |
|--------|----------------|-------------------|-------|
| Customer | `POST /v1/customer` | `GET /v1/customer?businessId={id}` | Requires: name, businessId |
| Job | `POST /v1/job` | `GET /v1/job?businessId={id}` | Requires: title, customerId, businessId |
| Quote | `POST /v1/quote` | `GET /v1/quote?businessId={id}` | Requires: jobId, businessId |
| Job Type | -- (created by onboarding) | `GET /v1/job-type?businessId={id}` | Read-only for verification |
| Tax Rate | -- (created by onboarding) | `GET /v1/tax-rate?businessId={id}` | Read-only for verification |
| Item | -- (created by onboarding) | `GET /v1/item?businessId={id}` | Read-only for verification |
| Business | `POST /v1/business` | `GET /v1/business` | Created via wizard; GET returns user's businesses |

## MongoDB Collections for Cleanup

Source: `INTEGRATIONS.md` collections list + API repository static COLLECTION constants.

| Collection | Cleaned by businessId | Notes |
|------------|----------------------|-------|
| `customers` | Yes | Standard cleanup |
| `jobs` | Yes | Standard cleanup |
| `quotes` | Yes | Standard cleanup |
| `quote_line_items` | Yes | Standard cleanup |
| `items` | Yes | Standard cleanup |
| `tax_rates` | Yes | Standard cleanup |
| `job_types` | Yes | Standard cleanup |
| `business` | Onboarding only | Only delete in onboarding test cleanup (by _id) |
| `business_users` | Onboarding only | Only delete in onboarding test cleanup (by businessId) |
| `users` | Never delete | Update to clear businessIds array in onboarding cleanup |

Database name: `trade-flow-db` (from INTEGRATIONS.md).

## Environment Variables (New in Phase 25)

Added to `.env.e2e` / `.env.e2e.example`:

| Variable | Purpose | Example |
|----------|---------|---------|
| `E2E_MONGO_URL` | MongoDB connection string for cleanup helper | `mongodb://localhost:27017` |
| `E2E_ONBOARDING_USER_EMAIL` | Second Firebase test user (no business) | `e2e-onboarding@your-project.com` |
| `E2E_ONBOARDING_USER_PASSWORD` | Password for onboarding test user | `your-test-password` |
| `E2E_TEST_BUSINESS_ID` | Business ID of pre-onboarded user (seeding needs this) | `ObjectId string` |

## Playwright Config Changes

Phase 25 extends the Phase 24 config:

1. **New auth setup step:** `auth.setup.ts` authenticates both users (pre-onboarded + fresh onboarding user)
2. **New storageState file:** `e2e/.auth/onboarding-user.json` -- gitignored
3. **Test file organization:** Auth tests go in `e2e/tests/auth/`, onboarding tests in `e2e/tests/onboarding/`
4. **The `chromium` project's `testMatch`** must exclude `*unauth*` files (these run in `chromium-unauth`). Phase 24 already handles this with `testMatch: /.*unauth\.spec\.ts/` on `chromium-unauth`.

The onboarding test needs a custom `storageState` override via `test.use({ storageState: "e2e/.auth/onboarding-user.json" })` -- this overrides the project-level `user.json` for just that test file.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Page Object Model for test helpers | Playwright fixtures via `test.extend()` | Playwright 1.20+ (2022) | Fixtures auto-manage lifecycle; POM requires manual setup/teardown |
| Database seeding via ORM/direct insert | API-based seeding via HTTP | Community best practice ~2023+ | Decouples tests from database schema; tests break only when API contract changes |
| Global test database reset | Per-businessId scoped cleanup | N/A -- project-specific | Enables shared database without global wipes; safe for serial execution |

## Open Questions

1. **Onboarding wizard step count and field names**
   - What we know: The wizard has business name and trade selection steps. `OnboardingDialogs.tsx` manages the flow.
   - What's unclear: Exact number of wizard steps, exact field labels/ARIA roles (need to inspect the live UI or read the component source)
   - Recommendation: The implementation task should inspect `trade-flow-ui/src/components/onboarding/` before writing selectors. May need 1-2 `data-testid` attributes if ARIA selectors are ambiguous.

2. **Business ID discovery for pre-onboarded user**
   - What we know: The seeding client needs the pre-onboarded user's businessId to scope API calls
   - What's unclear: Whether the business ID is available from storageState, or whether the seeding client should make a `GET /v1/business` call to discover it
   - Recommendation: Make a `GET /v1/business` call in the fixture setup and cache the businessId. Avoids hardcoding and works even if the test user's business is recreated.

3. **Visit types collection name**
   - What we know: AUTH-03 requires verifying "visit types" exist after onboarding. The INTEGRATIONS.md collections list does not include a `visit_types` collection.
   - What's unclear: Whether visit types are stored as a separate collection or as part of another entity. The API structure shows a `job-type/` module but no explicit `visit-type/` module.
   - Recommendation: Inspect the API codebase during implementation to locate visit types. They may be a sub-type of job types, or may use a different collection name. The cleanup helper collection list should be finalized after this is confirmed.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | @playwright/test 1.58.2 |
| Config file | trade-flow-ui/playwright.config.ts (created in Phase 24) |
| Quick run command | `npx playwright test e2e/tests/auth/` |
| Full suite command | `npx playwright test` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-03 | Seeding client creates and cleans up entities via API | integration (fixture test) | `npx playwright test e2e/tests/ --grep "seed"` | No -- Wave 0 |
| AUTH-01 | Valid login lands on dashboard | e2e | `npx playwright test e2e/tests/auth/login.spec.ts` | No -- Wave 0 |
| AUTH-02 | Unauthenticated redirect to login | e2e | `npx playwright test e2e/tests/auth/redirect-unauth.spec.ts --project=chromium-unauth` | No -- Wave 0 |
| AUTH-03 | Onboarding wizard creates business with defaults | e2e | `npx playwright test e2e/tests/onboarding/onboarding-wizard.spec.ts` | No -- Wave 0 |

### Sampling Rate

- **Per task commit:** `npx playwright test --list` (verify no syntax errors)
- **Per wave merge:** `npx playwright test` (full suite including Phase 24 smoke tests)
- **Phase gate:** All 4 new tests + 2 Phase 24 smoke tests pass green

### Wave 0 Gaps

- [ ] `e2e/helpers/api-client.ts` -- seeding client
- [ ] `e2e/helpers/db-cleanup.ts` -- MongoDB cleanup
- [ ] `e2e/helpers/seed-factories.ts` -- Faker factories
- [ ] `e2e/fixtures/test-base.ts` -- extended test with fixtures
- [ ] `e2e/tests/auth/login.spec.ts` -- AUTH-01
- [ ] `e2e/tests/auth/redirect-unauth.spec.ts` -- AUTH-02
- [ ] `e2e/tests/onboarding/onboarding-wizard.spec.ts` -- AUTH-03
- [ ] `.env.e2e.example` updated with new env vars
- [ ] `mongodb@7.0.0` installed as devDep
- [ ] Second Firebase test user created in Firebase Console

## Sources

### Primary (HIGH confidence)
- Phase 24 CONTEXT.md and 24-01-PLAN.md -- auth setup pattern, project layout, storageState with IndexedDB
- INTEGRATIONS.md -- MongoDB collections list, connection pattern, Firebase auth flow
- STRUCTURE.md -- API module layout, onboarding components, repository patterns
- TESTING.md -- existing backend test patterns, no frontend tests yet
- CONTEXT.md (Phase 25) -- all locked decisions (D-01 through D-10)

### Secondary (MEDIUM confidence)
- v1.5 research SUMMARY.md, ARCHITECTURE.md, PITFALLS.md -- seeding patterns, fixture architecture, Firebase pitfalls
- npm registry -- verified mongodb@7.0.0, @playwright/test@1.58.2, @faker-js/faker@10.4.0 are current

### Tertiary (LOW confidence)
- Onboarding wizard UI step details -- not verified against live component source; exact selectors need validation during implementation (Open Question 1)
- Visit types collection/endpoint -- not found in INTEGRATIONS.md or STRUCTURE.md; needs API codebase inspection (Open Question 3)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all packages verified, no new packages beyond mongodb devDep
- Architecture: HIGH -- builds directly on Phase 24 patterns; decisions are specific and well-constrained
- Pitfalls: HIGH -- sourced from v1.5 research + Phase 24 experience + domain-specific analysis
- Onboarding wizard details: MEDIUM -- wizard exists but exact UI steps need live validation

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable domain -- Playwright, MongoDB driver, Faker are mature)
