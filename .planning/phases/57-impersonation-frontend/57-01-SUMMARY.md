---
phase: 57-impersonation-frontend
plan: 01
subsystem: ui
tags: [redux, rtk-query, impersonation, token-switching, react, typescript]

# Dependency graph
requires:
  - phase: 56-impersonation-backend
    provides: POST /v1/impersonation/start and POST /v1/impersonation/end API endpoints with JWT token response
provides:
  - Redux impersonationSlice with startImpersonation/endImpersonation actions and selectImpersonation selector
  - impersonationApi RTK Query endpoints (useStartImpersonationMutation, useEndImpersonationMutation)
  - Token-switching prepareHeaders in api.ts that uses impersonation token when active, Firebase token otherwise
  - baseQuery wrapper that detects 401 during active impersonation and clears state + resets cache
affects: [57-02, 57-03, 57-04, impersonation-banner, impersonation-dialog, guard-bypass]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - createSlice for Redux impersonation state (first Redux slice in this project)
    - RTK Query baseQuery wrapper for cross-cutting error interception
    - Token switching via prepareHeaders getState() reading from Redux

key-files:
  created:
    - trade-flow-ui/src/store/slices/impersonationSlice.ts
    - trade-flow-ui/src/features/support/api/impersonationApi.ts
  modified:
    - trade-flow-ui/src/store/index.ts
    - trade-flow-ui/src/services/api.ts

key-decisions:
  - "Use api/resetApiState string action type to avoid circular reference between baseQueryWithImpersonationExpiry and apiSlice"
  - "Impersonation state stored in Redux only (no sessionStorage/localStorage) — page refresh terminates session by design"
  - "baseQuery wrapper dispatches endImpersonation + resetApiState on 401 during active impersonation; navigation handled by React component watching impersonation.active"

patterns-established:
  - "Pattern: Redux slice created with createSlice — first use in project, sets pattern for future slices"
  - "Pattern: api.ts baseQuery wrapper for cross-cutting response interception without per-endpoint handling"
  - "Pattern: Token switching in prepareHeaders using getState() — impersonation token takes priority over Firebase token"

requirements-completed: [IMP-03]

# Metrics
duration: 8min
completed: 2026-04-19
---

# Phase 57 Plan 01: Impersonation Infrastructure Layer Summary

**Redux impersonation slice with token-switching prepareHeaders and expired-session baseQuery wrapper — complete data layer for impersonation**

## Performance

- **Duration:** ~8 min
- **Started:** 2026-04-19T17:05:00Z
- **Completed:** 2026-04-19T17:06:38Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Created `impersonationSlice` (first Redux slice in project) with `startImpersonation`/`endImpersonation` actions and `selectImpersonation` selector, registered in the Redux store
- Created `impersonationApi` with RTK Query mutations for `POST /v1/impersonation/start` and `POST /v1/impersonation/end` using the standard `injectEndpoints` pattern
- Modified `api.ts` prepareHeaders to use the impersonation token when `state.impersonation.active` is true, falling back to Firebase token otherwise
- Added `baseQueryWithImpersonationExpiry` wrapper that detects 401 responses during active impersonation and automatically dispatches `endImpersonation()` + `api/resetApiState`

## Task Commits

Each task was committed atomically:

1. **Task 1: Create impersonation Redux slice, register in store, and create RTK Query API endpoints** - `28f03a0` (feat)
2. **Task 2: Modify api.ts for token switching and expired session detection** - `a6d06c5` (feat)

**Plan metadata:** (committed with docs commit below)

## Files Created/Modified

- `trade-flow-ui/src/store/slices/impersonationSlice.ts` - Redux slice for impersonation session state (active, token, sessionId, targetUser)
- `trade-flow-ui/src/features/support/api/impersonationApi.ts` - RTK Query mutations for start/end impersonation endpoints
- `trade-flow-ui/src/store/index.ts` - Registered impersonation reducer in the Redux store
- `trade-flow-ui/src/services/api.ts` - Token-switching prepareHeaders and expired-token baseQuery wrapper

## Decisions Made

- **Circular reference avoidance:** `baseQueryWithImpersonationExpiry` dispatches `{ type: "api/resetApiState" }` (string action type) rather than `apiSlice.util.resetApiState()` to avoid the circular dependency that would arise from referencing `apiSlice` before it is defined.
- **Navigation not in baseQuery wrapper:** The baseQuery wrapper only dispatches Redux actions; navigation to `/support` after expiry is handled by a React component watching `impersonation.active` (per RESEARCH.md Pitfall 4 guidance).

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None - both tasks completed cleanly. The potential circular reference issue flagged in the plan was resolved using the string action type dispatch approach.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Impersonation data layer is complete and ready for Plan 02 (UI components: banner, dialog, guard bypass)
- `useStartImpersonationMutation` and `useEndImpersonationMutation` hooks are available for `useImpersonation` hook in Plan 02/03
- `selectImpersonation` and `startImpersonation`/`endImpersonation` actions are ready for consumption
- No blockers

## Threat Flags

No new threat surface beyond what is documented in the plan's threat model. All three mitigations implemented:
- T-57-01: Token stored in Redux only (no localStorage/sessionStorage)
- T-57-02: Both `state.impersonation.active` AND `state.impersonation.token` checked before token switching
- T-57-03: 401 during active impersonation triggers `endImpersonation()` + `resetApiState`

---
*Phase: 57-impersonation-frontend*
*Completed: 2026-04-19*
