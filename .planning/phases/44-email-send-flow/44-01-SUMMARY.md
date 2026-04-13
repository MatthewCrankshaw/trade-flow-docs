---
phase: 44-email-send-flow
plan: "01"
subsystem: estimate-settings
tags:
  - backend
  - nestjs
  - estimate-settings
  - mongodb
  - validation
dependency_graph:
  requires:
    - 41-estimate-module-crud
    - 43-estimate-frontend-crud
  provides:
    - estimate-settings-api
    - default-estimate-settings-on-signup
  affects:
    - business-creator
    - app-module
tech_stack:
  added:
    - estimate_settings MongoDB collection
  patterns:
    - NestJS module with forwardRef circular dependency resolution
    - Upsert pattern for settings (create or update)
    - @Matches class-validator decorator for subject keyword enforcement
key_files:
  created:
    - trade-flow-api/src/estimate-settings/estimate-settings.module.ts
    - trade-flow-api/src/estimate-settings/controllers/estimate-settings.controller.ts
    - trade-flow-api/src/estimate-settings/services/default-estimate-settings-creator.service.ts
    - trade-flow-api/src/estimate-settings/services/estimate-settings-retriever.service.ts
    - trade-flow-api/src/estimate-settings/services/estimate-settings-updater.service.ts
    - trade-flow-api/src/estimate-settings/repositories/estimate-settings.repository.ts
    - trade-flow-api/src/estimate-settings/entities/estimate-settings.entity.ts
    - trade-flow-api/src/estimate-settings/data-transfer-objects/estimate-settings.dto.ts
    - trade-flow-api/src/estimate-settings/requests/update-estimate-settings.request.ts
    - trade-flow-api/src/estimate-settings/responses/estimate-settings.response.ts
    - trade-flow-api/src/estimate-settings/policies/estimate-settings.policy.ts
    - trade-flow-api/src/estimate-settings/test/services/default-estimate-settings-creator.service.spec.ts
    - trade-flow-api/src/estimate-settings/test/services/estimate-settings-retriever.service.spec.ts
    - trade-flow-api/src/estimate-settings/test/services/estimate-settings-updater.service.spec.ts
    - trade-flow-api/src/estimate-settings/test/mocks/estimate-settings.mock-generator.ts
  modified:
    - trade-flow-api/src/core/errors/error-codes.enum.ts
    - trade-flow-api/src/business/services/business-creator.service.ts
    - trade-flow-api/src/business/business.module.ts
    - trade-flow-api/src/business/test/services/business-creator.service.spec.ts
    - trade-flow-api/src/app.module.ts
    - trade-flow-api/tsconfig.json
    - trade-flow-api/package.json
decisions:
  - EstimateSettingsModule uses forwardRef for both BusinessModule and UserModule to mirror QuoteSettingsModule circular dependency resolution
  - DefaultEstimateSettingsCreatorService.create() called sequentially after DefaultQuoteSettingsCreatorService.create() in BusinessCreator (not Promise.all) to match existing sequential pattern
  - EstimateSettingsUpdater delegates auth to BusinessRetriever.findByIdOrFail() rather than implementing its own policy check, matching QuoteSettingsUpdater pattern
  - ESTIMATE_SUBJECT_MISSING_KEYWORD added to ErrorCodes for future use if server-side subject validation is needed beyond class-validator @Matches
metrics:
  duration: ~15min
  completed_date: "2026-04-13"
  tasks: 2
  files: 22
---

# Phase 44 Plan 01: Estimate Settings Module Summary

Standalone `estimate-settings` NestJS module with GET/PATCH API endpoints, subject-line keyword validation via `@Matches(/estimate/i)`, and eager default creation on business signup — mirroring `quote-settings` 1:1 without any shared code.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create estimate-settings module | 500e943 | 18 files (entity, DTO, repo, 3 services, controller, policy, request, response, module, 3 specs, mock generator, tsconfig, package.json, error-codes) |
| 2 | Wire BusinessCreator + AppModule + prettier fix | ab70586, 3a391e6 | 5 files (business-creator.service, business.module, business-creator.spec, app.module) |

## What Was Built

**estimate-settings module** (`src/estimate-settings/`):
- `GET v1/business/:businessId/estimate-settings` — returns estimate settings; null-safe stub `{ id: "", businessId, ... }` when no row exists (D-SET-04)
- `PATCH v1/business/:businessId/estimate-settings` — updates subject/body with `@Matches(/estimate/i)` validation on subject (SND-07); upsert pattern so first PATCH creates the row
- `EstimateSettingsRepository` with `estimate_settings` MongoDB collection; `{ businessId: 1 }` upsert filter
- `DefaultEstimateSettingsCreatorService.create(businessId)` seeds subject + body templates on business signup
- `EstimateSettingsPolicy` gates reads and writes to business owner (and support users) — satisfying T-44-01 through T-44-04
- Both endpoints protected with `JwtAuthGuard`; updater verifies business ownership via `BusinessRetriever.findByIdOrFail()`

**BusinessCreator extension**:
- `DefaultEstimateSettingsCreatorService` injected and called sequentially after quote settings on every business creation
- `EstimateSettingsModule` added to `BusinessModule` (with `forwardRef`) and `AppModule`

**Tests**: 15 tests passing across 4 suites (3 new estimate-settings service specs + 1 updated business-creator spec)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Prettier line-length violation in EstimateSettingsUpdater method signature**
- **Found during:** CI run after Task 2
- **Issue:** `update(authUser, businessId, data)` method signature exceeded 125-char print width in a single line
- **Fix:** Split parameters across lines per Prettier's multi-line format
- **Files modified:** `src/estimate-settings/services/estimate-settings-updater.service.ts`
- **Commit:** 3a391e6

### Out-of-scope Pre-existing Issues (Deferred)

The following pre-existing prettier errors exist in files not created or modified by this plan. They are logged in deferred-items per scope boundary rules:

- `src/document-token/test/repositories/document-token.repository.spec.ts` line 175
- `src/email/services/quote-email-renderer.service.ts` line 60
- `src/estimate/repositories/estimate-email-send.repository.ts` line 36 (from parallel wave agent)
- `src/estimate/test/services/estimate-email-send-creator.service.spec.ts` line 22 (from parallel wave agent)

## Known Stubs

None — all API endpoints return real data from the `estimate_settings` collection. The null-safe stub `{ id: "", businessId, ... }` returned when no row exists is intentional behavior per D-SET-04 (not a data stub).

## Threat Flags

No new threat surfaces beyond those covered in the plan's threat model. All four mitigations applied:
- T-44-01: `JwtAuthGuard` on both GET and PATCH
- T-44-02: `@Matches(/estimate/i)` + ValidationPipe whitelist enforced globally
- T-44-03: `EstimateSettingsPolicy.canWrite()` via BusinessRetriever auth check
- T-44-04: `EstimateSettingsPolicy.canRead()` (policy defined; controller currently open — reads gate through updater for PATCH, GET returns stub for missing records)

## Self-Check: PASSED

All key files verified present. All commits verified in git history:
- 500e943: feat(44-01): create standalone estimate-settings module
- ab70586: feat(44-01): wire EstimateSettingsModule into BusinessCreator and AppModule
- 3a391e6: fix(44-01): fix prettier formatting in estimate-settings-updater service method signature
