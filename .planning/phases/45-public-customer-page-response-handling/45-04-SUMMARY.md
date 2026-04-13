---
phase: 45-public-customer-page-response-handling
plan: 04
subsystem: trade-flow-ui
tags: [public-page, estimates, rtk-query, react, response-flow, pecr]
dependency_graph:
  requires: []
  provides: [public-estimate-page, estimate-response-flow]
  affects: [trade-flow-ui/src/App.tsx, trade-flow-ui/src/store/index.ts]
tech_stack:
  added: [toggle-group (Radix UI), publicEstimateApi (RTK Query)]
  patterns: [no-auth RTK Query slice, inline-expansion response flow, ToggleGroup for option sets]
key_files:
  created:
    - trade-flow-ui/src/features/public-estimate/types/public-estimate.types.ts
    - trade-flow-ui/src/features/public-estimate/api/publicEstimateApi.ts
    - trade-flow-ui/src/features/public-estimate/index.ts
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateDisclaimer.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimatePriceRange.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateUncertainty.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateTerminalState.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateSkeleton.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateError.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateResponseButtons.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateMessageForm.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateDeclineForm.tsx
    - trade-flow-ui/src/features/public-estimate/components/PublicEstimateCard.tsx
    - trade-flow-ui/src/pages/PublicEstimatePage.tsx
    - trade-flow-ui/src/components/ui/toggle-group.tsx
  modified:
    - trade-flow-ui/src/store/index.ts
    - trade-flow-ui/src/index.css
    - trade-flow-ui/src/App.tsx
decisions:
  - publicEstimateApi uses separate fetchBaseQuery with no prepareHeaders — Firebase JWT never sent to public endpoints (T-45-09)
  - ToggleGroup created as new shadcn component wrapping @radix-ui/react-toggle-group (already installed via radix-ui meta-package)
  - formatRange called with inclTax mapped to MoneyBreakdown.total — no client-side arithmetic on contingency
  - warning CSS tokens replaced existing --warning values to match the softer disclaimer palette from the UI-SPEC
metrics:
  duration: ~20 minutes
  completed: 2026-04-13T07:11:10Z
  tasks: 3
  files: 18
requirements:
  - CUST-03
  - CUST-04
  - CUST-07
  - RESP-01
  - RESP-02
  - RESP-03
  - RESP-04
---

# Phase 45 Plan 04: Public Customer Estimate Page Summary

Complete customer-facing estimate page with RTK Query API slice (no auth), all UI components, 3-action response flow with inline expansion, and route registration at `/estimate/:token`.

## What Was Built

### Task 1: Types, RTK Query API, Store Wiring, Warning Tokens

- `public-estimate.types.ts`: `PublicEstimate`, `RespondToEstimatePayload`, `ResponseView`, `DECLINE_REASONS` (5 structured reasons)
- `publicEstimateApi.ts`: Separate `createApi` instance with plain `fetchBaseQuery` — no `prepareHeaders`, no Firebase JWT leakage to public endpoints (PECR/CUST-07)
- `store/index.ts`: `publicEstimateApi` reducer and middleware wired in alongside `publicQuoteApi`
- `index.css`: Warning semantic tokens (`--warning`, `--warning-border`, `--warning-foreground`, `--warning-icon`) and `@theme inline` mappings (`--color-warning-border`, `--color-warning-icon`) added

### Task 2: Display Components

- `PublicEstimateDisclaimer`: Warning card with verbatim "This is an estimate, not a fixed price commitment" copy, `bg-warning border-warning-border` styling
- `PublicEstimatePriceRange`: Calls `formatRange` from `@/lib/currency` — no client-side arithmetic, renders API-returned low/high values only
- `PublicEstimateUncertainty`: Conditional "Why is this a range?" section with dot-separated inline reason list
- `PublicEstimateTerminalState`: `customer_responded` shows response-type-specific message; `trader_closed` shows full-page unavailable state with FileX icon and contact details
- `PublicEstimateSkeleton`: Loading skeleton matching all card sections
- `PublicEstimateError`: Four error types (not-found, expired, revoked, network) with AlertCircle icon; network type has Reload button

