# Requirements: Trade Flow

**Defined:** 2026-04-18
**Milestone:** v1.9 Support & Admin Tools
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.9 Requirements

Requirements for Support & Admin Tools milestone. Each maps to roadmap phases.

### Roles & Permissions Infrastructure

- [ ] **RBAC-01**: System defines workflow-based permissions (e.g., `send_quote`, `manage_schedules`, `view_financials`, `manage_users`, `impersonate_user`) rather than CRUD-based permissions
- [ ] **RBAC-02**: Permissions are stored as a defined set in a `permissions` collection with name, description, and category
- [ ] **RBAC-03**: Roles are stored in a `roles` collection with name, description, type (support or customer), and an array of associated permission IDs
- [ ] **RBAC-04**: Super User support role is seeded with all permissions and cannot be restricted
- [ ] **RBAC-05**: Admin support role is seeded with a default set of permissions and is configurable (permissions can be added or removed)
- [ ] **RBAC-06**: Business Administrator customer role is seeded with full business-scoped permissions (default for solo operators)
- [ ] **RBAC-07**: User-role assignments are stored per user, scoped to either support (global) or business (business-specific)
- [ ] **RBAC-08**: A permission-checking guard/decorator infrastructure validates user permissions on API endpoints (e.g., `@RequiresPermission('manage_users')`)
- [ ] **RBAC-09**: Existing hardcoded role checks (SubscriptionGuard support bypass, PaywallGuard support bypass) are migrated to use the new permission system
- [ ] **RBAC-10**: Solo business users never see role management UI; the data model supports future team roles without exposing complexity prematurely

### Support Access

- [ ] **SACC-01**: Support user can log in and bypass onboarding entirely (no business association required)
- [ ] **SACC-02**: Support user is redirected to `/support` dashboard after login instead of the main app dashboard
- [ ] **SACC-03**: Support routes (`/support/*`) are protected and only accessible to users with a support role
- [ ] **SACC-04**: Support user bypasses subscription gating (existing behaviour, preserved through RBAC migration)

### User Management

- [ ] **UMGT-01**: Support user can view a paginated list of all users (support and customer) with search by name or email
- [ ] **UMGT-02**: User list displays each user's role (support badge), subscription status (trialing, active, past_due, canceled, expired), and associated business name
- [x] **UMGT-03**: Support user can view a user detail page showing profile, business association, subscription status, and role assignments
- [x] **UMGT-04**: Support dashboard shows membership summary cards (total users, active trials, active subscriptions, expired, canceled)

### Role Administration

- [ ] **RADM-01**: Super user can grant the support admin role to any customer user
- [ ] **RADM-02**: Super user can revoke the support admin role from any support user
- [ ] **RADM-03**: Super user cannot revoke their own super user role (last-admin protection)
- [ ] **RADM-04**: Role changes take effect immediately without requiring the target user to re-login
- [ ] **RADM-05**: Role grant/revoke actions require a confirmation dialog in the UI

### Customer Impersonation

- [ ] **IMP-01**: Support user with impersonation permission can initiate a "login as" session for any customer user
- [ ] **IMP-02**: Support user cannot impersonate other support users (prevents lateral privilege movement)
- [ ] **IMP-03**: During impersonation, the app renders exactly what the customer sees (same data, same subscription state, same permissions)
- [ ] **IMP-04**: A fixed impersonation banner is visible at all times during an impersonation session showing the impersonated user's name and a "Return to Support" button
- [ ] **IMP-05**: Support user can terminate the impersonation session and return to their support dashboard cleanly
- [ ] **IMP-06**: Impersonation sessions are time-limited (maximum duration enforced)

### Impersonation Audit

- [ ] **IAUD-01**: Every impersonation session is logged with: who (support user), whom (target user), when (start/end timestamps), and reason (required text field)
- [ ] **IAUD-02**: Impersonation audit log is stored in a dedicated `impersonation_audit` collection
- [ ] **IAUD-03**: Impersonation audit entries are append-only (cannot be modified or deleted)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Team Management

- **TEAM-01**: Business administrator can invite users to join their business as team members
- **TEAM-02**: Business administrator can assign team roles with different permission sets
- **TEAM-03**: Team members can access the business scoped to their assigned role permissions
- **TEAM-04**: Business administrator can remove team members from their business

### Advanced Support

- **ASUP-01**: Support user can view a business detail page (customers, jobs, quotes) without impersonating
- **ASUP-02**: Support user can view user activity timeline (recent actions, login history)
- **ASUP-03**: Support users can add internal notes on user accounts

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Full audit logging system | Entire milestone unto itself; requires event sourcing infrastructure |
| Direct database editing via admin panel | Security risk; bypasses business logic and validation |
| Multi-tier support roles (L1/L2/L3) | Over-engineering for a small team; two support roles (super_user + admin) are sufficient |
| Customer-facing "contact support" feature | Different domain (ticketing/helpdesk); scope creep |
| Real-time user session monitoring | Complex WebSocket infrastructure; surveillance concerns |
| Automated account actions (suspend, delete, force-reset) | Dangerous without comprehensive audit logging |
| Support user editing subscriptions directly | Bypasses Stripe as source of truth; use Stripe Dashboard |
| Bulk role management | Premature optimisation for a small support team |
| Customer-facing role assignment UI | Deferred to team management milestone |
| Permission management UI for admin role | Super users configure admin permissions via seed data for now; UI deferred |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| RBAC-01 | Phase 51 | Pending |
| RBAC-02 | Phase 51 | Pending |
| RBAC-03 | Phase 51 | Pending |
| RBAC-04 | Phase 51 | Pending |
| RBAC-05 | Phase 51 | Pending |
| RBAC-06 | Phase 51 | Pending |
| RBAC-07 | Phase 51 | Pending |
| RBAC-08 | Phase 52 | Pending |
| RBAC-09 | Phase 52 | Pending |
| RBAC-10 | Phase 52 | Pending |
| SACC-01 | Phase 53 | Pending |
| SACC-02 | Phase 53 | Pending |
| SACC-03 | Phase 53 | Pending |
| SACC-04 | Phase 53 | Pending |
| UMGT-01 | Phase 54 | Pending |
| UMGT-02 | Phase 54 | Pending |
| UMGT-03 | Phase 54 | Complete |
| UMGT-04 | Phase 54 | Complete |
| RADM-01 | Phase 55 | Pending |
| RADM-02 | Phase 55 | Pending |
| RADM-03 | Phase 55 | Pending |
| RADM-04 | Phase 55 | Pending |
| RADM-05 | Phase 55 | Pending |
| IMP-01 | Phase 56 | Pending |
| IMP-02 | Phase 56 | Pending |
| IMP-03 | Phase 57 | Pending |
| IMP-04 | Phase 57 | Pending |
| IMP-05 | Phase 57 | Pending |
| IMP-06 | Phase 56 | Pending |
| IAUD-01 | Phase 56 | Pending |
| IAUD-02 | Phase 56 | Pending |
| IAUD-03 | Phase 56 | Pending |

**Coverage:**
- v1.9 requirements: 32 total
- Mapped to phases: 32
- Unmapped: 0

---
*Requirements defined: 2026-04-18*
*Last updated: 2026-04-18 after roadmap creation*
