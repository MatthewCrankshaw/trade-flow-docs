# Project Research Summary

**Project:** Trade Flow v1.4 — Background Worker Infrastructure
**Domain:** NestJS background job processing with BullMQ + Redis
**Researched:** 2026-03-22
**Confidence:** HIGH

## Executive Summary

Trade Flow v1.4 adds asynchronous background job processing to the existing NestJS API. The core challenge is adding a worker service without breaking the established 21.9k LOC codebase or introducing unnecessary structural complexity. Research conclusively recommends a **dual entry-point approach** — keeping the existing `src/` structure intact and adding `src/worker.ts` as a second NestJS application context alongside `src/main.ts`, rather than migrating to NestJS CLI monorepo mode (`nest g app`). This keeps `tsc` as the compiler, preserves all 20+ path aliases, and avoids a risky full-project restructure.

The recommended stack additions are minimal and well-aligned with the existing codebase: `@nestjs/bullmq` + `bullmq` + `ioredis` for queue processing, Redis 7.4 (Docker) as the backing store, and a new `worker-cli.json` to build and run the worker independently. The worker uses `NestFactory.createApplicationContext()` (no HTTP server) and imports only the specific feature modules it needs — not the entire `AppModule`. A shared `src/queue/` module provides the BullMQ configuration and queue name constants used by both the API (producer) and worker (consumer).

The primary risk is a quiet configuration mismatch across the many files that reference source paths — `tsconfig.json`, `tsconfig.build.json`, `tsconfig-check.json`, `nest-cli.json`, `nodemon.json`, Jest config, and the Dockerfile. The mitigation is strict ordering: infrastructure changes (Redis, deps, path aliases) before worker scaffolding, with explicit verification (`npm run validate && npm run test && node dist/main`) at each phase boundary before proceeding. The Redis `maxmemory-policy noeviction` requirement is a silent production killer that must be configured from day one.

## Key Findings

### Recommended Stack

The stack additions are narrow and purposeful. Only three new npm packages are required: `@nestjs/bullmq`, `bullmq`, and `ioredis`. Everything else (NestJS CLI, ConfigModule, nodemon, Docker Compose, TypeScript) is already in place and used as-is. The worker is not a separate repo, separate `package.json`, or separate monorepo app — it is a second entry point in the same codebase sharing the same `node_modules` and `tsconfig.json`.

See [STACK.md](STACK.md) for full version compatibility matrix and alternatives considered.

**Core technologies:**
- `@nestjs/bullmq` ^11.0.4: NestJS integration for BullMQ — official package, version-aligned with NestJS 11, provides `@Processor()` decorator and `WorkerHost` base class
- `bullmq` ^5.71.0: Job queue library — modern successor to legacy `bull`, Lua-based atomic operations, TypeScript-first, required peer dependency
- `ioredis` ^5.6.1: Redis client — BullMQ peer dep, must be installed explicitly; the `redis` npm package is incompatible
- Redis 7.4-alpine (Docker): Local queue backing store — proven stable branch, BullMQ requires 6.2+ and recommends 7.0+; 8.x is too new
- `worker-cli.json`: Separate NestJS CLI config with `"entryFile": "worker"` — lets `nest build` and `nest start` target the worker independently without triggering monorepo mode

### Expected Features

All v1.4 features are infrastructure — the visible deliverable is a provably working end-to-end pipeline: API enqueues a test job, Redis holds it, worker picks it up and logs success. No user-facing features ship in v1.4. The value is that v1.5+ can migrate email sending and future async work (PDF generation, Stripe webhooks) to the worker without architectural changes.

See [FEATURES.md](FEATURES.md) for full prioritization matrix and existing feature impact analysis.

**Must have (table stakes — v1.4 P1):**
- Dual entry point (`src/main.ts` + `src/worker.ts`) without CLI monorepo mode
- `src/queue/` shared module with `BullModule.forRootAsync()`, queue name constants, Redis config
- Redis service in Docker Compose with `maxmemory-policy noeviction`
- `REDIS_HOST` / `REDIS_PORT` env vars via existing ConfigService pattern
- Worker `main.ts` using `NestFactory.createApplicationContext()` — no HTTP server
- At least one `@Processor` class (smoke-test echo processor) proving end-to-end job flow
- `nodemon-worker.json` for worker hot reload in development
- End-to-end job enqueue and process verification (API enqueues, worker logs receipt)

