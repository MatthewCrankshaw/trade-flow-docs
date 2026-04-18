# Phase 54: User Management - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 54-user-management
**Areas discussed:** User list & search, User detail page, Subscription data, Dashboard metrics

---

## User list & search

| Option | Description | Selected |
|--------|-------------|----------|
| Server-side (Recommended) | API handles search + pagination. New GET /v1/support/users endpoint. | ✓ |
| Client-side | Fetch all users, filter in browser. Simpler but won't scale. | |

**User's choice:** Server-side
**Notes:** None

---

| Option | Description | Selected |
|--------|-------------|----------|
| Full row (Recommended) | Name, email, role badge, subscription status badge, business name | |
| Minimal row | Name, email, role badge only | |
| You decide | Claude picks | |

**User's choice:** Name, email, subscription status badge (custom selection)
**Notes:** User specified exact columns — narrower than recommended. No role badge or business name in list view.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, role tabs | Tabs to filter by All / Support / Customer | ✓ |
| No tabs, search only | Just the search bar | |
| You decide | Claude picks | |

**User's choice:** Yes, role tabs
**Notes:** None

---

**User's choice (free-text):** New API filtering/sorting convention
**Notes:** User defined a structured query format: `?filter:<fieldName>:<operator>=value`. Operators: in, eq, nq, lt, gt, le, ge, bt. Nested fields via dot notation. Sorting: `?sort=<field>` (asc) or `?sort=-<field>` (desc). This is a new project-wide convention, not yet implemented anywhere.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Core utility (Recommended) | Build in src/core/ for project-wide reuse | ✓ |
| Support-scoped | Build in support module only | |
| You decide | Claude picks | |

**User's choice:** Core utility
**Notes:** None

---

**User's choice (free-text):** Filter flow architecture
**Notes:** User specified that DtoCollection carries paged/filtered results. IBaseQueryOptionsDto must be extended to carry filters from request → service → repository. Repository maps structured filters to MongoDB queries.

---

**User's choice (free-text):** Document conventions
**Notes:** User requested adding the new filtering/sorting standards to CLAUDE.md and associated conventions docs as part of this phase.

---

## User detail page

| Option | Description | Selected |
|--------|-------------|----------|
| Sectioned card layout (Recommended) | Separate cards for Profile, Business, Subscription, Roles | ✓ |
| Single page with headers | One continuous page with section headers | |
| You decide | Claude picks | |

**User's choice:** Sectioned card layout
**Notes:** None

---

| Option | Description | Selected |
|--------|-------------|----------|
| New route (Recommended) | /support/users/:id — dedicated page with back navigation | ✓ |
| Slide-over panel | Drawer/panel from user list | |
| You decide | Claude picks | |

**User's choice:** New route
**Notes:** Matches existing pattern (job detail, customer detail are separate pages)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Read-only (Recommended) | View-only for Phase 54. Phase 55 adds role actions. | ✓ |
| Include role actions | Add grant/revoke buttons now | |
| You decide | Claude picks | |

**User's choice:** Read-only
**Notes:** Clean phase separation

---

| Option | Description | Selected |
|--------|-------------|----------|
| Status + key dates | Status badge plus trial start/end, subscription start, period end | ✓ |
| Status only | Just the status badge | |
| You decide | Claude picks | |

**User's choice:** Status + key dates
**Notes:** None

---

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, show dates | Account created and last sign-in dates from Firebase | ✓ |
| No auth metadata | Skip Firebase metadata | |
| You decide | Claude picks | |

**User's choice:** Yes, show dates
**Notes:** Useful for support troubleshooting

---

## Subscription data

| Option | Description | Selected |
|--------|-------------|----------|
| Aggregation join (Recommended) | MongoDB $lookup to join subscription onto user results | ✓ |
| Separate queries | Fetch users first, then batch-fetch subscriptions | |
| You decide | Claude picks | |

**User's choice:** Aggregation join
**Notes:** User clarified that subscriptions are on users (via userId), not on businesses. No business indirection needed.

---

| Option | Description | Selected |
|--------|-------------|----------|
| All statuses as filter options | Tabs/dropdown for each status: All/Trialing/Active/Past Due/Canceled/Expired | ✓ |
| No subscription filter | Only role filter, subscription visible but not filterable | |
| You decide | Claude picks | |

**User's choice:** All statuses as filter options
**Notes:** None

---

## Dashboard metrics

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated endpoint (Recommended) | GET /v1/support/dashboard/metrics returns pre-computed counts | ✓ |
| Client-side from list | Fetch users and aggregate in browser | |
| You decide | Claude picks | |

**User's choice:** Dedicated endpoint
**Notes:** None

---

| Option | Description | Selected |
|--------|-------------|----------|
| Top of page (Recommended) | Metrics cards first, then existing content below | ✓ |
| Below existing content | Keep migration controls at top | |
| You decide | Claude picks | |

**User's choice:** Top of page
**Notes:** Summary is the primary purpose of the dashboard

---

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, clickable (Recommended) | Cards navigate to filtered user list | ✓ |
| No, display only | Static summary numbers | |
| You decide | Claude picks | |

**User's choice:** Yes, clickable
**Notes:** Clicking navigates to /support/users with appropriate filter pre-applied

---

## Claude's Discretion

- MongoDB aggregation pipeline structure
- Pagination UI component design
- Role/subscription filter interaction (combined or separate)
- Firebase Admin SDK integration approach
- Dashboard metrics aggregation details
- Empty state design for user list

## Deferred Ideas

None — discussion stayed within phase scope.
