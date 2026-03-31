# Phase 33: Trial Banner and Billing Settings Tab - Research

**Researched:** 2026-03-29
**Domain:** React frontend -- subscription UI components, tab routing, date calculation
**Confidence:** HIGH

## Summary

Phase 33 is a frontend-only phase adding two UI features to `trade-flow-ui`: (1) a TrialChip in the DashboardLayout header for trialing users showing days remaining, and (2) a Billing tab on the Settings page with subscription status card and management CTAs.

The foundation is solid. Phase 32 delivered `useSubscription()` which already exposes the full `subscription` object (including `status`, `trialEnd`, `currentPeriodEnd`, `cancelAtPeriodEnd`) via `SubscriptionContext`. The shadcn Tabs, Badge, Card, Button, and Skeleton components are all installed. The `subscriptionApi` RTK Query endpoint for `getSubscription` exists but `createPortalSession` does NOT -- it must be added. `useSearchParams` from react-router-dom is already used in the codebase (SubscribeSuccessPage).

**Primary recommendation:** This phase requires three distinct deliverables: (1) add `createPortalSession` mutation to `subscriptionApi` + a `usePortalSession` hook, (2) build `TrialChip` and wire it into `DashboardLayout` header, (3) refactor `SettingsPage` to use shadcn Tabs with `?tab=billing` query param routing and build `SubscriptionStatusCard` + `BillingTab` components.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: TrialChip is a small chip/badge in the app header (inside DashboardLayout), NOT a full-width banner. Near user avatar.
- D-02: Visible on ALL authenticated pages inside DashboardLayout while status is trialing. Not dismissible.
- D-03: Hidden when status is active, past_due, canceled, or no subscription record. Only shown for trialing.
- D-04: Chip is clickable -- navigates to /settings?tab=billing.
- D-05: Copy format: "X days left in trial".
- D-06: When 0 days remaining: "Last day of trial".
- D-07: If days remaining is negative (trialEnd passed but status still trialing): hide chip entirely.
- D-08: Days remaining calculated from trialEnd date. Round down (floor).
- D-09: Billing tab via ?tab=billing query param. Matches Phase 31 D-05 portal return_url.
- D-10: Settings page reads tab query param on mount, pre-selects Billing tab if ?tab=billing.
- D-11: Active subscription -> "Manage Billing" CTA -> Stripe Portal. No subscription/past_due/canceled -> "Start Subscription" CTA -> /subscribe.
- D-12: No in-app Cancel button. Cancellation via Stripe Portal only.
- D-13: cancelAtPeriodEnd=true shows "Cancels on [date]" status label. Manage Billing CTA remains.
- D-14: Manage Billing calls POST /v1/subscription/portal -> redirect to portalUrl.
- D-15: Loading state on Manage Billing button (Loader2 spinner, disabled).
- D-16: Skeleton for status card while getSubscription is loading.

### Claude's Discretion
- Exact chip styling (color, size, typography) -- subtle but noticeable
- Date formatting (e.g., "Jan 15" vs "15 Jan 2026")
- Skeleton shape for Billing tab loading state
- Tab component pattern for Settings page
- Component name: TrialChip or TrialBadge or similar

### Deferred Ideas (OUT OF SCOPE)
- None
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TRIAL-01 | Persistent banner for trialing users showing days remaining and Subscribe CTA | TrialChip component in DashboardLayout header; uses `useSubscription()` which already provides `subscription.trialEnd` and `subscription.status`; days calculation via `Math.floor((trialEnd - now) / 86400000)` |
| TRIAL-02 | Trial banner not shown when subscription status is active | TrialChip visibility gated on `subscription?.status === "trialing"` only; all other statuses result in hidden chip |
| BILL-04 | Settings > Billing tab with subscription status, dates, and CTAs | SubscriptionStatusCard using existing Card/Button/Skeleton shadcn components; new `createPortalSession` mutation in `subscriptionApi`; Settings page refactored to use shadcn Tabs with `?tab=billing` routing |
| BILL-05 | cancelAtPeriodEnd=true shows "Cancels on [date]" | CTA matrix in SubscriptionStatusCard: when `subscription.cancelAtPeriodEnd === true`, display "Cancels on [formatted date]" instead of "Active -- renews [date]" |
</phase_requirements>

