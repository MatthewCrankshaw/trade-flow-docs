# Phase 20: Infrastructure Foundation - Research

**Researched:** 2026-03-22
**Domain:** Docker/Redis infrastructure, NestJS BullMQ dependencies, TypeScript path aliases
**Confidence:** HIGH

## Summary

Phase 20 is a pure infrastructure setup phase -- no application logic, no queue modules, no workers. It installs three npm packages, adds a Redis container to Docker Compose, registers a new environment variable, and creates two path aliases across four configuration surfaces (tsconfig.json, tsconfig.build.json, tsconfig-check.json, Jest moduleNameMapper).

The codebase already has 21 path aliases registered across all four surfaces, a Docker Compose file with MongoDB + API services on a shared bridge network, and a ConfigService pattern using `.get<string>(key) ?? fallback`. Every task in this phase follows an existing pattern -- there is nothing novel.

**Primary recommendation:** Follow existing codebase patterns exactly. Every change in this phase has a direct precedent in the current codebase.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `@queue/*` maps to `src/queue/` -- follows existing flat alias pattern
- **D-02:** `@worker/*` maps to `src/worker/` -- same convention
- **D-03:** Both aliases added to all three tsconfig files and Jest moduleNameMapper in package.json
- **D-04:** Redis 7.4 service added to docker-compose.yaml on `app-network` bridge network, exposed on port 6379
- **D-05:** Ephemeral storage (no volume mount) -- queue jobs are transient
- **D-06:** `maxmemory-policy noeviction` enforced via Redis command override
- **D-07:** Health check on Redis service using `redis-cli ping`
- **D-08:** Use `.get<string>("REDIS_URL")` with `??` fallback to `redis://localhost:6379`
- **D-09:** `REDIS_URL` added to `.env.example` with `redis://localhost:6379` default
- **D-10:** `REDIS_URL` added to docker-compose.yaml api service environment section
- **D-11:** Install `@nestjs/bullmq`, `bullmq`, and `ioredis` as production dependencies
- **D-12:** Use latest stable versions compatible with NestJS 11 and Node 22

### Claude's Discretion
- Exact Redis Docker image tag (within 7.4.x)
- Order of operations (deps first vs Docker first)
- Whether to create placeholder `src/queue/` and `src/worker/` directories or leave for later phases

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| INFRA-01 | Redis 7.4 service runs in Docker Compose with `maxmemory-policy noeviction` | Docker Compose redis service config with command override; verified Redis 7.4.8 image available |
| INFRA-02 | `REDIS_URL` environment variable configured via existing ConfigService pattern | Existing `configService.get<string>(key) ?? fallback` pattern found in 3 services; .env.example pattern documented |
| INFRA-03 | `@nestjs/bullmq`, `bullmq`, and `ioredis` npm dependencies installed with correct versions | Versions verified via npm registry: @nestjs/bullmq@11.0.4, bullmq@5.71.0, ioredis@5.10.1; peer deps compatible with NestJS 11 |
| INFRA-04 | `@queue/*` and `@worker/*` path aliases added to tsconfig.json, tsconfig.build.json, tsconfig-check.json, and Jest moduleNameMapper | All four config surfaces documented with exact existing patterns; 21 aliases already registered as precedent |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/bullmq | 11.0.4 | NestJS integration for BullMQ queues | Official NestJS queue package; peer deps support NestJS 10/11 |
| bullmq | 5.71.0 | Redis-backed job queue library | Industry standard for Node.js job queues; used by @nestjs/bullmq |
| ioredis | 5.10.1 | Redis client for Node.js | Required by bullmq (bundled as dep); explicit install for type access and direct Redis operations |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| redis:7.4 (Docker) | 7.4.8 | Redis server container | Local development via Docker Compose |

### Alternatives Considered
None -- all three packages are locked decisions from CONTEXT.md.

**Installation:**
```bash
npm install @nestjs/bullmq bullmq ioredis
```