**Should have (design-for now, implement v1.5):**
- Job retry with exponential backoff (`attempts: 3, backoff: exponential`)
- `@OnWorkerEvent('completed'/'failed')` logging via existing Pino AppLogger
- Queue-per-concern naming convention (`email`, `pdf`, `billing`) rather than a single `default`
- `removeOnComplete: { count: 100 }` and `removeOnFail: { count: 500 }` on all queues

**Defer (v2+):**
- Scheduled/recurring jobs via BullMQ `repeat` option
- Bull Board admin UI for queue monitoring
- Multiple worker instances with concurrency tuning
- Redis persistence configuration (AOF/RDB) for production
- Stripe webhook processing queue

### Architecture Approach

The architecture research explicitly rejects NestJS CLI monorepo mode in favor of a lightweight dual entry-point pattern. Both `main.ts` (API) and `worker.ts` (worker) live in the same `src/` directory and share all feature modules via NestJS DI — no code duplication, no `libs/` restructure, no webpack. The worker's `WorkerModule` cherry-picks only the modules it needs (CoreModule for logging and DB, ConfigModule, LoggerModule, QueueModule, and specific feature modules). Processors follow the same thin-handler pattern as existing controllers: extract job payload, delegate to existing service, handle result.

See [ARCHITECTURE.md](ARCHITECTURE.md) for full component diagram, build configs, and implementation patterns with code examples.

**Major components:**
1. `src/worker.ts` — standalone NestJS app entry point using `createApplicationContext(WorkerModule)`, SIGTERM handler for graceful shutdown
2. `src/worker/worker.module.ts` — root worker module, imports CoreModule + QueueModule + specific feature modules only
3. `src/worker/processors/*.processor.ts` — BullMQ `@Processor` classes extending `WorkerHost`, thin handlers delegating to services
4. `src/queue/queue.module.ts` — `BullModule.forRootAsync()` + `BullModule.registerQueue()`, exports `BullModule`, imported by both AppModule and WorkerModule
5. `src/queue/queue.constants.ts` — queue name constants (`QUEUE_NAMES.EMAIL` etc.), single source of truth shared by producer and consumer
6. `worker-cli.json` — `"entryFile": "worker"`, `"deleteOutDir": false` to prevent clobbering API dist output
7. `nodemon-worker.json` — watches `src/`, exec `nest start --watch --config worker-cli.json`

### Critical Pitfalls

See [PITFALLS.md](PITFALLS.md) for all pitfalls with detection, prevention, recovery strategies, and the "Looks Done But Isn't" verification checklist.

1. **Path aliases break during any source restructure** — `tsconfig.json`, `tsconfig.build.json`, `tsconfig-check.json`, `nest-cli.json`, `nodemon.json`, Jest `moduleNameMapper`, and Dockerfile COPY paths ALL reference `src/`; update atomically and verify with `npm run validate && npm run test && node dist/main` before touching anything else
2. **Accidentally triggering NestJS monorepo mode** — do NOT run `nest g app worker`; it rewrites `nest-cli.json` with `"monorepo": true`, switches to webpack, and changes the entire build output structure; use `worker-cli.json` manually instead
3. **Redis `maxmemory-policy` not set to `noeviction`** — Docker's `redis:7` ships with `volatile-lru` which silently evicts BullMQ keys under memory pressure; jobs vanish without errors; always set `command: redis-server --maxmemory-policy noeviction` in Docker Compose
4. **Worker importing entire AppModule** — pulls in HTTP controllers, `JwtAuthGuard`, `ValidationPipe`, Express, and CORS middleware; use `createApplicationContext()` and cherry-pick only needed modules in `WorkerModule`
5. **Producer-consumer queue name or Redis connection mismatch** — silently operates on different queues; jobs pile up, nothing processes, no errors thrown; always use shared `queue.constants.ts` and verify end-to-end with a test job before any real processor work