## Standard Stack

### Core (Already Installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| react | 19.2.0 | UI framework | Project standard |
| react-router-dom | 7.13.0 | Routing + useSearchParams | Already used in SubscribeSuccessPage |
| @reduxjs/toolkit | 2.11.2 | RTK Query for subscriptionApi | Phase 32 established pattern |
| sonner | 2.0.7 | Toast notifications | Used in useCheckout for error toasts |
| lucide-react | 0.563.0 | Icons (Clock, CreditCard, ExternalLink, Loader2) | Project icon library |

### shadcn UI Components (Already Installed)
| Component | File | Purpose |
|-----------|------|---------|
| Tabs, TabsList, TabsTrigger, TabsContent | `src/components/ui/tabs.tsx` | Settings page tab navigation |
| Badge | `src/components/ui/badge.tsx` | Status badges on SubscriptionStatusCard (custom colors override variants) |
| Card, CardHeader, CardTitle, CardContent | `src/components/ui/card.tsx` | SubscriptionStatusCard container |
| Button | `src/components/ui/button.tsx` | CTAs (Manage Billing, Start Subscription) |
| Skeleton | `src/components/ui/skeleton.tsx` | Loading state for SubscriptionStatusCard |

### No New Dependencies
No new npm packages required. All needed components and libraries are already installed.

## Architecture Patterns

### New Component Structure
```
src/features/subscription/
  api/subscriptionApi.ts          # ADD createPortalSession mutation
  hooks/useSubscription.ts        # UNCHANGED -- already exposes subscription object
  hooks/useCheckout.ts            # UNCHANGED -- reusable pattern reference
  hooks/usePortalSession.ts       # NEW -- mirrors useCheckout for portal redirect
  components/TrialChip.tsx        # NEW -- header chip for trialing users
  components/SubscriptionStatusCard.tsx  # NEW -- billing tab status display
  components/BillingTab.tsx       # NEW -- composition wrapper for billing tab content
  index.ts                        # UPDATE -- export new components and hook
```

### Pattern 1: usePortalSession Hook (mirrors useCheckout)
**What:** A hook wrapping the `createPortalSession` RTK Query mutation with redirect logic and error handling.
**When to use:** In the SubscriptionStatusCard "Manage Billing" button.
**Example:**
```typescript
// Follows exact pattern of useCheckout.ts
import { toast } from "sonner";
import { useCreatePortalSessionMutation } from "../api/subscriptionApi";

export function usePortalSession() {
  const [createPortal, { isLoading }] = useCreatePortalSessionMutation();

  const openPortal = async () => {
    try {
      const result = await createPortal().unwrap();
      window.location.href = result.portalUrl;
    } catch {
      toast.error("Unable to open billing portal. Please try again.");
    }
  };

  return { openPortal, isLoading };
}
```

### Pattern 2: Tab Routing with useSearchParams
**What:** Read `?tab=` query param to set active tab, sync tab changes back to URL.
**When to use:** SettingsPage refactoring.
**Example:**
```typescript
// useSearchParams already used in SubscribeSuccessPage.tsx
import { useSearchParams } from "react-router-dom";

function SettingsContent() {
  const [searchParams, setSearchParams] = useSearchParams();
  const activeTab = searchParams.get("tab") || "general";

  const handleTabChange = (value: string) => {
    setSearchParams(value === "general" ? {} : { tab: value });
  };

  return (
    <Tabs value={activeTab} onValueChange={handleTabChange}>
      <TabsList>
        <TabsTrigger value="general">General</TabsTrigger>
        <TabsTrigger value="billing">Billing</TabsTrigger>
      </TabsList>
      <TabsContent value="general">...</TabsContent>
      <TabsContent value="billing"><BillingTab /></TabsContent>
    </Tabs>
  );
}
```

