# Architecture Research

**Domain:** NestJS monorepo with BullMQ worker service integration
**Researched:** 2026-03-22
**Confidence:** HIGH

## Recommendation: Lightweight Monorepo (Not NestJS CLI Monorepo Mode)

**Do NOT use `nest g app` or NestJS CLI monorepo mode.** The NestJS CLI monorepo (`"monorepo": true` in nest-cli.json) restructures the entire project into `apps/` and `libs/` directories, switches the compiler from `tsc` to `webpack`, and changes how path aliases resolve. This is a massive migration for an existing 21.9k LOC codebase with 40+ path aliases and established conventions.

Instead, use a **lightweight approach**: keep the existing `src/` structure intact, add a second entry point (`src/worker.ts`) and a worker module (`src/worker/`), and use a second `nest-cli` config file to build/run the worker independently. Both API and worker share the same codebase, same `node_modules`, same path aliases, same TypeScript compilation -- but boot different NestJS applications.

This pattern is well-documented and widely used for NestJS + BullMQ setups. It avoids the webpack migration, preserves all existing conventions, and achieves the goal of independently deployable services.

## System Overview

```
                    ┌─────────────────────────────────────┐
                    │         trade-flow-api repo          │
                    │                                      │
                    │  ┌──────────┐    ┌──────────────┐   │
                    │  │ main.ts  │    │  worker.ts   │   │
                    │  │ (API)    │    │  (Worker)    │   │
                    │  └────┬─────┘    └──────┬───────┘   │
                    │       │                 │            │
                    │  ┌────┴─────┐    ┌──────┴───────┐   │
                    │  │AppModule │    │ WorkerModule │   │
                    │  │(HTTP+    │    │ (No HTTP,    │   │
                    │  │ queues)  │    │  processors) │   │
                    │  └────┬─────┘    └──────┬───────┘   │
                    │       │                 │            │
                    │       └────────┬────────┘            │
                    │                │                     │
                    │  ┌─────────────┴──────────────┐     │
                    │  │    Shared Feature Modules   │     │
                    │  │  (business, customer, job,  │     │
                    │  │   quote, email, core, etc.) │     │
                    │  └─────────────┬──────────────┘     │
                    │                │                     │
                    └────────────────┼─────────────────────┘
                                     │
                    ┌────────────────┼─────────────────────┐
                    │     External Infrastructure          │
                    │                │                     │
                    │  ┌─────────┐  │  ┌──────────────┐   │
                    │  │ MongoDB │◄─┤  │    Redis      │   │
                    │  │  7.0    │  │  │ (BullMQ)     │   │
                    │  └─────────┘  │  └──────────────┘   │
                    │               │                     │
                    └───────────────┴─────────────────────┘
```

### How It Works

1. **API process** (`main.ts`): Starts HTTP server, registers BullMQ queues (producer side), handles all REST endpoints. Enqueues jobs when async work is needed.
2. **Worker process** (`worker.ts`): Starts as a standalone NestJS application context (no HTTP server), registers BullMQ processors (consumer side), processes jobs from Redis queues.
3. **Shared modules**: Both processes import the same feature modules (CoreModule, EmailModule, etc.) for database access, services, and business logic. No code duplication.
4. **Redis**: Connects both processes via BullMQ queues. API produces jobs, worker consumes them.

## Component Responsibilities

| Component | Responsibility | New vs Existing |
|-----------|----------------|-----------------|
| `main.ts` | HTTP server bootstrap, CORS, validation pipes | **EXISTING** -- no changes |
| `worker.ts` | Standalone NestJS app context, worker bootstrap | **NEW** |
| `AppModule` | HTTP app root, imports all feature modules + QueueModule | **MODIFIED** -- add QueueModule import |
| `WorkerModule` | Worker app root, imports shared modules + processor providers | **NEW** |
| `CoreModule` | MongoDB connection, shared services (global) | **EXISTING** -- no changes |
| Feature modules | Business logic, repositories, services | **EXISTING** -- no changes |
| `src/worker/processors/` | BullMQ `@Processor` classes that consume queue jobs | **NEW** |
| `src/queue/` | Queue name constants and shared BullMQ/Redis configuration | **NEW** |
| `nest-cli.json` | API build config (existing) | **EXISTING** -- no changes |
| `worker-cli.json` | Worker build config (separate entry point) | **NEW** |

