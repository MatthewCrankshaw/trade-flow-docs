# Phase 55: Role Administration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 55-role-administration
**Areas discussed:** API design, Immediate effect, UI placement & flow, Self-protection

---

## API Design

| Option | Description | Selected |
|--------|-------------|----------|
| Separate endpoints (Recommended) | POST /v1/users/:id/support-role to grant, DELETE to revoke. Clear HTTP semantics, REST conventions. | ✓ |
| Single PATCH endpoint | PATCH with { action: 'grant' \| 'revoke' }. One endpoint, action in body. | |
| Role-specific endpoints | POST/DELETE /v1/users/:id/roles/:roleId. More generic but over-engineering. | |

**User's choice:** Separate endpoints
**Notes:** Protected by @RequiresPermission('manage_roles')

---

| Option | Description | Selected |
|--------|-------------|----------|
| Reuse seeded Admin role (Recommended) | Admin role seeded once, granting adds its ID to user's supportRoleIds[]. Mirrors BusinessRoleAssigner. | ✓ |
| Create per-user role document | New SupportRole document per granted user. Redundant data. | |

**User's choice:** Reuse seeded Admin role

---

| Option | Description | Selected |
|--------|-------------|----------|
| Single SupportRoleAssigner (Recommended) | One service with grant() and revoke(). Mirrors BusinessRoleAssigner pattern. | ✓ |
| Separate services | SupportRoleGranter and SupportRoleRevoker. Strict separation but heavy for two related ops. | |

**User's choice:** Single SupportRoleAssigner

---

| Option | Description | Selected |
|--------|-------------|----------|
| New controller | Dedicated controller at v1/users/:id/support-role. Keeps role admin separate. | ✓ |
| Extend UserController | Add routes to existing UserController. Fewer files but mixes concerns. | |
| You decide | Let Claude choose. | |

**User's choice:** New controller

---

## Immediate Effect

| Option | Description | Selected |
|--------|-------------|----------|
| Next API call picks it up (Recommended) | Backend hydrates roles per-request. Target user's next call has updated roles. | ✓ |
| Force re-auth on target | Revoke Firebase session/JWT. Guarantees immediate but disruptive. | |
| WebSocket notification | Push role-change event. Requires infrastructure that doesn't exist. | |

**User's choice:** Next API call picks it up

---

| Option | Description | Selected |
|--------|-------------|----------|
| RTK Query invalidation (Recommended) | Mutation invalidates user detail cache tag. Page refetches automatically. | ✓ |
| Optimistic update | Update UI before API responds, roll back on error. | |
| You decide | Let Claude choose. | |

**User's choice:** RTK Query invalidation

---

| Option | Description | Selected |
|--------|-------------|----------|
| Both invalidate (Recommended) | Invalidate user detail and user list cache tags. Role badges update everywhere. | ✓ |
| Detail page only | Only invalidate single user detail. List shows stale badges. | |

**User's choice:** Both invalidate

---

## UI Placement & Flow

| Option | Description | Selected |
|--------|-------------|----------|
| Roles section with action button (Recommended) | Roles section on user detail page. Super user sees Grant/Revoke button inline. | ✓ |
| Page-level action dropdown | Dropdown in page header with Grant/Revoke items. Less prominent. | |
| Dedicated role management panel | Separate card with role list and toggles. Over-engineering for one role. | |

**User's choice:** Roles section with action button

---

| Option | Description | Selected |
|--------|-------------|----------|
| User name + clear consequence (Recommended) | Dialog shows name and consequence: "Grant admin access to X? She will..." | ✓ |
| Minimal confirmation | Simple "Are you sure?" with Confirm/Cancel. | |
| Detailed with reason field | Same as recommended + required reason text field. | |

**User's choice:** User name + clear consequence

---

| Option | Description | Selected |
|--------|-------------|----------|
| Super users only (Recommended) | Only super users see Grant/Revoke button. Admin sees read-only roles. | ✓ |
| Visible to all, disabled for non-super | All support users see disabled button with tooltip. | |

**User's choice:** Super users only

---

## Self-Protection

| Option | Description | Selected |
|--------|-------------|----------|
| Backend + hidden UI (Recommended) | Backend rejects self-revocation. Frontend hides button on own profile. | ✓ |
| Backend only | Backend rejects, frontend shows error toast. | |
| UI only | Frontend hides button. No backend check. Unsafe. | |

**User's choice:** Backend + hidden UI

---

| Option | Description | Selected |
|--------|-------------|----------|
| Self-revocation only (Recommended) | Only prevent revoking own super user role. Simple, matches RADM-03. | |
| Last-admin count check | Count super users before revocation. Block if would leave zero. Extra safety. | ✓ |
| You decide | Let Claude choose. | |

**User's choice:** Last-admin count check

---

| Option | Description | Selected |
|--------|-------------|----------|
| Hidden entirely (Recommended) | No Revoke button on super user profile. System-managed role. | ✓ |
| Disabled with explanation | Disabled button with tooltip "Super User role is system-managed". | |

**User's choice:** Hidden entirely

---

| Option | Description | Selected |
|--------|-------------|----------|
| ForbiddenError with details (Recommended) | Existing ForbiddenError with { type: "last_admin_protection" }. Follows patterns. | ✓ |
| New InvalidRequestError | 422 for business rule violation. Semantically precise but less conventional. | |

**User's choice:** ForbiddenError with details

---

## Claude's Discretion

- Exact controller and service file naming
- Request validation decorators
- Toast notification messages
- RTK Query endpoint naming and tag structure
- Error toast content

## Deferred Ideas

None -- discussion stayed within phase scope.
