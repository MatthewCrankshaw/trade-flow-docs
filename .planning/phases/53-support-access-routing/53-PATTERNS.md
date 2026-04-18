# Phase 53: Support Access & Routing - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 8 (1 new, 7 modified)
**Analogs found:** 8 / 8

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `src/features/auth/components/SupportGuard.tsx` | guard (NEW) | request-response | `src/features/auth/components/OnboardingGuard.tsx` | exact |
| `src/App.tsx` | route-config (MODIFY) | request-response | self (current structure) | exact |
| `src/pages/LoginPage.tsx` | page (MODIFY) | request-response | self + `src/features/auth/components/ProtectedRoute.tsx` | role-match |
| `src/config/navigation.ts` | config (MODIFY) | transform | self (current structure) | exact |
| `src/components/layouts/DashboardLayout.tsx` | component (MODIFY) | transform | self (current structure) | exact |
| `src/pages/support/SupportDashboardPage.tsx` | page (MODIFY) | request-response | self (remove guard code) | exact |
| `src/pages/support/SupportBusinessesPage.tsx` | page (MODIFY) | request-response | self (remove guard code) | exact |
| `src/pages/support/SupportUsersPage.tsx` | page (MODIFY) | request-response | self (remove guard code) | exact |

## Pattern Assignments

### `src/features/auth/components/SupportGuard.tsx` (guard, NEW)

**Analog:** `src/features/auth/components/OnboardingGuard.tsx`

**Imports pattern** (lines 1-4):
```typescript
import { Navigate, Outlet, useLocation } from "react-router-dom";
import { Loader2 } from "lucide-react";

import { useCurrentBusiness } from "@/hooks/useCurrentBusiness";
```
Note: SupportGuard will import `useGetCurrentUserQuery` from `@/services` instead of `useCurrentBusiness`, since it checks `supportRoles` rather than business state.

**Loading state pattern** (lines 20-26):
```typescript
if (isLoading) {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <Loader2 className="size-8 animate-spin text-muted-foreground" />
    </div>
  );
}
```

**Guard check + redirect pattern** (lines 32-42):
```typescript
// User has completed onboarding -- pass through
if (hasDisplayName && hasBusiness) {
  return <Outlet />;
}

// ...

// User needs onboarding -- redirect
return <Navigate to="/onboarding" replace />;
```
SupportGuard adaptation: check `user.supportRoles.length > 0`, render `<Outlet />` if true, `<Navigate to="/dashboard" replace />` if false.

**Secondary analog:** `src/features/auth/components/PaywallGuard.tsx` lines 29-33 shows how to fetch user data within a guard:
```typescript
export function PaywallGuard() {
  const { data: user } = useGetCurrentUserQuery();
  const { isActive, isLoading, subscription } = useSubscription();

  // Support role bypass (D-15)
  if ((user?.supportRoles?.length ?? 0) > 0) {
```

**Barrel export to update:** `src/features/auth/components/index.ts` -- add `export { SupportGuard } from "./SupportGuard";`

---

### `src/App.tsx` (route-config, MODIFY)

**Analog:** self

**Current support routes location** (lines 114-118) -- these must be moved:
```typescript
{/* Support Routes */}
<Route path="/support" element={<SupportDashboardPage />} />
<Route path="/support/businesses" element={<SupportBusinessesPage />} />
<Route path="/support/users" element={<SupportUsersPage />} />
```
Currently nested inside `<OnboardingGuard>` > `<PaywallGuard>` (lines 100-101). Must move to a sibling branch under `<AuthenticatedLayout>`.

**Existing guard nesting pattern** (lines 84-119):
```typescript
<Route element={<AuthenticatedLayout />}>
  {/* Onboarding route */}
  <Route element={<OnboardingGuard />}>
    <Route path="/onboarding" element={<OnboardingPage />} />
  </Route>

  {/* ... standalone routes ... */}

  {/* Onboarding + Paywall guards wrap business routes */}
  <Route element={<OnboardingGuard />}>
    <Route element={<PaywallGuard />}>
      {/* business routes here */}
    </Route>
  </Route>
</Route>
```
New SupportGuard branch goes as a sibling, following the same nesting pattern.

**Import to add** (line 9 area):
```typescript
import { OnboardingGuard, PaywallGuard, ProtectedRoute, SupportGuard } from "@/features/auth";
```

---

### `src/pages/LoginPage.tsx` (page, MODIFY)

**Analog:** self + `src/features/auth/components/ProtectedRoute.tsx`

