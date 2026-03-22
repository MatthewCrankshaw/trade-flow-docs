# Phase 23: Developer Experience - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

Both API and worker can be developed and deployed independently with hot reload, production scripts, and containerized local development. No new queue processors, no new features -- just the developer and deployment tooling to make the existing dual entry-point setup usable day-to-day.

</domain>

<decisions>
## Implementation Decisions

### Docker Compose worker service
- **D-01:** Worker runs as a separate Docker Compose service with its own development target and volume mounts -- fully isolated from the API service
- **D-02:** Worker gets trimmed environment variables: only `MONGO_URL`, `REDIS_URL`, `NODE_ENV` -- no PORT, CORS_ORIGIN, FIREBASE_*, or RESEND_API_KEY
- **D-03:** Worker skips `cap_add: SYS_ADMIN` and `security_opt: seccomp=unconfined` -- those are for Playwright in the API service
- **D-04:** Container name `trade-flow-worker` following existing pattern (`trade-flow-api`, `trade-flow-db`, `trade-flow-redis`)
- **D-05:** Worker depends on `mongo` and `redis` (same as API), mounts source and runs `npm run worker:dev`

### Dockerfile worker stages
- **D-06:** Builder stage runs both `npm run build` (API) and `nest build --config worker-cli.json` (worker) sequentially -- produces both `dist/main.js` and `dist/worker.js`
- **D-07:** No separate `worker-development` Dockerfile stage -- Docker Compose worker service mounts source and runs `npm run worker:dev` from a generic Node image
- **D-08:** New `worker` production stage: copies `dist/` from builder, uses `node:22-slim` base (same as API production), runs `node dist/worker.js`
- **D-09:** No port exposed on worker production stage -- worker has no HTTP server

### Nodemon worker config
- **D-10:** New `nodemon-worker.json` file for worker hot reload
- **D-11:** `npm run worker:dev` script uses `nodemon --config nodemon-worker.json`
- **D-12:** `npm run worker:prod` script runs `node dist/worker.js`

### Claude's Discretion
- Whether worker nodemon uses `--inspect=9230` (different debug port) or skips `--inspect` entirely
- Watch scope: full `src/` or narrowed to `src/worker/`, `src/queue/`, `src/core/`
- Delay and signal settings (likely same 1500ms / SIGTERM as API)
- Exec command format (`node -r ts-node/register src/worker.ts` vs `ts-node src/worker.ts`)
- Whether to add a `build:worker` or `build:all` convenience script
- Worker Docker Compose development image choice (reuse `development` stage or plain `node:22`)

</decisions>

<specifics>
## Specific Ideas

- Worker Docker Compose service should be minimal and clean -- no Playwright/browser baggage
- Worker only needs database and Redis connectivity, not HTTP-facing config

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Developer experience requirements
- `.planning/REQUIREMENTS.md` -- DEVX-01, DEVX-02, DEVX-03, DEVX-04 define exact success criteria
- `.planning/ROADMAP.md` §Phase 23 -- Success criteria with verification conditions

### Phase 20 foundation (prerequisite)
- `.planning/milestones/v1.4-phases/20-infrastructure-foundation/20-CONTEXT.md` -- D-04 through D-07 (Redis Docker config), D-08/D-09 (REDIS_URL env pattern)

### Phase 21 queue module (prerequisite)
- `.planning/milestones/v1.4-phases/21-queue-module/21-CONTEXT.md` -- D-01 (QueueModule @Global), D-08/D-10 (Redis connection in AppModule)

### Phase 22 worker scaffold (prerequisite)
- `.planning/milestones/v1.4-phases/22-worker-service-scaffold/22-CONTEXT.md` -- D-06/D-07 (file organization), D-08/D-09 (worker log identity)

### Existing files to modify/reference
- `trade-flow-api/package.json` -- Scripts section (add worker:dev, worker:prod)
- `trade-flow-api/nodemon.json` -- Existing API nodemon config (template for nodemon-worker.json)
- `trade-flow-api/docker-compose.yaml` -- Current 3-service structure (add worker service)
- `trade-flow-api/Dockerfile` -- Current 3-stage build (add worker production stage, update builder)
- `trade-flow-api/worker-cli.json` -- Worker NestJS CLI config (Phase 22, entryFile: "worker")
- `trade-flow-api/nest-cli.json` -- API NestJS CLI config (reference for builder stage)
- `trade-flow-api/src/worker.ts` -- Worker entry point (Phase 22)

### Prior decisions
- `.planning/STATE.md` §Accumulated Context -- Locked: dual entry-point pattern, deleteOutDir:false in worker-cli.json

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `nodemon.json`: Template for worker config -- same structure, different exec command and potentially different watch/debug settings
- `docker-compose.yaml` api service: Template for worker service definition -- same depends_on pattern, different env vars and command
- `Dockerfile` production stage: Template for worker production stage -- same base image, different CMD

### Established Patterns
- Docker Compose service naming: `trade-flow-{role}` (db, redis, api, worker)
- npm script naming: `start:dev`, `start:prod` for API -- `worker:dev`, `worker:prod` for worker
- Nodemon config: separate JSON file per entry point
- Dockerfile multi-stage: builder compiles, runtime stages copy dist/
- NestJS CLI: separate config files per entry point (nest-cli.json, worker-cli.json)

### Integration Points
- `package.json` scripts -- new worker:dev and worker:prod scripts
- `docker-compose.yaml` -- new worker service alongside existing 3 services
- `Dockerfile` -- builder stage updated to build both, new worker production stage
- `nodemon-worker.json` -- new file (not modifying existing nodemon.json)

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope.

</deferred>

---

*Phase: 23-developer-experience*
*Context gathered: 2026-03-22*
