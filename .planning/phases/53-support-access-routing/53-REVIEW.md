---
phase: 53-support-access-routing
reviewed: 2026-04-19T00:00:00Z
depth: standard
files_reviewed: 9
files_reviewed_list:
  - trade-flow-ui/src/features/auth/components/SupportGuard.tsx
  - trade-flow-ui/src/features/auth/components/index.ts
  - trade-flow-ui/src/config/navigation.ts
  - trade-flow-ui/src/App.tsx
  - trade-flow-ui/src/pages/LoginPage.tsx
  - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
  - trade-flow-ui/src/pages/support/SupportDashboardPage.tsx
  - trade-flow-ui/src/pages/support/SupportBusinessesPage.tsx
  - trade-flow-ui/src/pages/support/SupportUsersPage.tsx
findings:
  critical: 0
  warning: 4
  info: 3
  total: 7
status: issues_found
---

# Phase 53: Code Review Report

**Reviewed:** 2026-04-19
**Depth:** standard
**Files Reviewed:** 9
**Status:** issues_found

## Summary

This phase adds a support user routing layer: a `SupportGuard` component, support navigation config, three support pages, and wiring across `App.tsx`, `LoginPage.tsx`, and `DashboardLayout.tsx`. The overall structure is sound — the guard chain is logically correct, role-based navigation switching works as intended, and the login redirect logic is clean.

Four issues warrant attention before shipping:

1. `navigation.ts` uses `React.ComponentType` without importing `React`, which is a type error under the project's `verbatimModuleSyntax` + `react-jsx` config.
2. `SupportGuard` conflates API error with "no access", silently redirecting on network failure.
3. `SupportBusinessesPage` ships hardcoded mock data with non-functional search/filter/action controls — it reads as production code but has no real implementation.
4. `LoginPage` uses a type assertion (`as`) for `location.state`, which the project's coding standards explicitly discourage.

## Warnings

### WR-01: `React.ComponentType` referenced without importing `React` in a `.ts` file

**File:** `trade-flow-ui/src/config/navigation.ts:16`
**Issue:** `NavItem.icon` is typed as `React.ComponentType<{ className?: string }>`. The file is `navigation.ts` (not `.tsx`), and there is no `import React from 'react'` or `import type { ComponentType } from 'react'`. With `"jsx": "react-jsx"` and `"verbatimModuleSyntax": true` the `React` global namespace is not injected — this is a type error that `npm run typecheck` will catch.
**Fix:**
```typescript
// Replace the bare React.ComponentType reference:
import type { ComponentType } from "react";

export interface NavItem {
  title: string;
  href: string;
  icon: ComponentType<{ className?: string }>;
  description?: string;
}
```

---

### WR-02: `SupportGuard` treats API error as "no access" — silent failure mode

**File:** `trade-flow-ui/src/features/auth/components/SupportGuard.tsx:17`
**Issue:** When `useGetCurrentUserQuery` fails (network error, 401, 500), `user` is `undefined` and `isLoading` is `false`. The guard falls through to `if (!user || ...)` and redirects to `/dashboard`. A support user hitting a transient network error is silently bounced to the business dashboard with no explanation. The two failure modes — "not a support user" and "couldn't fetch user" — should be handled separately.
**Fix:**
```typescript
export function SupportGuard() {
  const { data: user, isLoading, isError } = useGetCurrentUserQuery();

  if (isLoading) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <Loader2 className="size-8 animate-spin text-muted-foreground" />
      </div>
    );
  }

  if (isError) {
    // Don't redirect — surface the error so the user can retry
    return (
      <div className="flex min-h-screen items-center justify-center">
        <p className="text-muted-foreground">Unable to verify access. Please refresh.</p>
      </div>
    );
  }

  if (!user || user.supportRoles.length === 0) {
    return <Navigate to="/dashboard" replace />;
  }

  return <Outlet />;
}
```

---

### WR-03: `SupportBusinessesPage` ships hardcoded mock data with non-functional controls

**File:** `trade-flow-ui/src/pages/support/SupportBusinessesPage.tsx:13-54, 113-120, 150`
**Issue:** The page renders from `mockBusinesses` — a hardcoded constant — not from an API call. The search `<Input>` has no `value` or `onChange`. The `Filter` button has no handler. The `MoreHorizontal` action button per row has no handler. If this ships as-is, support users see fabricated data and interactions silently do nothing, which is misleading. Compare `SupportUsersPage`, which is fully wired to real data via `useSupportUsers()`.
**Fix:** Either wire the page to a real API endpoint (the pattern used by `SupportUsersPage` is the right model) before merging, or gate the page behind a feature flag / mark it clearly as a placeholder. At minimum, remove the mock data constant and render an empty state or a "coming soon" message so the gap is explicit.

---

### WR-04: Type assertion `as` used for `location.state` in `LoginPage`

**File:** `trade-flow-ui/src/pages/LoginPage.tsx:33`
**Issue:** `(location.state as { from?: Location })?.from` casts `location.state` (typed as `unknown`) to a specific shape without validation. The project's coding standards (memory entry: "prefer type guards/validation over `as` casts") flag this pattern. If `location.state` contains anything other than the expected shape — e.g., a stale history entry from a different state object — the navigation will silently misbehave.
**Fix:**
```typescript
function isFromLocation(state: unknown): state is { from: Location } {
  return (
    typeof state === "object" &&
    state !== null &&
    "from" in state &&
    typeof (state as { from: unknown }).from === "object"
  );
}

// In the useEffect:
const savedLocation = isFromLocation(location.state) ? location.state.from : undefined;
```

---

## Info

### IN-01: `SUPER_USER_ROLE` magic string defined locally in a page component

**File:** `trade-flow-ui/src/pages/support/SupportDashboardPage.tsx:36`
**Issue:** `const SUPER_USER_ROLE = "super_user"` is defined at the top of the page file. This string must match the backend enum value exactly. If the value ever changes or another component needs to check for `super_user`, the constant will be duplicated or diverge. It belongs in a shared location alongside other role constants.
**Fix:** Move to a shared constants file (e.g., `src/features/support/constants.ts` or `src/config/roles.ts`) and import from there.

---

### IN-02: `console.error` in `DashboardLayout` sign-out handler

**File:** `trade-flow-ui/src/components/layouts/DashboardLayout.tsx:89`
**Issue:** `console.error("Error signing out:", error)` is a debug artifact in production code. The project convention (and the existing `toast` infrastructure) favours surfacing errors to the user rather than logging to the console.
**Fix:**
```typescript
import { toast } from "@/lib/toast";

const handleSignOut = async () => {
  try {
    await signOut(auth);
  } catch {
    toast.error("Failed to sign out. Please try again.");
  }
};
```

---

### IN-03: `SupportBusinessesPage` imports `useGetCurrentUserQuery` but only passes it to layout

**File:** `trade-flow-ui/src/pages/support/SupportBusinessesPage.tsx:10, 164-171`
**Issue:** `useGetCurrentUserQuery` is imported and called only to supply `user` and `isLoading` to `DashboardLayout`. `SupportUsersPage` follows the same pattern, so this is not unusual. However, if the page gains content-level access checks in the future (e.g., checking `canManage` like `SupportDashboardPage` does), the hook is already in place. No action required — noted for awareness only.

---

_Reviewed: 2026-04-19_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
