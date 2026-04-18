# Phase 57: Impersonation Frontend - Pattern Map

**Mapped:** 2026-04-18
**Files analyzed:** 10 (5 new, 5 modified)
**Analogs found:** 10 / 10

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `src/store/slices/impersonationSlice.ts` | store | event-driven | `src/store/index.ts` (store config) | partial -- no existing slices |
| `src/features/support/api/impersonationApi.ts` | service | request-response | `src/features/customers/api/customerApi.ts` | exact |
| `src/features/support/hooks/useImpersonation.ts` | hook | request-response | `src/features/customers/hooks/useCustomerActions.ts` | exact |
| `src/features/support/components/ImpersonationBanner.tsx` | component | event-driven | `src/features/subscription/components/TrialBadge.tsx` | role-match |
| `src/features/support/components/ImpersonateUserDialog.tsx` | component | request-response | `src/features/estimates/components/MarkAsLostDialog.tsx` | exact |
| `src/services/api.ts` | config | request-response | (self -- modifying) | exact |
| `src/components/layouts/DashboardLayout.tsx` | component | event-driven | (self -- modifying) | exact |
| `src/config/navigation.ts` | config | transform | (self -- modifying) | exact |
| `src/features/auth/components/OnboardingGuard.tsx` | middleware | request-response | (self -- modifying) | exact |
| `src/features/auth/components/PaywallGuard.tsx` | middleware | request-response | (self -- modifying) | exact |

## Pattern Assignments

### `src/store/slices/impersonationSlice.ts` (store, event-driven)

**Analog:** No existing Redux slices in the codebase. This is the first `createSlice` usage. Follow Redux Toolkit conventions and the existing store registration pattern.

**Store registration pattern** (`src/store/index.ts` lines 1-23):
```typescript
import { configureStore } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query";

import { apiSlice } from "@/services";

export const store = configureStore({
  reducer: {
    [apiSlice.reducerPath]: apiSlice.reducer,
    // Add impersonation slice here: impersonation: impersonationSlice.reducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ["auth/setUser"],
        ignoredPaths: ["auth.user"],
      },
    }).concat(apiSlice.middleware),
  devTools: import.meta.env.DEV,
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

**Typed hooks pattern** (`src/store/hooks.ts` lines 1-6):
```typescript
import { useDispatch, useSelector } from "react-redux";
import type { RootState, AppDispatch } from "./index";

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

---

### `src/features/support/api/impersonationApi.ts` (service, request-response)

**Analog:** `src/features/customers/api/customerApi.ts`

**Imports pattern** (lines 1-3):
```typescript
import { apiSlice } from "@/services/api";

import type { Customer, CreateCustomerRequest, UpdateCustomerRequest, StandardResponse } from "@/types";
```

**RTK Query injectEndpoints pattern** (lines 5-57):
```typescript
export const customerApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    createCustomer: builder.mutation<Customer, { businessId: string; data: CreateCustomerRequest }>({
      query: ({ businessId, data }) => ({
        url: `/v1/business/${businessId}/customer`,
        method: "POST",
        body: data,
      }),
      transformResponse: (response: StandardResponse<Customer>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No customer data returned");
      },
      invalidatesTags: [{ type: "Customer", id: "LIST" }, "User"],
    }),
  }),
});

export const { useCreateCustomerMutation } = customerApi;
```

**Key adaptation:** Impersonation endpoints use `builder.mutation` for both start and end. No cache tag invalidation needed since `resetApiState()` handles full cache clearing.

---

### `src/features/support/hooks/useImpersonation.ts` (hook, request-response)

**Analog:** `src/features/customers/hooks/useCustomerActions.ts`

**Imports pattern** (lines 1-6):
```typescript
import { useCallback } from "react";

import { useCreateCustomerMutation, useUpdateCustomerMutation } from "@/services";
import { toast } from "@/lib/toast";

import type { Customer, CreateCustomerRequest, UpdateCustomerRequest } from "@/types";
```