### Pattern 3: Days Remaining Calculation
**What:** Calculate whole days between now and `trialEnd` ISO string, floor to integer.
**When to use:** TrialChip and SubscriptionStatusCard trial state.
**Example:**
```typescript
function calculateDaysRemaining(trialEnd: string | undefined): number | null {
  if (!trialEnd) return null;
  const now = new Date();
  const end = new Date(trialEnd);
  const diffMs = end.getTime() - now.getTime();
  return Math.floor(diffMs / (1000 * 60 * 60 * 24));
}
```

### Pattern 4: Date Formatting (UK Locale)
**What:** Format dates as "15 Jan 2026" per UI-SPEC contract.
**When to use:** SubscriptionStatusCard date displays.
**Example:**
```typescript
function formatBillingDate(isoDate: string): string {
  return new Intl.DateTimeFormat("en-GB", {
    day: "numeric",
    month: "short",
    year: "numeric",
  }).format(new Date(isoDate));
}
```

### Pattern 5: TrialChip Placement in DashboardLayout
**What:** TrialChip renders in the header right actions area, before PersistentCta.
**When to use:** DashboardLayout modification.
**Key insight:** The "Header Right Actions" div (line 214 of DashboardLayout.tsx) already has `flex items-center gap-2`. TrialChip slots in naturally as a sibling of PersistentCta. The TrialChip shows for trialing users while PersistentCta shows for non-active users -- they are mutually exclusive (trialing = active, so PersistentCta hides).

### Anti-Patterns to Avoid
- **Don't create a separate subscription fetch in TrialChip:** Use `useSubscription()` from context -- the subscription data is already fetched once by `SubscriptionProvider` and available everywhere.
- **Don't use Badge component variants for status colors:** The existing badgeVariants only has default/secondary/destructive/outline. Status badges need custom amber/green/orange/muted colors. Use `className` overrides on plain divs or Badge with custom classes, not new variant definitions.
- **Don't auto-redirect from Billing tab after portal session:** Use `window.location.href` for portal redirect (same as checkout). The portal will redirect back to `/settings?tab=billing` via the return_url configured in Phase 31.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Tab routing | Custom state + URL sync | shadcn Tabs + useSearchParams | Radix Tabs handles keyboard navigation, ARIA, active state |
| Date formatting | Manual string concatenation | Intl.DateTimeFormat("en-GB") | Handles locale, month names, ordinals correctly |
| Portal redirect | Custom fetch + redirect logic | RTK Query mutation + usePortalSession hook | Consistent with useCheckout pattern, handles loading/error |
| Subscription state | Local fetch in each component | useSubscription() context hook | Single source of truth, already fetched by SubscriptionProvider |

## Common Pitfalls

### Pitfall 1: Tab State Not Preserved on Portal Return
**What goes wrong:** User clicks "Manage Billing" -> goes to Stripe Portal -> returns to `/settings?tab=billing` but tab resets to default.
**Why it happens:** If tab state is managed in React state only (not URL), page reload loses it.
**How to avoid:** Tab state MUST be driven by `useSearchParams` reading from the URL. The `?tab=billing` param is preserved across the full-page redirect to/from Stripe Portal.
**Warning signs:** Tab defaults to "General" after returning from Stripe Portal.

### Pitfall 2: TrialChip Flash on Initial Load
**What goes wrong:** TrialChip briefly appears then disappears for active users, or briefly shows wrong day count.
**Why it happens:** Subscription data hasn't loaded yet.
**How to avoid:** TrialChip returns `null` when `isLoading` is true (D-16 in UI-SPEC). Only render once subscription data is confirmed.
**Warning signs:** Brief amber chip flash on page load for active subscribers.

