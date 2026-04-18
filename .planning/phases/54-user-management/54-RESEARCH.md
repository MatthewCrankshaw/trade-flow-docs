# Phase 54: User Management - Research

**Researched:** 2026-04-18
**Domain:** Support user management (API + UI), query filtering infrastructure, MongoDB aggregation
**Confidence:** HIGH

## Summary

Phase 54 delivers three interconnected capabilities: (1) a new project-wide query filtering/sorting convention built as reusable core utilities, (2) support-facing user list and detail pages with real data replacing mock skeletons, and (3) membership summary metric cards on the support dashboard. The phase spans both codebases and introduces two patterns new to the project: MongoDB aggregation pipelines (for `$lookup` joins) and server-side filtered/sorted/paginated queries (all existing list endpoints use client-side filtering or simple pagination).

The critical technical challenges are: extending `MongoDbFetcher` to support aggregation pipelines (it currently only wraps `find` and `findOne`), installing and integrating `firebase-admin` SDK for auth metadata (D-13), and designing the query filter parsing convention (D-01/D-02) to be both URL-friendly and extensible. The frontend must also be extended to handle paginated responses -- the existing `StandardResponse` type does not include a `pagination` field despite the API supporting it.

**Primary recommendation:** Build the core query infrastructure first (filter parser, aggregation support, pagination response types), then layer the support user endpoints and UI on top.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Filter format: `?filter:<fieldName>:<operator>=value,value2`. Operators: `in`, `eq`, `nq`, `lt`, `gt`, `le`, `ge`, `bt`. Nested fields use dot notation. Sorting: `?sort=<field>` (ascending) or `?sort=-<field>` (descending).
- **D-02:** Build reusable query parsing utilities in `src/core/` -- available to all modules.
- **D-03:** Extend `IBaseQueryOptionsDto` to carry structured filter and sort definitions.
- **D-04:** Document the filtering and sorting standards in CLAUDE.md and conventions documents.
- **D-05:** Server-side search and pagination via `GET /v1/support/users`.
- **D-06:** Each row displays: name, email, subscription status badge. No role badge or business name in the list.
- **D-07:** Role filter tabs above the table: All / Support / Customer.
- **D-08:** Subscription status filterable via the filter convention.
- **D-09:** Sectioned card layout for user detail: Profile, Business, Subscription, Roles.
- **D-10:** New route at `/support/users/:id`.
- **D-11:** Read-only in this phase.
- **D-12:** Subscription section shows status badge plus key dates.
- **D-13:** Show Firebase auth metadata: account created date and last sign-in date. API must fetch from Firebase Admin SDK.
- **D-14:** Subscriptions stored per user via `userId` field.
- **D-15:** User list endpoint uses MongoDB `$lookup` aggregation to join subscription status.
- **D-16:** Dedicated `GET /v1/support/dashboard/metrics` endpoint for membership counts.
- **D-17:** Metric cards at top of support dashboard.
- **D-18:** Metric cards are clickable with pre-applied filters.

### Claude's Discretion
- Exact MongoDB aggregation pipeline structure for user list with subscription lookup
- Pagination UI component design (page numbers, prev/next, or load more)
- How role filter tabs and subscription status filter interact
- Firebase Admin SDK integration approach (new service or extend UserRetriever)
- Dashboard metrics aggregation pipeline details
- Empty state design for user list

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| UMGT-01 | Support user can view a paginated list of all users with search by name or email | D-05 user list endpoint, D-01/D-02 query convention, D-15 aggregation pipeline, pagination infrastructure |
| UMGT-02 | User list displays role (support badge), subscription status, and business name | D-06 defines list columns (name, email, subscription status badge); D-09 puts role badges and business name on detail page instead |
| UMGT-03 | Support user can view user detail page with profile, business, subscription, roles | D-09 sectioned card layout, D-10 route, D-12/D-13 subscription and Firebase metadata |
| UMGT-04 | Support dashboard shows membership summary cards | D-16 metrics endpoint, D-17 card placement, D-18 clickable navigation |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Query filter parsing | API / Backend | -- | Server-side filter/sort parsing is a backend concern; URL construction is frontend |
| User list with subscription join | Database / Storage | API / Backend | MongoDB `$lookup` aggregation runs at database tier; API orchestrates and returns |
| User detail with Firebase metadata | API / Backend | Browser / Client | Firebase Admin SDK runs server-side; frontend renders |
| Dashboard metrics computation | Database / Storage | API / Backend | Aggregation pipeline computes counts; API exposes endpoint |
| Filter/search UI state | Browser / Client | -- | URL search params, debounced input, tab state are frontend concerns |
| Pagination navigation | Browser / Client | API / Backend | UI renders controls; API handles skip/limit |

