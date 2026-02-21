# Codebase Concerns

**Analysis Date:** 2026-02-21

## Tech Debt

**Missing Error Monitoring in Production:**
- Issue: Error boundary component has a TODO comment for sending errors to monitoring service (Sentry, etc.)
- Files: `trade-flow-ui/src/components/error-boundary/ErrorBoundary.tsx` (line 37)
- Impact: Production errors in UI are only logged to console, making incident detection and debugging difficult in production environments
- Fix approach: Integrate Sentry (or equivalent) error tracking. Uncomment the Sentry integration code and configure with appropriate DSN in environment variables

**Overly Large Components with Complex State Management:**
- Issue: Multiple UI components exceed 500 lines with complex local state management
- Files:
  - `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` (622 lines)
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (526 lines)
  - `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` (495 lines)
  - `trade-flow-ui/src/features/auth/components/AuthForm.tsx` (453 lines)
- Impact: Components are difficult to test, maintain, and reason about. Multiple state variables (`editingSection`, `formData`, `selectedCustomer`, `selectedJobType`, `customerPopoverOpen`, `jobTypePopoverOpen`, `customerSearch`, `jobTypeSearch`, etc.) increase cognitive load and bug surface area
- Fix approach: Extract reusable sub-components (e.g., separate edit card components), move state to custom hooks, or consider Redux for complex shared state

**Hardcoded Constants in Controllers:**
- Issue: Default job types, tax rates, and items are created with hardcoded templates during business creation
- Files:
  - `trade-flow-api/src/business/services/default-job-types-creator.service.ts` (490 lines)
  - `trade-flow-api/src/business/services/default-business-items-creator.service.ts` (324 lines)
- Impact: Adding new trades or modifying templates requires code changes; limited flexibility for different business types; maintenance burden grows with each new trade type
- Fix approach: Move templates to configuration files or database; implement a template registry system; consider feature flags for different business types

## Known Issues

**Batch Operations Without Atomicity:**
- Issue: Business creation orchestrates multiple sequential async operations without transaction support
- Files: `trade-flow-api/src/business/services/business-creator.service.ts` (lines 31-50)
- Trigger: Create business endpoint with default items/tax rates/job types
- Symptoms: If creation fails midway (e.g., after business is created but before default items), database is left in inconsistent state
- Workaround: Currently no workaround; requires manual cleanup
- Fix approach: Implement MongoDB transactions for the create operation sequence, or implement a rollback mechanism that cleans up orphaned data

**UI Form Submission Without Optimistic Updates:**
- Issue: Dialog forms show "Creating..." state but don't optimistically update Redux cache
- Files:
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (line 220)
  - `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx` (line 161)
  - `trade-flow-ui/src/features/items/components/ItemFormDialog.tsx` (line 83)
- Symptoms: After form submission, user sees loading state then dialog closes, but list doesn't refresh until next page reload or manual refetch
- Trigger: Create any entity through dialog form
- Workaround: Manually navigate away and back or refresh the page
- Fix approach: Update RTK Query cache optimistically on success, or implement manual Redux cache updates in mutation callbacks

**Type Assertions in Quote Line Item Factory:**
- Issue: Complex nested object transformations in `quote-bundle-line-item-factory.service.ts` may have implicit any types
- Files: `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` (143 lines)
- Impact: Runtime errors possible if bundle component structure doesn't match expectations
- Fix approach: Add strict type definitions for bundle components and validate structure before transformation

## Security Considerations

**Firebase Configuration Exposed in Source:**
- Risk: Firebase API key and configuration are stored in environment variables but file structure is in version control
- Files: `trade-flow-ui/src/config/firebase.ts`
- Current mitigation: Uses `import.meta.env.VITE_*` for secrets (good), but .env files should be in .gitignore
- Recommendations:
  - Verify `.env` and `.env.*.local` are in `.gitignore`
  - Add pre-commit hook to prevent accidental .env commits
  - Review .env files for any committed secrets
  - Consider using Husky (already installed) to add `no-env-secrets` rule

**Auth Guard Token Extraction Without Error Handling:**
- Risk: JWT token parsing could fail with misleading error messages
- Files: `trade-flow-api/src/auth/auth.guard.ts` (line with `extractUserInfoFromToken`)
- Current mitigation: Guard has basic error handling, but error messages could leak information
- Recommendations:
  - Add specific error types for auth failures
  - Return generic 401 Unauthorized without exposing JWT parsing details
  - Add request logging for failed auth attempts (fraud detection)

