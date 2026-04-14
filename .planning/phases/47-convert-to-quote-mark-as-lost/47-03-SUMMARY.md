---
phase: 47-convert-to-quote-mark-as-lost
plan: 03
subsystem: estimate
tags: [frontend, ui, rtk-query, mutations, dialog, convert, lost]
dependency_graph:
  requires: [47-02]
  provides: [convert-ui, mark-lost-ui, locked-state-display]
  affects: [estimate types, quote types, estimateApi, EstimateActionStrip, EstimateDetailPage]
tech_stack:
  added: []
  patterns: [conditional-render-by-status, key-remount-for-state-reset, idempotency-key-generation]
key_files:
  created:
    - trade-flow-ui/src/features/estimates/components/MarkAsLostDialog.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLostReasonCard.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateConvertedLink.tsx
  modified:
    - trade-flow-ui/src/types/estimate.ts
    - trade-flow-ui/src/types/quote.ts
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
    - trade-flow-ui/src/features/estimates/components/index.ts
    - trade-flow-ui/src/pages/EstimateDetailPage.tsx
decisions:
  - Used key-based remount pattern (conditional render with open flag) instead of useEffect+setState for MarkAsLostDialog state reset to satisfy react-hooks/set-state-in-effect ESLint rule
  - EstimateActionStrip returns null for lost and other terminal statuses; EstimateLostReasonCard renders in detail page content area instead
metrics:
  duration: 4min
  completed: "2026-04-14T07:31:00Z"
  tasks: 2
  files: 9
---

# Phase 47 Plan 03: Frontend UI for Convert to Quote and Mark as Lost Summary

RTK Query mutations for convert and mark-lost endpoints with idempotency key header, MarkAsLostDialog with 5 reason chips and freeform textarea, EstimateLostReasonCard and EstimateConvertedLink display components, and EstimateActionStrip rewired with status-based button visibility for all estimate lifecycle states.

## Task Results

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Frontend types, RTK Query mutations, and static display components | 6f72866 | 7 files (types, estimateApi, MarkAsLostDialog, EstimateLostReasonCard, EstimateConvertedLink, barrel) |
| 2 | EstimateActionStrip integration and EstimateDetailPage wiring | 7e911b2 | 5 files (EstimateActionStrip, MarkAsLostDialog, EstimateConvertedLink, EstimateLostReasonCard, EstimateDetailPage) |

## Decisions Made

1. **Key-based remount for dialog state reset**: Used conditional rendering (`{open && <MarkAsLostDialogBody />}`) instead of `useEffect` + `setState` to reset dialog form state when opened. This satisfies the `react-hooks/set-state-in-effect` ESLint rule that the project enforces.

2. **EstimateActionStrip returns null for terminal statuses**: Lost estimates show no action buttons in the strip; the `EstimateLostReasonCard` renders in the detail page content area above line items (not inside the action strip).

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed ESLint set-state-in-effect violation**
- **Found during:** Task 1 (CI verification)
- **Issue:** `useEffect` with `setState` in MarkAsLostDialog triggered `react-hooks/set-state-in-effect` lint error
- **Fix:** Extracted dialog body into `MarkAsLostDialogBody` inner component, conditionally rendered only when `open` is true, so `useState` initializers reset naturally on remount
- **Files modified:** trade-flow-ui/src/features/estimates/components/MarkAsLostDialog.tsx
- **Commit:** 7e911b2

## Verification

- TypeScript compilation: zero errors
- Unit tests: 94/94 pass
- ESLint: zero errors (1 pre-existing warning in unrelated BusinessStep.tsx)
- Formatting: all plan files pass Prettier check (5 pre-existing format issues in unrelated files)
- Acceptance criteria: all 18 acceptance criteria verified

## Self-Check: PASSED
