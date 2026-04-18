# Phase 55: Role Administration - Research

**Researched:** 2026-04-18
**Domain:** NestJS API endpoints + React UI for support role grant/revoke
**Confidence:** HIGH

## Summary

Phase 55 delivers role administration -- the ability for a super user to grant and revoke the support admin role on other users. The phase spans both API (new controller, service, repository methods) and UI (role action buttons, confirmation dialogs on the user detail page from Phase 54). All decisions are locked in CONTEXT.md with high specificity, leaving minimal ambiguity.

The backend work follows the established `BusinessRoleAssigner` pattern directly -- a `SupportRoleAssigner` service with `grant()` and `revoke()` methods, a dedicated `SupportRoleAdminController` at `v1/users/:id/support-role`, and repository methods for adding/removing support role IDs from the user document. The frontend work adds three components to the Phase 54 user detail page: `RoleActions`, `GrantRoleDialog`, and `RevokeRoleDialog`, with RTK Query mutations and cache invalidation.

**Primary recommendation:** Follow the locked decisions exactly. Mirror the `BusinessRoleAssigner` pattern for the backend service, use Radix AlertDialog for confirmation dialogs, and leverage RTK Query cache tag invalidation for immediate UI updates after role changes.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Separate endpoints -- `POST /v1/users/:id/support-role` to grant, `DELETE /v1/users/:id/support-role` to revoke. Protected by `@RequiresPermission('manage_roles')`.
- D-02: Reuse the seeded Admin support role (Phase 51). Granting adds that role's ID to the target user's `supportRoleIds[]`. No per-user role documents -- one shared Admin role document referenced by all admin users.
- D-03: Single `SupportRoleAssigner` service with `grant()` and `revoke()` methods. Mirrors the existing `BusinessRoleAssigner` pattern.
- D-04: Dedicated controller (e.g., `SupportRoleAdminController` or `UserRoleController`) at `v1/users/:id/support-role`.
- D-05: Backend already hydrates roles on every request via `UserRetriever.getByExternalAuthUserId()`. No push mechanism or forced re-auth needed.
- D-06: After grant/revoke, frontend uses RTK Query cache invalidation. Both user detail and user list cache tags invalidated.
- D-07: A "Roles" section on user detail page shows current roles. Super user sees Grant/Revoke button inline.
- D-08: Role action buttons visible to super users only. Admin support users see roles as read-only.
- D-09: Confirmation dialog with target user name and consequence statement.
- D-10: Backend rejects self-revocation with `ForbiddenError`. Frontend hides Revoke button for own profile.
- D-11: Last-admin count check before super user revocation. Block with `ForbiddenError` details: `{ type: "last_admin_protection" }`.
- D-12: No Revoke button for super user profiles. Super user role is system-managed.

### Claude's Discretion
- Exact controller and service file naming (within conventions)
- Request validation decorators on grant/revoke endpoints
- Toast notification messages after successful grant/revoke
- Error toast content when backend rejects an operation
- RTK Query endpoint naming and tag structure

### Deferred Ideas (OUT OF SCOPE)
None.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| RADM-01 | Super user can grant the support admin role to any customer user | D-01 (POST endpoint), D-02 (seeded Admin role), D-03 (SupportRoleAssigner.grant()), UI-SPEC GrantRoleDialog |
| RADM-02 | Super user can revoke the support admin role from any support user | D-01 (DELETE endpoint), D-03 (SupportRoleAssigner.revoke()), UI-SPEC RevokeRoleDialog |
| RADM-03 | Super user cannot revoke their own super user role (last-admin protection) | D-10 (self-revocation rejection), D-11 (last-admin count check), D-12 (no revoke on super user profiles) |
| RADM-04 | Role changes take effect immediately without re-login | D-05 (roles hydrated on every request), D-06 (RTK Query cache invalidation) |
| RADM-05 | Role grant/revoke actions require a confirmation dialog | D-09 (confirmation with name and consequence), UI-SPEC GrantRoleDialog + RevokeRoleDialog |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Role grant/revoke logic | API / Backend | -- | Business logic, authorization checks, and data mutation belong in backend services |
| Self-revocation / last-admin guard | API / Backend | Browser / Client | Backend enforces via ForbiddenError (primary), frontend hides buttons (defense in depth) |
| Confirmation dialogs | Browser / Client | -- | Pure UI concern -- AlertDialog before API call |
| Cache invalidation for immediate effect | Browser / Client | -- | RTK Query cache tags trigger refetch, backend already hydrates on every request |
| Permission check (`manage_roles`) | API / Backend | -- | PermissionGuard (Phase 52) validates on endpoint |