## Standard Stack

### Core (already installed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| mongodb | 7.0.0 | Native MongoDB driver with aggregation support | Already in project; `$lookup` and `$facet` available via `collection.aggregate()` [VERIFIED: trade-flow-api/package.json] |
| class-validator | 0.14.1 | Request DTO validation | Already used project-wide [VERIFIED: trade-flow-api/package.json] |
| luxon | 3.5.1 | Date/time handling in DTOs and responses | Project standard [VERIFIED: trade-flow-api/package.json] |
| @reduxjs/toolkit | 2.11.2 | RTK Query for API data fetching | Project standard [VERIFIED: trade-flow-ui/package.json] |

### New Dependency Required

| Library | Version | Purpose | Why Needed |
|---------|---------|---------|------------|
| firebase-admin | 13.8.0 | Firebase Auth user metadata (creation time, last sign-in) | D-13 requires server-side Firebase Auth access; current project only has client-side Firebase SDK [VERIFIED: npm registry] |

**Installation (trade-flow-api):**
```bash
npm install firebase-admin
```

**Version verification:**
- `firebase-admin@13.8.0` is current as of 2026-04-18 [VERIFIED: npm registry]

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| firebase-admin SDK | Store auth metadata in local users collection | Avoids new dependency but requires migration to backfill existing users; metadata drifts from Firebase |
| MongoDB `$lookup` | Multiple queries + manual join in service | Simpler but N+1 problem for paginated lists; D-15 explicitly mandates `$lookup` |

## Architecture Patterns

### System Architecture Diagram

```
Browser                          API (NestJS)                         Database (MongoDB)
  |                                |                                      |
  |-- GET /support/users --------->|                                      |
  |   ?filter:sub.status:eq=active |                                      |
  |   &search=john&page=2          |                                      |
  |                                |-- Parse query filters (core util) -->|
  |                                |-- Build aggregation pipeline ------->|
  |                                |   [$match, $lookup, $facet]          |
  |                                |<-- Paginated user+subscription docs -|
  |                                |                                      |
  |                                |-- Firebase Admin: getUser(uid) ----->| Firebase Auth
  |                                |<-- { creationTime, lastSignIn } -----|
  |                                |                                      |
  |<-- { data: [], pagination } ---|                                      |
  |                                |                                      |
  |-- GET /support/dashboard/      |                                      |
  |   metrics ------------------->|                                      |
  |                                |-- Aggregation $group pipeline ------>|
  |                                |<-- { total, trialing, active, ... } -|
  |<-- { data: [metrics] } --------|                                      |
```

### Recommended Project Structure

**Backend (new files in trade-flow-api):**
```
src/
├── core/
│   ├── data-transfer-objects/
│   │   └── base-query-options.dto.ts    # Extended with filters[]
│   ├── interfaces/
│   │   └── query-filter.interface.ts    # IQueryFilter definition
│   └── utilities/
│       ├── query-filter-parser.utility.ts   # Parse ?filter:field:op=val
│       └── query-sort-parser.utility.ts     # Parse ?sort=field / ?sort=-field
├── support/
│   ├── support.module.ts
│   ├── controllers/
│   │   └── support-user.controller.ts
│   ├── services/
│   │   ├── support-user-retriever.service.ts
│   │   └── support-dashboard-metrics.service.ts
│   ├── repositories/
│   │   └── support-user.repository.ts       # Aggregation pipelines
│   ├── data-transfer-objects/
│   │   ├── support-user.dto.ts              # User + subscription + business
│   │   └── dashboard-metrics.dto.ts
│   ├── responses/
│   │   ├── support-user.response.ts
│   │   ├── support-user-detail.response.ts
│   │   └── dashboard-metrics.response.ts
│   ├── services/
│   │   └── firebase-auth-metadata.service.ts  # Firebase Admin wrapper
│   └── test/
│       ├── services/
│       ├── repositories/
│       └── mocks/
```

