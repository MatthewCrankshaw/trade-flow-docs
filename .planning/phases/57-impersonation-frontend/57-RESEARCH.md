# Phase 57: Impersonation Frontend - Research

**Researched:** 2026-04-18
**Domain:** React frontend impersonation UX -- token switching, persistent banner, session lifecycle
**Confidence:** HIGH

## Summary

Phase 57 delivers the frontend impersonation experience for Trade Flow's support tool. A support user navigates to a customer's detail page (Phase 54), clicks "Impersonate User", enters a reason, and receives an impersonation token from Phase 56's `POST /v1/impersonation/start`. The app then switches to using that token for all API requests, shows the customer's data and navigation, and displays a fixed amber banner with a "Return to Support" button. Session termination calls `POST /v1/impersonation/end`, clears Redux state, resets the RTK Query cache, and navigates to `/support`.

The implementation touches six existing files and creates four new ones. The core technical challenge is modifying RTK Query's `prepareHeaders` to conditionally read from Redux state (impersonation token) instead of Firebase auth. RTK Query's `prepareHeaders` callback receives a second argument with `getState()` which provides direct access to the Redux store -- this is the standard, documented approach for token switching. [VERIFIED: Context7 /reduxjs/redux-toolkit `prepareHeaders` docs]

**Primary recommendation:** Create an `impersonationSlice` in the Redux store, modify `prepareHeaders` to check it, wrap `baseQuery` to detect expired impersonation tokens, and add the `ImpersonationBanner` above `DashboardLayout`'s header.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Replace the Bearer token during impersonation -- store the impersonation token in Redux state and use it instead of the Firebase token in RTK Query's `prepareHeaders`. When impersonation is active, `prepareHeaders` reads from the impersonation slice; otherwise, it uses `auth.currentUser.getIdToken()` as normal.
- **D-02:** Dedicated `impersonationSlice` in the Redux store holding: impersonation token, target user info (name, email, ID), session ID, and active flag. No sessionStorage persistence.
- **D-03:** Page refresh ends impersonation -- Redux state is lost, support user returns to their own dashboard via normal Firebase auth. No persistence complexity.
- **D-04:** Fixed banner at the very top of the viewport, above the DashboardLayout header. Position fixed, z-50 (above header's z-20). Cannot be scrolled away, dismissed, or covered by modals/dropdowns.
- **D-05:** Warning amber visual style -- amber/yellow background with dark text.
- **D-06:** Banner content: warning icon, "Impersonating: [user name]", and a "Return to Support" button. Compact single-line bar.
- **D-07:** During impersonation, the sidebar shows the customer's business navigation -- not the support navigation.
- **D-08:** "Impersonate" button on the user detail page only (from Phase 54). No quick-action in the users list table.
- **D-09:** Confirmation dialog with a required reason text field before starting impersonation.
- **D-10:** The "Impersonate" button is hidden entirely for support users who lack the `impersonate_user` permission.
- **D-11:** Cannot impersonate other support users -- the button should not appear when viewing a support user's detail page.
- **D-12:** "Return to Support" calls `POST /v1/impersonation/end` to record the end timestamp, then clears Redux impersonation state and resets the RTK Query cache.
- **D-13:** After ending impersonation, navigate to `/support` (support dashboard).
- **D-14:** Expired session handling: when the backend rejects the impersonation token (expired), show a toast, clear impersonation state, and navigate back to `/support`. No modal required.

### Claude's Discretion
- RTK Query cache reset strategy during impersonation start/end (full reset vs selective tag invalidation)
- Exact impersonation banner component structure and Tailwind classes
- How to detect expired tokens in RTK Query (baseQuery error interceptor vs per-endpoint handling)
- Whether to show a loading state during impersonation start (while calling backend + switching token)
- Mobile responsive behavior of the impersonation banner

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| IMP-03 | During impersonation, the app renders exactly what the customer sees (same data, same subscription state, same permissions) | Token switching in `prepareHeaders` + route guard bypass + navigation switching ensures customer-identical experience |
| IMP-04 | A fixed impersonation banner is visible at all times during an impersonation session showing the impersonated user's name and a "Return to Support" button | `ImpersonationBanner` component with `fixed top-0 z-[60]` positioning above all UI layers |
| IMP-05 | Support user can terminate the impersonation session and return to their support dashboard cleanly | `POST /v1/impersonation/end` + Redux state clear + `resetApiState()` + navigate to `/support` |

</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Token switching | Frontend (Redux + RTK Query) | -- | `prepareHeaders` reads impersonation token from Redux store; API treats it as any other Bearer token |
| Impersonation banner | Frontend (React component) | -- | Pure UI concern -- fixed position overlay above app layout |
| Session start (API call) | API / Backend | Frontend (trigger) | Backend creates session + signs JWT; frontend stores result in Redux |
| Session termination | API / Backend | Frontend (trigger) | Backend records end timestamp; frontend clears state + resets cache |
| Expired token detection | Frontend (baseQuery wrapper) | API / Backend | Backend returns 401 on expired token; frontend wrapper intercepts and cleans up |
| Route guard bypass | Frontend (React Router guards) | -- | `OnboardingGuard` and `PaywallGuard` must pass through during impersonation |
| Navigation switching | Frontend (navigation config) | -- | `getNavigationItems()` returns customer nav when impersonation is active |

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @reduxjs/toolkit | 2.11.2 | Impersonation Redux slice + RTK Query mutations | Already installed, `createSlice` + `injectEndpoints` are project standard [VERIFIED: trade-flow-ui/package.json] |
| react-redux | 9.2.0 | `useAppSelector` / `useAppDispatch` for impersonation state | Already installed, typed hooks in `src/store/hooks.ts` [VERIFIED: codebase] |
| react-router-dom | 7.13.0 | `useNavigate` for post-impersonation redirect | Already installed, used throughout [VERIFIED: codebase] |
| lucide-react | 0.563.0 | `AlertTriangle`, `UserCheck` icons for banner and button | Already installed [VERIFIED: codebase] |
| sonner | 2.0.7 | Toast notifications for session ended/expired/error | Already installed, used via `@/lib/toast` wrapper [VERIFIED: codebase] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Radix AlertDialog | Already installed | Confirmation dialog with reason field | Impersonation start flow |
| Radix Label | Already installed | Reason field label | Dialog form |
| shadcn Textarea | Already installed | Reason text input | Dialog form |
| shadcn Button | Already installed | Action buttons | Banner + dialog + user detail page |

No new packages need to be installed.

## Architecture Patterns

### System Architecture Diagram

```
Support User clicks "Impersonate"
        |
        v
[ImpersonateUserDialog] -- reason field + confirm
        |
        v
[POST /v1/impersonation/start] -- { targetUserId, reason }
        |
        v
Backend returns { token, sessionId, targetUser }
        |
        v
[impersonationSlice] -- stores token, targetUser, sessionId, active=true
        |
        v
[apiSlice.util.resetApiState()] -- clears all cached data
        |
        v
[Navigate to "/"] -- customer dashboard
        |
        v
[prepareHeaders] reads impersonation token from Redux state
        |
        v
All API calls use impersonation token --> Backend resolves as target user
        |
        v
[ImpersonationBanner] visible (fixed, z-[60], amber)
[getNavigationItems] returns customer nav
[OnboardingGuard / PaywallGuard] bypass when impersonation active
        |
        v
Support User clicks "Return to Support"
        |
        v
[POST /v1/impersonation/end] -- { sessionId }
        |
        v
[impersonationSlice] -- cleared
[apiSlice.util.resetApiState()] -- clears impersonated data
[Navigate to "/support"]
[Toast: "Impersonation session ended"]
```

### Recommended Project Structure

```
src/
├── store/
│   └── slices/
│       └── impersonationSlice.ts           # NEW: Redux slice for impersonation state
├── features/
│   └── support/
│       ├── components/
│       │   ├── ImpersonationBanner.tsx      # NEW: Fixed amber banner
│       │   └── ImpersonateUserDialog.tsx    # NEW: Confirmation dialog with reason
│       ├── hooks/
│       │   └── useImpersonation.ts          # NEW: Start/end impersonation logic
│       └── api/
│           └── impersonationApi.ts          # NEW: RTK Query endpoints for start/end
├── services/
│   └── api.ts                              # MODIFIED: prepareHeaders + baseQuery wrapper
├── components/
│   └── layouts/
│       └── DashboardLayout.tsx             # MODIFIED: Render banner + padding
├── config/
│   └── navigation.ts                       # MODIFIED: Impersonation-aware nav
└── App.tsx                                 # MODIFIED: Guard bypass during impersonation
```

### Pattern 1: Redux Slice for Impersonation State (D-02)

**What:** A dedicated `impersonationSlice` holding the impersonation session state.
**When to use:** Whenever checking impersonation status from any component or middleware.
**Example:**
```typescript
// Source: Follows existing project Redux patterns + D-02 decisions
import { createSlice } from "@reduxjs/toolkit";
import type { PayloadAction } from "@reduxjs/toolkit";

interface ImpersonationState {
  active: boolean;
  token: string | null;
  sessionId: string | null;
  targetUser: {
    id: string;
    name: string;
    email: string;
  } | null;
}

const initialState: ImpersonationState = {
  active: false,
  token: null,
  sessionId: null,
  targetUser: null,
};

export const impersonationSlice = createSlice({
  name: "impersonation",
  initialState,
  reducers: {
    startImpersonation(
      state,
      action: PayloadAction<{
        token: string;
        sessionId: string;
        targetUser: { id: string; name: string; email: string };
      }>,
    ) {
      state.active = true;
      state.token = action.payload.token;
      state.sessionId = action.payload.sessionId;
      state.targetUser = action.payload.targetUser;
    },
    endImpersonation(state) {
      state.active = false;
      state.token = null;
      state.sessionId = null;
      state.targetUser = null;
    },
  },
});

export const { startImpersonation, endImpersonation } = impersonationSlice.actions;
```

### Pattern 2: prepareHeaders Token Switching (D-01)

**What:** Modify `prepareHeaders` in `api.ts` to read impersonation token from Redux state before falling back to Firebase.
**When to use:** Every API request -- the switch is automatic based on Redux state.
**Example:**
```typescript
// Source: RTK Query prepareHeaders docs (Context7 /reduxjs/redux-toolkit)
// + existing api.ts pattern
import type { RootState } from "@/store";

const baseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:3000",
  prepareHeaders: async (headers, { getState }) => {
    const state = getState() as RootState;

    if (state.impersonation.active && state.impersonation.token) {
      headers.set("Authorization", `Bearer ${state.impersonation.token}`);
      return headers;
    }

    const user = auth.currentUser;
    if (user) {
      const token = await user.getIdToken();
      headers.set("Authorization", `Bearer ${token}`);
    }
    return headers;
  },
});
```

### Pattern 3: baseQuery Wrapper for Expired Token Detection (D-14)

**What:** Wrap `baseQuery` to intercept 401 responses during active impersonation and trigger cleanup.
**When to use:** Automatically on every API response when impersonation is active.
**Example:**
```typescript
// Source: RTK Query customizing-queries docs (Context7 /reduxjs/redux-toolkit)
import type { BaseQueryFn, FetchArgs, FetchBaseQueryError } from "@reduxjs/toolkit/query";

const baseQueryWithImpersonationExpiry: BaseQueryFn<
  string | FetchArgs,
  unknown,
  FetchBaseQueryError
> = async (args, api, extraOptions) => {
  const result = await rawBaseQuery(args, api, extraOptions);
  const state = api.getState() as RootState;

  if (
    result.error &&
    result.error.status === 401 &&
    state.impersonation.active
  ) {
    api.dispatch(endImpersonation());
    api.dispatch(apiSlice.util.resetApiState());
    // Navigation handled via listener or component-level effect
  }

  return result;
};
```

**Navigation note:** The baseQuery wrapper cannot call `useNavigate()` since it is not a React component. The recommended approach is to have a React component (e.g., `ImpersonationBanner` or a dedicated `ImpersonationSessionGuard`) that watches `impersonation.active` via `useAppSelector` and navigates when it transitions from `true` to `false` + shows the toast.

### Pattern 4: Route Guard Bypass During Impersonation

**What:** `OnboardingGuard` and `PaywallGuard` must let requests through during active impersonation, since the support user does not have onboarding or subscription context.
**When to use:** Both guards check impersonation state before their normal logic.
**Example:**
```typescript
// In OnboardingGuard and PaywallGuard, add early return:
const isImpersonating = useAppSelector((state: RootState) => state.impersonation.active);
if (isImpersonating) {
  return <Outlet />;
}
```

### Anti-Patterns to Avoid
- **Persisting impersonation state to sessionStorage/localStorage:** Violates D-03 and creates security risk. Redux-only means page refresh cleanly terminates the session.
- **Using a separate API slice for impersonation requests:** All requests during impersonation should go through the same `apiSlice` with the switched token -- no parallel API configuration.
- **Selective cache invalidation on impersonation start/end:** Use `apiSlice.util.resetApiState()` (full reset) not selective tag invalidation. Data belongs to different users -- stale data from one user leaking to another is a security issue.
- **Navigating from inside baseQuery wrapper directly:** baseQuery is not a React component. Use a React effect that watches impersonation state changes for navigation + toasts.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Token switching in API calls | Custom fetch wrapper | RTK Query `prepareHeaders` with `getState()` | Built-in support, documented pattern, type-safe [VERIFIED: Context7] |
| Confirmation dialog | Custom modal | Radix `AlertDialog` (already installed) | Accessibility (focus trap, Escape key), existing project pattern |
| Toast notifications | Custom notification system | Sonner via `@/lib/toast` | Already integrated, consistent with rest of app |
| Redux state management | React Context for impersonation | `createSlice` from Redux Toolkit | Needs to be accessible in `prepareHeaders` via `getState()` -- Context is not accessible there |
| Cache reset | Manual cache clearing per endpoint | `apiSlice.util.resetApiState()` | Used in AuthProvider already for user switching [VERIFIED: auth-provider.tsx] |

**Key insight:** Redux is the correct choice (not React Context) because the impersonation token must be readable from RTK Query's `prepareHeaders` callback, which has access to Redux state via `getState()` but not React Context.

## Common Pitfalls

### Pitfall 1: Stale Cache Data Leaking Between Users
**What goes wrong:** After starting impersonation, the support user sees their own cached data (customers, jobs) instead of the target customer's data. After ending, they see the customer's data in their support dashboard.
**Why it happens:** RTK Query aggressively caches responses by endpoint + params. Switching the token does not automatically invalidate cached data.
**How to avoid:** Call `apiSlice.util.resetApiState()` both when starting AND ending impersonation. This is already the pattern used in `AuthProvider` when Firebase user changes.
**Warning signs:** Support user sees wrong business name, wrong customer list, or wrong subscription status during/after impersonation.

### Pitfall 2: OnboardingGuard Redirecting to /onboarding During Impersonation
**What goes wrong:** Support user starts impersonation, navigates to customer dashboard, but `OnboardingGuard` checks the support user's profile (no business, no display name for the impersonated context) and redirects to `/onboarding`.
**Why it happens:** `OnboardingGuard` uses `useCurrentBusiness()` which queries `GET /v1/user/me`. After token switching, this returns the customer's data, but the guard fires before the cache is populated with the new user's data. Or if the cache was just reset, data is temporarily undefined.
**How to avoid:** Add impersonation bypass in `OnboardingGuard` -- check `state.impersonation.active` and return `<Outlet />` immediately.
**Warning signs:** Support user lands on onboarding page after starting impersonation.

### Pitfall 3: Firebase Auth State Change Triggered by Impersonation
**What goes wrong:** The `AuthProvider`'s `onAuthStateChanged` listener detects a "change" and calls `resetApiState()` at the wrong time, fighting with the impersonation flow.
**Why it happens:** Impersonation does NOT change Firebase auth state -- the support user remains logged in to Firebase. However, if any code inadvertently calls Firebase sign-out or sign-in during the impersonation flow, it would trigger the AuthProvider's cache reset.
**How to avoid:** Impersonation only touches Redux state and the `Authorization` header. Never call Firebase auth methods during impersonation start/end.
**Warning signs:** Double cache resets, user appears logged out unexpectedly.

### Pitfall 4: Navigation from baseQuery Wrapper
**What goes wrong:** Expired token detection in the baseQuery wrapper tries to call `navigate("/support")` but fails because `useNavigate()` is a React hook and cannot be used outside components.
**Why it happens:** The baseQuery wrapper is a plain function, not a React component or hook.
**How to avoid:** Use a React component or effect to watch `impersonation.active` state. When it transitions from `true` to `false`, navigate to `/support` and show the toast. The baseQuery wrapper only dispatches `endImpersonation()` and `resetApiState()`.
**Warning signs:** Console errors about hooks used outside components, or navigation not working after token expiry.

### Pitfall 5: Banner z-index Conflict with Radix Overlays
**What goes wrong:** Radix AlertDialog or DropdownMenu overlays cover the impersonation banner, making it impossible to click "Return to Support".
**Why it happens:** Radix portals to `document.body` and uses z-50 by default for overlay backdrops.
**How to avoid:** UI-SPEC specifies `z-[60]` for the banner to sit above Radix overlays. The banner's "Return to Support" button must remain clickable even when a modal is open.
**Warning signs:** Banner disappears behind a dimmed overlay.

## Code Examples

### RTK Query Impersonation Endpoints
```typescript
// Source: Existing RTK Query patterns in trade-flow-ui/src/services/userApi.ts
export const impersonationApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    startImpersonation: builder.mutation<
      { token: string; sessionId: string; targetUser: { id: string; name: string; email: string } },
      { targetUserId: string; reason: string }
    >({
      query: (body) => ({
        url: "/v1/impersonation/start",
        method: "POST",
        body,
      }),
      transformResponse: (response: StandardResponse</* response type */>) => {
        // Extract from standard response wrapper
      },
    }),
    endImpersonation: builder.mutation<void, { sessionId: string }>({
      query: (body) => ({
        url: "/v1/impersonation/end",
        method: "POST",
        body,
      }),
    }),
  }),
});
```

### Impersonation-Aware Navigation
```typescript
// Source: Existing getNavigationItems in trade-flow-ui/src/config/navigation.ts
// Modified to accept impersonation state
export function getNavigationItems(
  user: User | undefined,
  isImpersonating: boolean,
): NavSection[] {
  // During impersonation, show only customer navigation (Main + Business)
  // Do not show Support section -- the banner is the only escape hatch
  if (isImpersonating) {
    return [
      { title: "Main", items: [/* Dashboard, Jobs, Customers, Quotes, Estimates, Items */] },
      { title: "Business", items: [/* My Business, Settings */] },
    ];
  }

  // Normal flow -- existing logic
  // ...
}
```

### ImpersonationBanner Component Structure
```typescript
// Source: UI-SPEC Phase 57, section "Impersonation Banner"
// Uses: AlertTriangle from lucide-react, Button from shadcn
function ImpersonationBanner() {
  const { targetUser, active } = useAppSelector((state) => state.impersonation);
  const { endSession, isEnding } = useImpersonation();

  if (!active || !targetUser) return null;

  return (
    <div
      role="alert"
      aria-live="assertive"
      className="fixed top-0 left-0 right-0 z-[60] flex h-10 items-center justify-between bg-amber-500 px-4"
    >
      <div className="flex items-center gap-2">
        <AlertTriangle className="h-4 w-4 text-amber-950" />
        <span className="text-sm font-medium text-amber-950 truncate">
          Impersonating: {targetUser.name}
        </span>
      </div>
      <button
        onClick={endSession}
        disabled={isEnding}
        aria-label="End impersonation and return to support dashboard"
        className="bg-amber-950 text-amber-50 hover:bg-amber-900 text-sm font-medium h-7 px-3 rounded-md"
      >
        {isEnding ? "Ending..." : "Return to Support"}
      </button>
    </div>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Custom fetch wrappers for auth | RTK Query `prepareHeaders` with `getState()` | RTK Query 1.0+ (2021) | Standard pattern for token injection from Redux state [VERIFIED: Context7] |
| Manual API cache clearing | `apiSlice.util.resetApiState()` | RTK Query 1.0+ | Already used in this codebase in AuthProvider [VERIFIED: auth-provider.tsx] |
| React Context for cross-cutting state | Redux slice when middleware/non-component access needed | N/A | Context cannot be accessed from RTK Query's prepareHeaders |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 56 backend returns impersonation token + session ID + target user info from `POST /v1/impersonation/start` | Architecture Patterns | Frontend cannot populate impersonation slice without these fields; would need to separately fetch user info |
| A2 | Phase 56 backend returns 401 status code when impersonation token expires | Common Pitfalls | Expired token detection in baseQuery wrapper would not trigger; need to check for different status code |
| A3 | Phase 54 creates a `SupportUserDetailPage` at `/support/users/:userId` where the Impersonate button will be placed | Architecture Patterns | If user detail page doesn't exist yet, the impersonate entry point has no host component |
| A4 | The `User` type from `@/types/api.types.ts` has a `permissions` array or the permission check is done via role-based logic from Phase 52 | Code Examples | Cannot conditionally show/hide Impersonate button based on `impersonate_user` permission without this |

**Note on A1:** Phase 56 CONTEXT.md (D-10) confirms `POST /v1/impersonation/start` returns an impersonation JWT token, but the exact response shape (whether target user info is included or must be fetched separately) is at Claude's discretion in Phase 56. The frontend should handle either case. [CITED: Phase 56 CONTEXT.md D-10]

**Note on A2:** Phase 56 CONTEXT.md (D-06) confirms expired JWT returns 401 with a message. [CITED: Phase 56 CONTEXT.md D-06]

## Open Questions (RESOLVED)

1. **Phase 56 response shape for `POST /v1/impersonation/start`** (RESOLVED)
   - What we know: Returns an impersonation JWT token (D-10). Session includes `targetUserId`, `supportUserId`, `sessionId` (D-02, D-08).
   - What's unclear: Whether the response also includes the target user's name/email, or if the frontend must already have this from the user detail page.
   - Resolution: Frontend already has target user name/email from the SupportUserDetailPage context. The `useImpersonation.startSession()` accepts name and email as parameters and passes them directly into the `startImpersonation` Redux action alongside the API response token/sessionId. No extra API call needed.

2. **Permission checking mechanism on frontend** (RESOLVED)
   - What we know: Phase 52 builds `@RequiresPermission` and `hasPermission()` on the backend. Phase 51 hydrates permissions onto user DTOs.
   - What's unclear: Whether the frontend `User` type includes a `permissions` array or just `supportRoles`, and how to check `impersonate_user` permission client-side.
   - Resolution: The executor should read the User type shape at implementation time and use whichever mechanism is available (permissions array or role-based check). Plan 03 documents both approaches with fallback logic.

3. **Support routes currently inside OnboardingGuard + PaywallGuard** (RESOLVED)
   - What we know: In `App.tsx`, support routes (`/support/*`) are nested inside `OnboardingGuard` > `PaywallGuard`. Phase 53 may restructure this.
   - What's unclear: Whether Phase 53 moves support routes outside these guards or adds a `SupportGuard`.
   - Resolution: Plan 02 adds impersonation bypass checks in both OnboardingGuard and PaywallGuard. These bypasses are needed regardless of whether Phase 53 restructures the route hierarchy, since the support user must traverse these guards to reach business routes during impersonation.

## Project Constraints (from CLAUDE.md)

- Use `@/` path alias for all imports from `src/`
- Import types with `import type { ... }`
- Functional components only with TypeScript interfaces for props
- Use `useAppDispatch` and `useAppSelector` from `@/store/hooks` (not raw React Redux hooks)
- Use `cn()` from `@/lib/utils` for conditional Tailwind classes
- Toast via `@/lib/toast` wrapper (project convention, though some files import directly from `sonner`)
- Mobile-first responsive design
- Self-documenting code over comments
- No commented-out code
- Always run `npm run lint` and `npm run typecheck` before completing tasks
- `npm run ci` must pass (tests + lint + format + typecheck)
- No `eslint-disable`, `@ts-ignore`, `@ts-expect-error` to bypass checks
- Prefer type guards/validation over `as` casts (user feedback)
- Separation over DRY at entity boundaries (user feedback)

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest 4.1.3 |
| Config file | `vite.config.ts` (merged via `mergeConfig`) |
| Quick run command | `npm run test` |
| Full suite command | `npm run ci` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| IMP-03 | `prepareHeaders` uses impersonation token when active | unit | `npx vitest run src/services/__tests__/api.test.ts -t "impersonation" --reporter=verbose` | Wave 0 |
| IMP-03 | `impersonationSlice` correctly stores/clears state | unit | `npx vitest run src/store/slices/__tests__/impersonationSlice.test.ts --reporter=verbose` | Wave 0 |
| IMP-04 | `ImpersonationBanner` renders when active, hidden when not | unit | `npx vitest run src/features/support/components/__tests__/ImpersonationBanner.test.tsx --reporter=verbose` | Wave 0 |
| IMP-05 | `useImpersonation` hook handles start/end lifecycle | unit | `npx vitest run src/features/support/hooks/__tests__/useImpersonation.test.ts --reporter=verbose` | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test`
- **Per wave merge:** `npm run ci`
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `src/store/slices/__tests__/impersonationSlice.test.ts` -- covers IMP-03 (Redux state)
- [ ] `src/features/support/components/__tests__/ImpersonationBanner.test.tsx` -- covers IMP-04
- [ ] `src/features/support/hooks/__tests__/useImpersonation.test.ts` -- covers IMP-05

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | Impersonation token replaces Firebase token -- must not be persisted (D-03), no localStorage/sessionStorage |
| V3 Session Management | yes | 30-min expiry enforced server-side (Phase 56); frontend clears state on expiry detection |
| V4 Access Control | yes | Button visibility gated by `impersonate_user` permission (D-10); cannot impersonate support users (D-11) |
| V5 Input Validation | yes | Reason field required, minimum length validated by backend (Phase 56 D-12) |
| V6 Cryptography | no | JWT signing handled by backend (Phase 56) |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Impersonation token persisted in browser storage | Information Disclosure | Redux-only state, no persistence (D-03). Page refresh terminates session. |
| Support user retains customer data after session ends | Information Disclosure | `apiSlice.util.resetApiState()` clears all cached responses on session end |
| UI shows Impersonate button to unauthorized users | Elevation of Privilege | Permission check hides button entirely (D-10); backend also enforces (403) |
| Expired token used for continued access | Elevation of Privilege | baseQuery wrapper detects 401, dispatches cleanup; server rejects expired tokens |

## Sources

### Primary (HIGH confidence)
- Context7 /reduxjs/redux-toolkit -- `prepareHeaders` signature with `getState()`, baseQuery wrapper pattern for re-auth
- `trade-flow-ui/src/services/api.ts` -- Current `prepareHeaders` implementation (Firebase-only)
- `trade-flow-ui/src/providers/auth-provider.tsx` -- Existing `resetApiState()` pattern
- `trade-flow-ui/src/store/index.ts` -- Current store configuration
- `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` -- Layout structure, z-index baseline (z-20)
- `trade-flow-ui/src/config/navigation.ts` -- `getNavigationItems()` with role-conditional support section
- `trade-flow-ui/src/App.tsx` -- Route structure with guard nesting
- `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx` -- Guard implementation
- `trade-flow-ui/src/features/auth/components/PaywallGuard.tsx` -- Guard implementation with support bypass

### Secondary (MEDIUM confidence)
- Phase 56 CONTEXT.md -- Backend API contract (D-10 `POST /v1/impersonation/start`, D-11 `POST /v1/impersonation/end`, D-06 expiry returns 401)
- Phase 57 CONTEXT.md -- All locked decisions (D-01 through D-14)
- Phase 57 UI-SPEC -- Visual design contract (banner classes, dialog copy, z-index, accessibility)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and verified in codebase
- Architecture: HIGH -- patterns verified against RTK Query docs and existing codebase patterns
- Pitfalls: HIGH -- derived from actual codebase analysis (guard logic, cache behavior, z-index layering)

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable -- all dependencies are already locked in the project)
