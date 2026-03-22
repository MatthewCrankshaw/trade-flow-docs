# Stack Research

**Domain:** Monorepo restructure, BullMQ worker infrastructure, Redis backing store
**Researched:** 2026-03-22
**Confidence:** HIGH

## Scope

This research covers ONLY the new stack additions for v1.4. The existing validated stack (NestJS 11, MongoDB 7, Firebase Auth, Resend, class-validator, Docker Compose) is not re-evaluated.

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| NestJS Monorepo Mode | (CLI 11.x) | Multi-app workspace with shared code | Built into @nestjs/cli -- no extra tooling. `nest g app worker` converts the project automatically. Single `node_modules`, single `package.json`, independent build/deploy per app. |
| @nestjs/bullmq | ^11.0.4 | NestJS integration for BullMQ queues | Official NestJS package, version-aligned with NestJS 11. Provides `BullModule.forRoot()`, `@Processor()` decorator, `WorkerHost` base class. |
| bullmq | ^5.71.0 | Job queue library (peer dep of @nestjs/bullmq) | The modern successor to Bull. Required peer dependency (`^5.40.0`). Lua-based atomic operations, TypeScript-first, active maintenance. |
| ioredis | ^5.6.1 | Redis client (peer dep of bullmq) | BullMQ uses ioredis internally. Must be installed as a peer dependency. Connection options passed through BullMQ config. |
| Redis (Docker) | 7.4-alpine | Local development Redis server | Redis 7.4 is the proven stable branch. BullMQ requires minimum 6.2.0, recommends 7.0+. Use 7.4 over 8.x because 8.x is very new (released late 2025) and untested with BullMQ at scale. Alpine variant for smaller image. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @nestjs/config | ^4.0.2 (existing) | Environment variable management | Already installed. Use to configure `REDIS_HOST`, `REDIS_PORT` env vars for BullMQ connection. |
| nodemon | ^3.1.9 (existing) | Hot reload for worker service | Already installed. Create a second `nodemon-worker.json` config watching `apps/worker/src/`. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| @nestjs/cli | ^11.0.16 (existing) | Monorepo scaffolding and builds | `nest g app worker` converts to monorepo mode. `nest build worker` and `nest start worker` target specific apps. |
| Docker Compose | (existing) | Local Redis + MongoDB stack | Add a `redis` service to existing `docker-compose.yaml`. |

## Installation

```bash
# New dependencies (run from trade-flow-api root)
npm install @nestjs/bullmq bullmq ioredis
```

No new dev dependencies required -- existing @nestjs/cli, nodemon, and TypeScript tooling handle the worker service.

## Monorepo Conversion Details

### What `nest g app worker` Does

Running `nest g app worker` from the project root will:

1. Move existing `src/` to `apps/api/src/` (renaming the default app to match `nest-cli.json` root)
2. Create `apps/worker/src/` with a new `main.ts`, `app.module.ts`, and `app.controller.ts`
3. Create `apps/worker/tsconfig.app.json`
4. Update `nest-cli.json` to monorepo mode with `projects` map
5. Update root `tsconfig.json` with project references

### Resulting nest-cli.json Structure

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "monorepo": true,
  "root": "apps/api",
  "sourceRoot": "apps/api/src",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/api/tsconfig.app.json",
    "assets": ["email/templates/**/*.html"],
    "watchAssets": true,
    "deleteOutDir": true
  },
  "projects": {
    "api": {
      "type": "application",
      "root": "apps/api",
      "sourceRoot": "apps/api/src",
      "entryFile": "main",
      "compilerOptions": {
        "tsConfigPath": "apps/api/tsconfig.app.json",
        "assets": ["email/templates/**/*.html"],
        "watchAssets": true
      }
    },
    "worker": {
      "type": "application",
      "root": "apps/worker",
      "sourceRoot": "apps/worker/src",
      "entryFile": "main",
      "compilerOptions": {
        "tsConfigPath": "apps/worker/tsconfig.app.json"
      }
    }
  }
}
```

### Resulting Directory Structure

```
trade-flow-api/
  apps/
    api/
      src/           <- existing src/ moved here
        main.ts
        app.module.ts
        auth/
        business/
        core/
        ...
      tsconfig.app.json
    worker/
      src/
        main.ts      <- new worker entry point (no HTTP listener)
        worker.module.ts
      tsconfig.app.json
  libs/              <- optional, for shared code later
  nest-cli.json      <- updated to monorepo mode
  tsconfig.json      <- root config with project references
  package.json       <- single shared package.json
  docker-compose.yaml
  Dockerfile
```

### Critical: Webpack Switch

NestJS monorepo mode switches the default builder from `tsc` to `webpack`. This means:

- Build output changes from `dist/` to `dist/apps/api/main.js` and `dist/apps/worker/main.js` (single bundled files per app)
- Path aliases (`@auth/*`, `@core/*`, etc.) are resolved by webpack at build time instead of `tsconfig-paths`
- The `start:prod` script must update from `node dist/main` to `node dist/apps/api/main`
- `ts-node` with `tsconfig-paths/register` (used in nodemon) still works for development
- Asset copying (email templates) must be configured per-app in `nest-cli.json`

### Path Alias Updates

All existing `@module/*` path aliases in `tsconfig.json` must update from `./src/module/*` to `./apps/api/src/module/*`. The worker can import shared modules by referencing the API's source directly or via a `libs/` shared library.

## Docker Compose Addition

```yaml
# Add to existing docker-compose.yaml services:
redis:
  container_name: trade-flow-redis
  image: redis:7.4-alpine
  ports:
    - "6379:6379"
  volumes:
    - redis-data:/data
  networks:
    - app-network
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5

# Add to volumes:
volumes:
  mongo-data:
  node_modules:
  redis-data:
```

The `api` and future `worker` services add `depends_on: redis: condition: service_healthy`.

## Environment Variables

| Variable | Default (dev) | Purpose |
|----------|---------------|---------|
| `REDIS_HOST` | `localhost` (local) / `redis` (Docker) | Redis connection host |
| `REDIS_PORT` | `6379` | Redis connection port |

No Redis password needed for local development. Production Redis (Railway or similar) will provide a connection URL.

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| NestJS built-in monorepo | Nx workspace | If you need build caching, dependency graph visualization, or 5+ apps. Overkill for 2 apps. Adds significant tooling complexity. |
| NestJS built-in monorepo | Turborepo | If each app had its own `package.json`. NestJS monorepo shares one `package.json` by design -- Turborepo adds nothing here. |
| @nestjs/bullmq | @nestjs/bull (legacy) | Never. @nestjs/bull wraps the deprecated `bull` package. @nestjs/bullmq wraps the actively maintained `bullmq`. |
| BullMQ | Agenda.js | If you needed MongoDB-backed queues (avoid adding Redis). Not recommended -- BullMQ is significantly more mature for job processing, and Redis is needed for future features anyway (caching, rate limiting). |
| Redis 7.4 | Redis 8.x | When Redis 8 has been stable for 6+ months and BullMQ explicitly tests against it. Currently too new. |
| Redis 7.4 | Upstash (serverless Redis) | For production only, if Railway doesn't offer managed Redis. Not needed for local dev. |
| Webpack (monorepo default) | SWC builder | For faster builds. NestJS SWC support in monorepo mode has reported issues (see nestjs/nest#12977). Stick with webpack until SWC monorepo support matures. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `@nestjs/bull` | Wraps deprecated `bull` package (unmaintained). @nestjs/bullmq is the modern replacement. | `@nestjs/bullmq` |
| `bull` | Superseded by `bullmq`. No new features, limited TypeScript support. | `bullmq` |
| `redis` (npm package) | BullMQ requires `ioredis`, not the `redis` npm package. They are incompatible. | `ioredis` |
| Nx / Turborepo | Adds a build orchestration layer on top of NestJS's built-in monorepo support. Unnecessary complexity for 2 apps. | NestJS built-in monorepo mode |
| Separate `package.json` per app | NestJS monorepo mode uses a single shared `package.json`. Fighting this pattern creates dependency management headaches. | Single root `package.json` |
| Redis Cluster (local dev) | BullMQ supports clusters but single-node Redis is sufficient for this scale. Cluster adds operational complexity with no benefit. | Single Redis instance |
| `bull-board` or `@bull-board/nestjs` | Admin UI for queue monitoring. Premature -- add only when debugging queue issues in production. | Console logging via Pino |

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| @nestjs/bullmq@^11.0.4 | @nestjs/core@^11.x, bullmq@^5.40.0 | Version 11.x aligns with NestJS 11. Peer dep on bullmq 5.40+. |
| bullmq@^5.71.0 | ioredis@^5.x, Redis 6.2+ | Uses ioredis internally. Recommends Redis 7.0+ for full feature support. |
| ioredis@^5.6.1 | Redis 2.8+ (basic), 6.0+ (streams) | Mature Redis client. BullMQ passes connection options through to ioredis constructor. |
| Redis 7.4 | bullmq@^5.x | BullMQ has version-specific optimizations for Redis 7.0.8+. |
| @nestjs/cli@^11.0.16 | NestJS 11 monorepo mode | `nest g app`, `nest build`, `nest start` all support monorepo project targeting. |
| webpack (NestJS default) | NestJS 11 monorepo mode | Auto-configured when monorepo mode is enabled. No manual webpack config needed. |

## Worker Service Architecture Notes

The worker service is a separate NestJS application (its own `main.ts`) that:

- Does NOT listen on an HTTP port (no `app.listen()`)
- Imports shared modules from the API source (or from `libs/` if extracted)
- Registers `BullModule.forRoot()` with same Redis connection
- Uses `@Processor('queue-name')` decorated classes extending `WorkerHost`
- Runs independently via `nest start worker --watch` or a dedicated nodemon config

The API service produces jobs via `@InjectQueue('queue-name')` and `queue.add()`. The worker service consumes them via `@Processor()` classes. Both connect to the same Redis instance.

## Sources

- [NestJS Monorepo Documentation](https://docs.nestjs.com/cli/monorepo) -- Official monorepo mode configuration (HIGH confidence)
- [NestJS Libraries Documentation](https://docs.nestjs.com/cli/libraries) -- Shared library patterns (HIGH confidence)
- [BullMQ NestJS Integration Guide](https://docs.bullmq.io/guide/nestjs) -- Official @nestjs/bullmq setup (HIGH confidence)
- [BullMQ Redis Compatibility](https://docs.bullmq.io/guide/redis-tm-compatibility) -- Minimum Redis version requirements (HIGH confidence)
- [@nestjs/bullmq Releases](https://github.com/nestjs/bull/releases) -- Version 11.0.4 release notes, bullmq peer dep ^5.40.0 (HIGH confidence)
- [NestJS Monorepo SWC Issue #12977](https://github.com/nestjs/nest/issues/12977) -- SWC builder limitation in monorepo mode (MEDIUM confidence)
- [Redis Docker Hub](https://hub.docker.com/_/redis/) -- Official Redis Docker images, 7.4-alpine available (HIGH confidence)
- Codebase: `trade-flow-api/package.json` -- existing dependency versions (HIGH confidence)
- Codebase: `trade-flow-api/nest-cli.json` -- current standard mode config (HIGH confidence)
- Codebase: `trade-flow-api/tsconfig.json` -- existing path aliases (HIGH confidence)
- Codebase: `trade-flow-api/docker-compose.yaml` -- existing Docker services (HIGH confidence)
- Codebase: `trade-flow-api/nodemon.json` -- current hot reload config (HIGH confidence)

---
*Stack research for: Trade Flow v1.4 -- Monorepo & Worker Infrastructure*
*Researched: 2026-03-22*
