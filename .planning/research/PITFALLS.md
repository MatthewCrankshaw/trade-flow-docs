# Pitfalls Research

**Domain:** Adding monorepo structure, BullMQ worker service, and Redis to an existing NestJS single-service application
**Researched:** 2026-03-22
**Confidence:** HIGH

## Critical Pitfalls

### Pitfall 1: Path Aliases Break When Moving to Monorepo Structure

**What goes wrong:**
The existing `trade-flow-api` has 20+ path aliases (`@auth/*`, `@business/*`, `@core/*`, `@job/*`, `@quote/*`, etc.) defined in `tsconfig.json` with paths relative to `./src/*`. When restructuring to a monorepo layout (e.g., `apps/api/src/`, `apps/worker/src/`, `libs/shared/src/`), every single path alias breaks simultaneously. The TypeScript compiler resolves paths at compile time, but `ts-node` and the NestJS CLI resolve them at runtime via `tsconfig-paths`. Both must agree on the new paths or the app fails with `MODULE_NOT_FOUND` errors at startup.

**Why it happens:**
Developers update `tsconfig.json` paths for the new directory structure but forget that multiple other files reference the old paths:
1. `nest-cli.json` has `"sourceRoot": "src"` -- must change per project.
2. `nodemon.json` watches `"src"` -- must point to the new API source location.
3. `tsconfig.build.json` and `tsconfig-check.json` extend the base `tsconfig.json` -- their exclude paths are relative.
4. Jest config uses `moduleNameMapper` which must mirror tsconfig path changes.
5. The `nest start --watch` command resolves paths differently than `tsc --noEmit`.
6. The existing `start:dev` script (`nest start --watch`) and `test:debug` script (`-r tsconfig-paths/register`) both depend on path resolution.

**How to avoid:**
- Change directory structure first, update ALL config files that reference `src/` or `./src/` (tsconfig.json, tsconfig.build.json, tsconfig-check.json, nest-cli.json, nodemon.json, jest config, Dockerfile COPY commands), then verify the existing API starts and all tests pass BEFORE adding any worker code.
- Maintain a checklist of every config file that references source paths and update them atomically.
- Run `npm run validate` (typecheck + lint) and `npm run test` after the restructure to catch breakage immediately.

**Warning signs:**
- `MODULE_NOT_FOUND` errors mentioning `@auth/` or `@core/` paths at runtime
- Tests pass locally but `nest build` fails (or vice versa)
- `nodemon` restarts in a loop without successfully starting
- VS Code shows red squiggles on imports that previously worked

**Phase to address:**
Phase 1 (Monorepo Restructure) -- this must be the very first thing done and verified before any BullMQ/Redis work begins.

---

### Pitfall 2: NestJS Monorepo Mode vs Custom Directory Layout Confusion

**What goes wrong:**
NestJS has a built-in "monorepo mode" (triggered by `nest generate app <name>`) that changes the build tool from `tsc` to `webpack`, restructures `nest-cli.json` with a `"monorepo": true` flag and `projects` property, and expects a specific `apps/`/`libs/` layout. Developers either accidentally invoke monorepo mode when they only wanted a custom directory layout, or they try to use it without understanding that webpack bundling changes runtime behavior. The existing project uses `tsc` compilation (no webpack), has `nodemon` for dev hot reload, and copies email template assets via `nest-cli.json` `"assets": ["email/templates/**/*.html"]` -- all of which may break under webpack bundling.

**Why it happens:**
The NestJS docs describe monorepo mode as the "official" way to add multiple apps, so developers assume they must use it. But the trade-flow-api uses `tsc` with `tsconfig-paths` at runtime, nodemon for development, and asset copying for Maizzle email templates -- none of which are guaranteed to work identically under webpack.

**How to avoid:**
- Use a custom directory layout WITHOUT switching to NestJS monorepo mode. Keep `tsc` as the compiler.
- Do NOT run `nest generate app worker` -- this triggers monorepo mode conversion automatically and rewrites `nest-cli.json`.
- Manually create the worker app structure with its own `main.ts`, `tsconfig.json` (extending a shared base), and entry in package.json scripts.
- Keep the root `nest-cli.json` pointing to the API app as the default project.

**Warning signs:**
- `nest-cli.json` suddenly contains `"monorepo": true` that nobody explicitly added
- Build output changes from `dist/` to `dist/apps/api/` unexpectedly
- `webpack` appears in build logs when it was not there before
- Email template HTML assets stop being copied to the build output
- `start:prod` (`node dist/main`) fails because the output path changed

