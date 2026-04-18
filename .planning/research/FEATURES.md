# Feature Landscape: v1.9 Support & Admin Tools

**Domain:** SaaS internal tooling -- support user experience, user management dashboard, customer impersonation, and role administration for a tradespeople business management app.
**Researched:** 2026-04-18
**Confidence:** HIGH (well-established SaaS patterns with extensive prior art; existing codebase already has support role infrastructure)

## Executive Summary

Support and admin tooling for SaaS products follows highly standardised patterns. The four target features -- support login bypass, user management dashboard, customer impersonation, and role administration -- are all table-stakes for any SaaS product that has paying customers and a support team. The patterns are well-understood and the primary risks are security (impersonation audit trails, role escalation) rather than design ambiguity.

The existing Trade Flow codebase already has significant infrastructure in place: `supportRoles` on `IUserDto`, `SubscriptionGuard` support bypass, `PaywallGuard` support bypass, and placeholder support pages (`SupportDashboardPage.tsx`, `SupportBusinessesPage.tsx`, `SupportUsersPage.tsx`). The milestone is about completing and connecting these pieces rather than building from scratch.

## Table Stakes

Features users (support team members) expect. Missing = support workflow is broken.

| Feature | Why Expected | Complexity | Depends On | Notes |
|---------|--------------|------------|------------|-------|
| Support login bypasses onboarding | Support users have no business; onboarding wizard requires business creation | Low | Existing `OnboardingGuard`, `supportRoles` on user DTO | Guard already checks auth state; add support role check before onboarding requirement |
| Support user lands on support dashboard (not main app) | Support users don't have customers/jobs/quotes; main dashboard is meaningless to them | Low | Support route structure (pages already exist) | Redirect after login based on role; `/support` route already scaffolded |
| User list with search and pagination | Support must find specific users among potentially thousands | Medium | New API endpoint: `GET /v1/support/users` | Server-side search by name/email; paginated response matching existing `{ data, pagination }` contract |
| User detail showing membership status | Support needs to see subscription state (trialing, active, past_due, canceled, expired) at a glance | Medium | User list endpoint joins subscription data | Denormalize subscription status onto user list response to avoid N+1 |
| User detail showing business association | Support needs to see which business a user belongs to | Low | Existing `businessRoles` on user DTO | Already available in user data; surface in UI |
| Impersonation session initiation | "Login as" a customer to see exactly what they see | High | New API endpoint, custom JWT minting, session management | Core value of the milestone; requires Firebase Admin SDK `createCustomToken` |
| Impersonation banner | Clear visual indicator that support is viewing as another user | Medium | Impersonation session state | Fixed-position banner at top of viewport; must be impossible to miss |
| Impersonation session termination | "Return to my account" button to exit impersonation | Medium | Impersonation session state | Must restore original support session cleanly |
| Role grant (make user support) | Super user must be able to grant support role to other users | Medium | New API endpoint: `PATCH /v1/support/users/:id/roles` | Only existing support users can grant; prevents privilege escalation |
| Role revoke (remove support role) | Super user must be able to revoke support role | Medium | Same endpoint as grant | Must prevent revoking own role (last-admin protection) |

## Differentiators

Features that improve the support experience beyond minimum viability. Not expected day one, but valuable.

| Feature | Value Proposition | Complexity | Depends On | Notes |
|---------|-------------------|------------|------------|-------|
| Impersonation audit log | Track who impersonated whom, when, and for how long | Medium | Impersonation feature, new `impersonation_logs` collection | Security best practice; required for compliance if Trade Flow handles financial data |
| Impersonation reason field | Support must state why they're impersonating (accountability) | Low | Impersonation initiation flow | Simple text field on the "login as" action; stored in audit log |
| User membership summary cards | Dashboard overview: total users, active trials, active subscriptions, expired, canceled | Low | User list data aggregation | MongoDB aggregation pipeline on subscription status; support dashboard KPIs |
| Business detail view for support | View a business's customers, jobs, quotes without impersonating | Medium | New API endpoint scoped to support role | Lighter-weight than full impersonation for quick lookups |
| User activity timeline | See recent actions (login, quote sent, subscription changed) for a specific user | High | Requires activity/event logging infrastructure (not yet built) | Defer -- requires audit logging milestone first |
| Bulk role management | Grant/revoke roles for multiple users at once | Low | Role management endpoint | Minimal value with small support team; defer |
| Support-to-support chat/notes | Internal notes on user accounts visible to support team | Medium | New notes collection scoped to user IDs | Useful at scale; premature for initial support tooling |

## Anti-Features