### Pitfall 3: Negative Days Display
**What goes wrong:** Shows "-1 days left in trial" when webhook hasn't updated status after trial expiry.
**Why it happens:** `trialEnd` has passed but status is still `trialing` due to webhook processing delay.
**How to avoid:** Per D-07, if `calculateDaysRemaining(trialEnd) < 0`, hide the chip entirely.
**Warning signs:** Negative numbers in trial chip text.

### Pitfall 4: createPortalSession Response Shape
**What goes wrong:** Mutation expects `{ portalUrl }` but API returns standard `{ data: [{ portalUrl }] }` wrapper.
**Why it happens:** Forgetting the `StandardResponse<T>` wrapper used by all API endpoints.
**How to avoid:** Use `transformResponse` in the RTK Query mutation to unwrap `response.data[0]`, same as `createCheckoutSession`.
**Warning signs:** `undefined` portalUrl after mutation succeeds.

### Pitfall 5: Settings Page Profile Content Wrapped in Wrong Tab
**What goes wrong:** Existing Settings page content disappears or doesn't render when tabs are added.
**Why it happens:** Existing `SettingsContent` function has the page header + profile card. When wrapping in Tabs, the page header needs to stay outside the tab content.
**How to avoid:** Move the "Settings" heading and subtitle ABOVE the Tabs component. Only the profile card goes inside TabsContent value="general".
**Warning signs:** "Settings" heading duplicated per tab, or missing entirely.

## Code Examples

### createPortalSession Mutation (to add to subscriptionApi.ts)
```typescript
// Follows createCheckoutSession pattern exactly
createPortalSession: builder.mutation<{ portalUrl: string }, void>({
  query: () => ({
    url: "/v1/subscription/portal",
    method: "POST",
  }),
  transformResponse: (response: StandardResponse<{ portalUrl: string }>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No portal URL returned");
  },
}),
```

### TrialChip Component Shape
```typescript
// Located at: src/features/subscription/components/TrialChip.tsx
import { Clock } from "lucide-react";
import { useNavigate } from "react-router-dom";
import { useSubscription } from "../hooks/useSubscription";

export function TrialChip() {
  const { subscription, isLoading } = useSubscription();
  const navigate = useNavigate();

  if (isLoading || subscription?.status !== "trialing") return null;

  const daysRemaining = calculateDaysRemaining(subscription.trialEnd);
  if (daysRemaining === null || daysRemaining < 0) return null;

  const label = daysRemaining === 0
    ? "Last day of trial"
    : `${daysRemaining} days left in trial`;

  return (
    <button
      onClick={() => navigate("/settings?tab=billing")}
      className="flex items-center gap-1 rounded-full bg-amber-100 px-2 py-0.5 text-xs font-medium text-amber-800 transition-colors hover:bg-amber-200"
      aria-label={`${label}, view billing settings`}
    >
      <Clock className="h-3 w-3 text-amber-600" />
      {label}
    </button>
  );
}
```

### SubscriptionStatusCard CTA Matrix
```typescript
// Status -> display mapping
function getStatusDisplay(subscription: Subscription | null) {
  if (!subscription || subscription.status === "past_due" || subscription.status === "canceled") {
    return { badge: "Inactive", badgeClass: "bg-muted text-muted-foreground", line: "You don't have an active subscription.", cta: "subscribe" };
  }
  if (subscription.status === "trialing") {
    const days = calculateDaysRemaining(subscription.trialEnd);
    return { badge: "Trial", badgeClass: "bg-amber-100 text-amber-800", line: `Trial -- ${days} days remaining`, cta: "portal" };
  }
  if (subscription.cancelAtPeriodEnd) {
    const date = formatBillingDate(subscription.currentPeriodEnd!);
    return { badge: `Cancels on ${date}`, badgeClass: "bg-orange-100 text-orange-800", line: `Cancels on ${date}`, cta: "portal" };
  }
  const date = formatBillingDate(subscription.currentPeriodEnd!);
  return { badge: "Active", badgeClass: "bg-green-100 text-green-800", line: `Active -- renews ${date}`, cta: "portal" };
}
```

## Existing Codebase Facts (Verified)

