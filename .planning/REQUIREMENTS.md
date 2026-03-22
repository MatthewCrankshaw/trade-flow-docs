# Requirements: Trade Flow

**Defined:** 2026-03-22
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.4 Requirements

Requirements for Monorepo & Worker Infrastructure milestone. Each maps to roadmap phases.

### Infrastructure

- [x] **INFRA-01**: Redis 7.4 service runs in Docker Compose with `maxmemory-policy noeviction`
- [x] **INFRA-02**: `REDIS_URL` environment variable configured via existing ConfigService pattern
- [x] **INFRA-03**: `@nestjs/bullmq`, `bullmq`, and `ioredis` npm dependencies installed with correct versions
- [x] **INFRA-04**: `@queue/*` and `@worker/*` path aliases added to tsconfig.json, tsconfig.build.json, tsconfig-check.json, and Jest moduleNameMapper

### Queue System

- [x] **QUEUE-01**: Shared `QueueModule` with `BullModule.forRootAsync()` using ConfigService for Redis connection
- [x] **QUEUE-02**: Queue name constants file (`QUEUE_NAMES`) as single source of truth for producer and consumer
- [x] **QUEUE-03**: `QueueModule` imported into `AppModule` -- API connects to Redis and can enqueue jobs

### Worker Service

- [x] **WORK-01**: `src/worker.ts` entry point using `NestFactory.createApplicationContext()` with SIGTERM graceful shutdown
- [x] **WORK-02**: `WorkerModule` imports only CoreModule, ConfigModule, LoggerModule, and QueueModule (not full AppModule)
- [x] **WORK-03**: Echo `@Processor` that logs received jobs, proving end-to-end queue flow
- [x] **WORK-04**: `worker-cli.json` with `entryFile: "worker"` for independent build/start

### Developer Experience

- [x] **DEVX-01**: `nodemon-worker.json` and `npm run worker:dev` script for hot-reload development
- [x] **DEVX-02**: `npm run worker:prod` script for production worker startup
- [ ] **DEVX-03**: Worker runs as separate Docker Compose service in local dev environment
- [ ] **DEVX-04**: Multi-stage Dockerfile updated with worker production stage

## Future Requirements

Deferred to future milestones. Tracked but not in current roadmap.

### Async Job Processing

- **ASYNC-01**: Email sending migrated to background queue (worker processes instead of inline API)
- **ASYNC-02**: PDF generation processed via background queue
- **ASYNC-03**: Job retry with exponential backoff configured per queue
- **ASYNC-04**: `@OnWorkerEvent('completed'/'failed')` logging via AppLogger

### Billing & Subscriptions

- **BILL-01**: Stripe integration for subscription billing
- **BILL-02**: Scheduled payment processing via BullMQ delayed/repeatable jobs
- **BILL-03**: Reminder email scheduling for upcoming payments
- **BILL-04**: Subscription lifecycle management (trial, active, past-due, canceled)

### Queue Monitoring

- **QMON-01**: Bull Board admin UI for queue monitoring
- **QMON-02**: Multiple worker instances with concurrency tuning
- **QMON-03**: Redis persistence configuration (AOF/RDB) for production

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| NestJS CLI monorepo mode (`nest g app`) | Switches to webpack, breaks 20+ path aliases, massive migration for zero benefit |
| Nx or Turborepo | Overkill for 2-app setup; NestJS dual entry-point is simpler |
| Real job processors (email, PDF, billing) | v1.4 is infrastructure only; echo processor proves the pipeline |
| Stripe integration | Future milestone; requires billing design and subscription model decisions |
| Bull Board admin UI | Nice-to-have for monitoring; defer until real jobs exist |
| Redis Sentinel/Cluster | Solo operator scale; single Redis instance is sufficient |
| Shared `libs/` directory restructure | Dual entry-point shares `src/` directly; no restructure needed |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INFRA-01 | Phase 20 | Complete |
| INFRA-02 | Phase 20 | Complete |
| INFRA-03 | Phase 20 | Complete |
| INFRA-04 | Phase 20 | Complete |
| QUEUE-01 | Phase 21 | Complete |
| QUEUE-02 | Phase 21 | Complete |
| QUEUE-03 | Phase 21 | Complete |
| WORK-01 | Phase 22 | Complete |
| WORK-02 | Phase 22 | Complete |
| WORK-03 | Phase 22 | Complete |
| WORK-04 | Phase 22 | Complete |
| DEVX-01 | Phase 23 | Complete |
| DEVX-02 | Phase 23 | Complete |
| DEVX-03 | Phase 23 | Pending |
| DEVX-04 | Phase 23 | Pending |

**Coverage:**
- v1.4 requirements: 15 total
- Mapped to phases: 15
- Unmapped: 0

---
*Requirements defined: 2026-03-22*
*Last updated: 2026-03-22 after roadmap creation*
