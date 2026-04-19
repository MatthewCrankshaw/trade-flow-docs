---
phase: 57-impersonation-frontend
reviewed: 2026-04-19T00:00:00Z
depth: standard
files_reviewed: 14
files_reviewed_list:
  - trade-flow-ui/src/store/slices/impersonationSlice.ts
  - trade-flow-ui/src/features/support/api/impersonationApi.ts
  - trade-flow-ui/src/store/index.ts
  - trade-flow-ui/src/services/api.ts
  - trade-flow-ui/src/features/support/hooks/useImpersonation.ts
  - trade-flow-ui/src/features/support/components/ImpersonationBanner.tsx
  - trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx
  - trade-flow-ui/src/features/auth/components/PaywallGuard.tsx
  - trade-flow-ui/src/config/navigation.ts
  - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
  - trade-flow-ui/src/features/auth/components/__tests__/OnboardingGuard.test.tsx
  - trade-flow-ui/src/features/auth/components/__tests__/PaywallGuard.test.tsx
  - trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx
  - trade-flow-ui/src/pages/support/SupportUserDetailPage.tsx
findings:
  critical: 2
  warning: 4
  info: 3
  total: 9
status: issues_found
---

# Phase 57: Code Review Report — Impersonation Frontend

**Reviewed:** 2026-04-19
**Depth:** standard
**Files Reviewed:** 14
**Status:** issues_found

## Summary

This phase adds client-side impersonation support: a Redux slice for session state, an RTK Query API layer, a `useImpersonation` hook, an `ImpersonationBanner`, guard bypasses in `OnboardingGuard` and `PaywallGuard`, navigation switching, and a `ImpersonateUserDialog`. The overall architecture is sound — token injection via `prepareHeaders`, 401-triggered auto-expiry, navigation switching at start/end — but two critical issues require fixes before merge: the impersonation token is stored in Redux (plain JS memory, survives soft reloads on the same tab but is cleared by hard reload) with no handling of the resulting stale-state edge case on reload, and the `endImpersonation` dispatch/navigate ordering leaves a brief window where the Redux cache is reset before navigation completes, causing an uncached API request against the old (impersonated) token. There are also four warnings covering logic correctness and UX correctness, and three info-level items.

---

## Critical Issues

### CR-01: Impersonation token survives in Redux across `api/resetApiState` but the store is not persisted — a hard reload during an active session silently drops the token while leaving no recovery path

**File:** `trade-flow-ui/src/services/api.ts:35-38`

**Issue:** When a 401 fires during an active session the handler calls `endImpersonation()` and `api/resetApiState`. Both are correct individually. However the `ImpersonationBanner` and guards read `state.impersonation.active` which becomes `false` immediately — but only after the failed response. Because the Redux store is in-memory only, a hard refresh (F5 or browser restart) will clear `active = false` from the start, but any in-flight request that was already sent with the stale token will not be intercepted (the tab is gone). This is an inherent consequence of the chosen architecture, but the real critical problem is that `api/resetApiState` is dispatched as a plain action string rather than through the RTK `apiSlice.util.resetApiState()` utility:

```ts
// api.ts line 37 — plain string action, not the typed util
api.dispatch({ type: "api/resetApiState" });
```

`apiSlice.util.resetApiState()` is the canonical RTK Query action creator; dispatching the raw string works today only because the `reducerPath` happens to be `"api"`. If the `reducerPath` ever changes (or in test environments where a different `reducerPath` is configured), the cache will not be cleared. The `useImpersonation` hook already uses the correct form (`dispatch(apiSlice.util.resetApiState())`). This file should match.

**Fix:**
```ts
// src/services/api.ts — replace line 37
import { apiSlice } from "@/services/api"; // self-reference is fine since it is in the same file

// Inside baseQueryWithImpersonationExpiry:
if (result.error && result.error.status === 401 && state.impersonation.active) {
  api.dispatch(endImpersonation());
  api.dispatch(apiSlice.util.resetApiState());
}
```

Because `apiSlice` is defined in the same file, the import is not needed — just replace:
```ts
api.dispatch({ type: "api/resetApiState" });
```
with:
```ts
api.dispatch(apiSlice.util.resetApiState());
```
Note: `apiSlice` is already in scope at module level in `api.ts`, so no import is necessary.

---

### CR-02: `endSession` dispatches `endImpersonation` and `apiSlice.util.resetApiState` **after** `navigate("/support")`, creating a window where the reset-API-state triggers RTK Query to re-fetch using the **impersonated** token while the support page mounts

