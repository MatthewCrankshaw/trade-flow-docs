# Phase 20: Infrastructure Foundation - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

All prerequisites for BullMQ queue processing are installed and configured -- Redis runs locally, dependencies are available, and path aliases resolve correctly. No queue module, no worker, no processors -- just the foundation.

</domain>

<decisions>
## Implementation Decisions

### Path alias structure
- **D-01:** `@queue/*` maps to `src/queue/` — follows existing flat alias pattern (`@email/*` → `src/email/`, `@auth/*` → `src/auth/`)
- **D-02:** `@worker/*` maps to `src/worker/` — same convention
- **D-03:** Both aliases added to all three tsconfig files (tsconfig.json, tsconfig.build.json, tsconfig-check.json) and Jest moduleNameMapper in package.json

### Redis Docker configuration
- **D-04:** Redis 7.4 service added to docker-compose.yaml on `app-network` bridge network, exposed on port 6379
- **D-05:** Ephemeral storage (no volume mount) — queue jobs are transient, not persistent data; container restart just means in-flight jobs re-enter the queue
- **D-06:** `maxmemory-policy noeviction` enforced via Redis command override — BullMQ silently loses jobs otherwise (locked decision from roadmap)
- **D-07:** Health check on Redis service using `redis-cli ping` — matches existing MongoDB health check pattern

### REDIS_URL environment config
- **D-08:** Use `.get<string>("REDIS_URL")` with `??` fallback to `redis://localhost:6379` — matches existing ConfigService pattern (no `.getOrThrow()` used anywhere in codebase)
- **D-09:** `REDIS_URL` added to `.env.example` with `redis://localhost:6379` default
- **D-10:** `REDIS_URL` added to docker-compose.yaml api service environment section

### npm dependencies
- **D-11:** Install `@nestjs/bullmq`, `bullmq`, and `ioredis` as production dependencies
- **D-12:** Use latest stable versions compatible with NestJS 11 and Node 22

### Claude's Discretion
- Exact Redis Docker image tag (within 7.4.x)
- Order of operations (deps first vs Docker first)
- Whether to create placeholder `src/queue/` and `src/worker/` directories or leave for later phases

</decisions>

<specifics>
## Specific Ideas

No specific requirements -- standard infrastructure setup following established codebase patterns.

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Infrastructure requirements
- `.planning/REQUIREMENTS.md` — INFRA-01 through INFRA-04 define exact success criteria
- `.planning/ROADMAP.md` §Phase 20 — Success criteria with verification commands

### Existing patterns to follow
- `trade-flow-api/docker-compose.yaml` — Current Docker Compose structure (mongo + api services, app-network)
- `trade-flow-api/tsconfig.json` lines 20-43 — Path alias convention (21 existing aliases)
- `trade-flow-api/package.json` lines 83-118 — Jest moduleNameMapper entries for path aliases
- `trade-flow-api/.env.example` — Environment variable documentation pattern
- `trade-flow-api/src/email/services/email-sender.service.ts` lines 13-14 — ConfigService `.get<string>(key)` pattern

### Prior decisions
- `.planning/STATE.md` §Accumulated Context — Locked decisions: dual entry-point pattern, noeviction policy, createApplicationContext

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- ConfigService injection pattern: constructor injection with `.get<string>(key)` and `??` defaults
- Docker Compose health check pattern: `test`, `interval`, `timeout`, `retries` structure from MongoDB service
- Path alias registration: synchronized across tsconfig.json, tsconfig.build.json, tsconfig-check.json, and Jest moduleNameMapper

### Established Patterns
- All path aliases follow `@module-name/*` → `src/module-name/` convention (no nesting)
- Docker Compose services share `app-network` bridge network
- `.env.example` documents all required environment variables

### Integration Points
- `docker-compose.yaml` — Redis service joins existing network, api service gets REDIS_URL env var
- `tsconfig.json` paths section — New aliases added alongside 21 existing ones
- `package.json` jest.moduleNameMapper — New aliases added alongside 21 existing entries
- `.env.example` — REDIS_URL documented alongside existing vars

</code_context>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope.

</deferred>

---

*Phase: 20-infrastructure-foundation*
*Context gathered: 2026-03-22*
