---
phase: 02-inline-customer-and-job-creation-during-quote-estimate-creat
plan: 01
subsystem: job-api
tags: [auto-title, job-creation, backend]
dependency_graph:
  requires: []
  provides: [job-auto-title-generation, countByCustomerId-repository-method]
  affects: [job-creator-service, job-repository, create-job-request]
tech_stack:
  added: []
  patterns: [MongoConnectionService-direct-count, auto-title-generation]
key_files:
  created:
    - trade-flow-api/src/job/test/mocks/job-mock-generator.ts
    - trade-flow-api/src/job/test/services/job-creator.service.spec.ts
    - trade-flow-api/src/job/test/repositories/job.repository.spec.ts
  modified:
    - trade-flow-api/src/job/repositories/job.repository.ts
    - trade-flow-api/src/job/services/job-creator.service.ts
    - trade-flow-api/src/job/requests/create-job.request.ts
    - trade-flow-api/src/job/controllers/mappers/map-create-job-request-to-dto.utility.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json
decisions:
  - "Inject MongoConnectionService directly into JobRepository for countDocuments (follows ScheduleRepository pattern)"
  - "Default empty title in request-to-DTO mapper so service auto-generation triggers on empty string"
metrics:
  duration: 8m31s
  completed: 2026-04-22T17:48:51Z
---

# Phase 02 Plan 01: Job Auto-Title Generation Summary

Backend auto-title generation using `{JobType} - {CustomerName} #{N}` pattern when jobs are created without explicit titles, enabling minimal inline job creation from quote/estimate dialogs.

## What Was Done

### Task 1: Add countByCustomerId and auto-title generation (TDD)

**RED phase** (commit `9aa61a1`):
- Created `JobMockGenerator` for consistent test data
- Created service spec with 5 tests: auto-title on empty string, auto-title on undefined, sequential number 1 (0 existing), sequential number 4 (3 existing), preserve explicit title
- Created repository spec with 2 tests: countByCustomerId returns correct count, returns 0 for no jobs
- Added `@job-test/*` path alias to `tsconfig.json` and `package.json` jest config

**GREEN phase** (commit `7268a61`):
- Added `countByCustomerId(businessId, customerId)` to `JobRepository` using `MongoConnectionService.getDb()` and `countDocuments()` (follows existing `ScheduleRepository` pattern)
- Added auto-title conditional in `JobCreatorService.create()`: checks `!job.title || job.title.trim() === ""`, generates `${jobType.name} - ${customer.name} #${sequentialNumber}`
- Made `title` optional in `CreateJobRequest` (`@IsOptional()` instead of `@IsNotEmpty()`)
- Updated request-to-DTO mapper to default empty string when title is not provided

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Request mapper type mismatch**
- **Found during:** Task 1, GREEN phase
- **Issue:** Making `title` optional in `CreateJobRequest` caused `requestBody.title` to be `string | undefined`, incompatible with `IJobDto.title: string`
- **Fix:** Updated `map-create-job-request-to-dto.utility.ts` to use `requestBody.title ?? ""` so the service auto-generation logic handles empty strings
- **Files modified:** `trade-flow-api/src/job/controllers/mappers/map-create-job-request-to-dto.utility.ts`

**2. [Rule 3 - Blocking] Missing @job-test path alias**
- **Found during:** Task 1, RED phase
- **Issue:** No `@job-test/*` jest/tsconfig alias existed, preventing test imports following project convention
- **Fix:** Added `@job-test/*` alias to both `tsconfig.json` paths and `package.json` jest moduleNameMapper
- **Files modified:** `trade-flow-api/tsconfig.json`, `trade-flow-api/package.json`

## Verification

- `npm run ci` passes: 979 tests, 0 lint errors, 0 format issues, 0 type errors
- All 7 new tests pass (5 service, 2 repository)

## TDD Gate Compliance

- RED gate: `9aa61a1` (test commit with failing tests)
- GREEN gate: `7268a61` (feat commit with passing implementation)
- REFACTOR gate: not needed (implementation is minimal and clean)

## Self-Check: PASSED
