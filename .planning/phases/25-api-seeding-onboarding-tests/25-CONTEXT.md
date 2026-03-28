# Phase 25: API Seeding Infrastructure + Onboarding Tests - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Build a typed API seeding client for creating and cleaning up test data, then use it to test the auth redirect flows and the full onboarding wizard end-to-end. This phase delivers the data infrastructure that all subsequent E2E test phases (26-28) depend on.

</domain>

<decisions>
## Implementation Decisions

### Seeding client design
- **D-01:** Seeding client calls existing NestJS REST API endpoints (e.g. `POST /v1/customers`) rather than inserting directly into MongoDB. The API's creator services, access controllers, and policies handle data creation consistently — tests don't need to track schema changes.
- **D-02:** Authentication via the Firebase JWT extracted from Phase 24's saved storageState. Bearer token passed in fetch headers. No separate service account.
- **D-03:** Factory functions with auto-chaining dependencies. e.g. `seedJob()` auto-creates a customer if no `customerId` provided. Each function returns the created entity with its ID. Tests only specify what they care about.

### Test data isolation
- **D-04:** Single shared business per test user. Users cannot have multiple businesses, so "fresh business per test" is not viable.
- **D-05:** Cleanup via direct MongoDB connection. Test helper connects to the same MongoDB instance (via `MONGO_URL`) and deletes documents by `businessId` across all relevant collections after each test. No API delete endpoints needed.
- **D-06:** Two Firebase test users with distinct roles:
  - **Pre-onboarded user** — has a business already created. Used by most tests (seeding, CRUD, auth redirect). storageState from Phase 24.
  - **Fresh onboarding user** — no business. Used exclusively by the onboarding wizard test. Separate storageState.
- **D-07:** Unique naming via Faker + timestamp (e.g. "Test Plumbing 1711532400123") for any test-created entities. Human-readable for debugging.

### Onboarding test scope
- **D-08:** Full multi-step wizard walkthrough via UI. The test fills in each step of the onboarding wizard (business details, trade selection, etc.) using the fresh onboarding user.
- **D-09:** Four assertions after wizard completion:
  1. Business created with correct fields (name, trade match what was entered)
  2. Default entities exist (job types, tax rates, items, visit types) — verified via API calls (GET endpoints), not UI navigation
  3. User redirected to the dashboard after completing onboarding
  4. Subsequent login skips the onboarding wizard (log out, log back in, lands on dashboard)

### Auth redirect tests (not discussed — carried from roadmap)
- **D-10:** Auth tests use the pre-onboarded user's storageState. Valid login lands on dashboard; unauthenticated access to protected route redirects to login. Implementation details at Claude's discretion.

### Claude's Discretion
- MongoDB cleanup helper implementation details (connection management, collection list)
- Seeding client internal structure (file organization, TypeScript types)
- Faker library choice and data generation patterns
- Auth redirect test specifics (which protected route to test, exact assertions)
- How the fresh onboarding user's Firebase credentials are injected (env vars, fixture config)

</decisions>

<specifics>
## Specific Ideas

- Seeding client should be a thin typed HTTP wrapper — the API is the source of truth for data creation logic, not the test code
- "I would rather not have to expose deletion endpoints for every type of entity" — direct MongoDB cleanup avoids API surface area bloat
- The constraint that users can only have one business is load-bearing — all isolation strategies must work within this

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 24 infrastructure (foundation)
- `.planning/phases/24-playwright-bootstrap-auth/24-CONTEXT.md` — Auth strategy (programmatic Firebase), selector policy, storageState approach, directory structure, webServer config
- `.planning/phases/24-playwright-bootstrap-auth/24-01-PLAN.md` — Playwright config layout, global setup implementation, fixture patterns

### Codebase architecture
- `.planning/codebase/INTEGRATIONS.md` — MongoDB collections list, Firebase auth flow, API connection details
- `.planning/codebase/STRUCTURE.md` — API module layout, feature directory patterns
- `.planning/codebase/TESTING.md` — Existing test patterns and infrastructure

### API conventions (for seeding client)
- `CLAUDE.md` §Backend (API) Conventions — Controller/service/repository naming, DTO patterns, error handling, path aliases

### No external specs
- Requirements are fully captured in ROADMAP.md success criteria and decisions above

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Phase 24 storageState and global setup — seeding client reuses the saved auth token
- Phase 24 `e2e/fixtures/` directory — extend with seeding fixtures
- Phase 24 `e2e/helpers/` directory — home for the seeding client and MongoDB cleanup helper
- Existing API endpoints (POST /v1/customers, /v1/jobs, /v1/quotes, etc.) — seeding client wraps these

### Established Patterns
- Controller → Service → Repository layering — seeding goes through the full API stack
- Firebase JWT auth — same token works for both UI navigation and API seeding calls
- Playwright fixtures pattern (not POM) — seeding client exposed as a fixture

### Integration Points
- MongoDB connection string (MONGO_URL env var) — cleanup helper needs same connection as API
- Firebase auth for second test user — global setup needs to handle two storageState files
- API base URL — seeding client needs the running API's address (from Playwright config or env)

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 25-api-seeding-onboarding-tests*
*Context gathered: 2026-03-28*