## Standard Stack

### Core

All libraries are already installed in the project. No new dependencies required.

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/common | 11.1.12 | Controller, service, guard decorators | Already in project [VERIFIED: CLAUDE.md] |
| @nestjs/core | 11.1.12 | NestJS framework core | Already in project [VERIFIED: CLAUDE.md] |
| mongoose | 9.1.5 | MongoDB operations via MongoDbWriter/Fetcher | Already in project [VERIFIED: CLAUDE.md] |
| class-validator | 0.14.1 | Request DTO validation decorators | Already in project [VERIFIED: CLAUDE.md] |
| @reduxjs/toolkit | 2.11.2 | RTK Query mutations for grant/revoke | Already in project [VERIFIED: CLAUDE.md] |
| sonner | 2.0.7 | Toast notifications for success/error | Already in project [VERIFIED: CLAUDE.md] |
| lucide-react | 0.563.0 | ShieldPlus/ShieldMinus icons for buttons | Already in project [VERIFIED: CLAUDE.md] |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Radix AlertDialog | via shadcn/ui | Confirmation dialogs with focus trap | Grant/revoke confirmation before API call |

**Installation:** No new packages needed. All dependencies are already present in both repos.

## Architecture Patterns

### System Architecture Diagram

```
Super User Browser                        NestJS API                          MongoDB
     |                                       |                                  |
     |-- Click "Grant Admin Role" ---------->|                                  |
     |   (RoleActions component)             |                                  |
     |                                       |                                  |
     |-- Confirm in AlertDialog ------------>|                                  |
     |   (GrantRoleDialog)                   |                                  |
     |                                       |                                  |
     |-- POST /v1/users/:id/support-role --->|                                  |
     |   (RTK Query mutation)                |                                  |
     |                                       |-- JwtAuthGuard (auth check) ---->|
     |                                       |-- PermissionGuard (manage_roles) |
     |                                       |                                  |
     |                                       |-- SupportRoleAssigner.grant() -->|
     |                                       |   1. Find Admin role by name     |
     |                                       |   2. Check user not already admin|
     |                                       |   3. Add roleId to user's        |
     |                                       |      supportRoleIds[]            |
     |                                       |                                  |
     |<-- 200 { data: updatedUser } ---------|                                  |
     |                                       |                                  |
     |-- Invalidate ['User','UserList'] tags |                                  |
     |-- Show success toast                  |                                  |
     |                                       |                                  |
     |-- Target user's NEXT API request ---->|                                  |
     |                                       |-- UserRetriever hydrates roles ->|
     |                                       |   (supportRoleIds[] now includes |
     |                                       |    admin role -- immediate effect)|
```

### Recommended Project Structure

Backend additions:
```
trade-flow-api/src/user/
├── controllers/
│   └── support-role-admin.controller.ts   # NEW: POST/DELETE /v1/users/:id/support-role
├── services/
│   └── support-role-assigner.service.ts   # NEW: grant() and revoke() methods
├── repositories/
│   └── user.repository.ts                 # MODIFY: add setSupportRoleIds() or addSupportRoleId()/removeSupportRoleId()
│   └── support-role.repository.ts         # MODIFY: add findByRoleName() method
└── test/
    ├── services/
    │   └── support-role-assigner.service.spec.ts  # NEW
    └── controllers/
        └── support-role-admin.controller.spec.ts  # NEW
```

