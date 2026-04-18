# Phase 53: Support Access & Routing - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 53-support-access-routing
**Areas discussed:** Post-login routing, Onboarding bypass, Route structure, Shell dashboard, Sidebar navigation

---

## Post-login Routing

| Option | Description | Selected |
|--------|-------------|----------|
| Smart redirect in LoginPage | After Firebase auth, fetch user data, check supportRoles, navigate to /support or /dashboard | ✓ |
| Redirect in OnboardingGuard | Keep LoginPage navigating to /dashboard, OnboardingGuard detects support role and redirects | |
| Dedicated post-login resolver route | Login navigates to /resolve-redirect, a lightweight component that checks user state | |

**User's choice:** Smart redirect in LoginPage
**Notes:** All routing logic in one place

| Option | Description | Selected |
|--------|-------------|----------|
| Always /support | Support role takes priority for dual-role users | ✓ |
| Let user choose | Show role picker after login if both roles exist | |
| Always /dashboard | Business role takes priority | |

**User's choice:** Always /support
**Notes:** supportRoles.length > 0 always wins

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, honour location state | Check location.state?.from first, fall back to role-based redirect | ✓ |
| No, always role-based | After login always redirect based on role | |

**User's choice:** Yes, honour location state
**Notes:** ProtectedRoute already stores location state when redirecting to /login

---

## Onboarding Bypass

| Option | Description | Selected |
|--------|-------------|----------|
| Check in OnboardingGuard | Add supportRoles check at top of OnboardingGuard | ✓ (initially) |
| Separate route branch | Move /support/* routes outside OnboardingGuard entirely | |
| Permission-based check | Use Phase 52 hasPermission() pattern | |

**User's choice:** Initially selected OnboardingGuard check, but superseded by Route Structure decision — support routes in separate branch means OnboardingGuard never runs for them.

| Option | Description | Selected |
|--------|-------------|----------|
| supportRoles.length > 0 | Direct array check, simple | ✓ |
| Frontend hasSupportRole() utility | Create utility function for consistency | |
| You decide | Claude picks approach | |

**User's choice:** supportRoles.length > 0 directly
**Notes:** No frontend hasPermission() utility needed for a boolean check

---

## Route Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated SupportGuard component | Move /support/* into own branch under AuthenticatedLayout, outside OnboardingGuard/PaywallGuard | ✓ |
| Keep inside guards, rely on bypasses | Leave route structure as-is, guards handle bypass internally | |
| You decide | Claude picks cleanest approach | |

**User's choice:** Dedicated SupportGuard component
**Notes:** Clean separation — each guard handles its own concern

| Option | Description | Selected |
|--------|-------------|----------|
| /dashboard | Send non-support users to main app dashboard | ✓ |
| /login | Treat as unauthenticated request | |
| 404 page | Show not-found page | |

**User's choice:** /dashboard
**Notes:** Simple and expected

| Option | Description | Selected |
|--------|-------------|----------|
| SupportGuard only, skip OnboardingGuard change | Support routes in separate branch, OnboardingGuard never runs | ✓ |
| Both changes for safety | Belt and suspenders approach | |

**User's choice:** SupportGuard only
**Notes:** No OnboardingGuard bypass needed since routes are in separate branch

---

## Shell Dashboard

| Option | Description | Selected |
|--------|-------------|----------|
| Keep existing content | Migration controls + quick links, Phase 54 adds content above | ✓ |
| Strip to minimal placeholder | Remove everything, show placeholder | |
| Redesign with empty card slots | Show empty card placeholders for Phase 54 | |

**User's choice:** Keep existing content

| Option | Description | Selected |
|--------|-------------|----------|
| Remove per-page role checks | SupportGuard handles access, checks are redundant | ✓ |
| Keep as defence-in-depth | Leave checks as safety net | |

**User's choice:** Remove them

| Option | Description | Selected |
|--------|-------------|----------|
| You decide | Claude picks approach for migration controls check | ✓ |
| Permission-based check | Use permission to gate migration controls | |
| Keep role-name check | Simple roleName === 'super_user' check | |

**User's choice:** You decide (Claude's discretion)

---

## Sidebar Navigation

| Option | Description | Selected |
|--------|-------------|----------|
| Support items only | Dashboard, Users, Businesses — hide business items | ✓ |
| Support + business items for dual-role users | Show both sections if user has both roles + business | |
| All items always | Show everything regardless of role | |

**User's choice:** Support items only
**Notes:** Business items hidden since support users don't have a business context

| Option | Description | Selected |
|--------|-------------|----------|
| Reuse DashboardLayout | Pass different nav items based on role, same layout | ✓ |
| Separate SupportLayout | Create new layout component for /support routes | |
| You decide | Claude picks based on codebase patterns | |

**User's choice:** Reuse DashboardLayout
**Notes:** Same layout (header, sidebar, content area), different menu items

## Claude's Discretion

- Super_user-only migration controls check style (permission vs role-name)
- SupportGuard implementation details (loading states, error handling)
- Navigation config refactoring approach (separate function vs inline)

## Deferred Ideas

None — discussion stayed within phase scope.