## Recommended Project Structure

```
trade-flow-api/
├── src/
│   ├── main.ts                        # API entry point (EXISTING, unchanged)
│   ├── app.module.ts                  # API root module (MODIFIED -- add QueueModule import)
│   ├── worker.ts                      # Worker entry point (NEW)
│   ├── worker/                        # Worker-specific code (NEW)
│   │   ├── worker.module.ts           # Worker root module
│   │   └── processors/               # BullMQ processor classes
│   │       └── example.processor.ts   # Extends WorkerHost
│   ├── queue/                         # Shared queue infrastructure (NEW)
│   │   ├── queue.module.ts            # BullModule.forRoot + registerQueue
│   │   ├── queue.constants.ts         # Queue name constants
│   │   └── queue.config.ts            # Redis connection config
│   ├── auth/                          # (EXISTING, unchanged)
│   ├── business/                      # (EXISTING, unchanged)
│   ├── core/                          # (EXISTING, unchanged)
│   ├── customer/                      # (EXISTING, unchanged)
│   ├── email/                         # (EXISTING, unchanged)
│   ├── item/                          # (EXISTING, unchanged)
│   ├── job/                           # (EXISTING, unchanged)
│   ├── quote/                         # (EXISTING, unchanged)
│   ├── quote-settings/                # (EXISTING, unchanged)
│   ├── quote-token/                   # (EXISTING, unchanged)
│   ├── schedule/                      # (EXISTING, unchanged)
│   ├── tax-rate/                      # (EXISTING, unchanged)
│   ├── user/                          # (EXISTING, unchanged)
│   └── visit-type/                    # (EXISTING, unchanged)
├── nest-cli.json                      # API build config (EXISTING, unchanged)
├── worker-cli.json                    # Worker build config (NEW)
├── nodemon.json                       # API dev watch (EXISTING, unchanged)
├── nodemon-worker.json                # Worker dev watch (NEW)
├── docker-compose.yaml                # (MODIFIED -- add Redis + worker service)
├── tsconfig.json                      # (MODIFIED -- add @queue/* and @worker/* aliases)
├── package.json                       # (MODIFIED -- add deps + worker scripts)
└── Dockerfile                         # (MODIFIED -- add worker production stage)
```

### Structure Rationale

- **`src/worker/`**: Contains only worker-specific code (module + processors). Follows the existing feature-module pattern. Processors are the worker equivalent of controllers -- they handle incoming jobs rather than HTTP requests.
- **`src/queue/`**: Shared queue infrastructure imported by BOTH AppModule and WorkerModule. Contains BullModule configuration, queue name constants, and Redis connection config. Neither API-specific nor worker-specific.
- **No `apps/` or `libs/` restructure**: The existing `src/` layout is preserved. Feature modules are naturally shareable because NestJS DI handles wiring. The worker simply imports the modules it needs.

## Architectural Patterns

### Pattern 1: Dual Entry Point with Shared Modules

**What:** Two NestJS applications (API + Worker) in the same codebase, each with their own entry point and root module, sharing feature modules via NestJS DI.

**When to use:** When a worker needs access to the same repositories, services, and business logic as the API.

**Trade-offs:**
- Pro: Zero code duplication, shared type safety, single `npm install`
- Pro: Both apps stay in sync automatically (same codebase)
- Con: Worker pulls in modules it may not need (but NestJS tree-shakes unused providers)
- Con: Must be careful not to start HTTP-specific middleware in worker context

**API entry point (`main.ts` -- existing, no changes needed):**
```typescript
// Existing main.ts stays exactly as-is
const app = await NestFactory.create(AppModule, { bufferLogs: true });
// ... CORS, validation, listen on port
```

**Worker entry point (`worker.ts` -- new):**
```typescript
import { NestFactory } from "@nestjs/core";
import { Logger } from "@nestjs/common";
import { Logger as PinoLogger } from "nestjs-pino";
import { WorkerModule } from "./worker/worker.module";

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(WorkerModule, {
    bufferLogs: true,
  });

  app.useLogger(app.get(PinoLogger));

  const logger = new Logger("Worker");
  logger.log("Worker service started");

  // Graceful shutdown
  process.on("SIGTERM", async () => {
    logger.log("SIGTERM received, shutting down worker");
    await app.close();
    process.exit(0);
  });
}
bootstrap();
```