**Hook structure with mutation + toast + error handling** (lines 51-119):
```typescript
export function useCustomerActions({ businessId }: UseCustomerActionsOptions): UseCustomerActionsReturn {
  const [createMutation, { isLoading: isCreating }] = useCreateCustomerMutation();

  const createCustomer = useCallback(
    async (data: CreateCustomerRequest) => {
      try {
        const result = await createMutation({ businessId, data }).unwrap();
        return result;
      } catch (error) {
        console.error("Failed to create customer:", error);
        toast.error("Failed to create customer", {
          description: "Please try again.",
        });
        throw error;
      }
    },
    [businessId, createMutation],
  );

  return {
    createCustomer,
    isCreating,
    isLoading: isCreating,
  };
}
```

**Key adaptation:** Hook wraps `useStartImpersonationMutation` and `useEndImpersonationMutation`. Also dispatches `startImpersonation`/`endImpersonation` Redux actions and `apiSlice.util.resetApiState()`. Uses `useNavigate` for post-action redirect. Uses `useAppDispatch` and `useAppSelector` from `@/store/hooks`.

**Cache reset pattern** (`src/providers/auth-provider.tsx` lines 27-29):
```typescript
dispatch(apiSlice.util.resetApiState());
```

---

### `src/features/support/components/ImpersonationBanner.tsx` (component, event-driven)

**Analog:** `src/features/subscription/components/TrialBadge.tsx`

**Imports pattern** (lines 1-9):
```typescript
import { Clock } from "lucide-react";
import { toast } from "sonner";

import { Badge } from "@/components/ui/badge";
import { cn } from "@/lib/utils";

import { useCreatePortalSessionMutation } from "../api/subscriptionApi";
import { useSubscription } from "../hooks/useSubscription";
```

**Conditional render + Redux state + action handler** (lines 21-71):
```typescript
export function TrialBadge() {
  const { isLoading, subscription } = useSubscription();
  const [createPortal] = useCreatePortalSessionMutation();

  if (isLoading || subscription?.status !== "trialing") {
    return null;
  }

  async function handleClick() {
    try {
      const result = await createPortal().unwrap();
      window.open(result.url, "_blank", "noopener,noreferrer");
    } catch {
      toast.error("Unable to open billing portal. Please try again.");
    }
  }

  return (
    <button type="button" onClick={handleClick} aria-label="...">
      <Badge variant="secondary" className={cn("cursor-pointer", urgencyClasses)}>
        <Clock className="h-3.5 w-3.5 mr-1" />
        {desktopText}
      </Badge>
    </button>
  );
}
```

**Key adaptation:** Banner uses `useAppSelector` to read `state.impersonation` instead of a custom hook. Uses `fixed top-0 left-0 right-0 z-[60]` positioning instead of inline badge. Renders `null` when `active` is `false`. Calls `endSession` from `useImpersonation` hook on button click.

---

### `src/features/support/components/ImpersonateUserDialog.tsx` (component, request-response)

**Analog:** `src/features/estimates/components/MarkAsLostDialog.tsx`

**Imports pattern** (lines 1-7):
```typescript
import { useState } from "react";
import { Loader2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Textarea } from "@/components/ui/textarea";
import { useMarkEstimateLostMutation } from "../api/estimateApi";
import { toast } from "@/lib/toast";
```

**Dialog with split body pattern** (lines 19-33):
```typescript
interface MarkAsLostDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  estimateId: string;
}

export function MarkAsLostDialog({ open, onOpenChange, estimateId }: MarkAsLostDialogProps) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-md">
        {open && <MarkAsLostDialogBody estimateId={estimateId} onOpenChange={onOpenChange} />}
      </DialogContent>
    </Dialog>
  );
}
```