Features to explicitly NOT build in this milestone.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Full audit logging system | Entire milestone unto itself; requires event sourcing or change-data-capture infrastructure | Log impersonation events only; defer comprehensive audit logging |
| Direct database editing via admin panel | Massive security risk; bypasses all business logic and validation | Always go through existing service layer; impersonation lets support use the real UI |
| Support user creating businesses/jobs on behalf of users | Blurs ownership; creates data attribution problems | Use impersonation to act as the user; actions are attributed correctly |
| Multi-tier support roles (L1/L2/L3) | Over-engineering for a small team; role explosion risk | Single `support` role; all support users have equal access. Revisit when team grows past 5 |
| Customer-facing "contact support" feature | Different domain (ticketing/helpdesk); scope creep | Use existing channels (email, chat); build support tooling for the team, not customer-facing support |
| Real-time user session monitoring | Complex WebSocket infrastructure; surveillance concerns | Support uses impersonation to reproduce issues; no need to watch live sessions |
| Automated account actions (suspend, delete, force-reset) | Dangerous without proper safeguards; low frequency need | Manual database operations for now; automated actions need comprehensive audit logging first |
| Support user editing subscription directly | Bypasses Stripe as source of truth; creates sync issues | Use Stripe Dashboard for subscription management; Trade Flow reads from Stripe via webhooks |

## Feature Dependencies

```
Support login bypass ─────────────────────────────────────┐
                                                          │
Support dashboard redirect ──── depends on ──── Support login bypass
                                                          │
User list API endpoint ───────────────────────────────────┤
    │                                                     │
    ├── User detail with membership status                │
    │                                                     │
    └── User detail with business association             │
                                                          │
Role management (grant/revoke) ── depends on ── User list (need to find users first)
    │
    └── Self-revocation protection
                                                          
Impersonation session ── depends on ── User list (need to select who to impersonate)
    │
    ├── Impersonation banner
    │
    ├── Impersonation session termination
    │
    └── Impersonation audit log (differentiator, but should ship with impersonation)
```

### Critical Path

1. **Support login bypass** -- everything else is blocked without support users being able to log in properly
2. **User list API + UI** -- both impersonation and role management need a way to find and select users
3. **Role management** and **Impersonation** are independent of each other; can be built in parallel after user list

## Feature Details

### Support Login Bypass

**What it does:** When a user with `supportRoles.length > 0` logs in, they skip the onboarding wizard entirely and land on `/support` instead of `/dashboard`.

**Existing infrastructure:**
- `OnboardingGuard` already checks if user needs onboarding (missing name or business)
- `supportRoles` already exists on `IUserDto` and is populated from MongoDB
- Support pages already exist at `/support`, `/support/users`, `/support/businesses`
- `SubscriptionGuard` and `PaywallGuard` already bypass for support users

**What's needed:**
- `OnboardingGuard` must check `supportRoles` before checking onboarding completion
- Post-login redirect logic must route support users to `/support`
- Support routes must be protected (only support users can access)

**Complexity:** Low -- guard modification + route protection

### User Management Dashboard

**What it does:** Paginated, searchable list of all users with subscription status, business association, role indicators, and last login timestamp.

**Standard SaaS admin user list columns:**
- Name / email (with search)
- Role indicator (support badge)
- Subscription status (trialing, active, past_due, canceled, expired, none)
- Business name (if associated)
- Sign-up date
- Last login (if trackable from Firebase)

**What's needed:**
- API: New support-scoped controller with `GET /v1/support/users` endpoint
- API: Support role guard (decorator/guard that checks `supportRoles`)
- API: Server-side search (name/email partial match) and pagination
- API: Join subscription status onto user response (avoid N+1)
- UI: User list table with search bar, pagination controls, status badges
- UI: Membership summary cards (total users, by subscription status)

**Complexity:** Medium -- new API module + UI table with filters

### Customer Impersonation

**What it does:** Support user clicks "Login as" on a user row, and the app switches to show exactly what that customer sees -- same data, same subscription state, same permissions.

**Implementation pattern (Firebase-compatible):**
The recommended approach for Firebase auth is to use `firebase-admin` SDK's `createCustomToken(uid)` to mint a token for the target user. The support user's frontend receives this token, stores their original session, signs into Firebase with the custom token, and the app renders as if they are that user.

**Session architecture:**
- Original support session stored in memory/sessionStorage (not localStorage -- avoid persistence)
- Impersonation token has a short TTL (30 minutes recommended)
- All API requests during impersonation carry the impersonated user's JWT
- Impersonation state tracked via a flag (e.g., `x-impersonation-session` header or local state)

