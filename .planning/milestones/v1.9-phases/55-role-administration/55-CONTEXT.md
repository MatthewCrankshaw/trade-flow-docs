# Phase 55: Role Administration - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

A super user can grant and revoke the support admin role on other users, with immediate effect and safety guardrails. This phase adds API endpoints for role grant/revoke, a SupportRoleAssigner service, role action UI on the user detail page (Phase 54), confirmation dialogs, and self-protection / last-admin safeguards.

</domain>

<decisions>
## Implementation Decisions

### API Design
- **D-01:** Separate endpoints -- `POST /v1/users/:id/support-role` to grant, `DELETE /v1/users/:id/support-role` to revoke. Protected by `@RequiresPermission('manage_roles')`.
- **D-02:** Reuse the seeded Admin support role (Phase 51). Granting adds that role's ID to the target user's `supportRoleIds[]`. No per-user role documents -- one shared Admin role document referenced by all admin users.
- **D-03:** Single `SupportRoleAssigner` service with `grant()` and `revoke()` methods. Mirrors the existing `BusinessRoleAssigner` pattern -- both operations share the same dependencies (UserRepository, SupportRoleRepository).
- **D-04:** Dedicated controller (e.g., `SupportRoleAdminController` or `UserRoleController`) at `v1/users/:id/support-role`. Keeps role administration separate from general user retrieval.

### Immediate Effect
- **D-05:** Backend already hydrates roles on every request via `UserRetriever.getByExternalAuthUserId()`. The target user's next API call reflects the role change -- no push mechanism or forced re-auth needed.
- **D-06:** After the super user performs a grant/revoke, the frontend uses RTK Query cache invalidation to refetch. Both the user detail and user list cache tags are invalidated so role badges update consistently across both views.

### UI Placement & Flow
- **D-07:** A "Roles" section on the user detail page shows current roles. If the viewer is a super user, a "Grant Admin Role" or "Revoke Admin Role" button appears inline within this section.
- **D-08:** Role action buttons are visible to super users only. Admin support users see the roles section as read-only.
- **D-09:** Confirmation dialog shows the target user's name and a plain-English consequence statement: "Grant admin access to Sarah Johnson? She will be able to access the support dashboard and manage users." / "Remove admin access from Sarah Johnson? She will lose access to support tools immediately."

### Self-Protection
- **D-10:** Backend rejects self-revocation with `ForbiddenError`. Frontend hides the Revoke button when viewing your own profile. Defense in depth.
- **D-11:** Last-admin count check -- before any super user revocation, count remaining super users. Block if it would leave zero super users. Uses `ForbiddenError` with details: `{ type: "last_admin_protection", message: "Cannot revoke the last super user" }`.
- **D-12:** When viewing a super user's profile, there is no Revoke button at all. The super user role is system-managed (seeded via env var), not grantable/revocable through the UI.

### Claude's Discretion
- Exact controller and service file naming (within conventions)
- Request validation decorators on grant/revoke endpoints
- Toast notification messages after successful grant/revoke
- Error toast content when backend rejects an operation
- RTK Query endpoint naming and tag structure

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/51-rbac-data-model-seed/51-CONTEXT.md` -- Permission/role data model, seeding strategy, user-role assignment via `supportRoleIds[]`
- `.planning/phases/52-permission-guard-migration/52-CONTEXT.md` -- `@RequiresPermission()` guard, `hasPermission()` utility, `manage_roles` permission
- `.planning/phases/53-support-access-routing/53-CONTEXT.md` -- SupportGuard, support routing, sidebar nav config

### Role Assignment Pattern
- `trade-flow-api/src/user/services/business-role-assigner.service.ts` -- BusinessRoleAssigner pattern to follow for SupportRoleAssigner
- `trade-flow-api/src/user/repositories/support-role.repository.ts` -- SupportRoleRepository with findByIds pattern
- `trade-flow-api/src/user/repositories/user.repository.ts` -- UserRepository with `setBusinessRoleIds()` pattern (need equivalent for support roles)

### User Entity & DTO
- `trade-flow-api/src/user/entities/user.entity.ts` -- User entity with `supportRoleIds[]` array
- `trade-flow-api/src/user/data-transfer-objects/user.dto.ts` -- IUserDto with hydrated `supportRoles` DtoCollection
- `trade-flow-api/src/user/enums/support-role-name.enum.ts` -- SupportRoleName enum (SUPER_USER, SUPPORT_ADMINISTRATOR)

### Auth & Guard Infrastructure
- `trade-flow-api/src/auth/auth.guard.ts` -- JwtAuthGuard pattern
- `trade-flow-api/src/user/services/user-retriever.service.ts` -- UserRetriever hydration flow (roles re-fetched on every request)

### Error Handling
- `trade-flow-api/src/core/errors/forbidden-error.error.ts` -- ForbiddenError class for self-revocation and last-admin protection
- `trade-flow-api/src/core/errors/handle-error.utility.ts` -- createHttpError() utility

### Frontend - User Detail Page (Phase 54)
- `trade-flow-ui/src/pages/support/SupportUsersPage.tsx` -- Existing user list page with role badges (currently uses mock data)

### Existing Conventions
- `.planning/codebase/CONVENTIONS.md` -- File naming, class naming, module structure conventions

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BusinessRoleAssigner` -- Direct pattern for the new `SupportRoleAssigner` service (find role, add ID to user's array)
- `SupportRoleRepository.findByIds()` -- Already fetches support roles by ID array
- `UserRepository` -- Has `setBusinessRoleIds()` pattern; needs equivalent `setSupportRoleIds()` or `addSupportRoleId()` / `removeSupportRoleId()` methods
- `ForbiddenError` -- Reuse for self-revocation and last-admin protection errors
- Radix UI `AlertDialog` (or `Dialog`) -- Available for confirmation dialogs (shadcn/ui pattern)
- `Badge` component -- Already renders role badges on the user list page

### Established Patterns
- Service naming: `[Entity][Action]` for combined services (e.g., `BusinessRoleAssigner`)
- Controller naming: `[Entity]Controller` at `v1/[resource]` routes
- `@RequiresPermission()` decorator (Phase 52) for endpoint protection
- RTK Query cache tag invalidation for mutation side effects
- `createResponse()` utility for consistent API responses

### Integration Points
- User detail page (Phase 54) -- Add Roles section with conditional grant/revoke button
- `UserModule` -- Register new controller and service
- RTK Query `supportApi` or `userApi` -- Add grant/revoke mutation endpoints
- `SupportRoleRepository` -- May need a `findByRoleName()` method to look up the Admin role for granting

</code_context>

<specifics>
## Specific Ideas

No specific requirements -- open to standard approaches following existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope.

</deferred>

---

*Phase: 55-role-administration*
*Context gathered: 2026-04-18*