## Implications for Roadmap

Based on research, the build order has clear dependencies — infrastructure before configuration, configuration before code, API side (producer) before worker side (consumer), smoke test before real processors.

### Phase 1: Infrastructure Foundation

**Rationale:** Redis and npm dependencies have no code dependencies; they are prerequisites for everything else. Getting infrastructure in place first means every subsequent phase can immediately test its integrations. This is also where the biggest structural risk lives — adding `@queue/*` and `@worker/*` to tsconfig must be verified before any phase that uses them.
**Delivers:** Redis service running in Docker Compose with correct eviction policy; `@nestjs/bullmq`, `bullmq`, `ioredis` installed; `@queue/*` and `@worker/*` path aliases added to `tsconfig.json` and Jest `moduleNameMapper`; `REDIS_HOST` / `REDIS_PORT` in `.env.example`
**Addresses:** Table-stakes Redis env vars, Docker Compose service, ioredis config correctness
**Avoids:** Pitfall #3 (Redis eviction policy), Pitfall #6 (ioredis config)

### Phase 2: Queue Module (Shared Configuration)

**Rationale:** The queue module is the shared glue between API and worker. It must exist before AppModule can produce jobs or WorkerModule can consume them. Building it standalone lets it be validated in isolation — AppModule boots cleanly with QueueModule imported — before the worker is introduced.
**Delivers:** `src/queue/queue.constants.ts` (QUEUE_NAMES), `src/queue/queue.module.ts` (BullModule.forRootAsync + registerQueue), QueueModule imported into AppModule, API starts and connects to Redis without errors
**Uses:** `@nestjs/bullmq`, `BullModule.forRootAsync()` with ConfigService, existing ConfigModule pattern
**Implements:** Queue module architecture component; BullMQ producer side wired to API
**Avoids:** Pitfall #5 (producer-consumer mismatch — single source of truth for queue names established here)

### Phase 3: Worker Service Scaffold

**Rationale:** Once Redis is running and the queue module is validated, the worker can be built bottom-up: entry point first, then module, then first processor, then end-to-end verification. The smoke-test processor (echo/log only) validates the full pipeline before any real business logic is added.
**Delivers:** `src/worker.ts` entry point with `createApplicationContext()` and SIGTERM handling; `src/worker/worker.module.ts` importing CoreModule + QueueModule; one echo `@Processor` that logs received jobs; `worker-cli.json` and `nodemon-worker.json`; `npm run worker:dev` and `npm run worker:prod` scripts; Docker Compose worker service; worker Dockerfile production stage
**Implements:** Worker entry point, WorkerModule, first processor, full dual-process setup
**Avoids:** Pitfall #4 (worker importing HTTP stack), Pitfall #5 (end-to-end test job verification), Pitfall #7 (Dockerfile updated for worker)

### Phase Ordering Rationale

- Infrastructure before queue module because `BullModule.forRootAsync()` needs Redis to exist and npm deps to be installed
- Queue module before worker because AppModule must be able to produce jobs before the consumer side is meaningful
- Worker entry point before processors because `createApplicationContext()` must boot cleanly before processor DI wiring is attempted
- Smoke test (echo processor) before real processors because end-to-end validation must prove the pipeline works before any real job logic builds on top of it
- Dockerfile and Docker Compose worker service during Phase 3 (not deferred) so local development accurately reflects the production topology from day one

### Research Flags

All three phases have standard, well-documented patterns. None need `/gsd:research-phase` during planning.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Infrastructure):** Redis Docker Compose and npm install are mechanical; exact configuration captured in ARCHITECTURE.md
- **Phase 2 (Queue Module):** `BullModule.forRootAsync()` with ConfigService is a documented NestJS + BullMQ pattern; exact code examples in ARCHITECTURE.md
- **Phase 3 (Worker Scaffold):** `createApplicationContext()` pattern is fully documented in ARCHITECTURE.md with code examples and build config details

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified against official changelogs, peer dep requirements, and existing codebase `package.json`; alternatives explicitly evaluated and rejected with rationale |
| Features | HIGH | Feature dependencies mapped with clear rationale; MVP vs. v2+ distinction is well-justified by actual use cases; existing feature impact analysis done against real codebase files |
| Architecture | HIGH | Dual entry-point pattern verified against official NestJS standalone app docs and BullMQ NestJS guide; code examples grounded in actual codebase structure; 40+ path aliases and all config files verified |
| Pitfalls | HIGH | Each pitfall sourced from official docs, known GitHub issues, or codebase analysis; warning signs, recovery steps, and a "Looks Done But Isn't" checklist documented |

