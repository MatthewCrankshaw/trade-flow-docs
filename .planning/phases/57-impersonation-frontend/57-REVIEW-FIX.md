---
phase: 57-impersonation-frontend
fixed_at: 2026-04-19T00:00:00Z
review_path: .planning/phases/57-impersonation-frontend/57-REVIEW.md
iteration: 1
findings_in_scope: 6
fixed: 6
skipped: 0
status: all_fixed
---

# Phase 57: Code Review Fix Report

**Fixed at:** 2026-04-19
**Source review:** `.planning/phases/57-impersonation-frontend/57-REVIEW.md`
**Iteration:** 1

**Summary:**
- Findings in scope: 6 (2 Critical, 4 Warning)
- Fixed: 6
- Skipped: 0

## Fixed Issues

### CR-01: Use apiSlice.util.resetApiState() instead of plain string action

**Files modified:** `trade-flow-ui/src/services/api.ts`
**Commit:** bd24da5
**Applied fix:** Replaced `api.dispatch({ type: "api/resetApiState" })` with `api.dispatch(apiSlice.util.resetApiState())` in the 401 auto-expiry handler. `apiSlice` is declared in the same file so no import was needed. The call site inside `baseQueryWithImpersonationExpiry` resolves the module-level const correctly at runtime.

---

### CR-02: Dispatch endImpersonation and resetApiState before navigate in endSession

**Files modified:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts`
**Commit:** 696e093
**Applied fix:** Reordered the `endSession` success path so `dispatch(endImpersonation())` and `dispatch(apiSlice.util.resetApiState())` fire before `navigate("/support")`. This ensures Redux token state is cleared before the support page mounts and RTK Query fires any requests.

---

### WR-01: Document bypass ordering requirement in PaywallGuard

**Files modified:** `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx`
**Commit:** 5bf5329
**Applied fix:** Replaced the single-line impersonation bypass comment with a two-line comment explaining that impersonation and support-role bypasses must precede the loading check, and that support users must never see a paywall or loading spinner.

---

### WR-02: Use server-authoritative result.targetUser in startImpersonation dispatch

**Files modified:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts`, `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx`
**Commit:** 168ea90
**Applied fix:** Updated `startSession` to dispatch `targetUser: result.targetUser` (from the `ImpersonationStartResponse`) instead of `{ id: targetUserId, name: targetUserName, email: targetUserEmail }` from caller-supplied arguments. Removed `targetUserName` and `targetUserEmail` from both the `startSession` function signature and the `UseImpersonationReturn` interface type. Updated `ImpersonateUserDialog` call site from `startSession(targetUser.id, targetUser.name, targetUser.email, trimmedReason)` to `startSession(targetUser.id, trimmedReason)`.

---

### WR-03: Initialise wasImpersonatingRef from current Redux state on mount

**Files modified:** `trade-flow-ui/src/components/layouts/DashboardLayout.tsx`
**Commit:** 2d20389
**Applied fix:** Changed `useRef(false)` to `useRef(isImpersonating)` so that if `DashboardLayout` remounts while a session is active (e.g., hard navigation between routes), the ref correctly captures ongoing session state and subsequent expiry is not silently missed.

---

### WR-04: Clear Redux state in finally block so endSession cleans up on API failure

**Files modified:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts`
**Commit:** 4b031bf
**Applied fix:** Restructured `endSession` to use a `try/catch/finally` pattern. The `finally` block unconditionally dispatches `endImpersonation()`, `apiSlice.util.resetApiState()`, and `navigate("/support")`, ensuring client session state is cleared even when the end-session API call fails. The catch block now shows a specific error toast noting the client session was cleared for safety, with backend TTL responsible for expiring the server-side session.

---

_Fixed: 2026-04-19_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
