---
status: complete
phase: 51-rbac-data-model-seed
source: 51-01-SUMMARY.md, 51-02-SUMMARY.md
started: 2026-04-19T08:00:00Z
updated: 2026-04-19T08:30:00Z
---

## Current Test

<!-- OVERWRITE each test - shows where we are -->

[testing complete]

## Tests

### 1. Cold Start Smoke Test
expected: Kill any running API server. Start it fresh (npm run start:dev in trade-flow-api). Server should boot without errors. Watch logs for RbacSeeder output — permissions and support roles should seed via OnModuleInit. No crash, no unhandled exception.
result: pass

### 2. CI Gate Passes
expected: In the trade-flow-api directory, run `npm run ci`. All 4 checks must pass with zero failures: tests (25 new + all existing), lint, format, and typecheck. Exit code 0.
result: pass

### 3. 18 Permissions in Database
expected: After server boots, the `permissions` collection in MongoDB should contain exactly 18 documents — one per PermissionName enum entry. Each document should have name, description, and category fields populated.
result: pass

### 4. Support Roles Seeded
expected: The `supportroles` collection should contain at least two system roles: "Super User" (with all 18 permissionIds and isUnrestrictable: true) and "Support Administrator" (with 3 support-only permissionIds).
result: pass

### 5. Business Admin Roles Backfilled
expected: Any existing Business Administrator roles in the `businessroles` collection should now have 13 permissionIds populated (non-support permissions). Newly created business administrators will also receive these 13 permissions going forward.
result: pass

## Summary

total: 5
passed: 5
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

[none yet]
