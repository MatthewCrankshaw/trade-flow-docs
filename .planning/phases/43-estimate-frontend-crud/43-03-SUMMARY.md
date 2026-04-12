---
phase: 43-estimate-frontend-crud
plan: "03"
subsystem: trade-flow-ui/features/estimates
tags: [rtk-query, estimates, frontend, api-slice]
dependency_graph:
  requires:
    - 43-01 (Estimate TypeScript types in src/types/estimate.ts)
  provides:
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts (RTK Query slice with 8 hooks)
    - trade-flow-ui/src/features/estimates/index.ts (feature barrel)
    - trade-flow-ui/src/services/api.ts (Estimate tag registered)
  affects:
    - Plan 43-04 (CreateEstimateDialog uses useCreateEstimateMutation)
    - Plan 43-05 (EstimatesPage uses useGetEstimatesQuery, useDeleteEstimateMutation)
    - Plan 43-06 (EstimateDetailPage uses useGetEstimateQuery, useUpdateEstimateMutation, line-item hooks)
tech_stack:
  added: []
  patterns:
    - RTK Query injectEndpoints pattern (mirrors quoteApi.ts)
    - businessId as cache key only; JWT-scoped API (no query param)
    - Optimistic delete via onQueryStarted + updateQueryData
    - Cross-feature tag invalidation (Estimate mutations invalidate Job LIST)
key_files:
  created:
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts
    - trade-flow-ui/src/features/estimates/index.ts
  modified:
    - trade-flow-ui/src/services/api.ts
decisions:
  - "GET /v1/estimates is JWT-scoped (authUser.businessIds[0] on backend); businessId hook arg is RTK Query cache key only, not a URL query param"
  - "Estimate tag registered after Quote in tagTypes to keep related tags grouped"
  - "deleteEstimate uses optimistic update (mirrors quoteApi deleteQuote pattern exactly)"
  - "createEstimate and deleteEstimate both invalidate Job LIST so job rows reflect has-estimate state"
metrics:
  duration: "5 minutes"
  completed: "2026-04-12"
  tasks_completed: 2
  files_changed: 3
---

# Phase 43 Plan 03: RTK Query Estimate API Summary

**One-liner:** RTK Query estimateApi slice with 8 hooks targeting Phase 41 /v1/estimates routes, Estimate tag in apiSlice tagTypes, and estimates feature barrel — enabling Plans 04-06 to import hooks directly from @/features/estimates.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Register "Estimate" tag in src/services/api.ts tagTypes | 6fbefad (ui) | src/services/api.ts |
| 2 | Build estimateApi.ts with 8 endpoints + feature barrel | 565ad94 (ui) | src/features/estimates/api/estimateApi.ts, src/features/estimates/index.ts |

## What Was Built

**Task 1 — Estimate tag registration** (`trade-flow-ui/src/services/api.ts`):

Added `"Estimate"` to the `tagTypes` array immediately after `"Quote"` to keep related tags grouped. No other changes to the base slice.

**Task 2 — RTK Query estimateApi slice** (`trade-flow-ui/src/features/estimates/api/estimateApi.ts`):

Eight hooks exported:
- `useGetEstimatesQuery(businessId)` — GET /v1/estimates (JWT-scoped; businessId is cache key only)
- `useGetEstimateQuery(estimateId)` — GET /v1/estimates/:estimateId
- `useCreateEstimateMutation()` — POST /v1/estimates; invalidates Estimate LIST + Job LIST
- `useUpdateEstimateMutation()` — PATCH /v1/estimates/:estimateId; invalidates per-id + LIST
- `useAddEstimateLineItemMutation()` — POST /v1/estimates/:estimateId/line-item
- `useUpdateEstimateLineItemMutation()` — PATCH /v1/estimates/:estimateId/line-item/:lineItemId
- `useDeleteEstimateLineItemMutation()` — DELETE /v1/estimates/:estimateId/line-item/:lineItemId
- `useDeleteEstimateMutation()` — DELETE /v1/estimates/:estimateId; optimistic update + invalidates Estimate + Job LIST

**Feature barrel** (`trade-flow-ui/src/features/estimates/index.ts`):

Single re-export `export * from "./api/estimateApi"`. Plan 05 will append `export * from "./components"` when the components directory exists.

## Deviations from Plan

### Auto-adjusted: businessId query param

The plan's task description included a template using `?businessId=${encodeURIComponent(businessId)}` as the default. After reading the actual Phase 41 controller (`estimate.controller.ts`), confirmed that `GET /v1/estimates` is JWT-scoped — the backend retriever uses `authUser.businessIds[0]`, and `ListEstimatesRequest` has no `businessId` field. The query function was therefore written as `() => "/v1/estimates"` with businessId serving as the RTK Query cache key only. This matches the plan's own guidance: "If Phase 41 defines it as JWT-scoped, change getEstimates to query: () => '/v1/estimates'".

### Auto-fixed: ESLint unused variable

Initial implementation used `(_businessId) =>` in the query arrow function. ESLint's `@typescript-eslint/no-unused-vars` rule (from `tseslint.configs.recommended`) flagged it since the UI's `eslint.config.js` does not configure `argsIgnorePattern`. Fixed by dropping the parameter name entirely: `() => "/v1/estimates"`. No behavior change.

### Auto-fixed: Prettier formatting

After the ESLint fix, Prettier reformatted the `providesTags` array from multi-line to single-line (within 125 char width). Applied via `npx prettier --write`.

## Known Stubs

None. This plan is API slice only — no runtime rendering, no data wiring to UI components.

## Threat Flags

None. All 8 endpoints inject into the existing `apiSlice` whose `baseQuery` already attaches the Firebase JWT via `prepareHeaders`. No new auth path introduced. IDOR protection is server-side (Phase 41 EstimatePolicy). Tag invalidation is directed acyclic — no loop risk between Estimate and Job invalidation.

## Self-Check: PASSED

| Item | Result |
|------|--------|
| trade-flow-ui/src/features/estimates/api/estimateApi.ts | FOUND |
| trade-flow-ui/src/features/estimates/index.ts | FOUND |
| trade-flow-ui/src/services/api.ts contains "Estimate" | FOUND |
| Commit 6fbefad (ui: Estimate tag) | FOUND |
| Commit 565ad94 (ui: estimateApi + barrel) | FOUND |
| npm run ci exits 0 | PASSED (82 tests, 0 lint errors, 0 type errors) |
| 8 hooks exported | PASSED |
| No type assertions / suppressions | PASSED |
