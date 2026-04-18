# Pitfalls Research: v1.9 Support & Admin Tools

**Domain:** Adding support user experience, user management dashboard, customer impersonation, and role administration to existing Trade Flow SaaS (NestJS 11 / React 19 / MongoDB / Firebase Auth / RTK Query)
**Researched:** 2026-04-18
**Confidence:** HIGH for Firebase custom claims behaviour and impersonation security patterns (verified against Firebase official docs, Authress security analysis, and Pigment engineering blog). MEDIUM for frontend guard complexity (derived from existing codebase patterns and NestJS RBAC community patterns).

## Critical Pitfalls

### Pitfall 1: Firebase custom claims propagation delay creates a role-change window

**What goes wrong:** Admin grants a user the `support` role via `setCustomUserClaims`. The new claim does not appear in the user's JWT until the token refreshes -- which happens automatically every 60 minutes, or on explicit `getIdToken(true)`. During the gap, the user's old token (without the support claim) is still valid, and the new claim is invisible to both the API and the UI. The reverse is worse: revoking a support role does not take effect for up to 60 minutes, meaning a removed support user retains elevated access.

**Why it happens:** Firebase ID tokens are self-contained JWTs. The API validates them against Firebase public keys but does not call Firebase on every request to check current claims. This is by design for performance, but it means claim changes are eventually consistent, not immediate.

**How to avoid:**
- Store roles in MongoDB (the `user` document), NOT in Firebase custom claims. The API already reads user data from MongoDB on every authenticated request via `UserRetriever.getByExternalAuthUserId`. Adding a `roles: string[]` field to the user DTO makes role checks immediate with zero propagation delay.
- Firebase custom claims should contain ONLY the information needed for Firebase-side rules (if any). Trade Flow does not use Firestore security rules, so custom claims add zero value over a database-backed role field.
- On the frontend, the role is available from the `/v1/user/me` response (already fetched on every page load via RTK Query). No additional API call needed.
- If custom claims are used as a secondary cache for performance, they MUST be treated as advisory and the MongoDB role as authoritative. The API guard MUST read from MongoDB, never from the JWT claim.

**Warning signs:** Tests pass when run immediately after role assignment but fail with a delay. Support user reports "I can't see the admin dashboard" after being granted the role -- then it works an hour later.

**Phase to address:** Phase 1 (Role data model and guard infrastructure)

