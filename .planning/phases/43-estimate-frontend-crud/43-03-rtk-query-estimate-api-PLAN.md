---
phase: 43-estimate-frontend-crud
plan: 03
type: execute
wave: 2
depends_on:
  - 43-01
files_modified:
  - trade-flow-ui/src/services/api.ts
  - trade-flow-ui/src/features/estimates/api/estimateApi.ts
  - trade-flow-ui/src/features/estimates/index.ts
autonomous: true
requirements:
  - CONT-03
  - CONT-04

must_haves:
  truths:
    - "features/estimates/api/estimateApi.ts exports seven hooks corresponding to the Phase 41 CRUD surface"
    - "\"Estimate\" tag is registered in src/services/api.ts alongside \"Quote\""
    - "features/estimates/index.ts barrel re-exports the API module"
    - "Estimate mutations invalidate Job LIST so job rows reflect has-estimate state"
  artifacts:
    - path: "trade-flow-ui/src/features/estimates/api/estimateApi.ts"
      provides: "RTK Query slice with 7 endpoints targeting Phase 41 /v1/estimates routes"
      contains: "useGetEstimatesQuery"
    - path: "trade-flow-ui/src/services/api.ts"
      provides: "Base RTK Query slice with Estimate tag"
      contains: "\"Estimate\""
    - path: "trade-flow-ui/src/features/estimates/index.ts"
      provides: "Feature barrel export"
      contains: "./api/estimateApi"
  key_links:
    - from: "trade-flow-ui/src/features/estimates/api/estimateApi.ts"
      to: "trade-flow-ui/src/services/api.ts"
      via: "apiSlice.injectEndpoints"
      pattern: "apiSlice.injectEndpoints"
    - from: "trade-flow-ui/src/features/estimates/api/estimateApi.ts"
      to: "trade-flow-ui/src/types/estimate.ts"
      via: "type imports"
      pattern: "from \"@/types\""
---

<objective>
Build the RTK Query slice that Plans 04-06 consume to talk to Phase 41's CRUD surface, register the `"Estimate"` tag in the base slice, and seed the feature barrel so pages can import from `@/features/estimates`.

Purpose: Interface-first ordering — components cannot call hooks that do not yet exist, so this plan defines the API contracts before Plan 04's forms and Plan 05/06's pages reference them.

Output: `features/estimates/api/estimateApi.ts` exporting seven hooks, `src/services/api.ts` with `"Estimate"` tag, `features/estimates/index.ts` barrel.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@.planning/phases/41-estimate-module-crud-backend/41-08-controller-module-wiring-and-docs-PLAN.md
@trade-flow-ui/src/services/api.ts
@trade-flow-ui/src/features/quotes/api/quoteApi.ts
@trade-flow-ui/src/features/quotes/index.ts
@trade-flow-ui/src/types/estimate.ts

<interfaces>
<!-- Reference: quoteApi shape that estimateApi mirrors (with estimate endpoints) -->

From trade-flow-ui/src/features/quotes/api/quoteApi.ts:
```typescript
export const quoteApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getQuotes: builder.query<Quote[], string>({
      query: (businessId) => `/v1/business/${businessId}/quotes`,
      transformResponse: (response: StandardResponse<Quote>) => response.data,
      providesTags: (result) =>
        result
          ? [...result.map(({ id }) => ({ type: "Quote" as const, id })), { type: "Quote", id: "LIST" }]
          : [{ type: "Quote", id: "LIST" }],
    }),
    // getQuote, createQuote, addLineItem, updateLineItem, deleteLineItem, deleteQuote, etc.
  }),
});
export const { useGetQuotesQuery, useGetQuoteQuery, useCreateQuoteMutation, ... } = quoteApi;
```

From trade-flow-ui/src/services/api.ts:
```typescript
export const apiSlice = createApi({
  reducerPath: "api",
  baseQuery,
  tagTypes: ["User", "Business", "Customer", "Item", "Quote", "Migration", "TaxRate", "JobType", "Job", "VisitType", "Schedule", "Subscription"],
  endpoints: () => ({}),
});
```

Phase 41 locked endpoint surface (from 41-08 PLAN):
- `POST /v1/estimates` — create, body is `CreateEstimateRequest`, returns `IResponse<IEstimateResponse>`
- `GET /v1/estimates?businessId=...` — list (Phase 41 defines pagination + optional status filter; Phase 43 passes businessId as query param; confirm path during implementation against 41-08-controller-module-wiring-and-docs-PLAN.md)
- `GET /v1/estimates/:estimateId` — detail
- `PATCH /v1/estimates/:estimateId` — update (Draft only)
- `DELETE /v1/estimates/:estimateId` — soft delete
- `POST /v1/estimates/:estimateId/line-item` — add line item
- `PATCH /v1/estimates/:estimateId/line-item/:lineItemId` — update line item
- `DELETE /v1/estimates/:estimateId/line-item/:lineItemId` — delete line item