Frontend additions:
```
trade-flow-ui/src/features/support/
├── components/
│   ├── RoleActions.tsx        # NEW: Grant/Revoke button with visibility logic
│   ├── GrantRoleDialog.tsx    # NEW: AlertDialog for grant confirmation
│   └── RevokeRoleDialog.tsx   # NEW: AlertDialog for revoke confirmation
└── api/
    └── supportApi.ts          # MODIFY: add grantSupportRole and revokeSupportRole mutations
```

### Pattern 1: SupportRoleAssigner Service (mirrors BusinessRoleAssigner)

**What:** A single service class with `grant()` and `revoke()` methods that modify the target user's `supportRoleIds[]` array.
**When to use:** All role grant/revoke operations go through this service.

```typescript
// Pattern based on existing BusinessRoleAssigner [VERIFIED: CONTEXT.md canonical ref]
@Injectable()
export class SupportRoleAssigner {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly supportRoleRepository: SupportRoleRepository,
    private readonly logger: AppLogger,
  ) {
    this.logger = new AppLogger(SupportRoleAssigner.name);
  }

  async grant(authUser: IUserDto, targetUserId: string): Promise<IUserDto> {
    // 1. Find the seeded Admin role by name
    // 2. Validate target user exists and has no support role
    // 3. Add admin role ID to target user's supportRoleIds[]
    // 4. Return updated user DTO
  }

  async revoke(authUser: IUserDto, targetUserId: string): Promise<IUserDto> {
    // 1. Validate not self-revocation (authUser.id !== targetUserId)
    // 2. Validate not revoking a super user
    // 3. Validate last-admin protection (count remaining super users)
    // 4. Remove role ID from target user's supportRoleIds[]
    // 5. Return updated user DTO
  }
}
```

### Pattern 2: Dedicated Controller with Permission Guard

**What:** A controller at `v1/users/:id/support-role` with two endpoints, protected by `@RequiresPermission('manage_roles')`.
**When to use:** HTTP entry point for role administration.

```typescript
// Pattern based on existing controller conventions [VERIFIED: STRUCTURE.md]
@Controller("v1/users")
@UseGuards(JwtAuthGuard, PermissionGuard)
export class SupportRoleAdminController {
  constructor(
    private readonly supportRoleAssigner: SupportRoleAssigner,
  ) {}

  @Post(":id/support-role")
  @RequiresPermission("manage_roles")
  async grantRole(@Request() req, @Param("id") targetUserId: string) {
    const result = await this.supportRoleAssigner.grant(req.user, targetUserId);
    return createResponse([result]);
  }

  @Delete(":id/support-role")
  @RequiresPermission("manage_roles")
  async revokeRole(@Request() req, @Param("id") targetUserId: string) {
    const result = await this.supportRoleAssigner.revoke(req.user, targetUserId);
    return createResponse([result]);
  }
}
```

### Pattern 3: RTK Query Mutations with Cache Invalidation

**What:** Two mutation endpoints that call the API and invalidate both user detail and user list caches.
**When to use:** Frontend role grant/revoke actions.

```typescript
// Pattern based on existing RTK Query conventions [VERIFIED: STRUCTURE.md]
grantSupportRole: builder.mutation<UserResponse, string>({
  query: (userId) => ({
    url: `/v1/users/${userId}/support-role`,
    method: "POST",
  }),
  invalidatesTags: ["User", "UserList"],
}),
revokeSupportRole: builder.mutation<UserResponse, string>({
  query: (userId) => ({
    url: `/v1/users/${userId}/support-role`,
    method: "DELETE",
  }),
  invalidatesTags: ["User", "UserList"],
}),
```

### Pattern 4: AlertDialog Confirmation

**What:** Radix AlertDialog with destructive variant for revoke, default for grant.
**When to use:** Before executing any role change.