**Sources:**
- [Firebase -- Custom Claims and Security Rules](https://firebase.google.com/docs/auth/admin/custom-claims) -- HIGH
- [Firebase SDK Issue #9260 -- Claims disappear after profile update](https://github.com/firebase/firebase-js-sdk/issues/9260) -- HIGH
- [Firebase SDK Issue #6113 -- Claims not updating with createCustomToken](https://github.com/firebase/firebase-js-sdk/issues/6113) -- HIGH

---

### Pitfall 2: Impersonation session leaks write actions into customer data

**What goes wrong:** Support user impersonates a customer to debug their issue. While impersonated, the support user creates a test job, edits a quote, or changes a setting. These changes persist in the customer's real data with no indication that a support user made them. The customer later sees a phantom job or an edited quote they did not touch and loses trust in the product.

**Why it happens:** The simplest impersonation implementation swaps the authenticated user context entirely, making the API treat the support user as the customer. Every endpoint then behaves identically to a real customer session, including all write operations.

**How to avoid:**
- **Read-only by default.** Impersonation sessions MUST default to read-only mode. The API should reject all non-GET requests when an impersonation header is present, returning 403 with a clear message: "Write operations are not permitted during impersonation."
- If write access is genuinely needed for debugging (e.g. "fix the customer's broken quote"), it should require a separate explicit flag (e.g. `X-Impersonate-Write: true`) that is logged and auditable. This is a Phase 2+ feature, not Phase 1.
- The impersonation context must be carried as a separate header (`X-Impersonate-UserId`), NOT by replacing the authenticated user in the request. The API guard resolves the target user for data access but retains the original support user identity for audit purposes.
- Every response during impersonation should include a header (`X-Impersonating: true`) so the frontend can display a persistent banner: "Viewing as [customer name] -- Read-only mode."

**Warning signs:** No test asserting that POST/PUT/PATCH/DELETE fail during impersonation. The impersonation guard replaces `request.user` entirely instead of adding a parallel `request.impersonatedUser` field.

**Phase to address:** Phase 2 (Impersonation infrastructure) -- read-only enforcement is the first task, not an afterthought.

**Sources:**
- [Pigment Engineering -- Impersonation Done Right: Tokens, Read-Only Guarantees, and Audit Trail](https://engineering.pigment.com/2026/04/08/safe-user-impersonation/) -- MEDIUM
- [Authress -- The risks of user impersonation](https://authress.io/knowledge-base/academy/topics/user-impersonation-risks) -- MEDIUM
- [Clerk -- Empower Your Support Team With User Impersonation](https://clerk.com/blog/empower-support-team-user-impersonation) -- MEDIUM

---

### Pitfall 3: Missing audit trail makes impersonation actions untraceable

**What goes wrong:** Support user impersonates customer A, views their jobs and quotes, then impersonates customer B. Six months later, customer A files a GDPR subject access request asking who accessed their data. There is no record of the impersonation -- the access logs show only normal API calls with no indication a support user was involved.

**Why it happens:** Standard application logging records `userId` from the JWT. If impersonation replaces the user context, logs show the customer's ID, not the support user's. Even if impersonation uses a separate header, the default Pino HTTP logging middleware does not capture custom headers.

**How to avoid:**
- **Dedicated impersonation audit log.** Every impersonation session start/end MUST be logged to a separate `impersonation_audit` MongoDB collection with: `supportUserId`, `targetUserId`, `startedAt`, `endedAt`, `reason` (optional text field for the support ticket reference).
- Every API request during impersonation MUST log both `actorId` (support user) and `targetId` (impersonated customer) in the Pino request context. Add these fields to the `AppLogger` context enrichment.
- The impersonation guard should log session start. A session-end event fires when the support user returns to their own view or the session times out.
- Audit records are append-only. No delete endpoint. No update endpoint. Support users cannot modify their own audit trail.
- Retain audit records for the UK statutory limitation period (6 years England/Wales) to cover dispute resolution.

**Warning signs:** The `impersonation_audit` collection does not exist. API request logs during impersonation show only the customer's userId. No test asserting audit record creation on impersonation start.

**Phase to address:** Phase 2 (Impersonation infrastructure) -- audit logging ships WITH impersonation, never after.

**Sources:**
- [Authress -- The risks of user impersonation](https://authress.io/knowledge-base/academy/topics/user-impersonation-risks) -- MEDIUM
- [HubiFi -- What Is Audit Trail Software?](https://www.hubifi.com/blog/audit-trail-saas-platform) -- LOW

---

### Pitfall 4: Role escalation allows support user to grant themselves super-admin

**What goes wrong:** The role management endpoint (`PATCH /v1/users/:id/roles`) checks that the caller has the `support` role but does not prevent a support user from granting themselves a higher role (e.g. `super_admin`), or from granting the `support` role to a friend's account, or from revoking the role of the only remaining super-admin.

**Why it happens:** RBAC enforcement often checks "can this user access this endpoint?" but not "can this user assign THIS SPECIFIC role?" The distinction between "can manage roles" and "can assign any role" is the classic privilege escalation gap.

**How to avoid:**
- **Role hierarchy enforcement.** Define an explicit hierarchy: `super_admin > support > customer`. A user can only grant roles at or below their own level. Only `super_admin` can grant the `support` role. Only `super_admin` can grant `super_admin` (and there should be a seed super-admin that cannot be demoted).
- **Self-modification prevention.** No user can modify their own roles via the API. This prevents both self-escalation and accidental self-demotion.
- **Last-admin guard.** The API must reject role revocation if it would leave zero `super_admin` users. Query: `db.users.countDocuments({ roles: 'super_admin' })` before processing a demotion.
- The role management endpoint should be a separate controller with its own guard (`@Roles('super_admin')`), not a generic user update endpoint with a role field.

**Warning signs:** The role update endpoint accepts any role string without validation. No test for self-escalation. No test for last-admin protection. Role management is handled by the same `UserUpdater` service that handles profile updates.

**Phase to address:** Phase 3 (Role management) -- must be the FIRST feature in this phase, before any UI is built.

---

### Pitfall 5: Support user bypasses onboarding but app state assumes business exists

**What goes wrong:** Support user logs in and the OnboardingGuard correctly skips them (no business required). But downstream code -- `useCurrentBusiness()`, RTK Query calls to `/v1/business/:id`, the job list page, the quote creation dialog -- all assume a business exists. The support user sees blank pages, infinite loading spinners, or crashes from undefined business context.

**Why it happens:** The existing app was built for exactly one user type: a tradesperson with a business. Every feature implicitly depends on `user.businessRoles[0].businessId` being present. The three-tier guard architecture (auth > onboarding > paywall) was designed to ensure this invariant before the user reaches the app. Skipping onboarding for support users breaks the invariant.

**How to avoid:**
- **Separate route tree for support users.** Support users should land on `/support/dashboard` (user management, impersonation controls), NOT on the tradesperson's `/dashboard`. The support route tree does not import business-dependent features.
- `useCurrentBusiness()` must return `null` gracefully (not crash) when no business exists. All call sites must handle the null case -- but the support route tree should not call it at all.
- The OnboardingGuard should redirect support users to `/support/dashboard` instead of `/dashboard`. This is a single check: `if (user.roles.includes('support') && !hasBusiness) navigate('/support/dashboard')`.
- Navigation sidebar should render a different menu for support users (user management, impersonation log) vs tradespeople (jobs, quotes, customers).
- Do NOT add a "fake business" for support users. That creates orphan data and breaks business-scoped queries.

**Warning signs:** Support user sees "Create your business" onboarding step. Support user lands on the tradesperson dashboard and sees "No jobs yet." `useCurrentBusiness()` throws when `businessRoles` is empty.

**Phase to address:** Phase 1 (Support user login and routing) -- this must be solved BEFORE any other support features.

---

### Pitfall 6: Impersonation header injection from non-support users

**What goes wrong:** A regular customer discovers the `X-Impersonate-UserId` header (via browser dev tools, API docs, or source code inspection) and sends requests with it set to another customer's ID. If the API guard only checks for the header's presence and resolves the target user without verifying the caller is a support user, any authenticated user can view any other user's data.

**Why it happens:** The impersonation guard is added to the middleware chain but the check for "is the caller a support user?" is either missing, or is in the wrong order relative to the impersonation resolution.

**How to avoid:**
- The impersonation guard MUST verify `request.user.roles.includes('support')` BEFORE reading the `X-Impersonate-UserId` header. If the caller is not a support user, the header is silently ignored (not rejected -- to avoid leaking the header's existence).
- The guard should be a NestJS interceptor or guard applied globally, not per-controller, so no endpoint can accidentally skip it.
- The target user ID in the header must be validated: the user must exist, and the target user must NOT have a role >= the caller's role (support users cannot impersonate super-admins).
- Integration test: authenticate as a regular customer, send `X-Impersonate-UserId` with another customer's ID, assert the header is ignored and the response contains only the caller's own data.

**Warning signs:** No test for non-support user sending impersonation headers. The impersonation guard reads the header before checking the caller's role. The impersonation header name appears in the OpenAPI spec or public API docs.

**Phase to address:** Phase 2 (Impersonation infrastructure) -- security test is part of the acceptance criteria.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Store roles in Firebase custom claims only | No MongoDB schema change | 60-min propagation delay, untestable in unit tests, claims limited to 1000 bytes, cannot query "all support users" without scanning Firebase | Never -- always use MongoDB as source of truth |
| Skip audit logging for impersonation MVP | Ship faster | GDPR SAR requests unanswerable, no forensics after a data incident, customer trust erosion | Never -- audit ships with impersonation or impersonation does not ship |
| Single shared route tree for support + tradesperson | Less routing code | Every new feature must handle null-business, support user sees irrelevant UI, regression risk on every feature addition | Never -- separate `/support/` route tree from day one |
| Allow write operations during impersonation from the start | Support can fix customer issues directly | Accidental data corruption, audit trail ambiguity ("did the customer do this or support?"), liability risk | Only after read-only impersonation is proven stable and audit trail is comprehensive |
| Use a simple boolean `isSupport` field instead of a roles array | Simpler queries | Cannot add future roles (e.g. `billing_admin`, `content_moderator`) without migration, boolean does not express hierarchy | Acceptable for MVP if the field is `roles: string[]` from day one even if only `support` and `super_admin` exist initially |

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Firebase Auth (custom claims) | Setting claims via `createCustomToken` instead of `setCustomUserClaims` -- claims set via `createCustomToken` cannot be updated later even with forced token refresh | Always use `setCustomUserClaims` for role management. If claims are used at all, set them AFTER user creation, not during |
| Firebase Auth (token validation) | Trusting JWT claims for authorisation without server-side verification against MongoDB | JWT proves identity (who the user is). MongoDB roles prove authorisation (what they can do). Never conflate the two |
| RTK Query (impersonation) | Using the same API slice for normal and impersonated requests, causing cache poisoning -- impersonated data gets cached under the support user's key | Either use a separate RTK Query slice with an impersonation-aware `baseQuery` that includes the impersonation header, or use cache tags that include the impersonated user ID to prevent cross-contamination |
| RTK Query (role change) | Not invalidating the user cache after a role change, so the UI still shows the old role until page refresh | After a role mutation succeeds, invalidate the `['User']` tag to refetch `/v1/user/me`. If viewing another user's profile, invalidate `['Users', userId]` |
| SubscriptionGuard | Impersonation bypasses SubscriptionGuard because the support user's own subscription status is checked instead of the target customer's | During impersonation, the SubscriptionGuard should check the target user's subscription, not the support user's. Support users bypass subscription gating only when NOT impersonating |

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| User management dashboard fetches ALL users in one query | Page loads slowly, then crashes | Server-side pagination from day one. Use the existing `{ data: T[], pagination }` response format with `?page=1&limit=25` | 500+ users |
| Filtering users by subscription status via client-side filter | Dashboard fetches all users, then filters in the browser | Add `filter:subscription:eq=active` server-side filter on the user list endpoint, joining against the subscriptions collection | 200+ users |
| Audit log query without index | Impersonation audit page loads slowly | Compound index on `impersonation_audit: { supportUserId: 1, startedAt: -1 }` and `{ targetUserId: 1, startedAt: -1 }` | 1000+ audit entries |
| Role check on every API request via separate DB query | Added latency on every request | The existing `UserRetriever.getByExternalAuthUserId` already fetches the user document. Add `roles` to the same document -- zero additional queries | Immediate (adds latency from day one if separate query) |

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Impersonation header accepted from non-support users | Any authenticated user can view any other user's data | Guard checks caller role BEFORE reading impersonation header |
| Support user can impersonate another support/super-admin user | Horizontal privilege escalation between elevated roles | Impersonation target must have a strictly lower role than the caller |
| No session timeout on impersonation | Support user walks away from desk while impersonating; another person accesses the customer's data | Impersonation sessions expire after 30 minutes with no activity; explicit "end impersonation" action required |
| Role changes not requiring re-authentication | A compromised support account retains elevated access even after role revocation, for up to 60 minutes | Use MongoDB roles (immediate). If using Firebase claims as secondary, call `revokeRefreshTokens` on role revocation to force re-authentication |
| Audit log accessible to support users who could tamper with it | Support user deletes audit records of their own impersonation sessions | Audit collection has no delete/update endpoints. Only super-admin can view audit logs. MongoDB user permissions restrict write access at the database level if possible |
| User list endpoint exposes sensitive fields to support users | Support user sees Firebase UID, internal IDs, or subscription payment details they should not access | Dedicated `AdminUserResponse` that includes only: name, email, role, subscription status, created date. No financial details, no internal IDs beyond the user's public ID |

## UX Pitfalls

Common user experience mistakes in this domain.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No visual indicator during impersonation | Support user forgets they are impersonating and navigates away, thinking they are looking at their own view | Persistent top banner: "Viewing as [Customer Name] -- Read-only" with a prominent "End Impersonation" button. Banner uses a distinct colour (e.g. orange/amber) that is impossible to miss |
| Support dashboard uses the same nav as tradesperson app | Support user sees Jobs, Quotes, Customers nav items that are irrelevant to their role | Separate nav sidebar for support users: Users, Impersonation Log, Role Management. No tradesperson features visible |
| Role change confirmation is a simple click | Accidental role grant/revocation with no undo | Two-step confirmation dialog: "You are about to grant Support role to [name]. This gives them access to all user data. Confirm?" with explicit type-to-confirm for super_admin grants |
| Impersonation starts without stating the reason | No context for why the impersonation happened, making audit useless | Optional (but encouraged) "Reason" text field on the impersonation start dialog: "Debugging quote issue -- ticket #1234". Stored in audit log |
| User management dashboard shows raw subscription data | Support user cannot quickly assess if a customer's issue is billing-related | Show a human-readable subscription status badge: "Trial (12 days left)", "Active", "Past Due", "Cancelled", "Expired". Compute from the subscription record, do not show raw Stripe fields |

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Support login bypass:** Often missing the redirect to `/support/dashboard` -- verify support user does NOT land on tradesperson dashboard
- [ ] **Impersonation guard:** Often missing the role check on the caller -- verify non-support user with header is ignored, not 403'd (to avoid leaking the header name)
- [ ] **Impersonation read-only:** Often missing enforcement on PATCH/DELETE -- verify ALL non-GET HTTP methods are rejected during impersonation, not just POST
- [ ] **Role management:** Often missing self-modification prevention -- verify support user cannot change their own role
- [ ] **Role management:** Often missing last-admin guard -- verify the last super_admin cannot be demoted
- [ ] **Audit logging:** Often missing session-end event -- verify the audit log records both when impersonation started AND ended
- [ ] **User list pagination:** Often missing server-side pagination -- verify the endpoint returns paginated results, not all users
- [ ] **SubscriptionGuard during impersonation:** Often checks support user's subscription instead of target's -- verify impersonated view shows the customer's actual paywall state
- [ ] **RTK Query cache during impersonation:** Often caches impersonated data under the support user's key -- verify ending impersonation does not show stale customer data
- [ ] **Frontend route guards:** Often allow support user to navigate to `/jobs` or `/quotes` via direct URL -- verify these routes redirect support users to `/support/dashboard`

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Write action during impersonation corrupts customer data | HIGH | Must identify all writes made during impersonation session from audit log. Manually revert each change. If no audit log exists, recovery is nearly impossible -- must diff MongoDB oplog |
| Role escalation exploited | HIGH | Immediately revoke compromised role in MongoDB. Call Firebase `revokeRefreshTokens` for the user. Audit all actions taken with elevated privileges. Notify affected customers if data was accessed |
| Firebase claims out of sync with MongoDB roles | LOW | MongoDB is authoritative. Force token refresh on client. If using claims, run a sync script: read all users from MongoDB, call `setCustomUserClaims` for each |
| Support user sees tradesperson dashboard (no separate routing) | MEDIUM | Add `/support/` route tree. Update OnboardingGuard to redirect by role. Existing feature code unchanged. Tedious but not destructive |
| Audit log missing (shipped impersonation without it) | HIGH | Cannot reconstruct past impersonation history. Must add audit logging immediately and accept the gap. Communicate to affected customers that impersonation history prior to [date] is unavailable |
| Cache poisoning from impersonated RTK Query data | LOW | Clear the entire RTK Query cache on impersonation end (`api.util.resetApiState()`). One-line fix but must be discovered first |

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Firebase claims propagation delay | Phase 1: Role data model | Unit test: set role in MongoDB, assert API guard recognises it immediately without token refresh |
| Impersonation write leaks | Phase 2: Impersonation | Integration test: impersonate user, send POST/PATCH/DELETE, assert 403 on all |
| Missing audit trail | Phase 2: Impersonation | Integration test: start impersonation, assert `impersonation_audit` document created with both user IDs |
| Role escalation | Phase 3: Role management | Integration test: support user tries to grant super_admin, assert 403. Support user tries to modify own role, assert 403. Demote last super_admin, assert 400 |
| Support user hits null business | Phase 1: Support login | E2E test: login as support user (no business), assert redirect to `/support/dashboard`, assert no crash |
| Impersonation header injection | Phase 2: Impersonation | Integration test: authenticate as customer, send impersonation header, assert response contains only caller's data |
| SubscriptionGuard during impersonation | Phase 2: Impersonation | Integration test: impersonate a customer with expired subscription, assert paywall behaviour matches customer's state |
| RTK Query cache poisoning | Phase 2: Impersonation (frontend) | Unit test: start impersonation, fetch data, end impersonation, assert cache is cleared or namespaced |
| User list performance | Phase 1: User management dashboard | Assert pagination params in API response. Load test with 500 seed users |
| Impersonation session timeout | Phase 2: Impersonation | Unit test: impersonation started 31 minutes ago, assert next request returns 401/redirect to support dashboard |

## Sources

**Firebase Auth (HIGH -- official):**
- [Firebase -- Control Access with Custom Claims and Security Rules](https://firebase.google.com/docs/auth/admin/custom-claims)
- [Firebase SDK Issue #9260 -- Custom Claims Disappear After Profile Update](https://github.com/firebase/firebase-js-sdk/issues/9260)
- [Firebase SDK Issue #6113 -- Unable to Update Claims After createCustomToken](https://github.com/firebase/firebase-js-sdk/issues/6113)

**Impersonation Security (MEDIUM):**
- [Authress -- The Risks of User Impersonation](https://authress.io/knowledge-base/academy/topics/user-impersonation-risks)
- [Pigment Engineering -- Impersonation Done Right: Tokens, Read-Only Guarantees, and Audit Trail](https://engineering.pigment.com/2026/04/08/safe-user-impersonation/)
- [Clerk -- Empower Your Support Team With User Impersonation](https://clerk.com/blog/empower-support-team-user-impersonation)
- [Yaro Labs -- How to Build a Safe User Impersonation Tool](https://yaro-labs.com/blog/user-impersonation-tool-saas)

**NestJS RBAC (MEDIUM):**
- [NestJS -- Authorization Documentation](https://docs.nestjs.com/security/authorization)
- [Permit.io -- Implement RBAC Authorization in NestJS](https://www.permit.io/blog/how-to-protect-a-url-inside-a-nestjs-app-using-rbac-authorization)

**Internal (HIGH):**
- Trade Flow v1.6 key decisions (support role bypass on SubscriptionGuard)
- Trade Flow v1.7 key decisions (three-tier route guards, OnboardingGuard, @SkipSubscriptionCheck)
- Trade Flow existing patterns: UserRetriever, AppLogger, JwtAuthGuard, business-scoped policies

---
*Pitfalls research for: v1.9 Support & Admin Tools*
*Researched: 2026-04-18*
