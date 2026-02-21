# Codebase Concerns

**Analysis Date:** 2026-02-21

## Tech Debt

**Missing Error Monitoring in Production:**
- Issue: Error boundary component has a TODO comment for sending errors to monitoring service (Sentry, etc.)
- Files: `trade-flow-ui/src/components/error-boundary/ErrorBoundary.tsx` (line 37)
- Impact: Production errors in UI are only logged to console, making incident detection and debugging difficult in production environments
- Fix approach: Integrate Sentry (or equivalent) error tracking. Uncomment the Sentry integration code and configure with appropriate DSN in environment variables

**SendGrid Email Service Lacks Error Handling:**
- Issue: EmailSenderService has no error handling or retry logic for failed sends
- Files: `trade-flow-api/src/email/services/email-sender.service.ts` (lines 15-23)
- Impact: Email failures are silent; no user notification if SendGrid API fails; emails can be lost without trace
- Fix approach: Implement try-catch with proper logging, add retry mechanism or message queue (Bull/RabbitMQ) for failed emails, expose email status to API responses

**SendGrid API Key Not Validated at Startup:**
- Issue: SendGrid initialization doesn't validate API key is present or functional
- Files: `trade-flow-api/src/email/services/email-sender.service.ts` (lines 9-12)
- Impact: Service silently degrades if SENDGRID_API_KEY env var is missing; errors only surface on first email attempt
- Fix approach: Add startup validation in bootstrap service, throw error if API key missing in production environment

**Console Logging in Migration Template:**
- Issue: Example migration file uses console.log instead of proper logging
- Files: `trade-flow-api/src/migration/migrations/20240101000000-example-create-users-index.migration.ts`
- Impact: Developers may copy template and introduce console.log in production migrations, bypassing structured logging
- Fix approach: Update template to use AppLogger service, remove all console calls from example

**Overly Large Components with Complex State Management:**
- Issue: Multiple UI components exceed 500 lines with complex local state management
- Files:
  - `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` (622 lines)
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (526 lines)
  - `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` (495 lines)
  - `trade-flow-ui/src/features/auth/components/AuthForm.tsx` (453 lines)
- Impact: Components are difficult to test, maintain, and reason about. Multiple state variables increase cognitive load and bug surface area
- Fix approach: Extract reusable sub-components, move state to custom hooks, consider Redux for complex shared state

**Hardcoded Constants in Creator Services:**
- Issue: Default job types and items are created with hardcoded templates during business creation
- Files:
  - `trade-flow-api/src/business/services/default-job-types-creator.service.ts` (490 lines)
  - `trade-flow-api/src/business/services/default-business-items-creator.service.ts` (324 lines)
- Impact: Adding new trades or modifying templates requires code changes; limited flexibility; maintenance burden
- Fix approach: Move templates to configuration files or database; implement a template registry system

## Known Issues

**Hardcoded Email in Super User Migration:**
- Symptoms: Super user creation uses hardcoded email address instead of configuration
- Files: `trade-flow-api/src/migration/migrations/20260104120000-create-initial-super-user.migration.ts` (line 26)
- Value: `mhcrankshaw2@gmail.com` hardcoded
- Trigger: During initial database bootstrap
- Impact: Super user email bound to developer's personal email; cannot be changed without code modification
- Workaround: Manually edit migration before running, or update user email in database post-creation
- Fix approach: Use environment variable `INITIAL_SUPER_USER_EMAIL` with fallback validation

**Batch Operations Without Atomicity:**
- Issue: Business creation orchestrates multiple sequential async operations without transaction support
- Files: `trade-flow-api/src/business/services/business-creator.service.ts` (lines 31-50)
- Trigger: Create business endpoint with default items/tax rates/job types
- Symptoms: If creation fails midway (e.g., after business is created but before default items), database is left in inconsistent state
- Workaround: Currently no workaround; requires manual cleanup
- Fix approach: Implement MongoDB transactions for the create operation sequence, or implement a rollback mechanism

**UI Form Submission Without Optimistic Updates:**
- Issue: Dialog forms show "Creating..." state but don't optimistically update Redux cache
- Files:
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (line 220)
  - `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx` (line 161)
  - `trade-flow-ui/src/features/items/components/ItemFormDialog.tsx` (line 83)
- Symptoms: After form submission, user sees loading state then dialog closes, but list doesn't refresh until next page reload
- Trigger: Create any entity through dialog form
- Workaround: Manually navigate away and back or refresh the page
- Fix approach: Update RTK Query cache optimistically on success, or implement manual Redux cache updates in mutation callbacks