**Phase to address:**
Phase 1 (Monorepo Restructure) -- decide the approach upfront and document the decision before touching any files.

---

### Pitfall 3: Redis maxmemory-policy Not Set to noeviction

**What goes wrong:**
BullMQ stores job data, queue metadata, delayed job schedules, and rate limiter state in Redis keys. If Redis runs with its default eviction policy (`volatile-lru`), it may silently evict BullMQ keys when memory pressure rises. This causes jobs to vanish, queues to become corrupted, and BullMQ to throw `ReplyError` exceptions. In development this never surfaces (low memory), but in production under load it causes "jobs submitted but never processed" with no obvious error trail.

**Why it happens:**
Docker's `redis:7` image ships with `volatile-lru` as the default eviction policy. Developers add Redis to Docker Compose, confirm it starts, connect BullMQ, see jobs flow, and move on. The misconfiguration only manifests under memory pressure in production.

**How to avoid:**
- Always start Redis with `--maxmemory-policy noeviction` in the Docker Compose command.
- BullMQ itself checks this at startup and logs a warning -- but only a warning, not a fatal error. Treat it as fatal.
- Example Docker Compose Redis configuration:
  ```yaml
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory-policy noeviction
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
  ```
- After startup, verify with `redis-cli CONFIG GET maxmemory-policy` returning `noeviction`.

**Warning signs:**
- BullMQ log: `IMPORTANT! Eviction policy is volatile-lru. It should be "noeviction"`
- Jobs appear in the queue (Redis keys exist) but processors never receive them
- Intermittent `ReplyError` exceptions from ioredis under load

**Phase to address:**
Phase 2 (Redis Infrastructure) -- configure correctly from the very first Docker Compose addition.

---

### Pitfall 4: Worker Service Cannot Import API Services Without Pulling in HTTP Stack

**What goes wrong:**
The worker needs to import API services (e.g., `EmailModule` for sending notifications, `QuoteModule` for reading quote data) to process background jobs. Developers either: (a) import the entire API `AppModule` into the worker, which pulls in HTTP controllers, `JwtAuthGuard`, `ValidationPipe`, CORS middleware, and Express -- none of which belong in a worker; or (b) try to extract shared code into a `libs/` directory prematurely, breaking the existing API's module graph in the process. Both paths lead to bloated startup, circular dependency errors, or mysterious failures.

**Why it happens:**
The existing architecture has tight module coupling that works for a single HTTP app. `CoreModule` is `@Global()` and provides database services (needed by worker). `AuthModule` provides JWT guards (not needed by worker). Feature modules like `QuoteModule` import `CoreModule` implicitly via the global scope. When the worker imports `QuoteModule`, it transitively needs `CoreModule`, which is fine -- but if `QuoteModule` also depends on something that depends on `AuthModule`, the worker gets auth infrastructure it does not need.

**How to avoid:**
- The worker's `main.ts` should use `NestFactory.createApplicationContext()` (not `NestFactory.create()`) -- this bootstraps NestJS dependency injection without starting an HTTP server.
- Build the worker's own `WorkerModule` that imports only the specific feature modules it needs. Start minimal and add imports as needed.
- `CoreModule` (database, logging) is safe to import -- it has no HTTP dependencies.
- Do NOT import `AppModule`. Do NOT import `AuthModule` or any module that registers HTTP middleware/guards.
- If a feature module has a transitive dependency on auth infrastructure, that is a sign the feature module needs refactoring to separate its core logic from its HTTP layer.

**Warning signs:**
- Worker startup logs show Express/Fastify being initialized
- Worker bootstrap pulls in `JwtAuthGuard` or `PassportModule`
- Circular dependency warnings from NestJS DI container
- Worker takes as long to start as the API despite having no HTTP endpoints
- Worker crashes with `Cannot find module 'passport'` or similar

**Phase to address:**
Phase 3 (Worker Service Scaffold) -- design the worker's module imports carefully before writing any processor code.

---

### Pitfall 5: BullMQ Producer-Consumer Queue Name or Connection Mismatch