**Frontend (new files in trade-flow-ui):**
```
src/
├── features/
│   └── support/
│       ├── api/
│       │   ├── supportApi.ts           # RTK Query endpoints
│       │   └── index.ts
│       ├── components/
│       │   ├── MembershipMetrics.tsx
│       │   ├── UserListFilters.tsx
│       │   ├── UserListTable.tsx
│       │   ├── UserListPagination.tsx
│       │   ├── SubscriptionBadge.tsx
│       │   └── RoleBadge.tsx
│       ├── hooks/
│       │   └── useSupportUsers.ts
│       └── index.ts
├── pages/
│   └── support/
│       └── SupportUserDetailPage.tsx
└── types/
    └── api.types.ts                    # Extended with pagination + support types
```

### Pattern 1: Query Filter Convention (D-01)

**What:** A standardized URL query parameter format for filtering and sorting API results.
**When to use:** Any paginated list endpoint that needs server-side filtering.

```typescript
// Source: D-01 decision from CONTEXT.md
// URL: GET /v1/support/users?filter:subscription.status:eq=active&filter:supportRoleIds:in=id1,id2&sort=-email&page=2&limit=20

// Parsed filter structure
interface IQueryFilter {
  field: string;        // "subscription.status"
  operator: FilterOperator;  // "eq"
  value: string | string[];  // "active" or ["id1", "id2"]
}

type FilterOperator = "in" | "eq" | "nq" | "lt" | "gt" | "le" | "ge" | "bt";

// Extended IBaseQueryOptionsDto
interface IBaseQueryOptionsDto {
  readonly pagination?: {
    readonly limit: number;
    readonly page: number;
  };
  readonly sort?: Record<string, 1 | -1>;
  readonly filters?: IQueryFilter[];
  readonly search?: string;
}
```

### Pattern 2: MongoDB Aggregation via MongoDbFetcher

**What:** Add an `aggregate` method to `MongoDbFetcher` for pipelines.
**When to use:** When queries need `$lookup`, `$facet`, `$group`, or other aggregation stages.

```typescript
// Source: Existing MongoDbFetcher pattern [VERIFIED: trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts]
// New method to add:
public async aggregate<TResult>(
  collectionName: string,
  pipeline: Document[],
): Promise<TResult[]> {
  const db = await this.connection.getDb();
  this.logger.log(`aggregate on collection ${collectionName}`);
  return db.collection(collectionName).aggregate<TResult>(pipeline).toArray();
}
```

### Pattern 3: User List Aggregation Pipeline (D-15)

**What:** Single MongoDB aggregation query that joins users with subscriptions.
**When to use:** `GET /v1/support/users` endpoint.

```typescript
// Source: MongoDB $lookup documentation [CITED: https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/]
const pipeline = [
  // Stage 1: Match filters (search, role type)
  { $match: matchStage },
  // Stage 2: Lookup subscription
  {
    $lookup: {
      from: "subscriptions",
      localField: "_id",          // user._id (ObjectId)
      foreignField: "userId",     // subscription.userId (string) -- needs conversion
      as: "subscription",
      pipeline: [
        { $project: { status: 1, trialEnd: 1, currentPeriodEnd: 1, canceledAt: 1, createdAt: 1 } },
      ],
    },
  },
  // Stage 3: Unwind subscription (0 or 1)
  { $unwind: { path: "$subscription", preserveNullAndEmptyArrays: true } },
  // Stage 4: Apply subscription status filter (post-lookup)
  ...(subscriptionFilter ? [{ $match: { "subscription.status": subscriptionFilter } }] : []),
  // Stage 5: Facet for pagination
  {
    $facet: {
      data: [
        { $sort: sortStage },
        { $skip: (page - 1) * limit },
        { $limit: limit },
      ],
      totalCount: [{ $count: "count" }],
    },
  },
];
```

**Important note on `$lookup` join field types:** The `subscription.userId` field stores the user's internal ID as a string (not ObjectId). The `user._id` is an ObjectId. The `$lookup` pipeline needs to handle this type mismatch, likely by converting `_id` to string in a pipeline `$let` or by using a pipeline-style `$lookup` with `$expr` and `$toString`. [VERIFIED: subscription.entity.ts shows `userId: string`, user.entity.ts shows `_id: ObjectId`]

### Pattern 4: Firebase Admin SDK Integration (D-13)

**What:** Server-side Firebase Auth access for user metadata.
**When to use:** User detail page needs `creationTime` and `lastSignInTime`.