**UI indicators (standard patterns across SaaS products):**
- Fixed banner at top of page: "You are viewing as [User Name] ([email]). [Return to Support]"
- Banner uses warning/alert color (amber/orange) -- must be visually distinct from all other UI
- Banner persists across all page navigations during impersonation
- "Return to Support" button terminates impersonation session

**Security requirements:**
- Only support role users can initiate impersonation
- Cannot impersonate other support users (prevents lateral privilege movement)
- Impersonation sessions should be time-limited
- All impersonation events should be logged (who, whom, when, duration)
- Impersonation must be read-only OR clearly attributed in audit trails

**What's needed:**
- API: `POST /v1/support/impersonate/:userId` -- validates support role, mints custom token, logs event
- API: `POST /v1/support/impersonate/end` -- logs session end (optional, can be client-side)
- API: Impersonation audit log collection (userId, targetUserId, startedAt, endedAt, reason)
- UI: "Login as" button on user list/detail
- UI: Impersonation banner component (fixed position, high z-index)
- UI: Session swap logic (store original token, sign in with custom token, restore on exit)
- UI: Impersonation context provider (tracks active impersonation state)

**Complexity:** High -- touches auth layer, requires careful session management, security-critical

### Role Administration

**What it does:** Support users can grant or revoke the `support` role on other user accounts.

**Standard patterns:**
- Role changes are explicit actions (button/toggle), not form fields
- Confirmation dialog before granting elevated roles
- Cannot revoke own support role (prevents lockout)
- Cannot grant roles beyond your own level (no privilege escalation)
- Role changes take effect immediately (no approval workflow for small teams)

**What's needed:**
- API: `POST /v1/support/users/:userId/grant-support-role` 
- API: `DELETE /v1/support/users/:userId/revoke-support-role`
- API: Validation -- requester must have support role; cannot self-revoke
- API: Create/update `supportRoleIds` and `supportRoles` on user document
- UI: Role badge on user list showing current role
- UI: Grant/revoke buttons with confirmation dialogs
- UI: Self-revocation disabled with tooltip explanation

**Complexity:** Medium -- straightforward CRUD on role associations with validation

## MVP Recommendation

**Phase 1 -- Support Access (Low complexity, unblocks everything):**
1. Support login bypass (OnboardingGuard modification)
2. Support dashboard redirect
3. Support route protection (guard)

**Phase 2 -- User Management (Medium complexity, unblocks Phase 3):**
4. Support-scoped API module with SupportRoleGuard
5. User list endpoint with search/pagination
6. User list UI with membership status
7. Membership summary dashboard cards

**Phase 3 -- Role Management (Medium complexity, independent):**
8. Role grant/revoke API endpoints
9. Role management UI (buttons + confirmation dialogs)
10. Self-revocation protection

**Phase 4 -- Impersonation (High complexity, highest risk):**
11. Impersonation token minting API
12. Impersonation audit logging
13. Session swap logic (frontend)
14. Impersonation banner and exit flow

**Defer to future milestone:**
- User activity timeline (requires audit logging infrastructure)
- Business detail view for support (nice-to-have, impersonation covers the use case)
- Bulk role management (premature optimization)

## Sources

- [Yaro Labs: How to Build a Safe User Impersonation Tool for SaaS](https://yaro-labs.com/blog/user-impersonation-tool-saas) -- impersonation session architecture, audit patterns, security requirements
- [Authress: The risks of user impersonation](https://authress.io/knowledge-base/academy/topics/user-impersonation-risks) -- security considerations and risk mitigation
- [EnterpriseReady: Role Based Access Control](https://www.enterpriseready.io/features/role-based-access-control/) -- RBAC design patterns for SaaS
- [Cerbos: Role-Based Access Control Best Practices](https://www.cerbos.dev/blog/role-based-access-control-best-practices) -- least privilege, role explosion avoidance
- [IBM: RBAC Implementation Guide](https://www.ibm.com/think/topics/role-based-access-control-implementation) -- implementation patterns
- [Medium: How to build an admin mode to impersonate user accounts](https://parthi.medium.com/how-to-build-an-admin-mode-to-impersonate-user-accounts-da4d8092f9c7) -- Firebase-specific impersonation approach
- [DEV Community: How to Build a Secure Admin Panel for Your SaaS Application](https://dev.to/sodeira/how-to-build-a-secure-admin-panel-for-your-saas-application-2fid) -- general admin panel security
- Existing codebase: `supportRoles` on IUserDto, SubscriptionGuard bypass, PaywallGuard bypass, support page scaffolding