**Key detail:** `createApplicationContext` instead of `create` -- no HTTP server, no CORS, no ValidationPipe. The worker only processes queue jobs. BullMQ processors start automatically when the NestJS application context initializes because `@Processor` classes are registered as providers.

### Pattern 2: Queue Module as Shared Infrastructure

**What:** A dedicated `QueueModule` that configures BullMQ's Redis connection and registers queue names. Imported by both AppModule (to produce jobs) and WorkerModule (to consume jobs).

**When to use:** Always when API and worker share queue definitions.

**Trade-offs:**
- Pro: Single source of truth for queue names and Redis config
- Pro: Both sides reference the same queue constants
- Con: Must ensure processor providers are only in WorkerModule (not in QueueModule)

**Example:**
```typescript
// src/queue/queue.constants.ts
export const QUEUE_NAMES = {
  EMAIL: "email",
} as const;

// src/queue/queue.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bullmq";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { QUEUE_NAMES } from "./queue.constants";

@Module({
  imports: [
    BullModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        connection: {
          host: configService.get<string>("REDIS_HOST", "localhost"),
          port: configService.get<number>("REDIS_PORT", 6379),
        },
      }),
      inject: [ConfigService],
    }),
    BullModule.registerQueue(
      { name: QUEUE_NAMES.EMAIL },
    ),
  ],
  exports: [BullModule],
})
export class QueueModule {}
```

**AppModule imports QueueModule** -- services can `@InjectQueue("email")` to produce jobs.
**WorkerModule imports QueueModule** -- processors use `@Processor("email")` to consume jobs.

### Pattern 3: Processor as Worker-Layer Handler (Thin Processors)

**What:** BullMQ processors follow the same single-responsibility pattern as existing NestJS controllers. A processor handles one queue, delegates to existing services for business logic.

**When to use:** For every queue consumer in the worker.

**Trade-offs:**
- Pro: Keeps processors thin (like controllers), business logic stays in services
- Pro: Services remain testable without BullMQ dependency
- Con: Extra layer of indirection

**Example:**
```typescript
// src/worker/processors/email.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from "@nestjs/bullmq";
import { Job } from "bullmq";
import { AppLogger } from "@core/services/app-logger.service";
import { QUEUE_NAMES } from "@queue/queue.constants";

@Processor(QUEUE_NAMES.EMAIL)
export class EmailProcessor extends WorkerHost {
  private readonly logger = new AppLogger(EmailProcessor.name);

  constructor(private readonly emailService: EmailService) {
    super();
  }

  async process(job: Job): Promise<void> {
    this.logger.log(`Processing email job ${job.id}: ${job.name}`);
    // Delegate to existing service -- processor stays thin
    await this.emailService.sendQuoteEmail(job.data);
  }

  @OnWorkerEvent("failed")
  onFailed(job: Job, error: Error): void {
    this.logger.error(`Job ${job.id} failed: ${error.message}`, error.stack);
  }
}
```

**Naming convention:** `[queue-name].processor.ts` with class name `[QueueName]Processor`, matching the existing file naming pattern (`[name]-[role].service.ts`).

## Data Flow

### Job Production Flow (API side)

```
HTTP Request
    |
Controller -> Service -> @InjectQueue("email") -> queue.add("send-quote", payload)
    |                                                    |
Response <- (immediate)                          Redis Queue (async)
```

### Job Consumption Flow (Worker side)

```
Redis Queue
    |
@Processor("email") -> EmailProcessor.process(job)
    |
Service (shared) -> Repository (shared) -> MongoDB
    |
Job complete / Job failed (logged)
```

### Key Data Flows

1. **Quote email sending (future migration):** API controller calls service, service enqueues job with quote ID and recipient email, returns immediately. Worker picks up job, loads quote from DB via shared repository, renders email, sends via Resend. This replaces the current synchronous email sending in QuoteSender.