```typescript
// Source: Firebase Admin SDK [ASSUMED]
import * as admin from "firebase-admin";

// Initialize once (in module or service constructor)
admin.initializeApp({
  projectId: process.env.FIREBASE_PROJECT_ID,
});

// Fetch user metadata
const firebaseUser = await admin.auth().getUser(externalAuthUserId);
// firebaseUser.metadata.creationTime -> "Sat, 18 Apr 2026 10:00:00 GMT"
// firebaseUser.metadata.lastSignInTime -> "Sat, 18 Apr 2026 12:00:00 GMT"
```

### Anti-Patterns to Avoid

- **N+1 queries for subscription data in user list:** Do NOT fetch users then loop to fetch each subscription. Use `$lookup` aggregation as specified in D-15.
- **Client-side filtering for user list:** Do NOT fetch all users and filter in the browser. This is server-side (D-05).
- **Storing Firebase metadata in users collection:** Firebase Auth is the source of truth for creation/sign-in times. Fetching on-demand avoids data drift. Only fetch for detail page (single user), never for list.
- **Using `MongoDbFetcher.findMany()` for aggregation queries:** The existing `findMany` uses `find()` which cannot do `$lookup`. Must use `aggregate()`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Query parameter parsing | Custom regex parser | Structured parser utility with validated operators | Edge cases: multiple values, dot notation, type coercion, injection |
| Subscription status join | Service-layer loop with individual DB calls | MongoDB `$lookup` in aggregation pipeline | N+1 query elimination; pagination accuracy |
| Firebase user metadata | Storing creation/sign-in dates locally | `firebase-admin` SDK `auth().getUser()` | Single source of truth; no migration needed |
| Pagination math | Manual skip/limit calculation | Extend existing pagination utilities | Off-by-one errors, boundary conditions |
| Debounced search | `setTimeout`/`clearTimeout` in component | Custom hook with `useRef` for timer cleanup | Memory leaks, stale closure bugs, race conditions |

## Common Pitfalls

### Pitfall 1: ObjectId-to-String Mismatch in $lookup
**What goes wrong:** `$lookup` joins on `user._id` (ObjectId) against `subscription.userId` (string). Default `$lookup` with `localField`/`foreignField` requires matching types.
**Why it happens:** The subscription collection stores `userId` as a string representation of the ObjectId, not an ObjectId itself.
**How to avoid:** Use pipeline-style `$lookup` with `$expr` and `$toString($userId._id)` or `$toObjectId($subscription.userId)` to convert types during the join.
**Warning signs:** Empty subscription data in joined results despite subscriptions existing in the collection.

### Pitfall 2: Search Regex Injection
**What goes wrong:** User-supplied search strings containing regex special characters (`.*+?^${}()|[]\\`) cause MongoDB regex errors or unintended matches.
**Why it happens:** Passing raw user input to `$regex` without escaping.
**How to avoid:** Escape special regex characters before building the `$match` stage. Create a `escapeRegex()` utility.
**Warning signs:** 500 errors when users search for strings containing special characters.

### Pitfall 3: Pagination Total Count Accuracy
**What goes wrong:** Total count does not reflect post-`$lookup` filtering, leading to incorrect "Page X of Y" display.
**Why it happens:** If you count users before applying subscription status filter, the total is wrong.
**How to avoid:** Use `$facet` to compute both the paginated results and the total count from the same filtered pipeline, after all filter stages.
**Warning signs:** "Page 2 of 5" shows empty results because actual filtered count is much lower.

### Pitfall 4: Firebase Admin SDK Initialization
**What goes wrong:** Multiple calls to `admin.initializeApp()` throw "Firebase app already exists" error.
**Why it happens:** NestJS dependency injection creates service instances; if initialization is in constructor without checking existing apps, it fails.
**How to avoid:** Check `admin.apps.length` before initializing, or use `admin.app()` to get existing app. Better: initialize in a dedicated provider registered at module level.
**Warning signs:** Application crash on startup with "default Firebase app already exists" error.

### Pitfall 5: StandardResponse Missing Pagination Type
**What goes wrong:** Frontend `StandardResponse<T>` type does not include `pagination` field, so RTK Query `transformResponse` cannot access pagination metadata.
**Why it happens:** The frontend type was defined without the pagination field that the API `IResponse` interface includes.
**How to avoid:** Extend `StandardResponse` (or create a `PaginatedResponse<T>`) in `api.types.ts` to include `pagination?: { total: number; page: number; limit: number }`.
**Warning signs:** TypeScript errors when trying to read `response.pagination` in RTK Query endpoints.

