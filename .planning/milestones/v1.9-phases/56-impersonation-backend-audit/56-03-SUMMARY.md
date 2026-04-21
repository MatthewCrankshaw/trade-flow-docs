---
phase: 56-impersonation-backend-audit
plan: 03
subsystem: impersonation
tags: [impersonation, nestjs, module-wiring, ts-node, boot-fix, gap-closure]
dependency_graph:
  requires:
    - ImpersonationModule (plan 01)
    - ImpersonationAuditRepository (plan 01)
    - JwtAuthGuard with ImpersonationAuditRepository injection (plan 02)
  provides:
    - SupportModule with ImpersonationModule fully wired
    - Worker ts-node loading express.d.ts type augmentation
  affects:
    - src/support/support.module.ts
    - nodemon-worker.json
    - tsconfig.json
    - src/impersonation/controllers/impersonation.controller.ts
    - src/auth/test/guards/impersonation.guard.spec.ts
tech_stack:
  added: []
  patterns:
    - TS_NODE_FILES=true env var in nodemon exec to load tsconfig include files
    - ts-node.files=true in tsconfig.json for config-level declaration
key_files:
  created: []
  modified:
    - src/support/support.module.ts (added ImpersonationModule import)
    - nodemon-worker.json (added TS_NODE_FILES=true to exec command)
    - tsconfig.json (added ts-node.files=true section)
    - src/impersonation/controllers/impersonation.controller.ts (prettier fix)
    - src/auth/test/guards/impersonation.guard.spec.ts (prettier fix, committed from Plan 02)
decisions:
  - Used TS_NODE_FILES=true env var + ts-node.files=true in tsconfig for belt-and-suspenders fix to the worker express.d.ts pickup
  - Added ImpersonationModule directly to SupportModule imports (not forwardRef — no circular dependency)
  - Fixed prettier errors in Plan 02 impersonation files as part of CI gate requirement for this plan
metrics:
  duration: "~2 minutes"
  completed: "2026-04-19"
  tasks: 2
  files: 5
requirements:
  - IMP-01
  - IMP-06
---

# Phase 56 Plan 03: Boot Failure Gap Closure Summary

SupportModule gained an explicit ImpersonationModule import to resolve the UnknownDependenciesException for ImpersonationAuditRepository, and the worker ts-node config gained TS_NODE_FILES=true so express.d.ts type augmentation is loaded, eliminating TS2339 on request.impersonator.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Fix NestJS dependency resolution — add ImpersonationModule to SupportModule | 11e0faf | support.module.ts |
| 2 | Fix worker TypeScript compilation — ensure express.d.ts is included | c7ed1bf | nodemon-worker.json, tsconfig.json, impersonation.controller.ts (prettier), impersonation.guard.spec.ts (prettier) |

## What Was Built

**Task 1 — SupportModule dependency fix:**
- Added `ImpersonationModule` to `SupportModule`'s `imports` array
- `JwtAuthGuard` injects `ImpersonationAuditRepository` which is exported by `ImpersonationModule`
- `SupportUserController` uses `@UseGuards(JwtAuthGuard)` so NestJS needs `ImpersonationAuditRepository` in `SupportModule`'s scope
- `AuthModule` already imports `ImpersonationModule` via `forwardRef()` but `SupportModule` was only importing `UserModule`
- `npx nest build` completes with no errors after the fix

**Task 2 — Worker ts-node type augmentation fix:**
- `src/auth/types/express.d.ts` uses `declare global { namespace Express { interface Request { impersonator?: IUserDto } } }` — correct augmentation pattern
- ts-node in `-r ts-node/register` mode does NOT load all files from tsconfig `include` by default — it only compiles transitively imported files
- Since `express.d.ts` has no imports that chain back from `worker.ts`, ts-node never loaded it
- Fix 1: `TS_NODE_FILES=true` prepended to the `exec` command in `nodemon-worker.json`
- Fix 2: `"ts-node": { "files": true }` added to `tsconfig.json` as config-level declaration
- Both fixes are belt-and-suspenders; either alone would resolve TS2339

## Deviations from Plan

**1. [Rule 1 - Bug] Prettier errors in Plan 02 impersonation files blocked CI**
- **Found during:** Task 2 CI run
- **Issue:** `src/auth/test/guards/impersonation.guard.spec.ts` and `src/impersonation/controllers/impersonation.controller.ts` had prettier violations (jwt.sign() multi-line formatting and @Req() parameter formatting) that caused 14 lint errors
- **Fix:** `npx prettier --write` on both files, formatting normalized
- **Files modified:** impersonation.guard.spec.ts, impersonation.controller.ts
- **Commit:** c7ed1bf

## Known Stubs

None. This plan only fixes module wiring and TypeScript config.

## Threat Flags

None. No new trust boundaries introduced — additive module import and config change only.

## Self-Check: PASSED

| Check | Result |
|-------|--------|
| src/support/support.module.ts imports ImpersonationModule | FOUND |
| nodemon-worker.json has TS_NODE_FILES=true | FOUND |
| tsconfig.json has ts-node.files=true | FOUND |
| Commit 11e0faf (Task 1) | FOUND |
| Commit c7ed1bf (Task 2) | FOUND |
| npx nest build exit code 0 | PASSED |
| npm run ci exit code 0 | PASSED |
| 959 tests passing | PASSED |
| 0 lint errors | PASSED |
| format check clean | PASSED |
| typecheck clean | PASSED |