**Version verification:** All versions confirmed via `npm view [package] version` on 2026-03-22. @nestjs/bullmq peerDependencies: `@nestjs/common ^10.0.0 || ^11.0.0`, `@nestjs/core ^10.0.0 || ^11.0.0`, `bullmq ^3.0.0 || ^4.0.0 || ^5.0.0`. Current project uses NestJS 11.1.x -- fully compatible.

## Architecture Patterns

### Existing Path Alias Pattern (replicate exactly)

All four surfaces must stay in sync. The project has 21 aliases following this exact pattern:

**tsconfig.json** (line 20-43):
```json
"paths": {
  "@queue/*": ["./src/queue/*"],
  "@worker/*": ["./src/worker/*"]
}
```

**tsconfig.build.json**: Inherits from tsconfig.json via `"extends": "./tsconfig.json"` -- no paths section needed (aliases inherited automatically).

**tsconfig-check.json**: Also inherits from tsconfig.json via `"extends": "./tsconfig.json"` -- no paths section needed (aliases inherited automatically).

**package.json Jest moduleNameMapper** (line 94-117):
```json
"^@queue/(.*)$": "<rootDir>/queue/$1",
"^@worker/(.*)$": "<rootDir>/worker/$1"
```

### Existing Docker Compose Pattern (replicate exactly)

Current structure: `mongo` + `api` services on `app-network` bridge. Redis follows the same pattern:

```yaml
redis:
  container_name: trade-flow-redis
  image: redis:7.4-alpine
  ports:
    - "6379:6379"
  command: redis-server --maxmemory-policy noeviction
  networks:
    - app-network
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

Key details:
- Use `redis:7.4-alpine` (smaller image, sufficient for development)
- `command:` override sets `maxmemory-policy noeviction` without a custom redis.conf file
- No volume mount (D-05: ephemeral storage)
- Health check matches MongoDB pattern structure (`interval`, `timeout`, `retries`)
- Container name follows convention: `trade-flow-redis`

### Existing ConfigService Pattern (replicate exactly)

From `src/email/services/email-sender.service.ts` and `src/quote/services/quote-email-sender.service.ts`:
```typescript
const value = this.configService.get<string>("RESEND_API_KEY");
const appUrl = this.configService.get<string>("CORS_ORIGIN") ?? "https://tradeflowhub.uk";
```

For REDIS_URL:
```typescript
const redisUrl = this.configService.get<string>("REDIS_URL") ?? "redis://localhost:6379";
```

### Existing .env.example Pattern

Current file has grouped sections with comments. Add:
```
# Redis Configuration
REDIS_URL=redis://localhost:6379
```

### Docker Compose API Environment Pattern

Current api service environment section passes env vars. Add:
```yaml
REDIS_URL: redis://redis:6379
```
Note: Uses Docker service name `redis` (not `localhost`) for container-to-container communication.

### Anti-Patterns to Avoid
- **Forgetting Jest moduleNameMapper:** Path aliases compile with `tsc` but tests fail at runtime. All four surfaces must be updated together.
- **Using localhost in Docker Compose REDIS_URL:** Containers communicate via service names on the bridge network. The api service must use `redis://redis:6379`, not `redis://localhost:6379`.
- **Adding a redis.conf file:** Overkill for a single config flag. Use `command:` override instead.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Redis connection parsing | Custom URL parser | ioredis built-in URL parsing | ioredis natively accepts `redis://` URLs |
| Redis health monitoring | Custom ping loop | Docker Compose healthcheck | Built-in orchestration; `depends_on: condition: service_healthy` |
| Path alias registration | Manual webpack/jest config | tsconfig paths + ts-jest | Project already has this pattern working for 21 aliases |

## Common Pitfalls

### Pitfall 1: tsconfig.build.json and tsconfig-check.json path inheritance
**What goes wrong:** Developer adds paths only to tsconfig.json, assumes build and check configs need separate entries
**Why it happens:** Not realizing both files use `"extends": "./tsconfig.json"` which inherits paths
**How to avoid:** Verify both files use `extends` -- they do. Only add to tsconfig.json and Jest moduleNameMapper.
**Warning signs:** Duplicate path entries across tsconfig files (maintenance burden, divergence risk)