**Current redirect logic** (lines 14-16):
```typescript
const handleSignInSuccess = () => {
  // Redirect to dashboard after successful login
  navigate("/dashboard");
};
```
Must be updated to: (1) check `location.state?.from`, (2) fetch user data, (3) route to `/support` or `/dashboard` based on `supportRoles`.

**ProtectedRoute location state pattern** (lines 20-21) -- shows how `from` is set:
```typescript
if (!user) {
  return <Navigate to="/login" state={{ from: location }} replace />;
}
```
LoginPage must honour this by checking `location.state?.from` first.

**Current already-authenticated redirect** (lines 23-25):
```typescript
if (user) {
  return <Navigate to="/dashboard" replace />;
}
```
This also needs role-aware redirect logic for the case where a user is already logged in and navigates to `/login`.

**RTK Query import pattern** from `SupportDashboardPage.tsx` line 33:
```typescript
import { useGetCurrentUserQuery } from "@/services";
```

**Additional imports needed:**
```typescript
import { useLocation } from "react-router-dom"; // add useLocation
import { useGetCurrentUserQuery } from "@/services";
```

---

### `src/config/navigation.ts` (config, MODIFY)

**Analog:** self

**Current unified function** (lines 28-121):
```typescript
export function getNavigationItems(user: User | undefined): NavSection[] {
  const sections: NavSection[] = [];

  // Main navigation - always visible
  sections.push({ title: "Main", items: [...] });

  // Business section
  sections.push({ title: "Business", items: [...] });

  // Support section - only visible if user has support roles
  if (user && user.supportRoles.length > 0) {
    sections.push({ title: "Support", items: [...] });
  }

  return sections;
}
```

Must be refactored into two functions:
1. `getBusinessNavigationItems(user)` -- returns Main + Business sections (lines 32-91)
2. `getSupportNavigationItems()` -- returns Support section only (lines 94-118)

**Existing NavItem/NavSection interfaces** (lines 16-26) -- keep unchanged:
```typescript
export interface NavItem {
  title: string;
  href: string;
  icon: React.ComponentType<{ className?: string }>;
  description?: string;
}

export interface NavSection {
  title: string;
  items: NavItem[];
}
```

**Support nav items already defined** (lines 97-115):
```typescript
{
  title: "Support",
  items: [
    { title: "Support Dashboard", href: "/support", icon: HelpCircle, ... },
    { title: "All Businesses", href: "/support/businesses", icon: Building2, ... },
    { title: "All Users", href: "/support/users", icon: Users, ... },
  ],
}
```
Per UI-SPEC from RESEARCH.md, icons change to `LayoutDashboard` for Dashboard. Titles simplify to "Dashboard", "Users", "Businesses".

---

### `src/components/layouts/DashboardLayout.tsx` (component, MODIFY)

**Analog:** self

**Current NavContent function** (lines 48-82) reads nav items from config:
```typescript
function NavContent({ user }: { user: User | undefined }) {
  const sections = getNavigationItems(user);
  // renders sections...
}
```

Must be updated to call either `getSupportNavigationItems()` or `getBusinessNavigationItems(user)` based on `user.supportRoles.length > 0`.

**Current import** (line 20):
```typescript
import { getNavigationItems } from "@/config/navigation";
```
Must change to import the two new functions.

**Logo link** (line 132) currently points to `/dashboard`:
```typescript
<Link to="/dashboard">
```
Support users should link to `/support`. Similarly mobile logo link on line 181.

---

### `src/pages/support/SupportDashboardPage.tsx` (page, MODIFY -- remove guard)

**Analog:** self

**Code to remove** -- the inline role check in `SupportDashboardContent` (lines 370-375):
```typescript
const hasSupportRole = user && user.supportRoles.length > 0;
const canManageMigrations = user && isSuperUser(user.supportRoles);

if (!hasSupportRole) {
  return <Navigate to="/dashboard" replace />;
}
```
Remove the `hasSupportRole` check and the `Navigate` redirect. Keep `canManageMigrations` for migration controls gating.

**Page wrapper pattern to change** (lines 419-427) -- currently wraps itself in DashboardLayout:
```typescript
export default function SupportDashboardPage() {
  const { data: user, isLoading } = useGetCurrentUserQuery();

  return (
    <DashboardLayout user={user} isLoading={isLoading}>
      <SupportDashboardContent />
    </DashboardLayout>
  );
}
```
After restructure, DashboardLayout is rendered by App.tsx route tree (via `<Outlet />`), so support pages should not wrap themselves in DashboardLayout. The page export becomes just the content.