NOTE: Phase 41's route shape differs from quotes — quotes use `/v1/business/:businessId/quote/*` while estimates use `/v1/estimates/*` (no business prefix). The executor MUST read `.planning/phases/41-estimate-module-crud-backend/41-08-controller-module-wiring-and-docs-PLAN.md` to confirm the exact path (including how `businessId` is scoped — via query param, via JWT, or via path segment). Adjust query strings accordingly. Do NOT assume quote-style paths.
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Register "Estimate" tag in src/services/api.ts</name>
  <files>trade-flow-ui/src/services/api.ts</files>
  <read_first>
    - trade-flow-ui/src/services/api.ts (full file — only the tagTypes array needs changing)
  </read_first>
  <action>
EDIT `trade-flow-ui/src/services/api.ts` — add the string `"Estimate"` to the `tagTypes` array. Insert it immediately after `"Quote"` to keep related tags grouped:

```typescript
  tagTypes: [
    "User",
    "Business",
    "Customer",
    "Item",
    "Quote",
    "Estimate",
    "Migration",
    "TaxRate",
    "JobType",
    "Job",
    "VisitType",
    "Schedule",
    "Subscription",
  ],
```

Do NOT touch `baseQuery`, `reducerPath`, or `endpoints`. Only the tagTypes array changes.
  </action>
  <verify>
    <automated>grep -c "\"Estimate\"" trade-flow-ui/src/services/api.ts</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "\"Estimate\"" trade-flow-ui/src/services/api.ts` returns at least 1
    - `grep -c "\"Quote\"" trade-flow-ui/src/services/api.ts` returns at least 1 (pre-existing, must not be deleted)
    - `grep -c "tagTypes:" trade-flow-ui/src/services/api.ts` returns 1 (only one tagTypes declaration)
  </acceptance_criteria>
  <done>"Estimate" tag registered alongside existing tags; typecheck still passes.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Build features/estimates/api/estimateApi.ts with seven endpoints</name>
  <files>trade-flow-ui/src/features/estimates/api/estimateApi.ts, trade-flow-ui/src/features/estimates/index.ts</files>
  <read_first>
    - trade-flow-ui/src/features/quotes/api/quoteApi.ts (full file — exact structural template)
    - trade-flow-ui/src/features/quotes/index.ts (barrel pattern)
    - trade-flow-ui/src/types/estimate.ts (created in plan 01)
    - .planning/phases/41-estimate-module-crud-backend/41-08-controller-module-wiring-and-docs-PLAN.md (Phase 41 endpoint paths + request/response shapes — this is load-bearing, do not guess)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-MIR-04, D-MIR-05
  </read_first>
  <behavior>
    - 7 hooks exported: useGetEstimatesQuery, useGetEstimateQuery, useCreateEstimateMutation, useUpdateEstimateMutation, useAddEstimateLineItemMutation, useUpdateEstimateLineItemMutation, useDeleteEstimateLineItemMutation, useDeleteEstimateMutation (that's 8 counting delete estimate — update phase requirements block to match)
    - Tag invalidation mirrors quoteApi exactly: per-id + LIST sentinel; cross-feature invalidation on Job LIST for create/delete to match quote pattern
    - deleteEstimate uses optimistic update via onQueryStarted + updateQueryData (mirrors quote deleteQuote)
    - transformResponse unwraps StandardResponse<Estimate> exactly as quoteApi does
    - Path prefix is `/v1/estimates` (NOT `/v1/business/:businessId/estimate`) — per Phase 41 D-TXN-05 locked shape
  </behavior>
  <action>
Create NEW directory `trade-flow-ui/src/features/estimates/api/` and NEW file `estimateApi.ts`. Before writing, open Phase 41's `41-08-controller-module-wiring-and-docs-PLAN.md` and confirm:
1. The exact path prefix for each route (most likely `/v1/estimates` with no business prefix — Phase 41 D-TXN-05 scopes by JWT)
2. How `businessId` reaches the API for `getEstimates` (query param `?businessId=...` OR implicit via JWT). If unclear, default to using `useCurrentBusiness().businessId` at the call site and passing it as a query param: `` `/v1/estimates?businessId=${businessId}` ``
3. The exact request body for create (`CreateEstimateRequest` fields per Phase 41 DTOs) — verify against Phase 41 plan 04 `estimate-scaffold` plan's DTO definitions
4. Whether line-item endpoints return the parent estimate (they should, per Phase 41 plan 08 SC line "add line item, returns parent estimate")

File content:

```typescript
import { apiSlice } from "@/services/api";