**What goes wrong:**
The API (producer) registers a queue with `BullModule.forRoot()` and `BullModule.registerQueue({ name: 'email' })`. The worker (consumer) registers the same queue but with a different connection string, a different queue name string, or a different Redis database number. Jobs are added to one Redis namespace but the worker listens on another. Result: jobs pile up in Redis, nothing gets processed, and no errors are thrown because both sides think they are working correctly.

**Why it happens:**
In a monorepo with two separate NestJS apps, each has its own module registration and its own environment variable loading. If the Redis connection config (`host`, `port`, `db` number) or the queue name string differs by even one character between producer and consumer, they silently operate on different queues. This is especially insidious because BullMQ does not validate that a consumer exists for a queue.

**How to avoid:**
- Define queue names as shared constants in a common location imported by both API and worker (e.g., `libs/shared/constants/queue-names.constant.ts` or a shared TypeScript file both apps reference).
- Define Redis connection config via environment variables that both apps read identically (same `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB` env vars).
- Use `BullModule.forRoot()` with identical configuration in both the API and worker root modules.
- After initial setup, verify end-to-end by adding a trivial test job from the API and confirming the worker's processor receives and logs it. Do this verification in the scaffold phase, not after building real processors.

**Warning signs:**
- Redis shows `bull:email:waiting` keys growing but `bull:email:completed` stays at zero
- Worker logs "Processor registered for queue: email" but never "Processing job"
- Two different Redis `db` numbers in API vs worker `.env` files
- Queue name typo: `"emails"` in one place, `"email"` in another

**Phase to address:**
Phase 3 (Worker Service Scaffold) -- verify end-to-end job flow with a trivial test job before building real processors.

---

### Pitfall 6: ioredis Configuration Conflicts with BullMQ

**What goes wrong:**
Developers configure the ioredis client with options that are incompatible with BullMQ, causing silent failures or exceptions. The three most common mistakes: (1) using ioredis `keyPrefix` option, which conflicts with BullMQ's own prefix mechanism; (2) not setting `maxRetriesPerRequest: null` on connections used by Workers, causing BullMQ to throw; (3) setting `enableOfflineQueue: true` (the default) on producer connections, causing `queue.add()` calls to hang indefinitely when Redis is down instead of failing fast.

**Why it happens:**
Developers copy ioredis configuration from other projects or tutorials that were not BullMQ-specific. BullMQ has strict requirements on certain ioredis options that are not enforced at the configuration level -- they only manifest as runtime errors or hangs.

**How to avoid:**
- Never use ioredis `keyPrefix` with BullMQ -- use BullMQ's own `prefix` option if needed.
- Worker connections must set `maxRetriesPerRequest: null` (BullMQ uses blocking commands that need unlimited retries).
- Producer connections should set `enableOfflineQueue: false` so that `queue.add()` throws immediately if Redis is unreachable, preventing HTTP requests from hanging while waiting for reconnection.
- Use `@nestjs/bullmq`'s `BullModule.forRoot()` which handles most of these correctly, rather than manually constructing ioredis clients.

**Warning signs:**
- BullMQ throws `maxRetriesPerRequest` exception on worker startup
- `queue.add()` calls in API controllers hang for 30+ seconds then timeout
- Keys in Redis have unexpected double-prefixed names like `myprefix:bull:queue:`
- Worker connects to Redis but never receives jobs

**Phase to address:**
Phase 2 (Redis Infrastructure) for connection config, Phase 3 (Worker Scaffold) for verifying worker-specific settings.

---

### Pitfall 7: Dockerfile and Production Build Not Updated for Monorepo

**What goes wrong:**
The existing `Dockerfile` copies source from the root (`COPY . .`), runs `npm run build` (which executes `nest build`), and serves from `dist/main` (`node dist/main`). After monorepo restructuring, `dist/main` no longer exists at the expected path. The production Docker image builds successfully (TypeScript compiles) but crashes on `CMD ["npm", "run", "start:prod"]` because `node dist/main` points to a nonexistent file. Additionally, the worker has no Dockerfile or production entry point at all.

**Why it happens:**
Developers focus on getting local development working (nodemon, ts-node) and defer Docker/production concerns. The Dockerfile, Railway deployment config (`railway.json`), and production build paths all reference the old structure. The problem only surfaces when deploying, which may be days after the restructure.

