---
phase: 57-impersonation-frontend
reviewed: 2026-04-19T12:00:00Z
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
  critical: 0
  warning: 1
  info: 3
  total: 4
status: issues_found
---

# Phase 57: Code Review Report — Impersonation Frontend (Re-review)

**Reviewed:** 2026-04-19
**Depth:** standard
**Files Reviewed:** 14
**Status:** issues_found

## Summary

This is a re-review after the first-round fixes (57-REVIEW-FIX.md, iteration 1). All six previously identified issues (2 Critical, 4 Warning) have been verified as resolved in the current source. The impersonation architecture is sound: token injection in `prepareHeaders`, 401 auto-expiry with proper `apiSlice.util.resetApiState()` usage, correct dispatch-before-navigate ordering in `endSession`, `try/catch/finally` for safe client-state cleanup on API failure, server-authoritative `targetUser` from the API response, and properly initialised `wasImpersonatingRef`. Guard bypasses in `OnboardingGuard` and `PaywallGuard` are well-ordered with appropriate comments. Tests cover all key scenarios including impersonation bypass paths.

One new warning was identified regarding mobile layout during impersonation, plus three low-priority info items.

## Warnings

### WR-01: Sticky header overlaps fixed impersonation banner on mobile when user scrolls

**File:** `trade-flow-ui/src/components/layouts/DashboardLayout.tsx:152,179`
**Issue:** When impersonation is active, the outer container receives `pt-10` to push content below the fixed banner (`fixed top-0 z-[60] h-10`). On mobile (below the `lg:` breakpoint), the outer `div.flex.min-h-screen` is the scroll container. The header at line 179 has `sticky top-0 z-20`. Initially the header sits below the banner due to padding. However, as the user scrolls down, the sticky header resolves to `top-0` of the viewport, sliding behind the fixed banner (which has higher z-index). This means the header becomes invisible on mobile during impersonation once the user scrolls.

On desktop (`lg:h-screen` on the inner wrapper), the main content scrolls inside `main.overflow-y-auto` and the header stays within the flex column, so this only affects mobile.

**Fix:** Conditionally offset the sticky header top position when impersonating:
```tsx
<header
  className={cn(
    "sticky z-20 flex h-16 items-center justify-between border-b bg-card px-4 lg:px-6",
    isImpersonating ? "top-10" : "top-0",
  )}
>
```

## Info

### IN-01: `getNavigationItems` function is defined but never imported

**File:** `trade-flow-ui/src/config/navigation.ts:26-37`
**Issue:** The `getNavigationItems` function encapsulates the impersonation/support/business navigation selection logic, but no file imports it. `DashboardLayout.tsx` imports `getBusinessNavigationItems` and `getSupportNavigationItems` directly and duplicates the selection logic inline (lines 55-59). This is dead code that adds maintenance burden -- changes to the selection logic must be made in two places.

**Fix:** Either remove `getNavigationItems` from `navigation.ts` (if the inline pattern in `DashboardLayout` is preferred), or refactor `DashboardLayout.NavContent` to use `getNavigationItems(user, isImpersonating)` and remove the inline conditional.

### IN-02: `PaywallGuard.derivePaywallVariant` uses `new Date()` for parsing API date strings

**File:** `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx:19-21`
**Issue:** Per project CLAUDE.md: "Never use `new Date()` for parsing API date strings -- always use Luxon." The function uses `new Date(subscription.trialEnd).getTime()` and `new Date(subscription.canceledAt).getTime()`. This is a pre-existing convention violation that was present before this phase but remains in a file modified by this phase.

**Fix:**
```ts
import { DateTime } from "luxon";

const trialEnd = subscription.trialEnd ? DateTime.fromISO(subscription.trialEnd).toMillis() : 0;
const canceledAt = subscription.canceledAt ? DateTime.fromISO(subscription.canceledAt).toMillis() : 0;
```

### IN-03: Reason field validation allows very short strings with no audit value

**File:** `trade-flow-ui/src/features/support/components/ImpersonateUserDialog.tsx:47-49`
**Issue:** The validation `if (!trimmedReason)` blocks empty and whitespace-only strings, but a single character like `"x"` passes. For an audit-sensitive feature like impersonation, a minimum length (e.g., 10 characters) would improve the quality of the audit trail. The backend is the authoritative enforcement point, but frontend guidance helps support users provide meaningful reasons.

**Fix:** Consider a minimum length:
```ts
const MIN_REASON_LENGTH = 10;
if (trimmedReason.length < MIN_REASON_LENGTH) {
  setShowValidation(true);
  return;
}
```

---

_Reviewed: 2026-04-19_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
