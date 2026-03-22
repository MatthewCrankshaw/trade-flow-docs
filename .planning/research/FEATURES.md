# Feature Research

**Domain:** NestJS monorepo restructure, BullMQ worker service, Redis queue infrastructure
**Researched:** 2026-03-22
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Infrastructure features that must work correctly for the monorepo and worker to be useful. "Users" here means the developer (you) and future deployment pipelines.

| Feature | Why Expected | Complexity | Dependencies on Existing |
|---------|--------------|------------|--------------------------|
| `nest generate app worker` monorepo conversion | NestJS CLI's built-in monorepo mode relocates existing `src/` under `apps/api/`, creates `apps/worker/` as a sibling. Without this, no shared code. | MEDIUM | Requires canonical NestJS project structure (src/ and test/ at root). Existing trade-flow-api follows this. |
| `nest-cli.json` monorepo configuration | Sets `"monorepo": true`, defines `"projects"` map with `api` (default) and `worker` entries, each with own `tsconfig.app.json` and entry point. | LOW | Replaces current single-app nest-cli.json. Must preserve existing compiler options. |
| `nest generate library shared` for shared code | Creates `libs/shared/` with barrel exports, auto-registers path alias `@app/shared` in root tsconfig. Allows worker to import existing API modules (CoreModule, database config, repositories). | MEDIUM | Existing modules like CoreModule, MongoDbFetcher, AppLogger must be extractable to shared lib without breaking API imports. |
| BullModule.forRoot() with Redis connection | `@nestjs/bullmq` integration in both API and worker apps. API produces jobs, worker consumes them. Configured via `@nestjs/config` environment variables. | LOW | Depends on existing ConfigModule/ConfigService pattern already used for MongoDB, Firebase, Resend config. |
| BullModule.registerQueue() for named queues | Register at least one queue (e.g., `email` or `default`) so the infrastructure is testable end-to-end. Queue name shared between producer (API) and consumer (worker). | LOW | None beyond BullModule.forRoot(). |
| @Processor worker class extending WorkerHost | Worker app registers processor classes with `@Processor('queueName')` decorator, extending `WorkerHost` with `async process(job)` method. This is the BullMQ consumption pattern. | LOW | Worker app must import the queue module and any shared services the processor needs. |
| Redis in Docker Compose | Add `redis:7-alpine` service to existing `docker-compose.yaml` alongside MongoDB. Expose port 6379. | LOW | Existing docker-compose.yaml already has MongoDB service. Redis is an additive service. |
| Environment variables for Redis | `REDIS_HOST`, `REDIS_PORT` (minimum). Added to `.env.example` and loaded via ConfigService. | LOW | Follows existing pattern of env vars for MongoDB (`MONGODB_URI`), Firebase, Resend. |
| Worker entry point (`apps/worker/src/main.ts`) | Standalone NestJS application bootstrap (no HTTP listener) that initializes the worker app module and starts processing queues. | MEDIUM | Must bootstrap NestJS app without Express/Fastify. Uses `NestFactory.createApplicationContext()` or standard `NestFactory.create()` without `app.listen()`. |
| Nodemon hot reload for worker | Separate nodemon config (or npm script) that watches `apps/worker/src/` and `libs/` for changes and restarts the worker process. | LOW | Existing nodemon.json watches `src/`. Must be updated for monorepo paths, or create per-app configs. |

### Differentiators (Competitive Advantage)

