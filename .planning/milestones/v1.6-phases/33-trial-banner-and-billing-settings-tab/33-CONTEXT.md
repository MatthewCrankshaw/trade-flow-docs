# Phase 33: Trial Banner and Billing Settings Tab - Context

**Gathered:** 2026-03-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Frontend-only phase (`trade-flow-ui`). Deliver two UI additions on top of the Phase 32 subscription foundation:

1. **TrialChip** — A small persistent chip in the app header (inside `DashboardLayout`) showing days remaining for trialing users. Non-dismissible. Hidden when subscription status is `active`, `past_due`, `canceled`, or not `trialing`.

2. **Settings > Billing tab** — A new tab on the Settings page (`?tab=billing`) displaying subscription status and providing management CTAs. Any authenticated user can reach this tab regardless of subscription status.

Phase is frontend-only. Backend endpoints (`GET /v1/subscription`, `POST /v1/subscription/portal`) were built in Phases 31 and 32.

</domain>

<decisions>
## Implementation Decisions

### TrialChip — Placement and Appearance
- **D-01:** The component is a small chip/badge rendered in the app header (inside `DashboardLayout`) — NOT a full-width banner strip. Appears near the user avatar.
- **D-02:** Visible on ALL authenticated pages inside `DashboardLayout` while status is `trialing`. Not dismissible — always visible.
- **D-03:** Hidden entirely when status is `active`, `past_due`, `canceled`, or when there is no subscription record. Only shown for `trialing`.
- **D-04:** Chip is clickable → navigates to `Settings > Billing tab` (`/settings?tab=billing`).

### TrialChip — Days Remaining Display
- **D-05:** Copy format: `"X days left in trial"` (e.g., "7 days left in trial", "3 days left in trial").
- **D-06:** When trial ends today (0 days remaining after rounding down): show `"Last day of trial"` instead of "0 days left in trial".
- **D-07:** Edge case — if calculated days remaining is negative (trialEnd has passed but status is still `trialing` due to webhook lag): **hide the chip entirely**. Do not show a negative or "0 days" value.
- **D-08:** Days remaining is calculated from `trialEnd` date (from `GET /v1/subscription` response). Round down to whole days (floor).

### Settings > Billing Tab — Tab Routing
- **D-09:** The Billing tab is accessed via `?tab=billing` query param on the Settings page. This matches the portal `return_url` established in Phase 31 D-05 (`${FRONTEND_URL}/settings?tab=billing`).
- **D-10:** Settings page reads the `tab` query param on mount and pre-selects the Billing tab if `?tab=billing` is present. If no param, show the default tab.

### Settings > Billing Tab — CTA Matrix
State-driven CTAs based on subscription status:

- **trialing:** Status display: `"Trial — X days remaining"` | CTA: `[Manage Billing]` → Stripe Portal
- **active (no cancelAtPeriodEnd):** Status display: `"Active — renews [currentPeriodEnd date]"` | CTA: `[Manage Billing]` → Stripe Portal
- **active + cancelAtPeriodEnd=true:** Status display: `"Cancels on [currentPeriodEnd date]"` | CTA: `[Manage Billing]` → Stripe Portal
- **past_due / canceled / no subscription:** Status display: `"Inactive"` | CTA: `[Subscribe]` → navigates to `/subscribe`

- **D-11:** Once any subscription record exists (including `trialing`), the primary action is always `[Manage Billing]` → Stripe Portal (`POST /v1/subscription/portal` → redirect to `portalUrl`). The Subscribe CTA only appears when there is no subscription or it's `past_due`/`canceled`.
- **D-12:** No separate in-app Cancel button. Cancellation is handled entirely through the Stripe Billing Portal. `DELETE /v1/subscription` is not triggered from this UI.

### Cancel State Display (BILL-05)
- **D-13:** When `cancelAtPeriodEnd=true`, the status label changes from "Active — renews [date]" to `"Cancels on [date]"`. The `[Manage Billing]` CTA remains. No additional cancel warning UI needed — the status label is sufficient.

### Manage Billing CTA Flow
- **D-14:** Clicking `[Manage Billing]` calls `POST /v1/subscription/portal` → receives `{ portalUrl }` → `window.location.href = portalUrl`. Same pattern as the checkout redirect in Phase 32. No Stripe.js needed.
- **D-15:** Loading state while portal URL is being fetched (brief). Button shows loading indicator to prevent double-click.

### Loading State on Billing Tab
- **D-16:** While `getSubscription` is loading, show a skeleton for the status card. `useSubscription()` from Phase 32 already provides `isLoading`.