**How to avoid:**
- Update the `Dockerfile` immediately after restructuring: COPY paths, build commands, and `CMD` entry point.
- The existing `start:prod` script is `node dist/main` -- this must match the actual build output location.
- Create a worker Dockerfile (or a multi-stage Dockerfile with separate targets for API and worker).
- Update `railway.json` if it references specific build or start commands.
- Test the production build locally: `docker build --target production .` then `docker run` and verify the app starts.
- The `nest-cli.json` asset copy config (`"assets": ["email/templates/**/*.html"]`) must still resolve correctly after restructure.

**Warning signs:**
- `docker build` succeeds but `docker run` crashes with `Error: Cannot find module '/app/dist/main'`
- Email templates missing from production build (asset paths broken)
- Railway/CI deployment fails after a previously green local build
- Worker has no way to be built or deployed

**Phase to address:**
Phase 1 (Monorepo Restructure) for API Dockerfile, Phase 3 (Worker Scaffold) for worker Dockerfile.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Running worker in same process as API | No Redis needed, simpler setup | Cannot scale independently; CPU-heavy jobs block HTTP responses; no fault isolation | Never -- the purpose of v1.4 is separate worker infrastructure |
| Hardcoding Redis connection strings instead of shared config | Faster to get running | Connection details drift between API and worker; env var duplication | Only during initial spike; refactor before merging to main |
| Skipping `removeOnComplete`/`removeOnFail` on queues | Less config to write | Redis memory grows unbounded as completed/failed jobs accumulate; eventual OOM | Never -- configure retention from the start (e.g., keep 100 completed, 500 failed) |
| Not creating a worker Docker Compose service | Fewer moving parts locally | Developer forgets to start worker; bugs from "worker wasn't running" go unnoticed | Acceptable briefly during initial scaffold; add before any real processor work |
| Copying entire modules to `libs/shared/` instead of importing | Quick perceived "separation" | Duplicated code diverges; two copies of the same service with different bugs | Never -- share by module imports, not file duplication |
| Using `NestFactory.create()` for worker instead of `createApplicationContext()` | Worker "just works" like the API | Worker starts an unused HTTP server, binds a port, wastes resources, may conflict with API port | Never -- workers should not listen on HTTP |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Redis in Docker Compose | Adding Redis without `depends_on: condition: service_healthy` so API/worker start before Redis is ready | Use health check with `redis-cli ping` and `condition: service_healthy`, matching the existing Mongo health check pattern |
| Redis in Docker Compose | Not adding a named volume for Redis data | Add `redis-data` volume so queue state persists across `docker compose down`/`up` cycles (matches existing `mongo-data` pattern) |
| Redis in Docker Compose | Not adding Redis port to `.env.example` | Add `REDIS_HOST`, `REDIS_PORT` to `.env.example` and Docker Compose `environment` section for both API and worker services |
| `BullModule.forRoot()` | Registering `forRoot()` in a feature module instead of the root module | Register `forRoot()` with Redis connection in the root/app module; feature modules use `registerQueue()` only |
| `@nestjs/bullmq` | Using `@nestjs/bull` (the older package) instead of `@nestjs/bullmq` | `@nestjs/bull` wraps the legacy Bull library; `@nestjs/bullmq` wraps BullMQ which is the actively maintained successor. Use `@nestjs/bullmq` exclusively |
| BullMQ + ioredis | Passing shared ioredis client to Worker without `maxRetriesPerRequest: null` | Workers require `maxRetriesPerRequest: null`; producers should keep default or use a low value for fail-fast |
| BullMQ + ioredis | Using ioredis `keyPrefix` option | BullMQ has its own prefix mechanism; ioredis `keyPrefix` is incompatible and causes key lookup failures |
| Worker NestJS app | Using `NestFactory.create()` for the worker | Use `NestFactory.createApplicationContext()` -- bootstraps DI without HTTP server |
| Worker module imports | Importing the API's `AppModule` into the worker | Import only specific feature modules the worker needs; skip auth, controllers, HTTP middleware |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| No concurrency on worker processor | Worker processes one job at a time; queue backs up during bursts | Set `concurrency` option on `@Processor()` decorator; start with 3-5, tune by job type | When email volume exceeds ~1 job/second with >1s processing time |
| No job timeout or TTL | A hung job (e.g., Resend API timeout) blocks the worker slot indefinitely | Set `attempts` with `backoff` strategy on jobs; set `lockDuration` on processor | First time an external service (Resend) hangs or times out |
| Storing full objects in job payload | Putting entire quote/customer objects in job data; Redis memory balloons | Store only IDs in job data; worker fetches current data from MongoDB on processing | When quotes have many line items and thousands of jobs accumulate |
| No `removeOnComplete`/`removeOnFail` | Redis stores every completed and failed job forever | Configure `removeOnComplete: { count: 100 }` and `removeOnFail: { count: 500 }` | After a few thousand jobs; Redis memory grows linearly with job count |
| Single Redis connection for all BullMQ instances | Connection contention under load; blocking commands on worker affect producer | Let BullMQ manage its own connections (default behavior); do not share a single ioredis instance across queues/workers | At moderate throughput (~100 jobs/minute) |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Redis exposed on Docker host network without auth | Anyone on the local network can read/write queue data including job payloads with customer IDs | Bind Redis to `127.0.0.1` only; for production, set `requirepass` and use Redis ACLs |
| Sensitive data in BullMQ job payload | Redis data is unencrypted at rest; anyone with Redis access sees PII (emails, names) | Store only opaque entity IDs in job data; worker fetches sensitive data from MongoDB using existing service layer |
| Worker bypassing authorization policies | Worker services call repositories directly, skipping `AccessController` and `Policy` checks | Worker should use existing service classes (e.g., `QuoteRetriever`) that enforce policies; create a system-level `IUserDto` for worker authentication context if needed |
| Redis connection string with password in Docker Compose plain text | Credentials visible in `docker-compose.yaml` in version control | Use environment variable interpolation (`${REDIS_PASSWORD}`) and keep actual values in `.env` (which is gitignored) |

## "Looks Done But Isn't" Checklist

- [ ] **Monorepo restructure:** `npm run validate` (typecheck + lint) passes with new directory structure
- [ ] **Monorepo restructure:** `npm run test` passes -- Jest `moduleNameMapper` matches updated tsconfig paths
- [ ] **Monorepo restructure:** `npm run typecheck:strict` passes (the strict `tsconfig-check.json` has separate path resolution)
- [ ] **Monorepo restructure:** Email template assets still copy to build output (check `nest-cli.json` `assets` config after path changes)
- [ ] **Monorepo restructure:** `node dist/main` (production entry point) still works after restructure
- [ ] **Redis setup:** `docker compose up` starts Redis before API/worker (health check + `depends_on` with `condition: service_healthy`)
- [ ] **Redis setup:** `redis-cli CONFIG GET maxmemory-policy` returns `noeviction`
- [ ] **Redis setup:** `.env.example` updated with Redis connection variables
- [ ] **BullMQ integration:** A test job added by the API is received and processed by the worker (both sides log success)
- [ ] **BullMQ integration:** `removeOnComplete` and `removeOnFail` configured with retention limits on every queue
- [ ] **BullMQ integration:** `enableOfflineQueue: false` set on producer connections so API does not hang when Redis is down
- [ ] **Worker service:** Worker starts with `createApplicationContext()` -- no HTTP server, no Express, no port binding
- [ ] **Worker service:** Worker does NOT import `AuthModule`, `JwtAuthGuard`, or any HTTP middleware
- [ ] **Worker service:** Worker's nodemon/hot-reload works independently of the API's
- [ ] **Worker service:** `docker compose up` starts both API and worker services with separate containers and correct env vars
- [ ] **Graceful shutdown:** `docker compose down` sends SIGTERM; worker finishes in-flight jobs before exiting
- [ ] **Dockerfile:** `docker build --target production` works for API (and worker if Dockerfile exists)
- [ ] **Dockerfile:** Built production image starts without `MODULE_NOT_FOUND` errors

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Path aliases all broken after restructure | LOW | Revert directory changes via git, fix all config files atomically, re-apply structure change. Version control makes this safe. |
| Accidentally triggered NestJS monorepo mode | MEDIUM | Revert `nest-cli.json` to remove `"monorepo": true` and `projects` property, restore `tsc` build config, verify `dist/` output structure matches `start:prod` expectations. |
| Redis evicting BullMQ keys in production | HIGH | Stop producers, reconfigure `maxmemory-policy`, clear corrupted queue state with `FLUSHDB`, replay failed jobs from application logs if possible. Lost jobs cannot be recovered from Redis. |
| Worker imports entire AppModule with HTTP stack | LOW | Refactor worker module to cherry-pick specific feature modules. No data loss, just code restructuring. |
| Producer-consumer queue name mismatch | LOW | Fix queue name constants to match, drain the orphaned queue, redeploy. Jobs in the wrong queue can be moved with BullMQ API or re-enqueued. |
| Dockerfile broken in production | MEDIUM | Hotfix Dockerfile paths, rebuild, redeploy. Downtime depends on CI/CD pipeline speed. Test locally with `docker build && docker run` before pushing. |
| Worker not handling graceful shutdown | MEDIUM | Jobs may be partially processed and stuck in "active" state. Implement `onModuleDestroy()` lifecycle hook to close worker connections cleanly. Reprocess stuck active jobs. |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Path aliases break (#1) | Phase 1: Monorepo Restructure | `npm run validate && npm run test` pass; `node dist/main` starts |
| Monorepo mode confusion (#2) | Phase 1: Monorepo Restructure | `nest-cli.json` does NOT contain `"monorepo": true`; build uses `tsc` not `webpack` |
| Redis maxmemory-policy (#3) | Phase 2: Redis Infrastructure | `redis-cli CONFIG GET maxmemory-policy` returns `noeviction` |
| Worker imports HTTP stack (#4) | Phase 3: Worker Scaffold | Worker starts with `createApplicationContext()`; no Express/Passport in logs |
| Producer-consumer mismatch (#5) | Phase 3: Worker Scaffold | Test job added by API, processed by worker, both log success |
| ioredis config conflicts (#6) | Phase 2 + Phase 3 | No `keyPrefix` in config; `maxRetriesPerRequest: null` on worker connections |
| Dockerfile not updated (#7) | Phase 1 + Phase 3 | `docker build --target production` succeeds; `docker run` starts without errors |

## Sources

- [NestJS Monorepo Documentation](https://docs.nestjs.com/cli/monorepo) -- official monorepo mode configuration, webpack builder behavior -- HIGH confidence
- [BullMQ Connections Guide](https://docs.bullmq.io/guide/connections) -- `maxRetriesPerRequest`, `enableOfflineQueue`, `keyPrefix` incompatibility -- HIGH confidence
- [BullMQ Going to Production](https://docs.bullmq.io/guide/going-to-production) -- `maxmemory-policy noeviction` requirement -- HIGH confidence
- [BullMQ Reusing Redis Connections](https://docs.bullmq.io/bull/patterns/reusing-redis-connections) -- connection sharing constraints for workers -- HIGH confidence
- [BullMQ Failing Fast When Redis Is Down](https://docs.bullmq.io/patterns/failing-fast-when-redis-is-down) -- `enableOfflineQueue: false` for producers -- HIGH confidence
- [BullMQ NestJS Integration](https://docs.bullmq.io/guide/nestjs) -- official `@nestjs/bullmq` setup patterns -- HIGH confidence
- [NestJS Path Aliases in Monorepo (GitHub Issue #13558)](https://github.com/nestjs/nest/issues/13558) -- runtime path resolution failures in monorepo -- HIGH confidence
- [NestJS Standalone BullMQ Worker](https://medium.com/@omarae00/nestjs-standalone-bullmq-worker-6f44faefaf6b) -- `createApplicationContext()` pattern for background workers -- MEDIUM confidence
- [Running NestJS Queues in a Separate Process](https://medium.com/s1seven/running-nestjs-queues-in-a-separate-process-948f414c4b41) -- shared module architecture for worker services -- MEDIUM confidence
- [BullMQ Eviction Policy Issue #2737](https://github.com/taskforcesh/bullmq/issues/2737) -- error when Redis eviction policy is misconfigured -- HIGH confidence
- [Redis Docker Compose Health Check](https://www.baeldung.com/ops/redis-server-docker-image-health) -- health check configuration patterns -- MEDIUM confidence
- [NestJS Monorepos Without the Meltdown](https://medium.com/@bhagyarana80/nestjs-monorepos-without-the-meltdown-3a155795ea94) -- library structure anti-patterns, circular dependency risks -- MEDIUM confidence
- Trade Flow codebase analysis: `tsconfig.json` (20+ path aliases), `nest-cli.json` (asset copying), `nodemon.json` (watch paths), `docker-compose.yaml` (existing Mongo pattern), `Dockerfile` (multi-stage build), `package.json` (npm scripts) -- HIGH confidence

---
*Pitfalls research for: NestJS monorepo + BullMQ worker + Redis infrastructure*
*Researched: 2026-03-22*