**Dialog body with state + mutation + submit handler** (lines 40-118):
```typescript
function MarkAsLostDialogBody({ estimateId, onOpenChange }: MarkAsLostDialogBodyProps) {
  const [notesText, setNotesText] = useState("");
  const [markLost, { isLoading }] = useMarkEstimateLostMutation();

  const handleSubmit = async () => {
    try {
      await markLost({
        estimateId,
        reason: selectedReason ?? undefined,
        notes: notesText || undefined,
      }).unwrap();
      toast.success("Estimate marked as lost");
      onOpenChange(false);
    } catch {
      toast.error("Failed to mark estimate as lost. Please try again.");
    }
  };

  return (
    <>
      <DialogHeader>
        <DialogTitle>Mark estimate as lost?</DialogTitle>
        <DialogDescription>This will lock the estimate and cancel any pending follow-ups.</DialogDescription>
      </DialogHeader>

      <div className="space-y-4">
        <div className="space-y-2">
          <span className="text-sm font-semibold">Anything else? (optional)</span>
          <Textarea
            placeholder="Add a note if you'd like"
            value={notesText}
            onChange={(e) => setNotesText(e.target.value.slice(0, MAX_NOTES_LENGTH))}
            maxLength={MAX_NOTES_LENGTH}
            rows={3}
          />
        </div>
      </div>

      <DialogFooter>
        <Button type="button" variant="outline" onClick={() => onOpenChange(false)} disabled={isLoading}>
          Keep Estimate
        </Button>
        <Button type="button" variant="destructive" onClick={handleSubmit} disabled={isLoading}>
          {isLoading ? (
            <>
              <Loader2 className="mr-1 h-4 w-4 animate-spin" />
              Marking...
            </>
          ) : (
            "Mark as Lost"
          )}
        </Button>
      </DialogFooter>
    </>
  );
}
```

**Key adaptation:** Reason `Textarea` is required (not optional). Submit calls `startImpersonation` from `useImpersonation` hook. Props include target user info (id, name, email). Confirm button uses `variant="default"` not `"destructive"`. Contextual cancel label (e.g., "Go Back" not "Cancel").

---

### `src/services/api.ts` (config, request-response) -- MODIFIED

**Current file** (lines 1-36):
```typescript
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

import { auth } from "@/config/firebase";

const baseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:3000",
  prepareHeaders: async (headers) => {
    const user = auth.currentUser;
    if (user) {
      const token = await user.getIdToken();
      headers.set("Authorization", `Bearer ${token}`);
    }
    return headers;
  },
});

export const apiSlice = createApi({
  reducerPath: "api",
  baseQuery,
  tagTypes: [/* ... */],
  endpoints: () => ({}),
});
```

**Modifications needed:**
1. Add `getState` destructured param to `prepareHeaders` callback (RTK Query passes `{ getState }` as second arg)
2. Check `state.impersonation.active` and `state.impersonation.token` before Firebase fallback
3. Wrap `baseQuery` in a custom function that intercepts 401 during active impersonation, dispatches `endImpersonation()` and `apiSlice.util.resetApiState()`
4. Import `RootState` from `@/store`, `endImpersonation` from slice, types from RTK Query

---

### `src/components/layouts/DashboardLayout.tsx` (component, event-driven) -- MODIFIED

**Current header z-index** (line 153):
```typescript
<header className="sticky top-0 z-20 flex h-16 items-center justify-between border-b bg-card px-4 lg:px-6">
```

**Current NavContent** (line 49):
```typescript
function NavContent({ user }: { user: User | undefined }) {
  const sections = getNavigationItems(user);
```

**Modifications needed:**
1. Import and render `ImpersonationBanner` above the outer `<div>`, or add top padding to the outer container when impersonation is active
2. Pass impersonation state to `getNavigationItems` (or the function reads it internally)
3. Use `useAppSelector` to read `state.impersonation.active` for conditional padding

---

### `src/config/navigation.ts` (config, transform) -- MODIFIED

**Current function signature** (line 28):
```typescript
export function getNavigationItems(user: User | undefined): NavSection[] {
```

**Support section conditional** (lines 94-118):
```typescript
if (user && user.supportRoles.length > 0) {
  sections.push({
    title: "Support",
    items: [
      { title: "Support Dashboard", href: "/support", icon: HelpCircle },
      { title: "All Businesses", href: "/support/businesses", icon: Building2 },
      { title: "All Users", href: "/support/users", icon: Users },
    ],
  });
}
```

**Modifications needed:**
1. Add `isImpersonating: boolean` parameter
2. When `isImpersonating` is true, return only Main + Business sections (customer nav), omit Support section

---

### `src/features/auth/components/OnboardingGuard.tsx` (middleware) -- MODIFIED

