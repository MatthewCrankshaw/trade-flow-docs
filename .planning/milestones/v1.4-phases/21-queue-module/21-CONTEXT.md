# Phase 21: Queue Module - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

A shared queue module connects the API to Redis via BullMQ, with queue name constants as a single source of truth for producers and consumers. No processors, no worker — just the shared infrastructure that both API (producer) and worker (consumer) will import.

</domain>

<decisions>
## Implementation Decisions

### Module scope and structure
- **D-01:** `QueueModule` is `@Global()` — any module can inject `QueueProducer` without explicit imports (same pattern as `CoreModule`)
- **D-02:** QueueModule exports only `QueueProducer` service — no processor classes on the API side. Processors live exclusively in the worker (Phase 22)
- **D-03:** QueueModule contains `BullModule.forRootAsync()` (Redis connection) and `BullModule.registerQueue()` (queue registration) in one module

### QueueProducer service
- **D-04:** Single `QueueProducer` service with typed methods per queue (e.g., `enqueueEcho(payload)`) — central place to discover all queue operations
- **D-05:** QueueProducer injects `@InjectQueue(QUEUE_NAMES.ECHO)` internally — consumers of the service don't need to know queue names

### Queue naming
- **D-06:** `QUEUE_NAMES` typed const object in `src/queue/queue.constant.ts`: `export const QUEUE_NAMES = { ECHO: 'echo' } as const;`
- **D-07:** Only `ECHO` queue defined for Phase 21 — future queues (EMAIL, PDF) added in their respective milestones

### Redis connection configuration
- **D-08:** `REDIS_URL` added to `ConfigModule.forRoot()` load array in `AppModule`, alongside existing `MONGO_URL` — consistent env var loading
- **D-09:** `BullModule.forRootAsync()` uses `ConfigService.get<string>("REDIS_URL")` — NO fallback default. Fail fast if REDIS_URL is not set. Developers must configure it explicitly
- **D-10:** QueueModule imported into `AppModule` — API connects to Redis on startup

### Claude's Discretion
- Exact error message when REDIS_URL is missing
- Whether to use `useFactory` or `useClass` for BullModule.forRootAsync() configuration
- Test structure and mock approach for QueueProducer

</decisions>

<specifics>
## Specific Ideas

No specific requirements — standard NestJS BullMQ module setup following established codebase patterns.

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Queue requirements
- `.planning/REQUIREMENTS.md` — QUEUE-01, QUEUE-02, QUEUE-03 define exact success criteria
- `.planning/ROADMAP.md` §Phase 21 — Success criteria with verification conditions

### Phase 20 foundation (prerequisite)
- `.planning/milestones/v1.4-phases/20-infrastructure-foundation/20-CONTEXT.md` — Decisions D-08 (ConfigService pattern), D-01/D-02 (path aliases)
- `.planning/milestones/v1.4-phases/20-infrastructure-foundation/20-01-SUMMARY.md` — What was built: BullMQ deps + path aliases
- `.planning/milestones/v1.4-phases/20-infrastructure-foundation/20-02-SUMMARY.md` — What was built: Redis Docker + REDIS_URL

### Existing patterns to follow
- `trade-flow-api/src/app.module.ts` — AppModule structure, ConfigModule.forRoot() load array, module import pattern
- `trade-flow-api/src/core/core.module.ts` — @Global() module pattern (providers + exports)
- `trade-flow-api/src/email/services/email-sender.service.ts` lines 13-14 — ConfigService `.get<string>(key)` injection pattern
- `trade-flow-api/tsconfig.json` lines 43-44 — @queue/* and @worker/* path aliases (from Phase 20)

### Prior decisions
- `.planning/STATE.md` §Accumulated Context — Locked: dual entry-point, noeviction, createApplicationContext

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ConfigService` injection: constructor injection with `.get<string>(key)` — used in EmailSender, will be used in BullModule.forRootAsync()
- `@Global()` module pattern: CoreModule — same approach for QueueModule
- `@queue/*` path alias already registered (Phase 20) — import paths ready

### Established Patterns
- Module naming: `[Name]Module` class in `[name].module.ts` file
- Service naming: `[Name][Action]` class — QueueProducer follows this
- Constants: `[name].constant.ts` with UPPER_SNAKE_CASE object keys and lowercase string values
- `ConfigModule.forRoot({ isGlobal: true })` — ConfigService available everywhere without import

### Integration Points
- `AppModule` imports array — QueueModule added alongside existing 15 module imports
- `ConfigModule.forRoot()` load callback — REDIS_URL added alongside MONGO_URL
- `src/queue/` directory (placeholder from Phase 20) — QueueModule files live here

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 21-queue-module*
*Context gathered: 2026-03-22*