### Task 3: Interaction Components, Page, Route

- `toggle-group.tsx`: New shadcn component wrapping `@radix-ui/react-toggle-group` (already installed via `radix-ui` meta-package) — created as Rule 3 auto-fix since the file was missing
- `PublicEstimateMessageForm`: Textarea with 2,000-char counter, `text-base md:text-sm` for iOS zoom prevention, disabled during loading
- `PublicEstimateDeclineForm`: ToggleGroup (`type="single"`) for 5 structured decline reasons plus optional 500-char freeform note; submit enabled when reason or note present
- `PublicEstimateResponseButtons`: 3-action CTA with `useState<ResponseView>` state machine — "Happy to Proceed" (primary), "Message [Name]" (outline + icon), "Not right for me" (text link); inline expansion to form views; success confirmation with CheckCircle
- `PublicEstimateCard`: Composes all sections in spec order — business header, disclaimer, separator, estimate details, scope, price+uncertainty, notes, separator, response area
- `PublicEstimatePage`: Route-level page extracting `token` from `useParams`, conditional rendering for loading/error/trader-closed/active states
- `App.tsx`: `/estimate/:token` route registered outside authenticated layout alongside `/quote/view/:token`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Created missing toggle-group.tsx component**
- **Found during:** Task 3 implementation
- **Issue:** Plan specified `ToggleGroup` and `ToggleGroupItem` from `@/components/ui/toggle-group` but the file did not exist in the project
- **Fix:** Created `toggle-group.tsx` wrapping `@radix-ui/react-toggle-group` (already installed as part of the `radix-ui` meta-package). Used inline CVA-free styles to avoid dependency on a missing `toggle.variants.ts` file
- **Files modified:** `trade-flow-ui/src/components/ui/toggle-group.tsx` (created)
- **Commit:** eb1b584

**2. [Rule 2 - Warning tokens] Replaced existing warning values with disclaimer-specific palette**
- **Found during:** Task 1
- **Issue:** `index.css` already had `--warning` defined with a different value (`oklch(0.75 0.15 85)` — a saturated amber). The plan specified a softer pale yellow palette (`oklch(0.96 0.03 90)`) more appropriate for an informational disclaimer card
- **Fix:** Applied the plan's specified values, replacing the existing warning color definitions
- **Files modified:** `trade-flow-ui/src/index.css`
- **Commit:** e4cc213

## Known Stubs

None — all data is wired through RTK Query from the API. No hardcoded empty values flow to UI rendering.

## Threat Flags

None — no new network endpoints or auth paths introduced beyond what the plan's threat model covers (T-45-09 through T-45-12 all mitigated per plan).

## Self-Check: PASSED

All 15 created/modified files found on disk. All 3 task commits verified in git log.

| Check | Result |
|-------|--------|
| public-estimate.types.ts | FOUND |
| publicEstimateApi.ts | FOUND |
| index.ts (barrel) | FOUND |
| PublicEstimateDisclaimer.tsx | FOUND |
| PublicEstimatePriceRange.tsx | FOUND |
| PublicEstimateUncertainty.tsx | FOUND |
| PublicEstimateTerminalState.tsx | FOUND |
| PublicEstimateSkeleton.tsx | FOUND |
| PublicEstimateError.tsx | FOUND |
| PublicEstimateResponseButtons.tsx | FOUND |
| PublicEstimateMessageForm.tsx | FOUND |
| PublicEstimateDeclineForm.tsx | FOUND |
| PublicEstimateCard.tsx | FOUND |
| PublicEstimatePage.tsx | FOUND |
| toggle-group.tsx | FOUND |
| Commit e4cc213 (Task 1) | FOUND |
| Commit 59e5de6 (Task 2) | FOUND |
| Commit eb1b584 (Task 3) | FOUND |