---

### `src/pages/support/SupportBusinessesPage.tsx` (page, MODIFY -- remove guard)

**Analog:** self

**Code to remove** -- inline role check in `SupportBusinessesContent` (lines 57-63):
```typescript
const { data: user } = useGetCurrentUserQuery();
const hasSupportRole = user && user.supportRoles.length > 0;

if (!hasSupportRole) {
  return <Navigate to="/dashboard" replace />;
}
```

**Page wrapper to change** (lines 172-180) -- same DashboardLayout wrapping pattern as SupportDashboardPage.

---

### `src/pages/support/SupportUsersPage.tsx` (page, MODIFY -- remove guard)

**Analog:** self

**Code to remove** -- inline role check in `SupportUsersContent` (lines 57-63):
```typescript
const { data: user } = useGetCurrentUserQuery();
const hasSupportRole = user && user.supportRoles.length > 0;

if (!hasSupportRole) {
  return <Navigate to="/dashboard" replace />;
}
```

**Page wrapper to change** (lines 182-190) -- same DashboardLayout wrapping pattern.

---

## Shared Patterns

### Route Guard Pattern
**Source:** `src/features/auth/components/OnboardingGuard.tsx` (full file, 43 lines)
**Apply to:** `SupportGuard.tsx` (new file)

Guard structure:
1. Fetch user/role state via hook
2. Show `<Loader2>` spinner during loading (never render `<Outlet />` or `<Navigate />` while loading)
3. Check condition -- render `<Outlet />` if authorized
4. Render `<Navigate to="..." replace />` if unauthorized

```typescript
// Shared loading spinner pattern used in OnboardingGuard (line 21-26) and PaywallGuard (line 39-44)
<div className="flex min-h-screen items-center justify-center">
  <Loader2 className="size-8 animate-spin text-muted-foreground" />
</div>
```

### RTK Query User Data Fetching
**Source:** `src/features/auth/components/PaywallGuard.tsx` line 29
**Apply to:** `SupportGuard.tsx`, `LoginPage.tsx`

```typescript
import { useGetCurrentUserQuery } from "@/services";

const { data: user, isLoading } = useGetCurrentUserQuery();
```

### Support Role Check
**Source:** `src/features/auth/components/PaywallGuard.tsx` line 33
**Apply to:** `SupportGuard.tsx`, `DashboardLayout.tsx`, `navigation.ts`

```typescript
(user?.supportRoles?.length ?? 0) > 0
```
Or the simpler form used in support pages:
```typescript
user && user.supportRoles.length > 0
```

### DashboardLayout as Route-Level Layout
**Source:** Current App.tsx does NOT use DashboardLayout in the route tree -- each page wraps itself. Phase 53 changes this so DashboardLayout is rendered once per branch in the route tree, and pages render as children via `<Outlet />`.
**Apply to:** All three support pages must remove their `<DashboardLayout>` wrapper. App.tsx must add DashboardLayout as a route-level wrapper rendering `<Outlet />`.

**Current page pattern** (all three support pages):
```typescript
export default function SupportDashboardPage() {
  const { data: user, isLoading } = useGetCurrentUserQuery();
  return (
    <DashboardLayout user={user} isLoading={isLoading}>
      <SupportDashboardContent />
    </DashboardLayout>
  );
}
```

**Note:** DashboardLayout currently accepts `children` as a prop (line 126). To use it as a route-level layout with `<Outlet />`, it needs to either: (a) render `<Outlet />` internally when used as a route element, or (b) be wrapped in a small component that fetches user data and passes props. The implementer must decide which approach based on how business routes currently use DashboardLayout (check other pages like DashboardPage, CustomersPage).

### Barrel Export Pattern
**Source:** `src/features/auth/components/index.ts`
**Apply to:** Add `SupportGuard` export

```typescript
export { AuthForm } from "./AuthForm";
export { OnboardingGuard } from "./OnboardingGuard";
export { PaywallGuard } from "./PaywallGuard";
export { ProtectedRoute } from "./ProtectedRoute";
export { SupportGuard } from "./SupportGuard";  // NEW
```

## No Analog Found

No files lack analogs. All new/modified files have direct pattern sources in the existing codebase.

## Metadata

**Analog search scope:** `trade-flow-ui/src/`
**Files scanned:** 14 source files read in detail
**Pattern extraction date:** 2026-04-18