**Overall confidence:** HIGH

### Gaps to Address

- **Worker module's exact feature module imports for real processors:** WorkerModule's cherry-picked imports for the smoke-test phase only include CoreModule + QueueModule. When v1.5 adds the first real processor (likely email), the team will need to determine whether to import EmailModule directly or call it via a shared service abstraction. This is a Phase 3 / v1.5 concern, not a blocker for v1.4.
- **Production Redis host format:** Railway.app provides managed Redis but the exact connection URL format (separate host/port vs. single connection URL) is unverified. The architecture uses `REDIS_HOST` + `REDIS_PORT` env vars, which may need a small adjustment for a Railway Redis URL. Low risk — ConfigService can parse either format.
- **`removeOnComplete` / `removeOnFail` tuning values:** The research recommends `count: 100` completed and `count: 500` failed as starting defaults, but optimal values depend on actual job volumes. The constants should be configured from day one to prevent Redis memory growth; tune after v1.5 ships real jobs.

## Sources

### Primary (HIGH confidence)
- [NestJS Monorepo Documentation](https://docs.nestjs.com/cli/monorepo) — monorepo mode behavior, why to avoid it for this project
- [NestJS Standalone Applications](https://docs.nestjs.com/standalone-applications) — `createApplicationContext()` pattern
- [NestJS Queues Technique](https://docs.nestjs.com/techniques/queues) — `@nestjs/bullmq`, BullModule, processors
- [BullMQ NestJS Guide](https://docs.bullmq.io/guide/nestjs) — official `@nestjs/bullmq` setup patterns
- [BullMQ Connections Guide](https://docs.bullmq.io/guide/connections) — `maxRetriesPerRequest`, `enableOfflineQueue`, `keyPrefix` incompatibility
- [BullMQ Going to Production](https://docs.bullmq.io/guide/going-to-production) — `maxmemory-policy noeviction` requirement
- [@nestjs/bullmq Releases](https://github.com/nestjs/bull/releases) — version 11.0.4, bullmq peer dep `^5.40.0`
- [BullMQ Eviction Policy Issue #2737](https://github.com/taskforcesh/bullmq/issues/2737) — runtime behavior of misconfigured eviction policy
- [NestJS Path Aliases in Monorepo (GitHub Issue #13558)](https://github.com/nestjs/nest/issues/13558) — runtime path resolution failures
- Trade Flow codebase: `tsconfig.json`, `nest-cli.json`, `nodemon.json`, `docker-compose.yaml`, `Dockerfile`, `package.json` — all integration points verified against actual files

### Secondary (MEDIUM confidence)
- [NestJS Standalone BullMQ Worker](https://medium.com/@omarae00/nestjs-standalone-bullmq-worker-6f44faefaf6b) — `createApplicationContext()` worker pattern (consistent with official docs)
- [Running NestJS Queues in a Separate Process](https://medium.com/s1seven/running-nestjs-queues-in-a-separate-process-948f414c4b41) — shared module architecture for API + worker
- [NestJS Monorepos Without the Meltdown](https://medium.com/@bhagyarana80/nestjs-monorepos-without-the-meltdown-3a155795ea94) — anti-pattern guidance for monorepo structure, Jan 2026
- [NestJS Monorepo Setup (DEV Community)](https://dev.to/asibul_hasan_5fe57cd945b8/setup-monorepo-in-nestjs-dj4) — practical walkthrough confirming CLI monorepo side effects

---
*Research completed: 2026-03-22*
*Ready for roadmap: yes*
