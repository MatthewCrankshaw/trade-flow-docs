# Phase 51: RBAC Data Model & Seed - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 51-rbac-data-model-seed
**Areas discussed:** Existing role migration, Permission granularity, Seeding strategy, User-role assignment

---

## Existing Role Migration

| Option | Description | Selected |
|--------|-------------|----------|
| Extend existing | Add permissionIds[] to existing supportroles/businessroles. Add new permissions collection. Minimal disruption. | ✓ |
| Replace with unified roles | Single new roles collection with type field replacing both existing collections. Cleaner but requires migration. | |
| New collections alongside | Keep existing untouched, add brand new permissions/roles/assignments collections. Zero migration risk. | |

**User's choice:** Extend existing
**Notes:** Keeps the two-collection split (support vs business roles) that already works.

| Option | Description | Selected |
|--------|-------------|----------|
| Hydrate permissions | Role DTOs include permissions: DtoCollection alongside permissionIds[]. Guards can check permissions without extra DB calls. | ✓ |
| IDs only on role DTOs | Role DTOs only store permissionIds[]. Permission lookup happens separately when needed. | |

**User's choice:** Hydrate permissions
**Notes:** Follows same pattern as user entity (stores IDs, hydrates objects).

---

## Permission Granularity

| Option | Description | Selected |
|--------|-------------|----------|
| v1.9 needs only | ~5-6 permissions for support features only. Add more later. | |
| Comprehensive coverage | Permissions for ALL existing features plus v1.9 support. ~15-20 permissions. | ✓ |
| Grouped by domain | v1.9 permissions plus one broad permission per feature domain. ~8-10 permissions. | |

**User's choice:** Comprehensive coverage
**Notes:** Business Administrator role can be properly populated from the start.

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, category field | Each permission has a category string for grouping. Self-documenting. | ✓ |
| No categories yet | Just name and description. Add category later if needed. | |

**User's choice:** Yes, category field

---

## Seeding Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| App startup upsert | Idempotent upsert on every app boot via onModuleInit. Safe to run repeatedly. | ✓ |
| One-time migration script | Migration entry that runs once. Adding new permissions requires new migration. | |
| CLI seed command | Dedicated npm run seed command. Must be run manually per environment. | |

**User's choice:** App startup upsert

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, env-based bootstrap | SUPER_USER_EMAIL/UID env var identifies initial super user. Prevents lockout on DB reset. | ✓ |
| No, manual assignment only | No auto-assignment. First super user set via database operation. | |
| Seed file with user IDs | JSON config lists initial super user IDs. | |

**User's choice:** Yes, env-based bootstrap
**Notes:** Prevents lockout if database is reset.

---

## User-Role Assignment

| Option | Description | Selected |
|--------|-------------|----------|
| Keep existing arrays | supportRoleIds[] and businessRoleIds[] stay on user entity. Scope implicit from which array. | ✓ |
| Separate assignments collection | New user_role_assignments collection with explicit scope field. Better for audit trails. | |
| Hybrid approach | Keep arrays for fast reads, also write to assignments collection for audit. Dual-write complexity. | |

**User's choice:** Keep existing arrays

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, auto-assign on business creation | Business Administrator role auto-assigned when user creates a business. | ✓ |
| No, assign separately | Business creation doesn't touch roles. Separate step needed. | |

**User's choice:** Yes, auto-assign on business creation
**Notes:** Mirrors how default items/tax rates are auto-created. Solo operators never think about roles.

---

## Claude's Discretion

- Permission entity schema details (indexes, unique constraints)
- Exact permission names and descriptions for each feature domain
- Order of operations in the seeder
- Upsert implementation approach

## Deferred Ideas

None — discussion stayed within phase scope.