**API Base URL Fallback to Localhost:**
- Issue: UI defaults to `http://localhost:3000` if VITE_API_BASE_URL not set
- Files: `trade-flow-ui/src/services/api.ts` (line 6)
- Impact: Production builds could silently connect to wrong API if env var missing
- Fix approach: Require explicit VITE_API_BASE_URL, fail at startup in production if undefined

## Security Considerations

**Firebase Configuration Exposed in Source:**
- Risk: Firebase API key and configuration are stored in environment variables but file structure is in version control
- Files: `trade-flow-ui/src/config/firebase.ts`
- Current mitigation: Uses `import.meta.env.VITE_*` for secrets (good); Firebase credentials are public by design
- Recommendations:
  - Verify Firebase project has API key restrictions in Google Cloud Console (domain/IP restrictions enabled)
  - Ensure .env files are in .gitignore
  - Document that Firebase public keys are not considered secrets

**Auth Guard Token Extraction Without Error Handling:**
- Risk: JWT token parsing could fail with misleading error messages
- Files: `trade-flow-api/src/auth/auth.guard.ts` (extractUserInfoFromToken method)
- Current mitigation: Guard has basic error handling, but error messages could leak information
- Recommendations:
  - Add specific error types for auth failures
  - Return generic 401 Unauthorized without exposing JWT parsing details
  - Add request logging for failed auth attempts (fraud detection)

**Business ID Not Validated for Ownership:**
- Risk: Routes accept businessId as parameter; need to verify policy checks prevent cross-business data access
- Files: All controllers using BasePolicy
- Current mitigation: Access control factory and policy classes in place
- Recommendations:
  - Verify all endpoints using policy implementations
  - Add integration tests that explicitly test cross-business access denial
  - Consider adding middleware to log all access control checks for audit trail

**Missing Input Sanitization for Text Fields:**
- Risk: Customer names, job descriptions, item names are stored and displayed without explicit validation
- Files: Multiple form dialogs and display components
- Current mitigation: React's JSX prevents most XSS by default; class-validator on server-side
- Recommendations:
  - Add server-side max length validation (currently only client-side)
  - Consider adding schema validation for special characters in names
  - Verify API responses don't allow HTML/script tags in string fields

**No Rate Limiting on API Endpoints:**
- Risk: Email, authentication, and resource creation endpoints lack rate limiting
- Impact: Susceptible to spam, brute force attacks, and DDoS
- Files: API controllers missing @Throttle decorator from @nestjs/throttler
- Recommendations: Add rate limiting on auth endpoints (login, signup), email endpoints, and resource creation

## Performance Bottlenecks

**N+1 Queries in Quote Retrieval:**
- Problem: Quote service may retrieve quotes then iterate to fetch line items separately
- Files: `trade-flow-api/src/quote/repositories/quote.repository.ts`, `trade-flow-api/src/quote/services/quote-retriever.service.ts`
- Cause: If line items fetched per quote, causes N+1 query pattern
- Improvement path: Use MongoDB aggregation pipeline or batch `findMany` queries with multiple quote IDs

**Large Component Re-renders:**
- Problem: BusinessDetails component re-renders entire card UI when editing section changes
- Files: `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` (lines 140-300)
- Cause: Not using `React.memo()` for subcomponents; all tabs re-render even when inactive
- Improvement path: Extract `BusinessCard`, `LocationCard` as memoized components; use lazy loading for tab content

**Pagination Not Enforced by Default:**
- Problem: Query endpoints may return unlimited results if client doesn't specify pagination
- Files: `trade-flow-api/src/core/utilities/pagination-query-to-base-query-options.utility.ts`
- Cause: Pagination is optional in query parameters
- Improvement path: Set sensible defaults (50 items), add maximum page size enforcement

**Firebase Token Verification on Every Request:**
- Problem: Auth guard re-verifies JWT signature on every request
- Files: `trade-flow-api/src/auth/auth.guard.ts`
- Cause: No token caching or verification result caching
- Improvement path: Implement Redis-based token verification cache (5 min TTL)

**Selective Update Utility Complexity:**
- Problem: createSelectiveUpdateObject utility is 239 lines with complex nested object transformation logic
- Files: `trade-flow-api/src/core/utilities/selective-update.utility.ts` (239 lines)
- Cause: Attempts to handle multiple field mapping scenarios (nested paths, property extraction, transformations)
- Impact: Difficult to debug and maintain; possible edge cases with complex objects
- Improvement path: Simplify to handle only direct field mappings; create separate utilities for transformations

## Fragile Areas

