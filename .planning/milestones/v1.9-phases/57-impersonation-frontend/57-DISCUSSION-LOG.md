# Phase 57: Impersonation Frontend - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 57-impersonation-frontend
**Areas discussed:** Auth token switching, Banner design & placement, Impersonation entry point, Session termination

---

## Auth Token Switching

| Option | Description | Selected |
|--------|-------------|----------|
| Replace Bearer token | Store impersonation token in state, use instead of Firebase token in prepareHeaders | ✓ |
| Custom header alongside Firebase | Keep Firebase Bearer token, add X-Impersonation-Token header | |
| You decide | Claude picks based on existing RTK Query setup | |

**User's choice:** Replace Bearer token
**Notes:** Clean approach — backend already resolves impersonation token to target user identity.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Redux slice | Dedicated impersonationSlice in Redux store | ✓ |
| Redux slice + sessionStorage | Redux + sessionStorage for refresh persistence | |
| React Context | Separate ImpersonationProvider | |

**User's choice:** Redux slice
**Notes:** Pairs naturally with RTK Query prepareHeaders which already accesses the store.

---

| Option | Description | Selected |
|--------|-------------|----------|
| End impersonation | Redux state lost on refresh, support user returns to own dashboard | ✓ |
| Persist via sessionStorage | Mirror to sessionStorage, rehydrate on load | |
| You decide | Claude picks based on security | |

**User's choice:** End impersonation on refresh
**Notes:** Security best practice — no persistence of sensitive impersonation sessions.

---

## Banner Design & Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Above header | Fixed bar at very top of viewport, z-50, pushes content down | ✓ |
| Inside header bar | Integrated into existing header row | |
| Floating bottom bar | Fixed bar at bottom of viewport | |

**User's choice:** Above header
**Notes:** Shopify/Stripe pattern — impossible to miss, cannot be covered by modals.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Warning amber | Amber/yellow background with dark text | ✓ |
| Destructive red | Red background with white text | |
| Neutral dark | Dark gray/black background with white text | |

**User's choice:** Warning amber
**Notes:** Matches conventional impersonation bar pattern.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Customer navigation | Show business nav items during impersonation | ✓ |
| Support navigation | Keep support sidebar during impersonation | |
| You decide | Claude picks based on requirements | |

**User's choice:** Customer navigation
**Notes:** Requirement says "exactly what the customer sees" — sidebar must match.

---

## Impersonation Entry Point

| Option | Description | Selected |
|--------|-------------|----------|
| User detail page only | Button on Phase 54's user detail page | ✓ |
| Both list and detail | Icon in users table + button on detail page | |
| Users list only | Action button in each table row | |

**User's choice:** User detail page only
**Notes:** Keeps the action deliberate — review user info before impersonating.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, require reason | Confirmation dialog with required reason text field | ✓ |
| Confirm only, no reason | Simple confirmation without reason | |
| You decide | Claude picks based on Phase 56 contract | |

**User's choice:** Require reason
**Notes:** Maps to Phase 56's audit log reason field.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Hidden entirely | Only show if user has impersonate_user permission | ✓ |
| Visible but disabled | Greyed out with tooltip for users without permission | |

**User's choice:** Hidden entirely
**Notes:** Clean UI — no confusion about capabilities.

---

## Session Termination

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, call backend | Call POST /v1/impersonation/end, then clear state + cache | ✓ |
| Frontend-only cleanup | Just clear Redux state, let session expire naturally | |
| You decide | Claude picks based on audit requirements | |

**User's choice:** Call backend
**Notes:** Ensures audit trail has end timestamps (IAUD-01).

---

| Option | Description | Selected |
|--------|-------------|----------|
| Support dashboard | Navigate to /support after ending | ✓ |
| User detail page | Navigate back to impersonated user's detail | |
| You decide | Claude picks most practical destination | |

**User's choice:** Support dashboard
**Notes:** Clean break from customer context.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Toast + auto-return | Toast notification + auto-navigate to /support | ✓ |
| Modal with explanation | Modal requiring acknowledgment before navigating | |
| You decide | Claude picks least disruptive approach | |

**User's choice:** Toast + auto-return
**Notes:** Least disruptive — no user action required on expiry.

---

## Claude's Discretion

- RTK Query cache reset strategy (full reset vs selective)
- Impersonation banner Tailwind classes and component structure
- Expired token detection approach in RTK Query
- Loading state during impersonation start
- Mobile responsive banner behavior

## Deferred Ideas

None — discussion stayed within phase scope.
