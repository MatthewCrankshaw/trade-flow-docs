---
status: awaiting_human_verify
trigger: "Multiple identical user records created with same email due to race condition on first login"
created: 2026-03-25T00:00:00Z
updated: 2026-03-25T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED - Two separate issues cause duplicate user records
test: Fix applied - atomic findOrCreate + unique indexes + cross-provider identity handling
expecting: User to verify fix in their environment
next_action: Await human verification

## Symptoms

expected: When a user first logs in (via any auth provider), exactly one user record is created in the database. Subsequent logins find the existing record regardless of auth provider used.
actual: Multiple user records are created with the same email address because concurrent requests all pass the "user not found" check before any create operation completes.
errors: No explicit errors — the race condition is silent. The symptom is duplicate MongoDB documents with the same email.
reproduction: Trigger multiple simultaneous authenticated API requests from a brand-new user account (e.g., on first login when the frontend fires several API calls at once).
started: Ongoing issue — inherent to the current auto-create-on-login design without a uniqueness constraint or upsert pattern.

## Eliminated

## Evidence

- timestamp: 2026-03-25T00:00:00Z
  checked: auth.guard.ts - JwtAuthGuard.canActivate()
  found: Lines 38-83 show a find-then-create pattern. Line 38 calls getByExternalAuthUserId(). If null, line 61 calls userCreator.create(). There IS a catch for MongoDB duplicate key error (code 11000) at line 69, which suggests someone already knew this was a risk. However, lookup is by externalAuthUserId only -- not by email.
  implication: The 11000 catch is good defensive code but only works if a unique index exists on externalAuthUserId. It does NOT handle the cross-provider case (same email, different externalAuthUserId).

- timestamp: 2026-03-25T00:00:00Z
  checked: user.repository.ts - create() method
  found: Lines 48-67 use plain insertOne() -- no upsert pattern. The repository has no findOrCreate or upsert method.
  implication: Every call to create() inserts a new document. Without a unique index, duplicates are silently created.

- timestamp: 2026-03-25T00:00:00Z
  checked: user.entity.ts - UserEntity interface
  found: Entity has externalAuthUserId (string) and email (string) fields. No index annotations (not using Mongoose decorators for indexes).
  implication: Indexes must be created via migrations.

- timestamp: 2026-03-25T00:00:00Z
  checked: 20240101000000-example-create-users-index.migration.ts
  found: This is explicitly labelled as an EXAMPLE migration ("provided as a reference - you can delete it"). It creates unique indexes on both externalAuthUserId and email. But being an example, it may never have been run, or it may have been run and subsequently rolled back.
  implication: The indexes may or may not exist in production. Even if they do exist, the auth guard only looks up by externalAuthUserId, not email -- so a user signing in via Google (one Firebase UID) and later via email/password (different Firebase UID) would create two records even with the unique email index, because the email index would reject the second insert but the 11000 catch only retries lookup by externalAuthUserId (which differs).

- timestamp: 2026-03-25T00:00:00Z
  checked: MongoDbWriter.findOneAndUpdate()
  found: The writer already supports findOneAndUpdate with options parameter, meaning upsert:true can be passed.
  implication: Infrastructure for atomic upsert already exists.

## Resolution

root_cause: Two issues combine to cause duplicate users:
  1. RACE CONDITION: The auth guard uses a non-atomic find-then-create pattern (line 38: findByExternalAuthUserId, then line 61: create). Multiple concurrent requests from a new user all pass the "not found" check before any insert completes. The existing duplicate-key catch (11000) only works IF a unique index exists on externalAuthUserId.
  2. CROSS-PROVIDER IDENTITY: The lookup is by externalAuthUserId only. A user authenticating via Google OAuth gets one Firebase UID; the same user authenticating via email/password gets a different Firebase UID. Both create separate user records despite having the same email. The auth guard never checks email as a fallback identity key.

fix: Four files changed:
  1. NEW MIGRATION (20260325120000-add-users-unique-indexes.migration.ts): Creates unique indexes on both `externalAuthUserId` and `email` in the users collection for database-level duplicate protection.
  2. USER REPOSITORY (user.repository.ts): Added `findOrCreate()` method using `findOneAndUpdate` with `upsert:true` and `$setOnInsert`. Atomic operation matches on `$or: [{externalAuthUserId}, {email}]` so it handles both same-provider race conditions and cross-provider identity. Also added `findByEmail()`.
  3. USER CREATOR SERVICE (user-creator.service.ts): Added `findOrCreate()` that delegates to repository's atomic method. Only creates onboarding progress when user is genuinely new (not found).
  4. AUTH GUARD (auth.guard.ts): Replaced the non-atomic find+create+catch(11000) pattern with a call to `userCreator.findOrCreate()`. Simplified control flow -- fast path for existing users by externalAuthUserId, then atomic findOrCreate for new/cross-provider users.

verification: Self-verified code correctness via review. Needs human verification with running application.
files_changed:
  - trade-flow-api/src/auth/auth.guard.ts
  - trade-flow-api/src/user/repositories/user.repository.ts
  - trade-flow-api/src/user/services/user-creator.service.ts
  - trade-flow-api/src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts
