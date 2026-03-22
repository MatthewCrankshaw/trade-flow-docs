# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2020-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2020-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2020-03-15)
- v1.3 Send Quotes -- Phases 15-19 (shipped 2026-03-21)
- v1.4 Monorepo & Worker Infrastructure -- Phases 20-23 (in progress)

## Phases

<details>
<summary>v1.0 Scheduling (Phases 1-8) -- SHIPPED 2020-03-07</summary>

- [x] Phase 1: Visit Type Backend (2/2 plans) -- completed 2020-02-23
- [x] Phase 2: Visit Type Management UI (2/2 plans) -- completed 2020-02-28
- [x] Phase 3: Schedule Data Model and Create API (2/2 plans) -- completed 2020-03-01
- [x] Phase 4: Schedule Status and CRUD API (3/3 plans) -- completed 2020-03-01
- [x] Phase 5: Schedule Creation UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 6: Schedule List and Detail UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 7: Schedule Edit and Management UI (2/2 plans) -- completed 2020-03-07
- [x] Phase 8: Job Detail Integration (1/1 plan) -- completed 2020-03-07

Full details: `.planning/milestones/v1.0-ROADMAP.md`

</details>

<details>
<summary>v1.1 Item Tax Rate Linkage (Phases 9-10) -- SHIPPED 2020-03-08</summary>

- [x] Phase 9: Item Tax Rate API (4/4 plans) -- completed 2020-03-08
- [x] Phase 10: Item Tax Rate UI (2/2 plans) -- completed 2020-03-08

Full details: `.planning/milestones/v1.1-ROADMAP.md`

</details>

<details>
<summary>v1.2 Bundles & Quotes (Phases 11-14) -- SHIPPED 2020-03-15</summary>

- [x] Phase 11: Bundle Bug Fix and Foundation (1/1 plan) -- completed 2020-03-08
- [x] Phase 12: Bundle Component Editing (2/2 plans) -- completed 2020-03-08
- [x] Phase 13: Quote API Integration (3/3 plans) -- completed 2020-03-14
- [x] Phase 14: Quote Detail and Line Items (6/6 plans) -- completed 2020-03-14

Full details: `.planning/milestones/v1.2-ROADMAP.md`

</details>

<details>
<summary>v1.3 Send Quotes (Phases 15-19) -- SHIPPED 2026-03-21</summary>

- [x] Phase 15: Quote Deletion (2/2 plans) -- completed 2026-03-15
- [x] Phase 16: Token Infrastructure and Public API (2/2 plans) -- completed 2026-03-15
- [x] Phase 17: Customer Quote Page (4/4 plans) -- completed 2026-03-20
- [x] Phase 18: Quote Email Sending (7/7 plans) -- completed 2026-03-21
- [x] Phase 19: Customer Response (3/3 plans) -- completed 2026-03-21

Full details: `.planning/milestones/v1.3-ROADMAP.md`

</details>

### v1.4 Monorepo & Worker Infrastructure (In Progress)

**Milestone Goal:** Prepare trade-flow-api as a monorepo with two independently deployable NestJS services -- the existing API and a new background worker -- connected via BullMQ/Redis.

- [ ] **Phase 20: Infrastructure Foundation** - Redis, npm dependencies, path aliases, and environment config
- [ ] **Phase 21: Queue Module** - Shared BullMQ configuration and queue name constants wired into API
- [ ] **Phase 22: Worker Service Scaffold** - Worker entry point, WorkerModule, and echo processor proving end-to-end flow
- [ ] **Phase 23: Developer Experience** - Hot reload, production scripts, Docker Compose worker service, and Dockerfile stage

## Phase Details

### Phase 20: Infrastructure Foundation
**Goal**: All prerequisites for BullMQ queue processing are installed and configured -- Redis runs locally, dependencies are available, and path aliases resolve correctly
**Depends on**: Nothing (first phase of v1.4)
**Requirements**: INFRA-01, INFRA-02, INFRA-03, INFRA-04
**Success Criteria** (what must be TRUE):
  1. `docker compose up` starts a Redis 7.4 container alongside MongoDB with `maxmemory-policy noeviction` confirmed via `redis-cli CONFIG GET maxmemory-policy`
  2. `REDIS_URL` environment variable is read via ConfigService and documented in `.env.example`
  3. `@nestjs/bullmq`, `bullmq`, and `ioredis` are installed and `npm run validate` passes with zero TypeScript errors
  4. Imports using `@queue/*` and `@worker/*` path aliases compile successfully across `tsc`, Jest, and NestJS build
**Plans**: TBD

Plans:
- [ ] 20-01: TBD
- [ ] 20-02: TBD

### Phase 21: Queue Module
**Goal**: A shared queue module connects the API to Redis via BullMQ, with queue name constants as a single source of truth for producers and consumers
**Depends on**: Phase 20
**Requirements**: QUEUE-01, QUEUE-02, QUEUE-03
**Success Criteria** (what must be TRUE):
  1. `src/queue/queue.module.ts` exports `BullModule.forRootAsync()` configured via ConfigService for Redis connection
  2. Queue name constants in `src/queue/queue.constants.ts` define at least one queue name used by both producer and consumer code
  3. The existing API (`npm run start:dev`) boots without errors with QueueModule imported into AppModule, and connects to Redis on startup
**Plans**: TBD

Plans:
- [ ] 21-01: TBD

### Phase 22: Worker Service Scaffold
**Goal**: A standalone worker process boots as a separate NestJS application context, picks up jobs from the queue, and logs them -- proving end-to-end queue flow
**Depends on**: Phase 21
**Requirements**: WORK-01, WORK-02, WORK-03, WORK-04
**Success Criteria** (what must be TRUE):
  1. `src/worker.ts` boots via `NestFactory.createApplicationContext(WorkerModule)` without starting an HTTP server, and shuts down cleanly on SIGTERM
  2. `WorkerModule` imports only CoreModule, ConfigModule, LoggerModule, and QueueModule -- not AppModule or any HTTP-related modules
  3. An echo `@Processor` receives a test job enqueued by the API and logs the payload, proving end-to-end queue flow from API to worker
  4. `nest build --config worker-cli.json` produces a separate `dist/worker.js` entry point that runs independently of `dist/main.js`
**Plans**: TBD

Plans:
- [ ] 22-01: TBD
- [ ] 22-02: TBD

### Phase 23: Developer Experience
**Goal**: Both API and worker can be developed and deployed independently with hot reload, production scripts, and containerized local development
**Depends on**: Phase 22
**Requirements**: DEVX-01, DEVX-02, DEVX-03, DEVX-04
**Success Criteria** (what must be TRUE):
  1. `npm run worker:dev` starts the worker with nodemon hot reload -- saving a file in `src/` triggers automatic restart
  2. `npm run worker:prod` starts the worker from compiled `dist/worker.js` for production deployment
  3. `docker compose up` starts API, worker, MongoDB, and Redis as four separate services, with worker processing jobs enqueued by the API
  4. Multi-stage Dockerfile builds a production worker image that runs `node dist/worker.js` without dev dependencies
**Plans**: TBD

Plans:
- [ ] 23-01: TBD
- [ ] 23-02: TBD

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Visit Type Backend | v1.0 | 2/2 | Complete | 2020-02-23 |
| 2. Visit Type Management UI | v1.0 | 2/2 | Complete | 2020-02-28 |
| 3. Schedule Data Model and Create API | v1.0 | 2/2 | Complete | 2020-03-01 |
| 4. Schedule Status and CRUD API | v1.0 | 3/3 | Complete | 2020-03-01 |
| 5. Schedule Creation UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 6. Schedule List and Detail UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 7. Schedule Edit and Management UI | v1.0 | 2/2 | Complete | 2020-03-07 |
| 8. Job Detail Integration | v1.0 | 1/1 | Complete | 2020-03-07 |
| 9. Item Tax Rate API | v1.1 | 4/4 | Complete | 2020-03-08 |
| 10. Item Tax Rate UI | v1.1 | 2/2 | Complete | 2020-03-08 |
| 11. Bundle Bug Fix and Foundation | v1.2 | 1/1 | Complete | 2020-03-08 |
| 12. Bundle Component Editing | v1.2 | 2/2 | Complete | 2020-03-08 |
| 13. Quote API Integration | v1.2 | 3/3 | Complete | 2020-03-14 |
| 14. Quote Detail and Line Items | v1.2 | 6/6 | Complete | 2020-03-14 |
| 15. Quote Deletion | v1.3 | 2/2 | Complete | 2026-03-15 |
| 16. Token Infrastructure and Public API | v1.3 | 2/2 | Complete | 2026-03-15 |
| 17. Customer Quote Page | v1.3 | 4/4 | Complete | 2026-03-20 |
| 18. Quote Email Sending | v1.3 | 7/7 | Complete | 2026-03-21 |
| 19. Customer Response | v1.3 | 3/3 | Complete | 2026-03-21 |
| 20. Infrastructure Foundation | v1.4 | 0/? | Not started | - |
| 21. Queue Module | v1.4 | 0/? | Not started | - |
| 22. Worker Service Scaffold | v1.4 | 0/? | Not started | - |
| 23. Developer Experience | v1.4 | 0/? | Not started | - |