**Business ID Not Validated for Ownership:**
- Risk: Routes accept businessId as parameter; need to verify policy checks prevent cross-business data access
- Files: All controllers (e.g., `trade-flow-api/src/quote/controllers/quote.controller.ts`, `trade-flow-api/src/business/controllers/business.controller.ts`)
- Current mitigation: Access control factory and policy classes in place
- Recommendations:
  - Verify all endpoints using BasePolicy implementations
  - Add integration tests that explicitly test cross-business access denial
  - Consider adding middleware to log all access control checks for audit trail

**Missing Input Sanitization for Text Fields:**
- Risk: Customer names, job descriptions, item names are stored and displayed without HTML escaping
- Files: Multiple form dialogs and display components
- Current mitigation: React's JSX prevents most XSS by default
- Recommendations:
  - Add server-side validation for max lengths (currently only client-side)
  - Consider adding explicit schema validation for special characters in names
  - Verify API responses don't allow HTML/script tags in string fields

## Performance Bottlenecks

**N+1 Queries in Quote Retrieval:**
- Problem: Quote service retrieves quotes then iterates to fetch line items
- Files: `trade-flow-api/src/quote/repositories/quote.repository.ts`, `trade-flow-api/src/quote/services/quote-retriever.service.ts`
- Cause: Repository fetches quote collection, then requires separate query for each quote's line items (if implemented sequentially)
- Improvement path: Implement JOIN or aggregation pipeline in MongoDB; batch line item queries using `findMany` with multiple quote IDs; add caching layer for frequently accessed quotes

**Large Component Re-renders:**
- Problem: BusinessDetails component re-renders entire card UI when editing section changes
- Files: `trade-flow-ui/src/features/business/components/BusinessDetails.tsx` (lines 140-300)
- Cause: Not using `React.memo()` for subcomponents; all tabs re-render even when inactive
- Improvement path: Extract `BusinessCard`, `LocationCard` as memoized components; use lazy loading for tab content; consider suspense boundaries

**Pagination Not Enforced by Default:**
- Problem: Query endpoints may return unlimited results if client doesn't specify pagination
- Files: `trade-flow-api/src/core/utilities/pagination-query-to-base-query-options.utility.ts`
- Cause: Pagination is optional in query parameters
- Improvement path: Set sensible defaults (e.g., limit to 50 items), add maximum page size enforcement, add pagination metadata to all list responses

**Firebase Token Verification on Every Request:**
- Problem: Auth guard re-verifies JWT signature on every request
- Files: `trade-flow-api/src/auth/auth.guard.ts` (lines 1-120)
- Cause: No token caching or verification result caching
- Improvement path: Implement Redis-based token verification cache (short TTL of 5 minutes), add request signature caching to avoid repeated crypto operations

## Fragile Areas

**Selective Update Utility Complex Logic:**
- Files: `trade-flow-api/src/core/utilities/selective-update.utility.ts` (239 lines)
- Why fragile: Complex nested property extraction, field mapping transformation, path parsing with string operations
- Safe modification:
  - Add comprehensive unit tests for all field mapping scenarios
  - Test edge cases: null values, undefined values, nested objects, arrays
  - Add JSDoc examples for each usage pattern
  - Consider extracting path parsing to separate utility function
- Test coverage: Has spec file but coverage may be incomplete for all transformation paths

**Business User Role Assignment Flow:**
- Files:
  - `trade-flow-api/src/business/services/business-creator.service.ts`
  - `trade-flow-api/src/user/services/business-role-assigner.service.ts`
- Why fragile: Multiple sequential operations (create business, assign user role, update onboarding step); if any step fails, system is in inconsistent state
- Safe modification:
  - Add guards to ensure previous steps completed before continuing
  - Implement idempotent role assignment
  - Add validation that user has expected role after assignment
  - Document expected order of operations
- Test coverage: Should have tests for role assignment failure scenarios

