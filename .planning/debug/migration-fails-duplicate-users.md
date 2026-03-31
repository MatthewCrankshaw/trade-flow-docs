---
status: awaiting_human_verify
trigger: "migration-fails-duplicate-users"
created: 2026-03-25T00:00:00.000Z
updated: 2026-03-25T00:15:00.000Z
---

## Current Focus

hypothesis: CONFIRMED -- migration 20260325120000-add-users-unique-indexes.migration.ts attempted to create unique indexes without first deduplicating existing records. Fix applied: deduplication logic added before index creation.
test: TypeScript compilation passes (npx tsc --noEmit, no errors)
expecting: User confirms POST /v1/migrations/run succeeds against staging database with duplicate users
next_action: Awaiting human verification that the migration runs successfully end-to-end

## Symptoms

expected: Migration runs successfully, unique index is created on externalAuthUserId and email fields in the users collection.
actual: Migration fails with MongoServerError E11000 duplicate key error -- externalAuthUserId appears in multiple user documents, so the unique index cannot be built.
errors: |
  MongoServerError: Index build failed -- E11000 duplicate key error collection: trade-flow-db.users index: externalAuthUserId_1 dup key: { externalAuthUserId: "24c3M8wVmxaveYAgh6SCzyozaJk1" }
reproduction: Call POST /v1/migrations/run with existing duplicate user records in the database.
started: Blocking issue -- duplicates caused by race condition bug.

## Eliminated

## Evidence

- timestamp: 2026-03-25T00:10:00.000Z
  checked: 20260325120000-add-users-unique-indexes.migration.ts (committed state before this fix)
  found: Lines 22-23 called collection.createIndex with unique:true directly. No deduplication logic existed. Previous commit (244f079) added the file but omitted dedup.
  implication: The migration fails on any database with pre-existing duplicate users. The auth guard fix prevents future duplicates but does nothing about existing ones.

- timestamp: 2026-03-25T00:11:00.000Z
  checked: migration-runner.service.ts
  found: Runner executes pending migrations sorted by filename. If a migration throws, it propagates the error and stops. The 503 response from POST /v1/migrations/run is the API surface of that throw.
  implication: Fixing the migration's up() method is the correct approach -- no changes needed to the runner.

- timestamp: 2026-03-25T00:12:00.000Z
  checked: user.repository.ts and user-creator.service.ts
  found: Commit 244f079 already added findOrCreate() using MongoDB upsert ($or match on externalAuthUserId OR email, $setOnInsert for new records). auth.guard.ts updated to use this atomic method.
  implication: Future duplicates are prevented. Only the pre-existing dirty data in staging is the remaining blocker.

- timestamp: 2026-03-25T00:15:00.000Z
  checked: Migration fix applied and TypeScript checked
  found: npx tsc --noEmit exits with no output (no errors). Migration now runs two aggregation pipeline dedup passes before createIndex.
  implication: Migration should now succeed against a database with existing duplicates.

## Resolution

root_cause: Duplicate user documents exist in MongoDB (caused by a race condition prior to the findOrCreate fix in commit 244f079). The migration 20260325120000-add-users-unique-indexes.migration.ts attempted to create unique indexes directly without first removing the existing duplicate documents, causing MongoDB to reject the index build with E11000.
fix: |
  Updated migration 20260325120000-add-users-unique-indexes.migration.ts to deduplicate before creating indexes:
    - Aggregation pipeline groups documents by externalAuthUserId, identifies groups with count > 1, keeps the document with the lowest ObjectId (oldest), deletes all others.
    - Same dedup repeated for email field.
    - Only after both dedup passes does the migration call createIndex with unique:true.
  The previous commit (244f079) already fixed the race condition at the application layer (auth guard now uses atomic findOrCreate). This migration fix handles the pre-existing dirty data.
verification: TypeScript compilation passes (npx tsc --noEmit exits with no errors). Awaiting human verification of actual migration run against staging database.
files_changed:
  - trade-flow-api/src/migration/migrations/20260325120000-add-users-unique-indexes.migration.ts