**Current guard logic** (lines 16-43):
```typescript
export function OnboardingGuard() {
  const { user, hasBusiness, isLoading } = useCurrentBusiness();
  const location = useLocation();

  if (isLoading) { /* spinner */ }

  const hasDisplayName = Boolean(user?.name?.trim());
  const isOnboardingPath = location.pathname.startsWith("/onboarding");

  if (hasDisplayName && hasBusiness) {
    return <Outlet />;
  }
  if (isOnboardingPath) {
    return <Outlet />;
  }
  return <Navigate to="/onboarding" replace />;
}
```

**Modification needed:** Add impersonation bypass as first check after loading:
```typescript
const isImpersonating = useAppSelector((state: RootState) => state.impersonation.active);
if (isImpersonating) {
  return <Outlet />;
}
```

---

### `src/features/auth/components/PaywallGuard.tsx` (middleware) -- MODIFIED

**Current support role bypass** (lines 28-34):
```typescript
export function PaywallGuard() {
  const { data: user } = useGetCurrentUserQuery();
  const { isActive, isLoading, subscription } = useSubscription();

  if ((user?.supportRoles?.length ?? 0) > 0) {
    return <Outlet />;
  }
```

**Modification needed:** Add impersonation bypass alongside or before the support role bypass:
```typescript
const isImpersonating = useAppSelector((state: RootState) => state.impersonation.active);
if (isImpersonating) {
  return <Outlet />;
}
```

---

## Shared Patterns

### Toast Notifications
**Source:** `src/lib/toast.tsx`
**Apply to:** `useImpersonation.ts`, `ImpersonateUserDialog.tsx`
```typescript
import { toast } from "@/lib/toast";

// Success
toast.success("Impersonation session ended");

// Error
toast.error("Failed to start impersonation", {
  description: "Please try again.",
});

// Warning (for expired session)
toast.warning("Impersonation session expired");
```

### Redux Typed Hooks
**Source:** `src/store/hooks.ts`
**Apply to:** All new components and hooks that access Redux state
```typescript
import { useAppDispatch, useAppSelector } from "@/store/hooks";

const dispatch = useAppDispatch();
const isImpersonating = useAppSelector((state) => state.impersonation.active);
```

### Cache Reset Pattern
**Source:** `src/providers/auth-provider.tsx` (lines 27-29)
**Apply to:** `useImpersonation.ts` (on start and end)
```typescript
import { apiSlice } from "@/services";

dispatch(apiSlice.util.resetApiState());
```

### Loading Button Pattern
**Source:** `src/features/estimates/components/MarkAsLostDialog.tsx` (lines 106-114)
**Apply to:** `ImpersonateUserDialog.tsx` confirm button, `ImpersonationBanner.tsx` return button
```typescript
<Button type="button" onClick={handleSubmit} disabled={isLoading}>
  {isLoading ? (
    <>
      <Loader2 className="mr-1 h-4 w-4 animate-spin" />
      Starting...
    </>
  ) : (
    "Start Impersonation"
  )}
</Button>
```

### Dialog Wrapper Pattern
**Source:** `src/features/estimates/components/MarkAsLostDialog.tsx` (lines 25-33)
**Apply to:** `ImpersonateUserDialog.tsx`
```typescript
export function SomeDialog({ open, onOpenChange, ...rest }: SomeDialogProps) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-md">
        {open && <SomeDialogBody {...rest} onOpenChange={onOpenChange} />}
      </DialogContent>
    </Dialog>
  );
}
```

### Guard Bypass Pattern
**Source:** `src/features/auth/components/PaywallGuard.tsx` (lines 32-34)
**Apply to:** Both `OnboardingGuard.tsx` and `PaywallGuard.tsx`
```typescript
// Existing bypass pattern -- support role check
if ((user?.supportRoles?.length ?? 0) > 0) {
  return <Outlet />;
}
// Add impersonation bypass with same structure
```

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `src/store/slices/impersonationSlice.ts` | store | event-driven | No existing `createSlice` usage in the codebase. This is the first Redux slice. Follow Redux Toolkit `createSlice` conventions from RESEARCH.md Pattern 1. Store registration follows `src/store/index.ts`. |

## Metadata

**Analog search scope:** `trade-flow-ui/src/` (features, services, store, components, config, providers)
**Files scanned:** 15 analog candidates examined
**Pattern extraction date:** 2026-04-18