import type {
  Estimate,
  CreateEstimateRequest,
  UpdateEstimateRequest,
  UncertaintyReason,
  EstimateDisplayMode,
  StandardResponse,
} from "@/types";

function unwrapSingle(response: StandardResponse<Estimate>): Estimate {
  if (response.data && response.data.length > 0) {
    return response.data[0];
  }
  throw new Error("No estimate data returned");
}

export const estimateApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getEstimates: builder.query<Estimate[], string>({
      query: (businessId) => `/v1/estimates?businessId=${encodeURIComponent(businessId)}`,
      transformResponse: (response: StandardResponse<Estimate>) => response.data,
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: "Estimate" as const, id })),
              { type: "Estimate" as const, id: "LIST" },
            ]
          : [{ type: "Estimate" as const, id: "LIST" }],
    }),

    getEstimate: builder.query<Estimate, string>({
      query: (estimateId) => `/v1/estimates/${estimateId}`,
      transformResponse: unwrapSingle,
      providesTags: (_result, _error, id) => [{ type: "Estimate", id }],
    }),

    createEstimate: builder.mutation<Estimate, { businessId: string; data: CreateEstimateRequest }>({
      query: ({ data }) => ({
        url: `/v1/estimates`,
        method: "POST",
        body: data,
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: [
        { type: "Estimate", id: "LIST" },
        { type: "Job", id: "LIST" },
      ],
    }),

    updateEstimate: builder.mutation<
      Estimate,
      { businessId: string; estimateId: string; data: UpdateEstimateRequest }
    >({
      query: ({ estimateId, data }) => ({
        url: `/v1/estimates/${estimateId}`,
        method: "PATCH",
        body: data,
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: (_result, _error, { estimateId }) => [
        { type: "Estimate", id: estimateId },
        { type: "Estimate", id: "LIST" },
      ],
    }),

    addEstimateLineItem: builder.mutation<
      Estimate,
      {
        businessId: string;
        estimateId: string;
        itemId: string;
        quantity: number;
        priceStrategy?: string;
      }
    >({
      query: ({ estimateId, itemId, quantity, priceStrategy }) => ({
        url: `/v1/estimates/${estimateId}/line-item`,
        method: "POST",
        body: { itemId, quantity, ...(priceStrategy && { priceStrategy }) },
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: (_result, _error, { estimateId }) => [
        { type: "Estimate", id: estimateId },
        { type: "Estimate", id: "LIST" },
      ],
    }),

    updateEstimateLineItem: builder.mutation<
      Estimate,
      {
        businessId: string;
        estimateId: string;
        lineItemId: string;
        quantity?: number;
        unitPrice?: number;
      }
    >({
      query: ({ estimateId, lineItemId, quantity, unitPrice }) => ({
        url: `/v1/estimates/${estimateId}/line-item/${lineItemId}`,
        method: "PATCH",
        body: {
          ...(quantity !== undefined && { quantity }),
          ...(unitPrice !== undefined && { unitPrice }),
        },
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: (_result, _error, { estimateId }) => [
        { type: "Estimate", id: estimateId },
        { type: "Estimate", id: "LIST" },
      ],
    }),

    deleteEstimateLineItem: builder.mutation<
      Estimate,
      { businessId: string; estimateId: string; lineItemId: string }
    >({
      query: ({ estimateId, lineItemId }) => ({
        url: `/v1/estimates/${estimateId}/line-item/${lineItemId}`,
        method: "DELETE",
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: (_result, _error, { estimateId }) => [
        { type: "Estimate", id: estimateId },
        { type: "Estimate", id: "LIST" },
      ],
    }),

    deleteEstimate: builder.mutation<Estimate, { businessId: string; estimateId: string }>({
      query: ({ estimateId }) => ({
        url: `/v1/estimates/${estimateId}`,
        method: "DELETE",
      }),
      transformResponse: unwrapSingle,
      invalidatesTags: (_result, _error, { estimateId }) => [
        { type: "Estimate", id: estimateId },
        { type: "Estimate", id: "LIST" },
        { type: "Job", id: "LIST" },
      ],
      async onQueryStarted({ businessId, estimateId }, { dispatch, queryFulfilled }) {
        const patchResult = dispatch(
          estimateApi.util.updateQueryData("getEstimates", businessId, (draft) => {
            return draft.filter((e) => e.id !== estimateId);
          }),
        );
        try {
          await queryFulfilled;
        } catch {
          patchResult.undo();
        }
      },
    }),
  }),
});

export const {
  useGetEstimatesQuery,
  useGetEstimateQuery,
  useCreateEstimateMutation,
  useUpdateEstimateMutation,
  useAddEstimateLineItemMutation,
  useUpdateEstimateLineItemMutation,
  useDeleteEstimateLineItemMutation,
  useDeleteEstimateMutation,
} = estimateApi;

export type { UncertaintyReason, EstimateDisplayMode };
```

IMPORTANT: if Phase 41's 41-08 plan defines the GET list endpoint as `/v1/estimates` (business-scoped via JWT) rather than `/v1/estimates?businessId=...`, change `getEstimates` to `query: () => "/v1/estimates"` and either (a) change the hook signature from `Estimate[], string` to `Estimate[], void` or (b) keep the arg as a cache key but ignore it in the query function. Match whatever Phase 41 ships. Do NOT invent a path that does not exist in Phase 41.

Then create NEW file `trade-flow-ui/src/features/estimates/index.ts` as a barrel:

```typescript
export * from "./api/estimateApi";
```

(Plan 05 will append `export * from "./components"` once the components directory exists.)

Run `cd trade-flow-ui && npm run ci` — must exit 0.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/features/estimates src/services/api.ts && npm run format:check</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/features/estimates/api/estimateApi.ts` exists
    - `grep -c "export const useGetEstimatesQuery" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 1 (as part of the destructured export block)
    - `grep -c "useGetEstimatesQuery\|useGetEstimateQuery\|useCreateEstimateMutation\|useUpdateEstimateMutation\|useAddEstimateLineItemMutation\|useUpdateEstimateLineItemMutation\|useDeleteEstimateLineItemMutation\|useDeleteEstimateMutation" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 8
    - `grep -c "apiSlice.injectEndpoints" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 1
    - `grep -c "type: \"Estimate\"" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 4 (multiple invalidation sites)
    - `grep -c "/v1/estimates" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 5
    - `grep -c "onQueryStarted" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns at least 1 (optimistic delete)
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable\|: any" trade-flow-ui/src/features/estimates/api/estimateApi.ts` returns 0
    - `grep -c "from \"./api/estimateApi\"" trade-flow-ui/src/features/estimates/index.ts` returns at least 1
    - `cd trade-flow-ui && npm run typecheck` exits 0
    - `cd trade-flow-ui && npm run lint` exits 0
  </acceptance_criteria>
  <done>estimateApi slice exists with 8 hooks, uses correct Phase 41 paths, invalidates correct tags, optimistic delete wired, barrel re-exports the API, typecheck and lint green.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| client→API | All 8 mutations and 2 queries cross the HTTP boundary to `/v1/estimates/*` — Firebase JWT is attached by the existing `prepareHeaders` in `src/services/api.ts` |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-03-01 | Spoofing | Request reaches API without JWT header | mitigate | estimateApi injects into existing apiSlice whose baseQuery already attaches Firebase token — no new auth path introduced |
| T-43-03-02 | Tampering | Client manipulates URL to PATCH an estimate it does not own (IDOR) | mitigate | Phase 41 EstimatePolicy + DocumentSessionAuthGuard enforce ownership server-side; frontend is not the trust boundary |
| T-43-03-03 | Information disclosure | Optimistic delete leaks other businesses' rows into local cache if backend sends wrong scope | accept | Cache is keyed by businessId; transformResponse trusts API; Phase 41 test suite covers list scoping |
| T-43-03-04 | Tampering | Mass-assignment via UpdateEstimateRequest body bypasses Phase 41 Draft guard | mitigate | Phase 41 plan 07 (EstimateUpdater) whitelists fields server-side AND re-checks Draft; frontend cannot bypass |
| T-43-03-05 | Denial of service | Infinite invalidation loop on Job LIST + Estimate LIST cross-invalidation | mitigate | Invalidation tags are directed acyclic — Job LIST is only invalidated by create/delete, not by Job queries invalidating back |
</threat_model>

<verification>
`cd trade-flow-ui && npm run ci` must exit 0.

Interface integration check: after this plan, a downstream plan can write `import { useGetEstimatesQuery } from "@/features/estimates"` and typecheck without errors.
</verification>

<success_criteria>
- `"Estimate"` tag in src/services/api.ts tagTypes array
- features/estimates/api/estimateApi.ts exports 8 hooks matching Phase 41 routes
- features/estimates/index.ts barrel re-exports the api module
- `npm run ci` passes in trade-flow-ui
- No type assertions / suppressions
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-03-SUMMARY.md`
</output>