**File:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts:57-61`

**Issue:** The current order is:

```ts
await endMutation({ sessionId }).unwrap();  // 1 — server session closed
navigate("/support");                        // 2 — React navigates, support page mounts, useGetCurrentUserQuery fires
dispatch(endImpersonation());               // 3 — token cleared from Redux
dispatch(apiSlice.util.resetApiState());    // 4 — cache cleared
```

Between steps 2 and 3, the newly mounted support page makes RTK Query requests. `prepareHeaders` still sees `state.impersonation.active === true` and `state.impersonation.token` still set, so those requests are sent with the **impersonated user's JWT**. The API will likely reject them (the backend session is already closed after step 1), but they will return 401s and trigger the auto-expiry handler in `api.ts` — a redundant cascade that produces a confusing log trace and could result in a double-clear.

**Fix:** Clear Redux state and cache **before** navigating, so any new requests fired by the target page are made with the support user's own Firebase token:
```ts
const endSession = useCallback(async () => {
  const { sessionId } = impersonation;
  if (!sessionId) return;

  try {
    await endMutation({ sessionId }).unwrap();
    dispatch(endImpersonation());              // clear token first
    dispatch(apiSlice.util.resetApiState());   // clear cache
    navigate("/support");                      // then navigate
    toast.success("Impersonation session ended");
  } catch {
    toast.error("Unable to end impersonation session. Please try again.");
  }
}, [impersonation, endMutation, dispatch, navigate]);
```

---

## Warnings

### WR-01: `PaywallGuard` checks `isImpersonating` before the loading guard, meaning the paywall bypass fires even when subscription data has not yet loaded — this is correct today but the ordering creates a subtle precedence dependency that silently breaks if the loading check is moved above it

**File:** `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx:33-52`

**Issue:** The current order is: impersonation bypass → support-role bypass → loading spinner → active subscription → paywall. This means that if `isLoading` is `true` and `isImpersonating` is `true`, the component skips loading and renders the Outlet immediately. This is intentional (impersonating users should never see a spinner for the paywall). However the comment on line 45 references ticket `D-08` for the loading check while the impersonation check has no defensive comment. If a future developer reorders these checks to move loading above impersonation, support-impersonated sessions will incorrectly see the loading spinner and potentially flash a blank screen. The ordering should be documented with a comment to prevent regression.

**Fix:**
```tsx
// Impersonation and support-role bypasses must precede the loading check.
// Support users (real or impersonating) must never see a paywall or loading spinner.
if (isImpersonating) {
  return <Outlet />;
}
if ((user?.supportRoles?.length ?? 0) > 0) {
  return <Outlet />;
}
// Loading state — seamless transition (D-08) — only reached for real customers
if (isLoading) { ... }
```

---

### WR-02: `useImpersonation.startSession` populates `targetUser` in Redux from the **caller-supplied** name/email, not from the server response — a mismatch between the two is silently swallowed

**File:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts:32-38`

**Issue:**
```ts
const result = await startMutation({ targetUserId, reason }).unwrap();
dispatch(
  startImpersonation({
    token: result.token,
    sessionId: result.sessionId,
    targetUser: { id: targetUserId, name: targetUserName, email: targetUserEmail }, // caller values, not result
  }),
);
```

The server response `ImpersonationStartResponse` already includes `targetUser: { id, name, email }` (defined in `impersonationApi.ts:10-16`). The hook ignores `result.targetUser` and stores the values passed by the caller instead. If the caller's `targetUserName` or `targetUserEmail` differs from the authoritative server data (e.g., the user renamed themselves between page load and clicking "Impersonate"), the banner and any other consumer of `state.impersonation.targetUser` will display stale data. More importantly, the `id` used in Redux comes from the caller parameter, not `result.targetUser.id` — which is an authoritative source that the server chose to include.

**Fix:**
```ts
dispatch(
  startImpersonation({
    token: result.token,
    sessionId: result.sessionId,
    targetUser: result.targetUser, // use server-authoritative data
  }),
);
```
Remove `targetUserName` and `targetUserEmail` from `startSession`'s signature since they become unused (the caller already passes `targetUserId` for the API call; name and email come back from the server).

---

### WR-03: `DashboardLayout` useEffect navigates to `/support` on impersonation expiry, but the `wasImpersonatingRef` initialises to `false` — on first render when impersonation is already active (e.g., page loaded mid-session), the ref is `false` and `isImpersonating` is `true`, so no expiry fires. However if the component is unmounted and remounted (e.g., route change within the dashboard), `wasImpersonatingRef` resets to `false` again, meaning a subsequent session expiry on the new mount will be missed for one render cycle

**File:** `trade-flow-ui/src/components/layouts/DashboardLayout.tsx:140-148`

**Issue:**
```ts
const wasImpersonatingRef = useRef(false);

useEffect(() => {
  if (wasImpersonatingRef.current && !isImpersonating) {
    navigate("/support");
    toast.warning("Impersonation session expired");
  }
  wasImpersonatingRef.current = isImpersonating;
}, [isImpersonating, navigate]);
```

When `DashboardLayout` re-mounts (React Router can unmount/remount layouts when navigating between routes depending on tree structure), `wasImpersonatingRef.current` starts as `false`. If `isImpersonating` is `true` at mount time and the session expires, the first run of the effect sets `wasImpersonatingRef.current = true`. On the next render caused by `isImpersonating` flipping to `false`, the expiry logic fires correctly. This is actually fine in steady-state. The actual risk is: if the layout is remounted (e.g., hard navigation) exactly at the moment the 401 fires and `endImpersonation` has already dispatched, `isImpersonating` will be `false` on the very first render — the ref will be initialised to `false`, the condition `wasImpersonatingRef.current && !isImpersonating` is `false && true = false`, and no navigation or toast fires. The expiry is silently swallowed.