### Pitfall 6: Firebase Admin Credentials in Production
**What goes wrong:** Firebase Admin SDK requires a service account key in production (not just project ID).
**Why it happens:** Local development can use Application Default Credentials or emulator, but production needs explicit credentials.
**How to avoid:** Use `GOOGLE_APPLICATION_CREDENTIALS` environment variable pointing to service account JSON, or use `credential: admin.credential.cert()` with environment variables for key fields.
**Warning signs:** "Could not load the default credentials" error in production.

## Code Examples

### Query Filter Parser

```typescript
// Source: Custom implementation following D-01 convention
// File: src/core/utilities/query-filter-parser.utility.ts

interface IQueryFilter {
  field: string;
  operator: FilterOperator;
  value: string | string[];
}

type FilterOperator = "in" | "eq" | "nq" | "lt" | "gt" | "le" | "ge" | "bt";

const VALID_OPERATORS: FilterOperator[] = ["in", "eq", "nq", "lt", "gt", "le", "ge", "bt"];
const FILTER_PREFIX = "filter:";

export function parseQueryFilters(query: Record<string, string>): IQueryFilter[] {
  const filters: IQueryFilter[] = [];

  for (const [key, value] of Object.entries(query)) {
    if (!key.startsWith(FILTER_PREFIX)) continue;

    const withoutPrefix = key.slice(FILTER_PREFIX.length);
    const lastColonIndex = withoutPrefix.lastIndexOf(":");
    if (lastColonIndex === -1) continue;

    const field = withoutPrefix.slice(0, lastColonIndex);
    const operator = withoutPrefix.slice(lastColonIndex + 1) as FilterOperator;

    if (!VALID_OPERATORS.includes(operator)) continue;
    if (!field || !value) continue;

    const parsedValue = operator === "in" || operator === "bt"
      ? value.split(",")
      : value;

    filters.push({ field, operator, value: parsedValue });
  }

  return filters;
}
```

### RTK Query Endpoint with Pagination

```typescript
// Source: Extends existing customerApi pattern [VERIFIED: trade-flow-ui/src/features/customers/api/customerApi.ts]
// File: src/features/support/api/supportApi.ts

interface SupportUsersParams {
  page?: number;
  limit?: number;
  search?: string;
  roleFilter?: string;
  subscriptionStatus?: string;
}

interface PaginatedResponse<T> {
  data: T[];
  pagination?: {
    total: number;
    page: number;
    limit: number;
  };
}

export const supportApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getSupportUsers: builder.query<
      { users: SupportUser[]; pagination: { total: number; page: number; limit: number } },
      SupportUsersParams
    >({
      query: ({ page = 1, limit = 20, search, roleFilter, subscriptionStatus }) => {
        const params = new URLSearchParams();
        params.set("page", String(page));
        params.set("limit", String(limit));
        if (search) params.set("search", search);
        if (roleFilter && roleFilter !== "all") {
          params.set("filter:role.type:eq", roleFilter);
        }
        if (subscriptionStatus && subscriptionStatus !== "all") {
          params.set("filter:subscription.status:eq", subscriptionStatus);
        }
        return `/v1/support/users?${params.toString()}`;
      },
      transformResponse: (response: PaginatedResponse<SupportUser>) => ({
        users: response.data,
        pagination: response.pagination ?? { total: 0, page: 1, limit: 20 },
      }),
      providesTags: ["SupportUser"],
    }),
  }),
});
```

### Dashboard Metrics Aggregation