| Fact | Status | Evidence |
|------|--------|----------|
| `useSubscription()` exposes `subscription` object | Confirmed | SubscriptionContext.ts line 8: `subscription: Subscription \| null` |
| `Subscription` type has `trialEnd`, `currentPeriodEnd`, `cancelAtPeriodEnd` | Confirmed | subscription.ts lines 14-16 |
| `createPortalSession` endpoint does NOT exist in subscriptionApi | Confirmed | subscriptionApi.ts only has getSubscription, createCheckoutSession, verifySession |
| shadcn Tabs component installed | Confirmed | `src/components/ui/tabs.tsx` with Radix TabsPrimitive |
| shadcn Badge component installed | Confirmed | `src/components/ui/badge.tsx` with CVA variants |
| Badge has 4 variants only (default, secondary, destructive, outline) | Confirmed | badge.variants.ts -- no amber/green/orange variants |
| Settings page has NO tabs currently | Confirmed | SettingsPage.tsx is a flat layout with profile card only |
| DashboardLayout header has right actions area with PersistentCta | Confirmed | DashboardLayout.tsx line 214-231 |
| TrialChip and PersistentCta are mutually exclusive | Confirmed | PersistentCta hides when `isActive=true`; trialing = active per SubscriptionProvider line 25 |
| `useSearchParams` pattern already in codebase | Confirmed | SubscribeSuccessPage.tsx line 2 |
| `useNavigate` pattern already in codebase | Confirmed | 5 files use useNavigate |
| `toast` from sonner used for error states | Confirmed | useCheckout.ts uses `toast.error()` |
| `window.location.href` used for Stripe redirects | Confirmed | useCheckout.ts line 11 |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | ESLint + TypeScript (frontend has no test runner) |
| Config file | `eslint.config.js`, `tsconfig.app.json` |
| Quick run command | `npm run lint && npm run typecheck` |
| Full suite command | `npm run lint && npm run typecheck && npm run build` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| TRIAL-01 | TrialChip shows days remaining for trialing users | manual | Visual verification in browser | N/A |
| TRIAL-02 | TrialChip hidden when status is active | manual | Visual verification in browser | N/A |
| BILL-04 | Billing tab with status, dates, CTAs | manual | Visual verification in browser | N/A |
| BILL-05 | cancelAtPeriodEnd shows "Cancels on [date]" | manual | Visual verification in browser | N/A |

### Sampling Rate
- **Per task commit:** `npm run lint && npm run typecheck`
- **Per wave merge:** `npm run lint && npm run typecheck && npm run build`
- **Phase gate:** Full build green before `/gsd:verify-work`

### Wave 0 Gaps
None -- frontend has static analysis only (ESLint, TypeScript). No test framework to configure.

## Sources

### Primary (HIGH confidence)
- Codebase inspection: `trade-flow-ui/src/features/subscription/` -- all Phase 32 subscription infrastructure
- Codebase inspection: `trade-flow-ui/src/pages/SettingsPage.tsx` -- current Settings page structure
- Codebase inspection: `trade-flow-ui/src/components/layouts/DashboardLayout.tsx` -- header layout
- Codebase inspection: `trade-flow-ui/src/components/ui/tabs.tsx` -- shadcn Tabs component
- Codebase inspection: `trade-flow-ui/src/components/ui/badge.tsx` + `badge.variants.ts` -- Badge component

### Secondary (MEDIUM confidence)
- Phase 32 CONTEXT.md -- useSubscription hook shape, subscriptionApi pattern, portal redirect pattern
- Phase 33 CONTEXT.md -- all locked decisions
- Phase 33 UI-SPEC.md -- visual and interaction contract

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all components verified installed via codebase inspection
- Architecture: HIGH - extends existing Phase 32 patterns with zero new dependencies
- Pitfalls: HIGH - derived from actual code inspection (portal response shape, tab routing, flash prevention)

**Research date:** 2026-03-29
**Valid until:** 2026-04-28 (stable -- no external library research needed)