**Selective Update Utility Complex Logic:**
- Files: `trade-flow-api/src/core/utilities/selective-update.utility.ts` (239 lines)
- Why fragile: Complex nested property extraction, field mapping transformation, path parsing with string operations
- Safe modification:
  - Add comprehensive unit tests for all field mapping scenarios
  - Test edge cases: null values, undefined values, nested objects, arrays
  - Add JSDoc examples for each usage pattern
  - Consider extracting path parsing to separate utility function

**Business User Role Assignment Flow:**
- Files:
  - `trade-flow-api/src/business/services/business-creator.service.ts`
  - `trade-flow-api/src/user/services/business-role-assigner.service.ts`
- Why fragile: Multiple sequential operations (create business, assign user role, update onboarding); if any step fails, system is inconsistent
- Safe modification:
  - Add guards to ensure previous steps completed before continuing
  - Implement idempotent role assignment
  - Add validation that user has expected role after assignment

**CreateJobDialog Multi-Entity Creation:**
- Files: `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (lines 220-271)
- Why fragile: Creates customer, then job type, then job in sequence; if step 2 fails, step 1 succeeded (orphaned data)
- Safe modification:
  - Add error boundary to catch and display specific step failures
  - Implement rollback mechanism or "create with existing" fallback
  - Add validation at each step before proceeding

**Migration System:**
- Files: `trade-flow-api/src/migration/*`
- Why fragile:
  - Migration ordering depends on timestamp filenames (no explicit ordering)
  - No validation migrations run in correct order
  - Rollback (down) is not validated; could leave database inconsistent
  - Initial super user creation depends on INITIAL_SUPER_USER_EXTERNAL_ID env var
- Safe modification:
  - Test rollback operations locally before deploying
  - Add bootstrap checks to ensure migrations can run idempotently
  - Implement migration status validation before app startup

**OnboardingStorage Silent Failures:**
- Files: `trade-flow-ui/src/store/utils/onboarding-storage.ts`
- Why fragile: localStorage access wrapped in try-catch that silently fails
- Issue: State could get out of sync if localStorage unavailable (private mode, storage quota exceeded)
- Safe modification: Add logging for failures, consider sessionStorage fallback, provide UI feedback

## Scaling Limits

**Firebase Authentication JWT Verification:**
- Current capacity: Single request = 1 crypto operation per request
- Limit: At 1000 RPS, API will spend significant CPU on JWT verification
- Scaling path:
  - Implement token caching with Redis (5 min TTL)
  - Use Firebase Admin SDK's built-in token verification cache
  - Consider JWKS caching for public key rotation

**MongoDB Connection Pool:**
- Current capacity: Default MongoDB driver pool size
- Limit: Need to verify pool size matches expected concurrent connections
- Scaling path:
  - Increase `maxPoolSize` in Mongoose connection options
  - Monitor connection usage with APM
  - Consider connection pooling proxy

**RTK Query Cache Size:**
- Current capacity: In-memory Redux store cache
- Limit: Large numbers of quotes/jobs will consume memory; no cache eviction strategy visible
- Scaling path:
  - Implement RTK Query cache invalidation strategy
  - Add cache time-to-live (TTL) for stale data
  - Implement pagination to avoid loading entire datasets

**Single SendGrid API Key:**
- Resource: SendGrid rate limits
- Current capacity: ~100 emails/second on standard plan
- Limit: If business scales to thousands of users, email sending bottlenecked
- Scaling path: Monitor email queue depth, consider email service queue (Bull/RabbitMQ), upgrade SendGrid plan

## Dependencies at Risk

**Dinero.js Version Mismatch:**
- Risk: UI uses `^2.0.0-alpha.14`, API uses `^1.9.1` - version mismatch and alpha status
- Impact: Currency calculations could break on minor updates; API and UI handle money differently
- Files:
  - `trade-flow-ui/package.json` (line 14)
  - `trade-flow-api/package.json` (line 37)
- Migration plan:
  - Decide on v1.9.1 or stable v2.0 and standardize both
  - Add tests for currency calculations to catch breaking changes
  - Monitor Dinero.js releases for stability

**Mongoose 9.1.5 with MongoDB 7.0:**
- Risk: Mongoose 9.x is recent; may have compatibility issues
- Impact: Breaking changes in schema handling or query behavior possible
- Migration plan:
  - Run full test suite against each minor version
  - Monitor GitHub issues for reported breaking changes
  - Keep MongoDB driver updated in lock file

**Firebase SDK 12.x:**
- Risk: Recent major version; Auth API may have changes
- Impact: Token handling, sign-out, session management could change
- Migration plan:
  - Document current Firebase SDK usage patterns
  - Add tests for auth flow scenarios
  - Monitor Firebase release notes for breaking changes

## Missing Critical Features

**No Audit Logging:**
- Problem: Business operations (create, update, delete) are not logged for audit trail
- Blocks: Cannot track who made changes, when, or what changed; regulatory compliance issues
- Implementation path: Add audit log service, capture before/after state, store in separate collection

**No Rate Limiting:**
- Problem: API endpoints accept unlimited requests per client
- Blocks: DDoS attacks possible; brute force attacks on auth endpoints
- Implementation path: Add rate limiting middleware, implement sliding window rate limiter in Redis

**No Data Export/Backup:**
- Problem: Users cannot export their business data
- Blocks: Data portability requirement; user-requested feature
- Implementation path: Add export endpoint (CSV, JSON), implement scheduled backups

**No Undo/Soft Deletes:**
- Problem: Deletes are permanent in database
- Blocks: User recovery from accidental deletion; audit trail preservation
- Implementation path: Add `deletedAt` field to entities, implement soft delete pattern

**Job Scheduling System Not Implemented:**
- Problem: JobDetailTabs shows MOCK_SCHEDULES but no backend implementation
- Blocks: Users cannot schedule follow-up visits or view calendar
- Files: `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` (lines 35-50)
- Status: UI exists, API endpoints missing

**Invoice Generation Not Implemented:**
- Problem: MOCK_INVOICES in JobDetailTabs but no invoice creation/management endpoints
- Blocks: Users cannot generate invoices from quotes or jobs; cannot track payments
- Files: `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` (lines 62-77)
- Status: UI mockup only

**Quote Acceptance/Rejection Workflow Missing:**
- Problem: Quote status displayed but no workflow to accept/reject quotes
- Blocks: Cannot track quote pipeline or convert accepted quotes to jobs
- Files: `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx`
- Status: Read-only display, no user actions

**Note Taking on Jobs Not Implemented:**
- Problem: MOCK_NOTES displayed but no backend endpoint to create/edit notes
- Blocks: Users cannot document job progress or customer requests
- Files: `trade-flow-ui/src/features/jobs/components/JobDetailTabs.tsx` (lines 79-92)
- Status: Read-only mock data only

**File Attachment/Photo Upload Not Implemented:**
- Problem: Photo button in JobDetailTabs but no upload implementation
- Blocks: Cannot attach before/after photos, invoices, or documents to jobs
- Impact: Limited documentation capability for trade work
- Status: UI scaffolding only

## Test Coverage Gaps

**UI Component Integration Tests Missing:**
- What's not tested: Dialog submission workflows, form validation errors, async API integration
- Files:
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (no .test.tsx)
  - `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` (no .test.tsx)
  - `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx` (no .test.tsx)
- Risk: Form workflows could break without detection; users encounter runtime errors
- Priority: HIGH - these are critical user-facing workflows

**API Email Service Tests Missing:**
- What's not tested: EmailSenderService.sendEmail method, error handling, retry logic
- Files: `trade-flow-api/src/email/services/email-sender.service.ts` (no .spec.ts)
- Risk: Email failures undetected until production
- Priority: CRITICAL - email is critical path for onboarding

**API Policy/Authorization Tests Incomplete:**
- What's not tested: Cross-business access denial, role-based authorization edge cases
- Files: `trade-flow-api/src/*/policies/*.policy.ts`
- Risk: Authorization bypass possible; users could access other users' data
- Priority: CRITICAL - security issue

**Error Boundary Error Capture Not Tested:**
- What's not tested: Error boundary actually catches component errors, fallback rendering
- Files: `trade-flow-ui/src/components/error-boundary/ErrorBoundary.tsx` (no .test.tsx)
- Risk: Error boundary may not work; unhandled errors crash entire app
- Priority: HIGH - reliability issue

**Migration System Rollback Tests Missing:**
- What's not tested: Migration rollback functionality, data consistency after rollback
- Files: `trade-flow-api/src/migration/services/migration-runner.service.ts`
- Risk: Rollback fails silently; data becomes inconsistent
- Priority: MEDIUM - deployment risk

**Selective Update Utility Edge Cases Not Tested:**
- What's not tested: Field mappings with complex transformations, null handling, undefined values
- Files: `trade-flow-api/src/core/utilities/selective-update.utility.ts`
- Risk: Silent failures in update operations; partial updates committed
- Priority: MEDIUM - data integrity issue

---

*Concerns audit: 2026-02-21*