Features that make this infrastructure particularly well-suited for Trade Flow's future needs (PDF generation, Stripe webhooks, scheduled emails). Not required for v1.4 but should be designed-for.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Queue-per-concern naming convention | Name queues by domain (`email`, `pdf`, `billing`) rather than a single `default` queue. Enables independent scaling and monitoring per concern. | LOW | Define the convention now even if only one queue exists. Worker registers multiple processors. |
| Job retry with exponential backoff | BullMQ supports per-queue and per-job `attempts` and `backoff` configuration. Critical for email sending (Resend API transient failures) and future Stripe webhooks. | LOW | Built into BullMQ. Configure `defaultJobOptions: { attempts: 3, backoff: { type: 'exponential', delay: 1000 } }` on queue registration. |
| Job progress and event tracking | `@OnWorkerEvent('completed')`, `@OnWorkerEvent('failed')` decorators for logging job outcomes. Integrates with existing Pino logging. | LOW | Valuable for debugging in production. Wire to AppLogger. |
| Shared library for DTOs/interfaces | Extract shared types (e.g., job payload interfaces, queue name constants) into `libs/shared/` so both API and worker have a single source of truth. | MEDIUM | Prevents drift between what API enqueues and what worker expects. |
| BullModule.forRootAsync() with ConfigService | Async configuration reads Redis connection from ConfigService rather than hardcoded values. Supports different Redis configs per environment. | LOW | Matches existing pattern: `MongooseModule.forRootAsync()` already uses ConfigService. |
| Dead Letter Queue (DLQ) pattern | Failed jobs after max retries move to a DLQ for manual inspection. Prevents silent data loss on persistent failures. | MEDIUM | BullMQ supports this natively via `removeOnFail` and custom failed-job handlers. Defer implementation to when first real queue use case ships, but design the processor base class to support it. |
| Bull Board admin UI (development only) | Web dashboard at `/admin/queues` showing queue status, job counts, retry/fail rates. Only enabled in development. | LOW | `@bull-board/nestjs` and `@bull-board/api` packages. Wire to existing Express adapter. Guards prevent production exposure. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Nx or Turborepo for monorepo management | "Better caching, dependency graphs, affected commands" | Massive tooling overhead for a 2-app monorepo. NestJS CLI's built-in monorepo mode handles the use case natively. Adds learning curve, config complexity, and another dependency to maintain. | Use NestJS CLI monorepo mode (`nest generate app/lib`). It handles the apps/libs structure, tsconfig management, and build orchestration out of the box. Revisit Nx only if you add 5+ apps. |
| Redis Sentinel or Cluster for HA | "Production Redis needs high availability" | Solo tradesperson app with low traffic. Redis Sentinel adds operational complexity (3+ nodes minimum). Railway.app provides managed Redis with built-in persistence. | Use single Redis instance (Railway managed Redis or Docker for dev). Add Sentinel only when actual uptime SLA demands it. |
| BullMQ sandboxed processors | "Run processors in child processes for isolation" | Adds process management complexity, harder debugging, slower startup. Sandboxed processors lose access to NestJS dependency injection. | Use standard in-process WorkerHost processors. The worker is already a separate NestJS app from the API -- that IS the isolation boundary. |
| Microservice transport (e.g., Redis pub/sub via @nestjs/microservices) | "NestJS has a microservices module, use it" | BullMQ already provides the job queue pattern. Adding @nestjs/microservices for Redis transport introduces a second communication mechanism. Overengineered for "API enqueues job, worker processes it." | Stick with BullMQ queues as the sole inter-service communication mechanism. It handles persistence, retries, and backpressure. Microservice transport is for request-reply patterns you don't need. |
| Shared database connection pool between API and worker | "Both apps connect to same MongoDB, share the connection" | Each NestJS app must manage its own connection lifecycle. Sharing connections across processes is not possible (separate OS processes). | Each app (API and worker) independently connects to MongoDB via its own MongooseModule.forRootAsync(). Same connection string, independent pools. |
| Publishing shared libs to npm registry | "Proper package management for shared code" | Single repo, single team (you). Registry publishing adds versioning ceremony, publish steps, and version drift risk for zero benefit. | NestJS monorepo libs are resolved via tsconfig path aliases at compile time. No publishing needed. |

## Feature Dependencies

```
[Redis in Docker Compose]
    └──requires──> [Environment variables for Redis]
                       └──requires──> [BullModule.forRoot() config]
                                          └──requires──> [BullModule.registerQueue()]
                                                             └──requires──> [@Processor worker class]

[nest generate app worker (monorepo conversion)]
    └──requires──> [nest-cli.json monorepo config]
                       └──enables──> [nest generate library shared]
                                         └──enables──> [Worker imports shared modules]

[Worker entry point (main.ts)]
    └──requires──> [monorepo conversion complete]
    └──requires──> [BullModule config in worker app module]

[Nodemon hot reload for worker]
    └──requires──> [Worker entry point exists]
    └──requires──> [monorepo directory structure finalized]
```

### Dependency Notes

- **Monorepo conversion must happen first:** Everything else depends on the apps/libs directory structure being in place. This is a structural change that touches every import path in the existing API.
- **Redis infra before BullMQ config:** Docker Compose Redis service and env vars must exist before BullModule.forRoot() can connect.
- **Shared library before worker processors:** Worker processors need to import shared modules (CoreModule for logging, database access) to do useful work.
- **Worker entry point depends on both monorepo structure AND BullMQ config:** The worker app module imports BullModule and registers processors.

## MVP Definition

### Launch With (v1.4)

Minimum viable infrastructure -- what's needed so that v1.5+ can enqueue actual jobs.

- [x] Monorepo conversion via `nest generate app worker` -- structural foundation for everything
- [x] `nest-cli.json` updated for monorepo mode with `api` and `worker` projects
- [x] Shared library (`libs/shared/`) with CoreModule extraction (logging, config)
- [x] Redis service in Docker Compose (`redis:7-alpine`)
- [x] Environment variables (`REDIS_HOST`, `REDIS_PORT`) via ConfigService
- [x] `BullModule.forRootAsync()` configured in both API and worker app modules
- [x] At least one registered queue (e.g., `default` or `email`) to prove end-to-end
- [x] One @Processor class in worker that logs received jobs (smoke test)
- [x] Worker `main.ts` entry point bootstrapping NestJS app context
- [x] Nodemon config for worker hot reload in development
- [x] API can enqueue a job that the worker picks up and processes (integration proof)

### Add After Validation (v1.5+)

Features to add when first real queue use case ships (likely email or PDF).