```typescript
// Source: MongoDB aggregation [CITED: https://www.mongodb.com/docs/manual/reference/operator/aggregation/group/]
// File: src/support/repositories/support-user.repository.ts (metrics method)

public async getMetrics(): Promise<IDashboardMetricsDto> {
  const db = await this.connection.getDb();
  const result = await db.collection("users").aggregate([
    {
      $lookup: {
        from: "subscriptions",
        let: { odUserId: { $toString: "$_id" } },
        pipeline: [
          { $match: { $expr: { $eq: ["$userId", "$$odUserId"] } } },
          { $project: { status: 1 } },
        ],
        as: "subscription",
      },
    },
    { $unwind: { path: "$subscription", preserveNullAndEmptyArrays: true } },
    {
      $group: {
        _id: null,
        totalUsers: { $sum: 1 },
        activeTrials: {
          $sum: { $cond: [{ $eq: ["$subscription.status", "trialing"] }, 1, 0] },
        },
        activeSubscriptions: {
          $sum: { $cond: [{ $eq: ["$subscription.status", "active"] }, 1, 0] },
        },
        expiredSubscriptions: {
          $sum: { $cond: [{ $eq: ["$subscription.status", "incomplete"] }, 1, 0] },
        },
        canceledSubscriptions: {
          $sum: { $cond: [{ $eq: ["$subscription.status", "canceled"] }, 1, 0] },
        },
      },
    },
  ]).toArray();

  // ... map to DTO
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Client-side list filtering | Server-side filter/sort/paginate via query params | This phase (D-01) | All future list endpoints follow this convention |
| `MongoDbFetcher.findMany()` only | `MongoDbFetcher.aggregate()` added | This phase | Enables `$lookup` joins, `$facet` pagination |
| No Firebase Admin SDK | `firebase-admin` installed | This phase (D-13) | Enables server-side user metadata access for support tools |

**Deprecated/outdated:**
- Mock data in `SupportUsersPage.tsx`: Replaced with real RTK Query data
- `isSuperUser()` page-level checks in support pages: Removed in Phase 53 (SupportGuard handles access)

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `firebase-admin` can be initialized with just `FIREBASE_PROJECT_ID` env var in combination with Application Default Credentials | Architecture Patterns / Pattern 4 | Medium -- may need service account JSON; impacts environment setup |
| A2 | `subscription.userId` stores the string representation of `user._id` ObjectId | Pitfalls / Pitfall 1 | High -- if it stores Firebase UID instead, the `$lookup` join logic changes entirely |
| A3 | The `SubscriptionStatus` enum lacks an "expired" value; "incomplete" maps to expired display status | Code Examples / Metrics | Medium -- UI shows "Expired" but enum has `INCOMPLETE`; may need mapping logic or a new enum value |
| A4 | Firebase Admin SDK `getUser()` returns `metadata.creationTime` and `metadata.lastSignInTime` as RFC 2822 strings | Code Examples | Low -- well-documented API, but exact format affects Luxon parsing |

## Open Questions (RESOLVED)

All three open questions have been operationally resolved through planning decisions. Implementors should verify assumptions A2 during Plan 02 execution.

1. **What does "expired" mean in subscription status context?** (RESOLVED)
   - What we know: `SubscriptionStatus` enum has `TRIALING`, `ACTIVE`, `PAST_DUE`, `CANCELED`, `INCOMPLETE`. No `EXPIRED` value.
   - Resolution: Plan 03 treats "expired" as mapping to `INCOMPLETE` and `INCOMPLETE_EXPIRED` statuses in the metrics aggregation pipeline. The UI displays "Expired" as the user-facing label for these statuses. See Plan 03 Task 1 (dashboard metrics service) for implementation.

2. **Does `subscription.userId` match `user._id.toString()` or `user.externalAuthUserId`?** (RESOLVED)
   - What we know: `ISubscriptionEntity.userId` is typed as `string`. The subscription is created during Stripe checkout flow.
   - Resolution: Plan 02 uses pipeline-style `$lookup` with `$expr` and `{ $toString: "$_id" }` to handle the ObjectId-to-string conversion. The implementor should verify by inspecting the database during execution. If `userId` stores the Firebase UID instead, the `$lookup` `let` variable should use `$externalAuthUserId` instead of `$toString("$_id")`.

3. **Firebase Admin SDK credentials strategy for production (Railway)** (RESOLVED)
   - What we know: Railway uses environment variables. Firebase Admin needs credentials.
   - Resolution: Plan 03 Task 1 implements `FirebaseAuthMetadataService` that supports both service account credentials (`FIREBASE_CLIENT_EMAIL` + `FIREBASE_PRIVATE_KEY` env vars) and Application Default Credentials (just `FIREBASE_PROJECT_ID`). Plan 05 includes a `user_setup` section for generating the Firebase service account key.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| MongoDB | User list aggregation, metrics | Yes | 7.0+ | -- |
| firebase-admin | D-13 auth metadata | No (not installed) | 13.8.0 (npm) | Install via `npm install firebase-admin` |
| Node.js | Backend runtime | Yes | 22.x | -- |
| Firebase project | Auth metadata | Yes (configured) | -- | -- |

**Missing dependencies with no fallback:**
- `firebase-admin` must be installed in `trade-flow-api` for D-13

**Missing dependencies with fallback:**
- None

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 |
| Framework (UI) | Vitest 4.1.3 |
| Config file (API) | `trade-flow-api/package.json` (jest config) |
| Config file (UI) | `trade-flow-ui/vitest.config.ts` |
| Quick run command (API) | `npm run test -- --testPathPattern=support` |
| Quick run command (UI) | `npm run test -- --testPathPattern=support` |
| Full suite command (API) | `npm run ci` |
| Full suite command (UI) | `npm run ci` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| UMGT-01 | Query filter parser extracts filters from query params | unit | `npm run test -- --testPathPattern=query-filter-parser` | No -- Wave 0 |
| UMGT-01 | Support user repository returns paginated users with subscription | unit | `npm run test -- --testPathPattern=support-user.repository` | No -- Wave 0 |
| UMGT-01 | Support user controller returns filtered, paginated response | unit | `npm run test -- --testPathPattern=support-user.controller` | No -- Wave 0 |
| UMGT-03 | Firebase auth metadata service returns creation/sign-in dates | unit | `npm run test -- --testPathPattern=firebase-auth-metadata` | No -- Wave 0 |
| UMGT-04 | Dashboard metrics returns correct counts | unit | `npm run test -- --testPathPattern=dashboard-metrics` | No -- Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=support` (API) or `npm run test -- --testPathPattern=support` (UI)
- **Per wave merge:** `npm run ci` in both repos
- **Phase gate:** `npm run ci` green in both repos before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `src/core/test/utilities/query-filter-parser.utility.spec.ts` -- covers filter parsing
- [ ] `src/support/test/services/support-user-retriever.service.spec.ts` -- covers UMGT-01, UMGT-03
- [ ] `src/support/test/services/support-dashboard-metrics.service.spec.ts` -- covers UMGT-04
- [ ] `src/support/test/mocks/` -- shared mock generators for support user, subscription

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Firebase Auth (existing) |
| V3 Session Management | No | JWT (existing) |
| V4 Access Control | Yes | `@RequiresPermission('manage_users')` on support endpoints (Phase 52 infrastructure) |
| V5 Input Validation | Yes | class-validator on request DTOs; regex escaping for search input |
| V6 Cryptography | No | Not applicable |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Regex injection via search parameter | Tampering | Escape regex special characters before `$regex` |
| Unauthorized access to user list | Elevation of Privilege | `@RequiresPermission('manage_users')` guard + `JwtAuthGuard` |
| Information disclosure via user detail | Information Disclosure | Support role required; Firebase metadata only fetched server-side |
| Query parameter manipulation for data exfiltration | Tampering | Validate filter operators against allowlist; validate field names against allowlist |

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/core/services/mongo/mongo-db-fetcher.service.ts` -- Confirmed no aggregation method exists
- `trade-flow-api/src/core/data-transfer-objects/base-query-options.dto.ts` -- Current structure verified
- `trade-flow-api/src/subscription/entities/subscription.entity.ts` -- Entity structure with `userId: string`
- `trade-flow-api/src/user/entities/user.entity.ts` -- Entity structure with `_id: ObjectId`
- `trade-flow-api/src/subscription/enums/subscription-status.enum.ts` -- No EXPIRED value
- `trade-flow-ui/src/types/api.types.ts` -- No pagination in StandardResponse
- `trade-flow-ui/src/pages/support/SupportUsersPage.tsx` -- Mock skeleton structure
- `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx` -- Dashboard structure

### Secondary (MEDIUM confidence)
- npm registry: `firebase-admin@13.8.0` current version verified
- MongoDB aggregation documentation for `$lookup`, `$facet`, `$group` pipeline stages

### Tertiary (LOW confidence)
- Firebase Admin SDK initialization patterns (A1, A4 in assumptions)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all core dependencies verified in package.json; firebase-admin version confirmed via npm
- Architecture: HIGH -- existing patterns thoroughly examined; aggregation gap confirmed
- Pitfalls: HIGH -- type mismatch between user._id and subscription.userId verified in source code
- Firebase Admin integration: MEDIUM -- SDK is well-documented but initialization pattern for this specific project is assumed

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable dependencies, no fast-moving concerns)