**CreateJobDialog Multi-Entity Creation:**
- Files: `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (lines 220-271)
- Why fragile: Creates customer, then job type, then job in sequence; if step 2 fails, step 3 doesn't run but step 1 succeeded
- Safe modification:
  - Add error boundary to catch and display specific step failures
  - Implement rollback mechanism or "create with existing" fallback
  - Add validation at each step before proceeding
  - Consider extracting to custom hook with proper error handling
- Test coverage: No unit tests visible for this component; should add Jest tests

**Quote Line Item Bundle Component Transformation:**
- Files: `trade-flow-api/src/quote/services/quote-bundle-line-item-factory.service.ts` (143 lines)
- Why fragile: Transforms bundle configuration into line items; bundle component structure must match expectations
- Safe modification:
  - Add schema validation for bundle component structure
  - Add try-catch around transformation with detailed error messages
  - Test with all bundle types currently supported
  - Document bundle component interface clearly
- Test coverage: May lack tests for malformed bundle components

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
  - Consider connection pooling proxy (PgBouncer equivalent for MongoDB)

**RTK Query Cache Size:**
- Current capacity: In-memory Redux store cache
- Limit: Large numbers of quotes/jobs will consume memory; no cache eviction strategy visible
- Scaling path:
  - Implement RTK Query cache invalidation strategy
  - Add cache time-to-live (TTL) for stale data
  - Consider Redux persist with storage limits
  - Implement pagination to avoid loading entire datasets

## Dependencies at Risk

**Dinero.js Beta Version:**
- Risk: Project uses `dinero.js@1.9.1` and UI uses alpha `@dinero.js/currencies@2.0.0-alpha.14`
- Impact: Major version mismatch; alpha package may have breaking changes; v2 API is different from v1
- Migration plan:
  - Test complete money handling with v2 when stable release available
  - Create wrapper service for currency operations to ease migration
  - Pin versions and add deprecation warnings to code using Dinero directly
  - Consider moving to `currency.js` or `decimal.js` for better stability

**Mongoose 9.1.5 with MongoDB 7.0:**
- Risk: Mongoose 9.x is recent; May have compatibility issues with legacy code
- Impact: Breaking changes in schema handling or query behavior possible
- Migration plan:
  - Run full test suite against each minor version
  - Monitor GitHub issues for reported breaking changes
  - Keep MongoDB driver updated in lock file
  - Add integration tests that verify critical data operations

**Firebase SDK 12.x:**
- Risk: Recent major version; Auth API may have changes
- Impact: Token handling, sign-out, session management could have breaking changes
- Migration plan:
  - Document current Firebase SDK usage patterns
  - Add tests for auth flow scenarios
  - Monitor Firebase release notes for breaking changes
  - Consider feature flag for auth implementation to ease migration

## Missing Critical Features

**No Audit Logging:**
- Problem: Business operations (create, update, delete) are not logged for audit trail
- Blocks: Cannot track who made changes, when, or what changed; regulatory compliance issues
- Implementation path: Add audit log service, capture before/after state, store in separate collection

**No Rate Limiting:**
- Problem: API endpoints accept unlimited requests per client
- Blocks: DDoS attacks possible; brute force attacks on auth endpoints; resource exhaustion
- Implementation path: Add rate limiting middleware (using express-rate-limit or similar), implement sliding window rate limiter in Redis

**No Data Export/Backup:**
- Problem: Users cannot export their business data
- Blocks: Data portability requirement; user requested feature
- Implementation path: Add export endpoint (CSV, JSON), implement scheduled backups to S3

**No Undo/Soft Deletes:**
- Problem: Deletes are permanent in database
- Blocks: User recovery from accidental deletion; audit trail preservation
- Implementation path: Add `deletedAt` field to entities, implement soft delete pattern, add restore endpoint

## Test Coverage Gaps

**UI Component Integration Tests Missing:**
- What's not tested: Dialog submission workflows, form validation errors, async API integration
- Files:
  - `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx`
  - `trade-flow-ui/src/features/business/components/BusinessDetails.tsx`
  - `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx`
- Risk: Form workflows could break without detection; users would encounter runtime errors
- Priority: High - these are critical user workflows

**API Policy/Authorization Tests:**
- What's not tested: Cross-business access denial, role-based authorization edge cases
- Files: `trade-flow-api/src/*/policies/*.policy.ts`, `trade-flow-api/src/*/repositories/*.repository.ts`
- Risk: Authorization bypass possible; users could access other users' data
- Priority: Critical - security issue

**Error Boundary Error Capture:**
- What's not tested: Error boundary actually catches component errors, fallback rendering
- Files: `trade-flow-ui/src/components/error-boundary/*.tsx`
- Risk: Error boundary may not work; unhandled errors crash entire app
- Priority: High - reliability issue

**Migration System Rollback Tests:**
- What's not tested: Migration rollback functionality, data consistency after rollback
- Files: `trade-flow-api/src/migration/services/migration-runner.service.ts`
- Risk: Rollback fails silently; data becomes inconsistent
- Priority: Medium - deployment risk

**Selective Update Utility Edge Cases:**
- What's not tested: Field mappings with complex transformations, null handling, undefined values
- Files: `trade-flow-api/src/core/utilities/selective-update.utility.ts`
- Risk: Silent failures in update operations; partial updates committed
- Priority: Medium - data integrity issue

---

*Concerns audit: 2026-02-21*