### Pitfall 2: Docker Compose depends_on without health check
**What goes wrong:** API starts before Redis is ready, first BullMQ connection attempt fails
**Why it happens:** `depends_on` alone only waits for container start, not service readiness
**How to avoid:** Add `healthcheck` to redis service, use `depends_on: redis: condition: service_healthy` on api service
**Warning signs:** Intermittent "ECONNREFUSED" errors on first startup

### Pitfall 3: Forgetting REDIS_URL in Docker Compose environment
**What goes wrong:** API inside Docker uses fallback `redis://localhost:6379` which points to nothing inside the container
**Why it happens:** `.env` file works locally but Docker Compose needs explicit environment mapping
**How to avoid:** Set `REDIS_URL: redis://redis:6379` in docker-compose.yaml api service environment section (D-10)
**Warning signs:** Works locally with `npm run start:dev` but fails inside Docker

### Pitfall 4: maxmemory-policy not persisting
**What goes wrong:** Policy appears set but reverts after container restart
**Why it happens:** Using `redis-cli CONFIG SET` at runtime instead of `command:` override in Docker Compose
**How to avoid:** Set policy via `command: redis-server --maxmemory-policy noeviction` in docker-compose.yaml
**Warning signs:** `redis-cli CONFIG GET maxmemory-policy` returns different value after `docker compose down && docker compose up`

## Code Examples

### Path alias addition to tsconfig.json
```json
// Source: Existing pattern in trade-flow-api/tsconfig.json lines 20-42
// Add to the "paths" object alongside existing 21 aliases:
"@queue/*": ["./src/queue/*"],
"@worker/*": ["./src/worker/*"]
```

### Jest moduleNameMapper addition to package.json
```json
// Source: Existing pattern in trade-flow-api/package.json lines 94-116
// Add to the "moduleNameMapper" object:
"^@queue/(.*)$": "<rootDir>/queue/$1",
"^@worker/(.*)$": "<rootDir>/worker/$1"
```

### Redis Docker Compose service
```yaml
# Source: Follows existing mongo service pattern in docker-compose.yaml
redis:
  container_name: trade-flow-redis
  image: redis:7.4-alpine
  ports:
    - "6379:6379"
  command: redis-server --maxmemory-policy noeviction
  networks:
    - app-network
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### API service depends_on update
```yaml
# Add redis dependency alongside existing mongo dependency
depends_on:
  mongo:
    condition: service_healthy
  redis:
    condition: service_healthy
