# Phase 54: User Management - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase delivers the support user management feature: a paginated, searchable user list with role and subscription filtering, a read-only user detail page with profile/business/subscription/role sections, and membership summary metric cards on the support dashboard. It also establishes the project-wide API query filtering and sorting convention as a reusable core utility.

</domain>

<decisions>
## Implementation Decisions

### API Query Convention (New Standard)
- **D-01:** Establish a new project-wide query filtering and sorting convention. Filter format: `?filter:<fieldName>:<operator>=value,value2`. Operators: `in`, `eq`, `nq`, `lt`, `gt`, `le`, `ge`, `bt`. Nested fields use dot notation (`field1.fieldNested`). Sorting: `?sort=<field>` (ascending) or `?sort=-<field>` (descending).
- **D-02:** Build reusable query parsing utilities in `src/core/` — available to all modules, not scoped to support.
- **D-03:** Extend `IBaseQueryOptionsDto` to carry structured filter and sort definitions from the request layer through services to repositories. The repository layer determines how to apply filters to MongoDB queries.
- **D-04:** Document the filtering and sorting standards in CLAUDE.md and associated conventions documents (`.planning/codebase/CONVENTIONS.md`).

### User List & Search
- **D-05:** Server-side search and pagination via a new `GET /v1/support/users` endpoint. Supports the new filter/sort query convention.
- **D-06:** Each row displays: name, email, subscription status badge. No role badge or business name in the list — those are visible on the detail page.
- **D-07:** Role filter tabs above the table: All / Support / Customer.
- **D-08:** Subscription status filterable via the filter convention: All / Trialing / Active / Past Due / Canceled / Expired.

### User Detail Page
- **D-09:** Sectioned card layout with separate cards for Profile (name, email), Business (name, trade), Subscription (status, key dates), and Roles (badges).
- **D-10:** New route at `/support/users/:id` — dedicated page with back navigation, matching existing detail page patterns (job detail, customer detail).
- **D-11:** Read-only in this phase. Phase 55 adds role grant/revoke actions, Phase 56 adds impersonation.
- **D-12:** Subscription section shows status badge plus key dates: trial start/end, subscription start, current period end.
- **D-13:** Show Firebase auth metadata: account created date and last sign-in date. API must fetch this from Firebase Admin SDK.

### Subscription Data
- **D-14:** Subscriptions are stored per user (via `userId` field in subscriptions collection) — no business indirection needed.
- **D-15:** User list endpoint uses MongoDB `$lookup` aggregation to join subscription status onto user results in a single query. Efficient for paginated lists.

### Dashboard Metrics
- **D-16:** Dedicated API endpoint `GET /v1/support/dashboard/metrics` returns pre-computed membership counts: total users, active trials, active subscriptions, expired subscriptions, canceled subscriptions.
- **D-17:** Metric cards placed at the top of the support dashboard, above existing content (migration controls, quick links).
- **D-18:** Metric cards are clickable — each navigates to `/support/users` with the appropriate filter pre-applied (e.g., clicking "Active Trials" navigates to `?filter:subscription.status:eq=trialing`).

### Claude's Discretion
- Exact MongoDB aggregation pipeline structure for user list with subscription lookup
- Pagination UI component design (page numbers, prev/next, or load more)
- How role filter tabs and subscription status filter interact (combined or separate filter groups)
- Firebase Admin SDK integration approach for auth metadata (new service or extend existing UserRetriever)
- Dashboard metrics aggregation pipeline details
- Empty state design for user list with no results

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` — Permission/role data model, hydration strategy, seeding approach
- `.planning/phases/52-permission-guard-migration/52-CONTEXT.md` — Permission guard infrastructure, `@RequiresPermission` decorator
- `.planning/phases/53-support-access-routing/53-CONTEXT.md` — Support routing, SupportGuard, sidebar navigation, dashboard shell

### Subscription Infrastructure
- `trade-flow-api/src/subscription/entities/subscription.entity.ts` — Subscription entity with `userId`, `status`, `trialEnd`, `currentPeriodEnd`, `canceledAt`
- `trade-flow-api/src/subscription/data-transfer-objects/subscription.dto.ts` — ISubscriptionDto structure
- `trade-flow-api/src/subscription/enums/subscription-status.enum.ts` — SubscriptionStatus enum values
- `trade-flow-api/src/subscription/repositories/subscription.repository.ts` — Existing subscription repository patterns

### User Infrastructure
- `trade-flow-api/src/user/entities/user.entity.ts` — User entity with `supportRoleIds[]`, `businessRoleIds[]`
- `trade-flow-api/src/user/data-transfer-objects/user.dto.ts` — IUserDto with hydrated roles
- `trade-flow-api/src/user/services/user-retriever.service.ts` — UserRetriever hydration flow
- `trade-flow-api/src/user/repositories/user.repository.ts` — Existing user repository (needs findAll/search)

### Core Patterns
- `trade-flow-api/src/core/data-transfer-objects/query-results.dto.ts` — IQueryResultsDto for pagination
- `trade-flow-api/src/core/utilities/pagination.utility.ts` — Existing pagination utilities to extend
- `trade-flow-api/src/core/collections/dto.collection.ts` — DtoCollection for paged results
- `trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts` — MongoDbFetcher for query execution

### Frontend Patterns
- `trade-flow-ui/src/pages/support/SupportUsersPage.tsx` — Existing skeleton with mock data (stats cards, table, search)
- `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` — Dashboard to add metrics cards to
- `trade-flow-ui/src/features/customers/hooks/useCustomersList.ts` — Client-side list hook pattern (reference for structure, but this will be server-side)

### Convention Documentation
- `trade-flow-api/CLAUDE.md` — Must be updated with filtering/sorting standards
- `.planning/codebase/CONVENTIONS.md` — Must be updated with filtering/sorting conventions

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SupportUsersPage.tsx` — Skeleton already exists with stats cards, table layout, search input (needs real data)
- `SupportDashboardPage.tsx` — Dashboard shell exists with migration controls and quick links
- `DtoCollection<T>` — Already handles paged results, will carry filtered/sorted results
- `IQueryResultsDto` — Pagination metadata structure exists
- shadcn `Table`, `Card`, `Badge` components — all available for user list and detail page
- `useCustomersList` hook — Pattern reference for list state management

### Established Patterns
- Module structure: Controller → Service → Repository with separate entities/DTOs/responses
- Service naming: `*Creator`, `*Retriever`, `*Updater` pattern
- Repository: `findByIdOrFail()`, `findByIds()`, static `COLLECTION` constant
- Frontend: RTK Query endpoints in feature `api/` directories
- Frontend: `[Feature]DataView` components for table/card switching

### Integration Points
- `App.tsx` — New route `/support/users/:id` under SupportGuard branch (set up in Phase 53)
- `navigation.ts` — Users link already in support navigation (set up in Phase 53)
- `UserModule` — New support-specific controller and services
- `CoreModule` — New query parsing utilities

</code_context>

<specifics>
## Specific Ideas

- Metric cards should link to the user list with appropriate filters using the new query convention (e.g., `?filter:subscription.status:eq=trialing`)
- Firebase auth metadata (account created, last sign-in) adds support troubleshooting value on the detail page

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 54-user-management*
*Context gathered: 2026-04-18*
