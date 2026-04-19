---
phase: 57-impersonation-frontend
plan: 03
subsystem: ui
tags: [react, impersonation, dialog, confirmation, support, typescript]

# Dependency graph
requires:
  - phase: 57-02
    provides: useImpersonation hook with startSession/endSession lifecycle
provides:
  - ImpersonateUserDialog confirmation dialog with required reason field and loading state
  - Impersonate User button on SupportUserDetailPage gated by canImpersonate boolean
affects: [57-04, impersonation-entry-point]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Dialog wrapper + lazy-mount body pattern (MarkAsLostDialog analog)
    - canImpersonate boolean gating button/dialog visibility (hidden entirely, not disabled)

key-files:
  created:
    - trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx
  modified:
    - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx

key-decisions:
  - "Permission check uses currentUser.supportRoles.length > 0 as proxy — User type has no permissions array (Phase 52 not built); backend enforces the actual impersonate_user permission with 403"
  - "canImpersonate triple-condition: not currently impersonating AND target has no support roles AND viewer has at least one support role"
  - "ImpersonateUserDialog reuses Dialog (not AlertDialog) to match MarkAsLostDialog codebase pattern"
  - "Prettier auto-fix applied to ImpersonateUserDialog.tsx after initial write (line length for DialogDescription)"

requirements-completed: [IMP-03, IMP-05]

# Metrics
duration: 4min
completed: 2026-04-19
---

# Phase 57 Plan 03: Impersonation Entry Point Summary

**Confirmation dialog with required reason field and conditional Impersonate User button on the support user detail page — complete impersonation entry point**

## Performance

- **Duration:** ~4 min
- **Started:** 2026-04-19T17:18:00Z
- **Completed:** 2026-04-19T17:22:11Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments

- Created `ImpersonateUserDialog` following the `MarkAsLostDialog` wrapper + lazy-mount body split pattern. Dialog includes:
  - Required reason `Textarea` with inline validation message "A reason is required before impersonating a user."
  - "Start Impersonation" button with `Loader2` spinner and "Starting..." text during `isStarting` state
  - "Keep Reviewing" cancel button, both disabled during submission
  - Calls `useImpersonation().startSession()` on valid submit; dialog closes on success, stays open on error (toast handled by hook)
- Modified `SupportUserDetailPage` to:
  - Import and render `ImpersonateUserDialog` conditionally on `canImpersonate`
  - Read `state.impersonation.active` from Redux via `useAppSelector`
  - Compute `canImpersonate = !isImpersonating && targetIsCustomer && viewerHasPermission`
  - Show "Impersonate User" button with `UserCheck` icon and `aria-label` in the page header when `canImpersonate` is true
  - Hide button entirely when conditions are not met (D-10 — no disabled/greyed-out state)

## Task Commits

1. **Task 1: Create ImpersonateUserDialog and add Impersonate User button to SupportUserDetailPage** — `8c8b4d9` (feat)

## Files Created/Modified

- `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx` — Dialog component with required reason textarea, validation, and loading state; calls `useImpersonation().startSession()`
- `trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx` — Added conditional "Impersonate User" button in header area and `ImpersonateUserDialog` mount; `canImpersonate` boolean gates both

## Decisions Made

- **Permission check proxy:** The `User` type (`api.types.ts`) has `supportRoles: SupportRole[]` with only `id` and `roleName` — no `permissions` array (Phase 52 not yet built). Used `currentUser.supportRoles.length > 0` as the viewer permission check proxy. The backend enforces the actual `impersonate_user` permission with a 403, so this is safe for now.
- **Dialog variant:** Used `Dialog` (not `AlertDialog`) to match the `MarkAsLostDialog` pattern already established in the codebase. `AlertDialog` was listed in UI-SPEC but the codebase analog uses `Dialog`.
- **canImpersonate placement:** Computed in `SupportUserDetailContent` after `targetUser` is confirmed non-null to avoid undefined access.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Prettier formatting on ImpersonateUserDialog.tsx**

- **Found during:** Task 1 CI format:check run
- **Issue:** The `DialogDescription` text exceeded Prettier's 125-character line width.
- **Fix:** Ran `prettier --write` on `ImpersonateUserDialog.tsx`. Prettier reformatted the description across two lines.
- **Files modified:** `ImpersonateUserDialog.tsx`
- **Commit:** included in `8c8b4d9`

## Known Stubs

None — the dialog wires directly to `useImpersonation().startSession()` which calls the real API. The button visibility is wired to live Redux state (`state.impersonation.active`) and live API data (`targetUser.supportRoleIds`, `currentUser.supportRoles`). No placeholder data.

## Threat Flags

No new threat surface beyond the plan's threat model. All four mitigations implemented:
- T-57-08: Button hidden entirely when viewer lacks support roles (canImpersonate false); backend enforces 403
- T-57-09: Button hidden when target has any support role ids (targetIsCustomer false); backend enforces 403
- T-57-10: Required reason field ensures audit trail submitted to POST /v1/impersonation/start
- T-57-11: Reason is free text; backend responsible for sanitization (accepted disposition)

## Self-Check

- `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx` — FOUND
- `trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx` — modified
- Commit `8c8b4d9` — FOUND (trade-flow-ui main branch)
- `npm run ci` passes — CONFIRMED (13 test files, 102 tests, 0 lint errors, format clean, typecheck clean)

## Self-Check: PASSED

---
*Phase: 57-impersonation-frontend*
*Completed: 2026-04-19*