```tsx
// Pattern based on shadcn/ui AlertDialog [VERIFIED: UI-SPEC]
<AlertDialog>
  <AlertDialogTrigger asChild>
    <Button variant="destructive" aria-label={`Revoke admin role from ${name}`}>
      <ShieldMinus className="mr-2 h-4 w-4" />
      Revoke Admin Role
    </Button>
  </AlertDialogTrigger>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Revoke Admin Role</AlertDialogTitle>
      <AlertDialogDescription>
        Remove admin access from {name}? They will lose access to support tools immediately.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel disabled={isLoading}>Keep Admin Role</AlertDialogCancel>
      <AlertDialogAction
        variant="destructive"
        onClick={handleRevoke}
        disabled={isLoading}
      >
        {isLoading ? <Spinner /> : "Revoke Role"}
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

### Anti-Patterns to Avoid
- **Granting by creating a new role document:** D-02 is explicit -- reuse the seeded Admin role. All admin users reference the same role document ID.
- **Checking role names instead of permissions:** Phase 52 migrated away from `isSupportUser()`/`isSuperUser()`. Use `@RequiresPermission('manage_roles')` on endpoints, not role name checks.
- **Frontend-only protection:** D-10 requires both frontend hiding AND backend rejection. Never rely solely on hiding the button.
- **Invalidating only user detail cache:** D-06 requires both `['User']` and `['UserList']` tags to be invalidated so role badges update on the list page too.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Confirmation dialogs | Custom modal with focus management | Radix AlertDialog (shadcn/ui) | Focus trap, Escape key, accessibility built in [VERIFIED: UI-SPEC] |
| Permission checks | Manual role-name checks in controller | `@RequiresPermission('manage_roles')` decorator | Phase 52 guard infrastructure handles this [VERIFIED: 52-CONTEXT.md] |
| Cache invalidation | Manual refetch calls | RTK Query `invalidatesTags` | Automatic refetch on tag invalidation [VERIFIED: CLAUDE.md RTK Query pattern] |
| Error mapping to HTTP | Custom try/catch in controller | `createHttpError()` utility | Existing error handler maps ForbiddenError to 403 [VERIFIED: STRUCTURE.md] |

## Common Pitfalls

### Pitfall 1: Forgetting Last-Admin Protection for Super Users
**What goes wrong:** A super user revokes themselves and locks everyone out of the admin panel.
**Why it happens:** Self-revocation check (D-10) protects against revoking your own role, but D-11 is a separate check -- count remaining super users before ANY revocation.
**How to avoid:** Two separate guards in `revoke()`: (1) self-revocation check, (2) last-admin count check. Both throw `ForbiddenError` with different `details.type` values.
**Warning signs:** Test only covers self-revocation but not the case where one super user tries to revoke another super user who is the last one.

### Pitfall 2: Super User vs Admin Role Confusion
**What goes wrong:** Implementing revoke for super users when D-12 explicitly says no.
**Why it happens:** The revoke endpoint exists, so developers assume it works for all support roles.
**How to avoid:** Backend `revoke()` must reject attempts to revoke the super user role (it is system-managed, only grantable via env var seed). Frontend never shows Revoke on super user profiles.
**Warning signs:** Test expects revoke to work on a super user target.

### Pitfall 3: Grant to Already-Admin Users
**What goes wrong:** Granting the admin role to a user who already has it, creating duplicate entries in `supportRoleIds[]`.
**Why it happens:** No idempotency check before adding the role ID.
**How to avoid:** `grant()` must check if the target user already has any support role and reject with an appropriate error.
**Warning signs:** Duplicate role IDs in the `supportRoleIds` array after multiple grants.

### Pitfall 4: Stale UI After Role Change
**What goes wrong:** The user list page still shows old role badges after navigating back from the detail page.
**Why it happens:** Only the user detail cache tag was invalidated.
**How to avoid:** Invalidate both `['User']` (detail) and `['UserList']` (list) tags as specified in D-06.
**Warning signs:** Role badge on list page doesn't match what the detail page shows.

### Pitfall 5: Missing Error Differentiation in Frontend
**What goes wrong:** All 403 errors show the same generic message.
**Why it happens:** Frontend doesn't inspect `details.type` in the error response.
**How to avoid:** Parse the error response `details.type` to distinguish `self_revocation`, `last_admin_protection`, and generic `permission_denied`. Show appropriate toast for each (see UI-SPEC copywriting contract).
**Warning signs:** "Unable to update role" toast when it should say "You cannot revoke your own role."

## Code Examples

### SupportRoleRepository.findByRoleName() Addition

```typescript
// Add to existing SupportRoleRepository [VERIFIED: CONTEXT.md canonical ref]
async findByRoleName(name: SupportRoleName): Promise<ISupportRoleDto | null> {
  const entity = await this.mongoDbFetcher.findOne<ISupportRoleEntity>(
    SupportRoleRepository.COLLECTION,
    { name },
  );
  return entity ? this.mapToDto(entity) : null;
}
```

### UserRepository Support Role Methods

```typescript
// Add to existing UserRepository, following setBusinessRoleIds pattern [VERIFIED: CONTEXT.md canonical ref]
async addSupportRoleId(userId: string, roleId: string): Promise<void> {
  await this.mongoDbWriter.updateOne(
    UserRepository.COLLECTION,
    { _id: new ObjectId(userId) },
    { $addToSet: { supportRoleIds: new ObjectId(roleId) } },
  );
}