```

### API service environment update
```yaml
# Add to existing environment section
REDIS_URL: redis://redis:6379
```

### .env.example addition
```bash
# Redis Configuration
REDIS_URL=redis://localhost:6379
```

### Placeholder directories (discretion: recommend creating them)
```
src/queue/.gitkeep
src/worker/.gitkeep
```
Recommendation: Create placeholder directories with `.gitkeep` files so that the path aliases resolve to real directories. This prevents confusing "module not found" errors if someone accidentally imports from `@queue/*` or `@worker/*` before Phase 21/22. Phase 21 will replace `.gitkeep` with actual module files.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@nestjs/bull` (Bull v3) | `@nestjs/bullmq` (BullMQ v5) | 2023 | BullMQ is the maintained successor; Bull is in maintenance mode |
| `bull` package | `bullmq` package | 2023 | BullMQ has better TypeScript support, flows, and worker patterns |
| Redis 6.x | Redis 7.4 | 2024 | Better memory management, ACL improvements; 7.4 is current stable |

**Deprecated/outdated:**
- `@nestjs/bull`: Still works but routes to Bull v3 which is maintenance-only. Use `@nestjs/bullmq` for new projects.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | package.json `"jest"` section |
| Quick run command | `npm run typecheck` |
| Full suite command | `npm run validate` (typecheck + lint) |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| INFRA-01 | Redis container starts with noeviction policy | smoke | `docker compose up -d redis && docker exec trade-flow-redis redis-cli CONFIG GET maxmemory-policy` | N/A (manual Docker verification) |
| INFRA-02 | REDIS_URL in .env.example and docker-compose environment | manual-only | Visual inspection of .env.example and docker-compose.yaml | N/A |
| INFRA-03 | Dependencies installed, no TypeScript errors | unit | `npm run validate` | Existing infrastructure |
| INFRA-04 | Path aliases compile in tsc and Jest | unit | `npm run validate && npm run test` | Existing infrastructure |

### Sampling Rate
- **Per task commit:** `npm run validate`
- **Per wave merge:** `npm run validate && npm run test`
- **Phase gate:** `npm run validate` green + `docker compose up -d` starts Redis + `redis-cli CONFIG GET maxmemory-policy` returns `noeviction`

### Wave 0 Gaps
None -- existing test infrastructure (`npm run validate`, `npm run test`) covers all automated verification for this phase. Docker smoke tests are manual.

## Open Questions

1. **Placeholder directories vs deferred creation**
   - What we know: Path aliases pointing to non-existent directories will not cause TypeScript errors (tsc only resolves paths on import). Jest moduleNameMapper also only resolves on import.
   - What's unclear: Whether the planner should create placeholder directories now or defer to Phase 21/22
   - Recommendation: Create `src/queue/.gitkeep` and `src/worker/.gitkeep` to make the aliases immediately resolvable. Low cost, prevents confusion.

## Sources

### Primary (HIGH confidence)
- trade-flow-api/docker-compose.yaml -- Current Docker Compose structure (read directly)
- trade-flow-api/tsconfig.json -- Path alias convention with 21 existing aliases (read directly)
- trade-flow-api/tsconfig.build.json -- Extends tsconfig.json (read directly)
- trade-flow-api/tsconfig-check.json -- Extends tsconfig.json (read directly)
- trade-flow-api/package.json -- Jest moduleNameMapper with 21 existing entries (read directly)
- trade-flow-api/.env.example -- Environment variable documentation pattern (read directly)
- trade-flow-api/src/email/services/email-sender.service.ts -- ConfigService `.get<string>()` pattern (grep verified)
- npm registry -- @nestjs/bullmq@11.0.4, bullmq@5.71.0, ioredis@5.10.1 versions verified
- npm registry -- @nestjs/bullmq peerDependencies verified (NestJS 10/11, BullMQ 3/4/5)
- Docker Hub -- redis:7.4.8 image verified available

### Secondary (MEDIUM confidence)
- [NestJS BullMQ docs](https://docs.bullmq.io/guide/nestjs) -- BullModule.forRoot() synchronous pattern
- [NestJS Bull GitHub](https://github.com/nestjs/bull/blob/master/packages/bullmq/lib/bull.module.ts) -- forRootAsync method signature and SharedBullAsyncConfiguration interface
- [BullMQ in NestJS tutorial](https://dev.to/ronak_navadia/level-up-your-nestjs-app-with-bullmq-queues-dlqs-bull-board-5hnn) -- forRootAsync + ConfigService pattern
- [Redis eviction docs](https://redis.io/docs/latest/develop/reference/eviction/) -- noeviction policy behavior
- [Redis Docker memory config](https://peterkellner.net/2023-09-24-managing-redis-memory-limits-with-docker-compose/) -- command override pattern for maxmemory-policy

### Tertiary (LOW confidence)
None -- all findings verified against primary or secondary sources.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- versions verified against npm registry, peer deps confirmed compatible
- Architecture: HIGH -- every pattern has a direct precedent in the existing codebase (read and verified)
- Pitfalls: HIGH -- pitfalls derived from known Docker networking behavior and observed codebase patterns

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable infrastructure; package versions may increment but patterns won't change)
