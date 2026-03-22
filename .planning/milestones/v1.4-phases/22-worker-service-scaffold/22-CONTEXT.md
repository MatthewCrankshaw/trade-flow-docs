# Phase 22: Worker Service Scaffold - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

A standalone worker process boots as a separate NestJS application context, picks up jobs from the queue, and logs them -- proving end-to-end queue flow from API to worker. Includes a permanent diagnostic endpoint for queue health verification.

</domain>

<decisions>
## Implementation Decisions

### End-to-end proof strategy
- **D-01:** `POST /v1/queue/test-echo` endpoint enqueues a fixed-payload echo job -- no request body, no auth (open like ping)
- **D-02:** Endpoint lives in a new `QueueController` in the queue module -- keeps queue concerns together, separate from PingController
- **D-03:** Both ping and echo-test endpoints throttled at 6 requests per minute using existing throttler infrastructure
- **D-04:** Permanent diagnostic endpoint -- kept as a queue health check, not removed when real jobs arrive
- **D-05:** Fixed payload (no user input) -- worker logs a predetermined message proving the pipeline works

### Processor file organization
- **D-06:** `@Processor` classes live in `src/worker/processors/` -- worker-owned, never imported by API
- **D-07:** Clean split: `src/queue/` is shared infrastructure (module, constants, producer), `src/worker/` is consumer-side code (processors, WorkerModule, entry point)

### Worker log identity
- **D-08:** Pino `service` field distinguishes processes -- worker logs `"service": "worker"`, API logs `"service": "api"`
- **D-09:** Same Pino configuration for both, just the service identifier differs -- enables structured filtering with grep or log tools

### Claude's Discretion
- Exact fixed payload content for the echo job
- Worker bootstrap logging messages (startup, shutdown)
- WorkerModule provider registration order
- Test file structure and mock approach for echo processor

</decisions>

<specifics>
## Specific Ideas

- Echo-test endpoint should feel like the existing ping endpoint -- minimal, diagnostic, no auth
- Throttling at 6 req/min applies to both open endpoints (ping + echo-test) using existing throttler setup

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Worker requirements
- `.planning/REQUIREMENTS.md` -- WORK-01, WORK-02, WORK-03, WORK-04 define exact success criteria
- `.planning/ROADMAP.md` SS Phase 22 -- Success criteria with verification conditions

### Phase 20 foundation (prerequisite)
- `.planning/milestones/v1.4-phases/20-infrastructure-foundation/20-CONTEXT.md` -- D-01/D-02 (path aliases @queue/*, @worker/*), D-08 (ConfigService pattern)

### Phase 21 queue module (prerequisite)
- `.planning/milestones/v1.4-phases/21-queue-module/21-CONTEXT.md` -- D-01 (QueueModule @Global), D-04/D-05 (QueueProducer service), D-06 (QUEUE_NAMES constant)

### Existing patterns to follow
- `trade-flow-api/src/main.ts` -- API bootstrap pattern (NestFactory.create, graceful shutdown, Pino logger setup)
- `trade-flow-api/src/app.module.ts` -- Module import structure, ConfigModule.forRoot() load array
- `trade-flow-api/src/core/core.module.ts` -- @Global() module pattern
- `trade-flow-api/src/ping/` -- Open diagnostic endpoint pattern (no auth, simple response)
- `trade-flow-api/nest-cli.json` -- Existing NestJS CLI config (worker-cli.json follows same structure with entryFile override)

### Throttler
- Existing throttler setup in the codebase -- research agent must locate and follow the existing pattern for 6 req/min on open endpoints

### Prior decisions
- `.planning/STATE.md` SS Accumulated Context -- Locked: dual entry-point pattern, noeviction policy, createApplicationContext

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `AppLogger` from `@core/services/app-logger.service.ts` -- same structured logger for worker
- `ConfigModule.forRoot()` pattern -- worker reuses same env var loading
- `LoggerModule.forRoot(loggerConfig)` -- same Pino config with added `service` field
- Existing throttler infrastructure -- apply to both ping and echo-test endpoints
- `PingController` pattern -- model for open diagnostic endpoints

### Established Patterns
- Bootstrap: `NestFactory.create()` with `bufferLogs: true`, global pipes, SIGTERM/SIGINT handlers
- `@Global()` modules: CoreModule, ConfigModule -- available without explicit import
- Service naming: `[Name][Action]` classes (e.g., `QueueProducer`)
- File naming: `[name]-[action].service.ts` in `services/` subdirectory
- Test files: `[name].spec.ts` in `test/[layer]/` subdirectory

### Integration Points
- `src/worker.ts` -- new entry point alongside `src/main.ts`
- `src/worker/worker.module.ts` -- new module importing CoreModule, ConfigModule, LoggerModule, QueueModule
- `src/worker/processors/` -- echo processor consuming from QUEUE_NAMES.ECHO
- `src/queue/controllers/queue.controller.ts` -- new QueueController with POST /v1/queue/test-echo
- `worker-cli.json` -- new NestJS CLI config with `entryFile: "worker"`
- `nest-cli.json` -- reference for worker-cli.json structure

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope.

</deferred>

---

*Phase: 22-worker-service-scaffold*
*Context gathered: 2026-03-22*
