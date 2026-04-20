---
status: resolved
trigger: "Server crashes on boot: Nest can't resolve dependencies of JwtAuthGuard — ImpersonationAuditRepository at index [3] not available in TaxRateModule"
created: 2026-04-19
updated: 2026-04-19
---

## Symptoms

- **Expected behavior:** Server starts successfully
- **Actual behavior:** Server crashes with UnknownDependenciesException during bootstrap
- **Error messages:** Nest can't resolve dependencies of JwtAuthGuard (UserCreator, UserRetriever, ConfigService, ?). ImpersonationAuditRepository at index [3] not available in TaxRateModule.
- **Timeline:** Current — server cannot start
- **Reproduction:** Run server (`npm run start:dev`)

## Current Focus

- hypothesis: confirmed — ImpersonationAuditRepository added to JwtAuthGuard but ImpersonationModule not globally available
- test: Made ImpersonationModule @Global() so its exports are available in all module scopes
- expecting: Server boots past DI resolution
- next_action: none — resolved

## Evidence

- timestamp: 2026-04-19 — JwtAuthGuard constructor has 4 deps: UserCreator, UserRetriever, ConfigService, ImpersonationAuditRepository
- timestamp: 2026-04-19 — AuthModule imports ImpersonationModule (forwardRef) but does NOT re-export it
- timestamp: 2026-04-19 — 15 modules use @UseGuards(JwtAuthGuard) without importing AuthModule; they rely on per-module DI resolution
- timestamp: 2026-04-19 — Before ImpersonationAuditRepository was added, JwtAuthGuard deps (UserCreator, UserRetriever, ConfigService) were available in every module because UserModule is imported everywhere and ConfigModule is global
- timestamp: 2026-04-19 — Adding @Global() to ImpersonationModule resolves the DI error; server boots (fails on Redis connection, which is expected without local Redis)
- timestamp: 2026-04-19 — All 959 tests pass, typecheck clean

## Eliminated

- Making AuthModule @Global() — tried first but did not resolve the issue due to NestJS guard resolution behavior with @UseGuards(Class) creating instances per-module scope

## Resolution

- **root_cause:** ImpersonationAuditRepository was added as a constructor dependency to JwtAuthGuard, but ImpersonationModule was not globally available. NestJS resolves @UseGuards(JwtAuthGuard) per-module by creating instances in the controller's module scope. 15 modules use JwtAuthGuard without importing AuthModule or ImpersonationModule, so ImpersonationAuditRepository was unresolvable in their DI context.
- **fix:** Added @Global() decorator to ImpersonationModule so its exported ImpersonationAuditRepository is available in all module scopes, matching the pattern already used by CoreModule and QueueModule.
- **files_changed:** src/impersonation/impersonation.module.ts (added @Global() decorator)
- **verification:** Server boots past DI resolution, 959/959 tests pass, typecheck clean