async removeSupportRoleId(userId: string, roleId: string): Promise<void> {
  await this.mongoDbWriter.updateOne(
    UserRepository.COLLECTION,
    { _id: new ObjectId(userId) },
    { $pull: { supportRoleIds: new ObjectId(roleId) } },
  );
}
```

### Self-Revocation and Last-Admin Guard

```typescript
// Inside SupportRoleAssigner.revoke() [ASSUMED]
if (authUser.id === targetUserId) {
  throw new ForbiddenError(
    ErrorCodes.FORBIDDEN,
    "Cannot revoke your own role",
    { type: "self_revocation" },
  );
}

// Check target is not a super user (D-12: super user role is system-managed)
const targetUser = await this.userRepository.findByIdOrFail(targetUserId);
const hasSuperUserRole = targetUser.supportRoles.items.some(
  (role) => role.name === SupportRoleName.SUPER_USER,
);
if (hasSuperUserRole) {
  throw new ForbiddenError(
    ErrorCodes.FORBIDDEN,
    "Cannot revoke the super user role",
    { type: "system_managed_role" },
  );
}
```

### RoleActions Visibility Logic

```tsx
// RoleActions component visibility [VERIFIED: UI-SPEC interaction contracts]
function RoleActions({ targetUser, viewer }: RoleActionsProps) {
  const isSuperUser = viewer.supportRoles?.items.some(
    (role) => role.name === "super_user"
  );

  if (!isSuperUser) return null;

  const targetHasNoSupportRole = !targetUser.supportRoles?.items.length;
  const targetIsAdmin = targetUser.supportRoles?.items.some(
    (role) => role.name === "admin"
  );
  const targetIsSuperUser = targetUser.supportRoles?.items.some(
    (role) => role.name === "super_user"
  );
  const isViewingSelf = viewer.id === targetUser.id;

  if (targetHasNoSupportRole) {
    return <GrantRoleDialog targetUser={targetUser} />;
  }

  if (targetIsAdmin && !isViewingSelf) {
    return <RevokeRoleDialog targetUser={targetUser} />;
  }

  // Super user profile or own profile: no action buttons
  return null;
}
```

## State of the Art

No technology changes relevant to this phase. All patterns are stable and well-established in the codebase.

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `isSupportUser()`/`isSuperUser()` checks | `@RequiresPermission()` decorator + `hasPermission()` utility | Phase 52 | Endpoint protection uses RBAC, not role name checks |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | ForbiddenError constructor accepts `(code, message, details)` arguments | Code Examples | Minor -- adjust constructor call signature to match actual class |
| A2 | MongoDbWriter supports `$addToSet` and `$pull` operators directly | Code Examples | Medium -- if MongoDbWriter wraps operations differently, need to use native MongoDB driver approach |
| A3 | RTK Query mutation `invalidatesTags` accepts string array shorthand | Architecture Patterns | Low -- may need object format `{ type: 'User' }` instead |

## Open Questions

1. **SupportRoleRepository.findByRoleName() -- does it already exist?**
   - What we know: CONTEXT.md mentions it as a needed addition in Integration Points
   - What's unclear: Whether Phase 51 already adds this method during seeding
   - Recommendation: Check if Phase 51 plan includes this method; if not, add it in this phase

2. **UserRepository.addSupportRoleId() / removeSupportRoleId() -- which pattern?**
   - What we know: CONTEXT.md says there is a `setBusinessRoleIds()` pattern for business roles and needs "equivalent for support roles"
   - What's unclear: Whether to follow `setBusinessRoleIds()` (replace entire array) or use atomic `$addToSet`/`$pull` (add/remove individual IDs)
   - Recommendation: Use `$addToSet`/`$pull` for atomic add/remove -- safer for concurrent operations and more semantically correct for grant/revoke

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 (API), Vitest 4.1.3 (UI) |
| Config file | `jest.config.js` (API), `vite.config.ts` (UI) |
| Quick run command | `npm run test -- --testPathPattern=support-role` (API) |
| Full suite command | `npm run ci` (both repos) |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RADM-01 | Grant admin role via POST endpoint | unit | `npm run test -- --testPathPattern=support-role-assigner` | Wave 0 |
| RADM-02 | Revoke admin role via DELETE endpoint | unit | `npm run test -- --testPathPattern=support-role-assigner` | Wave 0 |
| RADM-03 | Self-revocation rejected + last-admin rejected | unit | `npm run test -- --testPathPattern=support-role-assigner` | Wave 0 |
| RADM-04 | Immediate effect (cache invalidation) | manual-only | N/A -- verify by observing UI after grant/revoke | N/A |
| RADM-05 | Confirmation dialog renders before action | unit | `npm run test -- --testPathPattern=RoleActions` (UI) | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=support-role` (API) or relevant UI test
- **Per wave merge:** `npm run ci` (both repos)
- **Phase gate:** Full suite green before `/gsd-verify-work`