- [ ] Email queue -- move Resend email sending from synchronous API call to async worker job
- [ ] Queue-per-concern naming (`email`, `pdf`, `billing`)
- [ ] Dead Letter Queue pattern for failed jobs
- [ ] Bull Board admin UI (development only)
- [ ] Job progress tracking and structured logging of outcomes
- [ ] PDF generation queue (quote PDF rendering in worker)

### Future Consideration (v2+)

Features to defer until queue usage patterns are proven.

- [ ] Scheduled/recurring jobs via BullMQ's `repeat` option (e.g., reminder emails)
- [ ] Multiple worker instances with concurrency tuning
- [ ] Redis persistence configuration (AOF/RDB) for production
- [ ] Queue metrics and alerting integration
- [ ] Stripe webhook processing queue

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Monorepo conversion | HIGH | MEDIUM | P1 |
| nest-cli.json config | HIGH | LOW | P1 |
| Shared library (libs/shared) | HIGH | MEDIUM | P1 |
| Redis in Docker Compose | HIGH | LOW | P1 |
| Redis env vars + ConfigService | HIGH | LOW | P1 |
| BullModule.forRootAsync() | HIGH | LOW | P1 |
| Queue registration (at least one) | HIGH | LOW | P1 |
| Worker entry point (main.ts) | HIGH | MEDIUM | P1 |
| @Processor smoke-test class | HIGH | LOW | P1 |
| Nodemon worker hot reload | MEDIUM | LOW | P1 |
| End-to-end job enqueue/process proof | HIGH | LOW | P1 |
| Bull Board admin (dev only) | LOW | LOW | P2 |
| DLQ pattern | MEDIUM | MEDIUM | P2 |
| Email queue migration | HIGH | MEDIUM | P2 (v1.5) |
| Queue-per-concern convention | MEDIUM | LOW | P2 |

**Priority key:**
- P1: Must have for v1.4 -- infrastructure foundation
- P2: Should have, add in v1.5 when first real queue use case ships
- P3: Nice to have, future consideration

## Existing Feature Impact Analysis

How the monorepo restructure affects what's already built.

| Existing Feature | Impact | Migration Effort |
|------------------|--------|------------------|
| All API modules (business, customer, job, quote, etc.) | File paths change from `src/` to `apps/api/src/`. All internal imports remain the same thanks to tsconfig path aliases. | LOW -- `nest generate app` handles relocation automatically. |
| Docker Compose (MongoDB) | Additive change only -- Redis service added alongside existing MongoDB service. | LOW -- add service block, no changes to MongoDB config. |
| Nodemon dev reload | Must update watched paths from `src/` to `apps/api/src/`. Or create per-app nodemon configs. | LOW |
| tsconfig path aliases (@auth/*, @business/*, etc.) | These are project-relative and should survive relocation. Verify after conversion. | LOW -- but verify |
| Environment config (.env, ConfigModule) | Additive -- new REDIS_* vars alongside existing MONGODB_*, FIREBASE_*, RESEND_* vars. | LOW |
| npm scripts (start:dev, build, test) | Must be updated to specify which app to run: `nest start api --watch`, `nest start worker --watch`. | LOW |
| Husky pre-commit hooks + lint-staged | Continue to work at repo root level. May need glob updates if lint-staged targets `src/`. | LOW -- verify globs |
| Jest test configuration | Each app gets its own `tsconfig.app.json`. Test config may need `roots` or `moduleNameMapper` updates. | MEDIUM -- test carefully |

## Sources

- [NestJS CLI Monorepo Documentation](https://docs.nestjs.com/cli/monorepo) -- official docs on monorepo mode, nest-cli.json config, generate app/lib commands
- [BullMQ NestJS Guide](https://docs.bullmq.io/guide/nestjs) -- official BullMQ integration pattern for NestJS
- [NestJS Queues Documentation](https://docs.nestjs.com/techniques/queues) -- official NestJS docs on @nestjs/bullmq, BullModule, processors
- [BullMQ Connections Guide](https://docs.bullmq.io/guide/connections) -- Redis connection options, ioredis config, retry strategies
- [BullMQ Going to Production](https://docs.bullmq.io/guide/going-to-production) -- production Redis configuration, maxRetriesPerRequest, enableReadyCheck
- [NestJS Standalone BullMQ Worker](https://medium.com/@omarae00/nestjs-standalone-bullmq-worker-6f44faefaf6b) -- pattern for separate worker NestJS app
- [Running NestJS Queues in a Separate Process](https://medium.com/s1seven/running-nestjs-queues-in-a-separate-process-948f414c4b41) -- architecture for API + worker separation
- [NestJS Monorepo Setup (DEV Community)](https://dev.to/asibul_hasan_5fe57cd945b8/setup-monorepo-in-nestjs-dj4) -- practical walkthrough of nest generate app conversion
- [NestJS Monorepos Without the Meltdown](https://medium.com/@bhagyarana80/nestjs-monorepos-without-the-meltdown-3a155795ea94) -- best practices for NestJS monorepo structure

---
*Feature research for: NestJS monorepo, BullMQ worker service, Redis queue infrastructure*
*Researched: 2026-03-22*