2. **PDF generation (future):** API enqueues PDF generation job with quote ID. Worker loads quote data, generates PDF, stores result. API can poll for completion or use a webhook.

3. **Stripe webhooks (future):** API receives webhook, enqueues processing job for payment state transitions and subscription updates.

## Integration Points

### New Components

| Component | Type | Purpose |
|-----------|------|---------|
| `src/worker.ts` | Entry point | Standalone NestJS app for background processing |
| `src/worker/worker.module.ts` | Module | Root module for worker app |
| `src/worker/processors/*.processor.ts` | Providers | BullMQ job consumers |
| `src/queue/queue.module.ts` | Module | Shared BullMQ + Redis configuration |
| `src/queue/queue.constants.ts` | Constants | Queue name constants |
| `src/queue/queue.config.ts` | Config | Redis connection configuration |
| `worker-cli.json` | Config | NestJS CLI config for worker builds |
| `nodemon-worker.json` | Config | Dev hot reload for worker |

### Modified Components

| Component | Change | Why |
|-----------|--------|-----|
| `app.module.ts` | Import `QueueModule` | API needs to produce jobs to BullMQ queues |
| `docker-compose.yaml` | Add Redis service + worker service | Worker needs Redis; worker runs as separate container |
| `tsconfig.json` | Add `@queue/*` and `@worker/*` path aliases | Follow existing alias convention for new modules |
| `package.json` | Add `@nestjs/bullmq`, `bullmq`, `ioredis` deps + worker scripts | BullMQ integration + `npm run worker:dev` / `npm run worker:prod` |
| `Dockerfile` | Add worker production stage | Worker deploys independently from API |
| `.env.example` | Add `REDIS_HOST`, `REDIS_PORT` | Document Redis config requirements |
| Jest `moduleNameMapper` in `package.json` | Add `@queue/*` and `@worker/*` mappings | Tests need to resolve new path aliases |

### Unchanged Components

All existing feature modules (`auth/`, `business/`, `core/`, `customer/`, `email/`, `item/`, `job/`, `migration/`, `ping/`, `quote/`, `quote-settings/`, `quote-token/`, `schedule/`, `tax-rate/`, `user/`, `visit-type/`) remain untouched. The worker imports them as-is through NestJS DI.

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| API <-> Worker | BullMQ queue via Redis | Async, decoupled. API produces, worker consumes. No direct process communication. |
| Worker <-> MongoDB | Direct via shared CoreModule | Same connection pattern as API, separate connection pool at runtime |
| API <-> Redis | BullMQ producer connection | Only for queue operations, not caching |
| Worker <-> Redis | BullMQ worker connection | Consumes jobs, reports completion/failure status |

## Build and Run Configuration

### worker-cli.json (NEW)

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "entryFile": "worker",
  "compilerOptions": {
    "deleteOutDir": false,
    "tsConfigPath": "tsconfig.build.json"
  }
}
```

**Key:** `"entryFile": "worker"` points to `src/worker.ts` instead of `src/main.ts`.
**Key:** `"deleteOutDir": false` prevents the worker build from deleting the API's `dist/` output. Both the API and worker compile into the same `dist/` directory but with different entry files (`dist/main.js` and `dist/worker.js`).

### nodemon-worker.json (NEW)

```json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "nest start --watch --config worker-cli.json",
  "delay": "1500",
  "signal": "SIGTERM"
}
```

### package.json script additions

```json
{
  "worker:dev": "nest start --watch --config worker-cli.json",
  "worker:debug": "nest start --debug 9230 --watch --config worker-cli.json",
  "worker:prod": "node dist/worker"
}
```

**Note:** Debug port `9230` avoids conflict with API's default `9229`.

### Docker Compose additions

```yaml
  # Redis for BullMQ
  redis:
    container_name: trade-flow-redis
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Worker Service
  worker:
    container_name: trade-flow-worker
    build:
      context: .
      target: development
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      MONGO_URL: mongodb://mongo:27017
      REDIS_HOST: redis
      REDIS_PORT: 6379
      FIREBASE_PROJECT_ID: ${FIREBASE_PROJECT_ID:-}
      FIREBASE_API_KEY: ${FIREBASE_API_KEY:-}
      RESEND_API_KEY: ${RESEND_API_KEY:-}
    networks:
      - app-network
    command: sh -c "npm install && npm run worker:dev"
