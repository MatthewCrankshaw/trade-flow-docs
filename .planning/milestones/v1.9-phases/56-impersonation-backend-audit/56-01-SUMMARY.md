---
phase: 56-impersonation-backend-audit
plan: 01
subsystem: impersonation
tags: [impersonation, audit, mongodb, repository, security]
dependency_graph:
  requires: []
  provides:
    - ImpersonationModule
    - ImpersonationAuditRepository
    - IImpersonationAuditEntity
    - IImpersonationAuditDto
    - ImpersonationStatus
  affects:
    - src/app.module.ts
    - tsconfig.json
    - package.json
tech_stack:
  added: []
  patterns:
    - Append-only repository (no update/delete methods -- IAUD-02, IAUD-03)
    - NestJS module with forwardRef for circular dependency (UserModule)
    - Luxon DateTime in DTOs, native Date in entities, toDateTime() at repository boundary
key_files:
  created:
    - src/impersonation/enums/impersonation-status.enum.ts
    - src/impersonation/entities/impersonation-audit.entity.ts
    - src/impersonation/data-transfer-objects/impersonation-audit.dto.ts
    - src/impersonation/requests/start-impersonation.request.ts
    - src/impersonation/requests/end-impersonation.request.ts
    - src/impersonation/responses/impersonation.response.ts
    - src/impersonation/repositories/impersonation-audit.repository.ts
    - src/impersonation/impersonation.module.ts
    - src/impersonation/test/mocks/impersonation-mock-generator.ts
    - src/impersonation/test/repositories/impersonation-audit.repository.spec.ts
  modified:
    - src/app.module.ts (added ImpersonationModule import)
    - tsconfig.json (added @impersonation/* and @impersonation-test/* path aliases)
    - package.json (added @impersonation/* and @impersonation-test/* jest moduleNameMapper entries)
decisions:
  - Used `declare` keyword on request DTO properties to satisfy strictPropertyInitialization (consistent with class-validator NestJS pattern)
  - Used toDateTime() from @core/utilities/to-date-time.utility for Date->DateTime conversion at repository boundary (API CLAUDE.md standard)
  - Used EntityCollection.map() for findBySupportUserId() since MongoDbFetcher.findMany() returns EntityCollection<T> not T[]
  - endedAt stored as Date | null in entity (not undefined) to align with explicit null semantics for session termination
metrics:
  duration: "~10 minutes"
  completed: "2026-04-19"
  tasks: 3
  files: 13
requirements:
  - IAUD-02
  - IAUD-03
---

# Phase 56 Plan 01: Impersonation Module Foundation Summary

Established the `src/impersonation/` module with typed data models, an append-only audit repository, NestJS module registration, and 6 passing repository unit tests verifying both behavior and structural IAUD-03 compliance.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create impersonation data models and request/response types | 7a23d86 | enum, entity, DTO, 2 requests, response, tsconfig.json |
| 2 | Create append-only audit repository, module, register in AppModule | 4555cb1 | repository, module, app.module.ts, package.json |
| 3 | Create mock generator and repository unit tests | d803360 | mock generator, repository spec |

## What Was Built

**Data Models:**
- `ImpersonationStatus` enum with ACTIVE, ENDED, EXPIRED values
- `IImpersonationAuditEntity` extending IBaseEntity with Date fields for MongoDB storage
- `IImpersonationAuditDto` extending IBaseResourceDto with Luxon DateTime fields for service layer
- `StartImpersonationRequest` with `@MinLength(10)` on reason (T-56-02 mitigation)
- `EndImpersonationRequest` with `@IsNotEmpty()` on sessionId
- `IImpersonationResponse` for the start endpoint JWT response

**Append-Only Repository (T-56-01 mitigation):**
- `ImpersonationAuditRepository` with only: `create()`, `findBySessionId()`, `findBySupportUserId()`, `markEnded()`
- Intentionally omits `update()` and `delete()` methods -- enforces write-once semantics at application layer
- `markEnded()` is the one controlled mutation -- sets endedAt + status ENDED for D-11 session termination
- Collection name: `impersonation_audit`

**Module & Registration:**
- `ImpersonationModule` follows one-feature-per-directory convention
- Imports CoreModule and UserModule (via forwardRef for circular dep)
- Exports ImpersonationAuditRepository for Plan 02 services to consume
- Registered in AppModule

**Test Coverage:**
- 6 unit tests, all passing
- Tests: create inserts to correct collection, DTO round-trip with string id and DateTime, findBySessionId ObjectId filter, null return when not found, markEnded sets status "ended", structural IAUD-03 check

## Deviations from Plan

**1. [Rule 1 - Bug] Used `declare` keyword on request DTO properties**
- **Found during:** Task 1 TypeScript compilation check
- **Issue:** `strictPropertyInitialization: true` in tsconfig-check.json flagged required class properties without initializers as TS2564 errors
- **Fix:** Added `declare` keyword to `targetUserId`, `reason`, and `sessionId` properties. This is the correct NestJS pattern for class-validator DTOs where properties are populated at runtime by the validation pipeline, not in a constructor.
- **Files modified:** start-impersonation.request.ts, end-impersonation.request.ts
- **Commit:** 7a23d86

**2. [Rule 1 - Bug] Adjusted findBySupportUserId to use EntityCollection.map()**
- **Found during:** Task 2 implementation
- **Issue:** PATTERNS.md showed MongoDbFetcher.findMany() but did not clarify it returns EntityCollection<T> not T[]. The actual signature returns EntityCollection which has a .map() method but is not a plain array.
- **Fix:** Called `collection.map()` on the EntityCollection result, which returns T[]. This is the correct usage pattern.
- **Files modified:** impersonation-audit.repository.ts
- **Commit:** 4555cb1

## Known Stubs

None. All data model files are complete with correct types. Repository methods are fully implemented. Plan 02 will add services and controller to complete the feature.

## Threat Flags

None. All new files implement mitigations for the threats already registered in the plan's threat model (T-56-01 append-only, T-56-02 input validation).

## Self-Check: PASSED

All created files verified to exist on disk. All task commits verified in git log.

| Check | Result |
|-------|--------|
| impersonation-status.enum.ts | FOUND |
| impersonation-audit.entity.ts | FOUND |
| impersonation-audit.dto.ts | FOUND |
| impersonation-audit.repository.ts | FOUND |
| impersonation.module.ts | FOUND |
| impersonation-mock-generator.ts | FOUND |
| impersonation-audit.repository.spec.ts | FOUND |
| Commit 7a23d86 (Task 1) | FOUND |
| Commit 4555cb1 (Task 2) | FOUND |
| Commit d803360 (Task 3) | FOUND |