### Wave 0 Gaps
- [ ] `src/user/test/services/support-role-assigner.service.spec.ts` -- covers RADM-01, RADM-02, RADM-03
- [ ] `src/user/test/controllers/support-role-admin.controller.spec.ts` -- covers endpoint routing and guard application
- [ ] `src/user/test/mocks/` -- mock factories for support role assigner dependencies

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | JwtAuthGuard already in place |
| V3 Session Management | no | Firebase manages sessions |
| V4 Access Control | yes | `@RequiresPermission('manage_roles')` guard (Phase 52), self-revocation prevention, last-admin protection |
| V5 Input Validation | yes | class-validator on `:id` param (valid ObjectId), no request body needed |
| V6 Cryptography | no | No cryptographic operations |

### Known Threat Patterns for NestJS Role Administration

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Privilege escalation via direct API call | Elevation of Privilege | `@RequiresPermission('manage_roles')` ensures only super users can grant/revoke |
| Self-revocation to lock out system | Denial of Service | Backend self-revocation check + last-admin count check |
| IDOR on user ID parameter | Tampering | Backend validates target user exists and current user has permission (not resource-ownership scoped, but permission-scoped) |
| Replay of grant/revoke requests | Tampering | Idempotent operations -- granting already-admin user rejected, revoking non-admin user is no-op |

## Sources

### Primary (HIGH confidence)
- `55-CONTEXT.md` -- All 12 locked decisions, canonical references, code context
- `55-UI-SPEC.md` -- Component inventory, interaction contracts, copywriting contract
- `51-CONTEXT.md` -- Role data model, seeding strategy, `supportRoleIds[]` pattern
- `52-CONTEXT.md` -- `@RequiresPermission()` guard, `hasPermission()` utility, migration scope
- `54-CONTEXT.md` -- User detail page structure, user list endpoint
- `.planning/codebase/STRUCTURE.md` -- File naming, directory patterns, where to add new code
- `CLAUDE.md` -- Technology stack versions, conventions, testing standards

### Secondary (MEDIUM confidence)
- None -- all claims derived from project-internal sources

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed, versions confirmed from CLAUDE.md
- Architecture: HIGH -- decisions are extremely specific (12 locked decisions + UI-SPEC)
- Pitfalls: HIGH -- derived from the specific decisions and established codebase patterns

**Research date:** 2026-04-18
**Valid until:** 2026-05-18 (stable -- all decisions locked, no external dependency changes)