**Fix:** Initialise the ref from the current Redux state at mount so remounts correctly capture ongoing session state:
```ts
const wasImpersonatingRef = useRef(isImpersonating);
```

---

### WR-04: `endImpersonation` API call failure in `useImpersonation.endSession` leaves the session open on the **server** while the user sees a toast and remains stuck — there is no retry affordance or forced local-clear fallback

**File:** `trade-flow-ui/src/features/support/hooks/useImpersonation.ts:50-64`

**Issue:**
```ts
try {
  await endMutation({ sessionId }).unwrap();
  // ... success path
} catch {
  toast.error("Unable to end impersonation session. Please try again.");
}
```

If the `endMutation` call fails (network error, server error), the catch block shows a toast and returns. The impersonation state in Redux remains active, the banner stays visible, and the button re-enables (since `isEnding` returns to `false`). The user can retry. However the server-side session remains open, which is a security concern: the impersonation token is still valid. The UX offers no graceful degradation — the user must retry indefinitely until the server-side session TTL expires naturally.

A safer pattern is to clear Redux state locally even on API failure, accepting that the server session may outlive the client session (the backend should enforce TTL independently). This ensures the support user cannot accidentally keep sending impersonated requests even if the end-session API is temporarily unavailable.

**Fix:**
```ts
try {
  await endMutation({ sessionId }).unwrap();
} catch {
  // Server-side session may still be active; clear client state regardless
  // to prevent continued impersonated requests. Backend TTL will expire the session.
  toast.error("Session end request failed. Client session cleared for safety.");
} finally {
  dispatch(endImpersonation());
  dispatch(apiSlice.util.resetApiState());
  navigate("/support");
}
```

---

## Info

### IN-01: `impersonationApi.ts` `endImpersonation` mutation has no `transformResponse` but the API returns `StandardResponse<void>` — if the backend returns a non-void body the mutation silently discards it with no type check

**File:** `trade-flow-ui/src/features/support/api/impersonationApi.ts:38-44`

**Issue:** The `endImpersonation` mutation is typed as `builder.mutation<void, ImpersonationEndRequest>`. If the backend actually returns `StandardResponse<void>` (a wrapper object), RTK Query will return the raw wrapper as the result but the TypeScript type says `void`. This is not a crash risk today but it means the return type and runtime value diverge. Add a `transformResponse` for consistency with the `startImpersonation` endpoint.

**Fix:**
```ts
endImpersonation: builder.mutation<void, ImpersonationEndRequest>({
  query: (body) => ({
    url: "/v1/impersonation/end",
    method: "POST",
    body,
  }),
  transformResponse: () => undefined,
}),
```

---

### IN-02: `SupportUserDetailPage` uses `new Date()` for `canceledAt` and `trialEnd` comparisons in `PaywallGuard.derivePaywallVariant` — this is in `PaywallGuard.tsx`, not `SupportUserDetailPage.tsx`, but `formatIsoDate` in `SupportUserDetailPage.tsx:21-24` correctly uses Luxon while `PaywallGuard.tsx:19-21` uses `new Date()` for comparisons

**File:** `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx:19-21`

**Issue:** Per the project CLAUDE.md: "Never use `new Date()` for parsing API date strings — always use Luxon." The `derivePaywallVariant` function uses `new Date(subscription.trialEnd).getTime()` and `new Date(subscription.canceledAt).getTime()` for the 24-hour proximity check:

```ts
const trialEnd = subscription.trialEnd ? new Date(subscription.trialEnd).getTime() : 0;
const canceledAt = subscription.canceledAt ? new Date(subscription.canceledAt).getTime() : 0;
```

**Fix:**
```ts
import { DateTime } from "luxon";

const trialEnd = subscription.trialEnd ? DateTime.fromISO(subscription.trialEnd).toMillis() : 0;
const canceledAt = subscription.canceledAt ? DateTime.fromISO(subscription.canceledAt).toMillis() : 0;
```

---

### IN-03: `ImpersonateUserDialog` validates that the reason field is non-empty but imposes no minimum meaningful length — a single space followed by trim correctly catches whitespace, but a single character (e.g., `"x"`) passes validation and reaches the API

**File:** `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx:47-49`

**Issue:** The current validation is `if (!trimmedReason)` which blocks empty and whitespace-only strings. A support user could submit a single-character reason that provides no audit value. This is a low-priority UX issue, not a security issue (the backend is the authoritative audit log), but it makes the audit trail less useful.

**Fix:** Consider a minimum length of 10–20 characters:
```ts
const MIN_REASON_LENGTH = 10;
if (trimmedReason.length < MIN_REASON_LENGTH) {
  setShowValidation(true);
  return;
}
```
Update the validation message accordingly: `"A reason of at least 10 characters is required."`

---

_Reviewed: 2026-04-19_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