```

**Note:** Worker does NOT expose any ports (no HTTP server). It also does NOT need CORS environment variables. The API service also needs `REDIS_HOST: redis` and `REDIS_PORT: 6379` added to its environment block.

### Dockerfile additions (worker production stage)

```dockerfile
# Worker production stage
FROM node:22-slim AS worker-production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

ENV NODE_ENV=production
# No EXPOSE -- worker has no HTTP server

CMD ["npm", "run", "worker:prod"]
```

### tsconfig.json path alias additions

```json
{
  "compilerOptions": {
    "paths": {
      "@queue/*": ["./src/queue/*"],
      "@worker/*": ["./src/worker/*"]
    }
  }
}
```

These follow the exact same pattern as all existing aliases (`@auth/*`, `@business/*`, `@core/*`, etc.).

### Jest moduleNameMapper additions (in package.json)

```json
{
  "jest": {
    "moduleNameMapper": {
      "^@queue/(.*)$": "<rootDir>/queue/$1",
      "^@worker/(.*)$": "<rootDir>/worker/$1"
    }
  }
}
```

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Current (solo operator) | Single API + single worker process. Redis + MongoDB on Docker Compose locally, managed services in production (Railway). |
| 100s of users | Same architecture. BullMQ handles job queuing naturally. If worker falls behind, spin up a second worker instance (same code, same Redis). BullMQ distributes jobs across workers automatically. |
| 1000s of users | Consider dedicated Redis instance (not shared with caching if added later). Separate worker deployments per queue type if processing characteristics differ significantly (e.g., CPU-heavy PDF vs. I/O-bound email). |

### Scaling Priorities

1. **First bottleneck:** Worker throughput on CPU-intensive tasks (PDF generation). Fix: Scale worker instances horizontally -- BullMQ handles job distribution across workers automatically.
2. **Second bottleneck:** Redis memory if queues back up. Fix: Configure job TTL and completed job cleanup in BullMQ defaultJobOptions.

## Anti-Patterns

### Anti-Pattern 1: Full NestJS CLI Monorepo Migration

**What people do:** Run `nest g app worker` which restructures `src/` into `apps/api/src/` and `apps/worker/src/`, switches compiler to webpack, and breaks all existing path aliases.
**Why it's wrong:** Massive migration for an established 21.9k LOC codebase. Webpack introduces new build complexity and different module resolution. All 40+ path aliases need reconfiguring. Every import in every file potentially changes. All CI/CD, Dockerfiles, and scripts break.
**Do this instead:** Use a separate `worker-cli.json` with `"entryFile": "worker"`. Same codebase, zero restructuring.

### Anti-Pattern 2: Worker Importing Entire AppModule

**What people do:** Have WorkerModule import AppModule to get access to all services.
**Why it's wrong:** AppModule includes HTTP middleware configuration (CORS, ValidationPipe, ThrottlerModule), logging config, and all controllers -- none of which apply to a worker. It can also cause port binding conflicts.
**Do this instead:** WorkerModule imports only the modules it needs: CoreModule (global, auto-imported), QueueModule, ConfigModule, LoggerModule, and specific feature modules (e.g., EmailModule, QuoteModule). Cherry-pick dependencies.

### Anti-Pattern 3: Business Logic in Processors

**What people do:** Put database queries, validation, email rendering, and error handling directly in processor classes.
**Why it's wrong:** Processors become untestable without BullMQ mocking. Business logic gets duplicated between API controllers and worker processors. Violates the existing Controller->Service->Repository layering.
**Do this instead:** Processors should be thin -- extract the job payload, call an existing service method, handle the result. Like controllers delegate to services, processors delegate to services.

### Anti-Pattern 4: Shared `libs/common` Dumping Ground

**What people do:** Create a `libs/common/` directory and dump every shared utility, type, constant, and helper there.
**Why it's wrong:** Becomes an unorganized grab-bag that everything depends on. Impossible to understand what depends on what. The NestJS CLI monorepo libs pattern encourages this.
**Do this instead:** The existing `src/core/` module already serves this purpose with clear organization (errors, factories, services, utilities). Keep using it. New shared queue infrastructure goes in `src/queue/`, following the same feature-module pattern.

## Suggested Build Order

Based on dependency analysis, implement in this order:

1. **Redis infrastructure** -- Docker Compose Redis service + env vars in `.env.example` (no code dependencies, can be tested with `redis-cli`)
2. **Dependencies** -- Install `@nestjs/bullmq`, `bullmq`, `ioredis`; add path aliases to `tsconfig.json` and Jest config
3. **Queue module** -- `src/queue/` with BullModule.forRoot config, queue constants, and module definition (depends on Redis + deps)
4. **AppModule integration** -- Import QueueModule into AppModule (depends on queue module; validates API still boots cleanly)
5. **Worker entry point + config** -- `src/worker.ts`, `worker-cli.json`, `nodemon-worker.json`, package.json scripts (depends on queue module)
6. **WorkerModule** -- `src/worker/worker.module.ts` importing CoreModule, ConfigModule, LoggerModule, QueueModule (depends on worker entry point)
7. **First processor** -- Scaffold one simple processor (e.g., a ping/echo processor) to validate the full pipeline end-to-end: API enqueues -> Redis -> Worker processes (depends on WorkerModule)
8. **Docker Compose worker service** -- Add worker container definition, test full `docker-compose up` with API + worker + Redis + MongoDB running together
9. **Dockerfile worker stage** -- Add worker-production build stage for deployment

**Rationale:** Each step can be tested in isolation before the next depends on it. Redis infra first because everything else needs it. Queue module before worker because the API side (producer) is simpler to validate than the consumer side. Worker entry point before processors to confirm the NestJS application context boots cleanly.

## Sources

- [BullMQ NestJS Guide](https://docs.bullmq.io/guide/nestjs) -- HIGH confidence (official BullMQ docs)
- [NestJS Standalone Applications](https://docs.nestjs.com/standalone-applications) -- HIGH confidence (official NestJS docs, documents `createApplicationContext`)
- [NestJS CLI Monorepo Docs](https://docs.nestjs.com/cli/monorepo) -- HIGH confidence (official NestJS docs, explains why NOT to use this for our case)
- [NestJS Queues Technique](https://docs.nestjs.com/techniques/queues) -- HIGH confidence (official NestJS docs)
- [NestJS Standalone BullMQ Worker](https://medium.com/@omarae00/nestjs-standalone-bullmq-worker-6f44faefaf6b) -- MEDIUM confidence (blog, but pattern matches official docs and multiple sources)
- [Running NestJS Queues in Separate Process](https://medium.com/s1seven/running-nestjs-queues-in-a-separate-process-948f414c4b41) -- MEDIUM confidence (blog, consistent with official patterns)
- [NestJS Monorepos Without the Meltdown](https://medium.com/@bhagyarana80/nestjs-monorepos-without-the-meltdown-3a155795ea94) -- MEDIUM confidence (Jan 2026, recent, good anti-pattern guidance)
- [NestJS Microservices Monorepo Setup](https://www.trpkovski.com/2025/10/12/how-to-structure-a-nestjs-project-for-microservices-monorepo-setup/) -- MEDIUM confidence (2025 guide, nest-cli.json patterns)
- Existing codebase (HIGH confidence -- all integration points verified against actual files):
  - `trade-flow-api/src/main.ts` -- existing entry point and bootstrap pattern
  - `trade-flow-api/src/app.module.ts` -- existing module imports and structure
  - `trade-flow-api/src/core/core.module.ts` -- global shared module pattern
  - `trade-flow-api/nest-cli.json` -- existing CLI configuration
  - `trade-flow-api/tsconfig.json` -- existing path aliases (40+ entries)
  - `trade-flow-api/nodemon.json` -- existing dev watch configuration
  - `trade-flow-api/docker-compose.yaml` -- existing Docker Compose structure
  - `trade-flow-api/Dockerfile` -- existing multi-stage build with builder/development/production stages
  - `trade-flow-api/package.json` -- existing scripts and Jest moduleNameMapper

---
*Architecture research for: NestJS monorepo with BullMQ worker service*
*Researched: 2026-03-22*