### Claude's Discretion
- Exact chip styling (color, size, typography) — should feel subtle but noticeable in the header
- Date formatting for `currentPeriodEnd` and `trialEnd` (e.g., "Jan 15" vs "15 Jan 2026")
- Skeleton shape for the Billing tab loading state
- Tab component pattern for Settings page (if not already tab-based, implement standard tabs pattern)
- Component name: `TrialChip` or `TrialBadge` or similar

</decisions>

<specifics>
## Specific Ideas

- The TrialChip should be near the user avatar in the header — small, non-intrusive, but clearly visible. Think "badge" or "pill" rather than a notification bar.
- The Billing tab `?tab=billing` param was locked in Phase 31 as the Stripe Portal return URL — this routing MUST work correctly so users land on the Billing tab after managing their subscription in the Portal.
- Phase 32 D-12 established outcome-driven copy: avoid generic "Subscribe" language. "Manage Billing" and "Cancels on [date]" fit this principle.

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.6 Requirements
- `.planning/REQUIREMENTS.md` — Phase 33 covers TRIAL-01, TRIAL-02, BILL-04, BILL-05

### Roadmap
- `.planning/ROADMAP.md` §Phase 33 — Phase goal, success criteria (SC-1 through SC-4), dependency on Phase 32

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/phases/32-subscription-gate-and-subscribe-pages/32-CONTEXT.md` — `useSubscription()` hook shape (D-08/D-09), `subscriptionApi` RTK Query setup (D-06/D-07), soft-gate model, copy philosophy (D-12), TrialBanner deferred from Phase 32 (D-14)
- `.planning/phases/31-subscription-api-endpoints-and-tests/31-CONTEXT.md` — Portal return URL = `settings?tab=billing` (D-05), `DELETE /v1/subscription` cancel semantics (D-01 through D-04)
- `.planning/phases/29-subscription-module-foundation/29-CONTEXT.md` — Subscription entity fields: `status`, `currentPeriodEnd`, `trialEnd`, `cancelAtPeriodEnd`, `SubscriptionStatus` enum values

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — React naming patterns, hook conventions, RTK Query endpoint structure
- `.planning/codebase/ARCHITECTURE.md` — Frontend feature module structure, RTK Query patterns

### Existing Frontend Patterns
- `trade-flow-ui/src/features/subscription/` — Phase 32 subscription feature module (`subscriptionApi`, `useSubscription`, `PaywallModal`) — this phase extends it
- `trade-flow-ui/src/pages/SettingsPage.tsx` — Settings page to add Billing tab to
- `trade-flow-ui/src/components/layout/DashboardLayout.tsx` — Where TrialChip renders (near user avatar in header)
- `trade-flow-ui/src/services/api.ts` — `apiSlice` base; `subscriptionApi` already injects here (Phase 32)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `useSubscription()` (Phase 32): returns `{ isActive, isLoading, openPaywall }` — Phase 33 may need to extend or complement this with `subscription` data object (status, trialEnd, currentPeriodEnd, cancelAtPeriodEnd) for the Billing tab display
- `subscriptionApi.getSubscription`: RTK Query endpoint returning full subscription record (or null for 404) — already built in Phase 32
- `useGetCurrentUserQuery()`: returns `User` with `supportRoles` — billing tab may use this for support user display
- `DashboardLayout`: header area where TrialChip renders; Phase 32 already added persistent CTA layer here

### Established Patterns
- RTK Query: `injectEndpoints` on `apiSlice`, `transformResponse` for `StandardResponse<T>`
- Feature modules: `src/features/subscription/` with `api/`, `components/`, `hooks/`, `index.ts` barrel — new components go here
- Settings page tab pattern: check if `SettingsPage.tsx` already uses tabs; if not, implement consistent tab routing via `?tab=` query param
- Stripe Portal redirect: `window.location.href = portalUrl` (same as checkout redirect in Phase 32 D-18)

### Integration Points
- `DashboardLayout` header — add TrialChip near user avatar
- `SettingsPage.tsx` — add Billing tab, wire up `?tab=billing` query param routing
- `subscriptionApi` — add `createPortalSession` endpoint (`POST /v1/subscription/portal`) if not already present from Phase 32
- `useSubscription()` hook — may need to expose raw `subscription` data object (not just `isActive`) for Billing tab display

</code_context>

<deferred>
## Deferred Ideas

- None — discussion stayed within phase scope.

</deferred>

---

*Phase: 33-trial-banner-and-billing-settings-tab*
*Context gathered: 2026-03-29*
