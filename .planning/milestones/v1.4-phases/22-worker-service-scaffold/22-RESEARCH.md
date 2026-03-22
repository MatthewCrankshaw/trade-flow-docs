# Phase 22: Worker Service Scaffold - Research

**Researched:** 2026-03-22
**Domain:** NestJS worker process with BullMQ processor, dual entry-point build
**Confidence:** HIGH

## Summary

This phase creates a standalone NestJS worker process that boots without an HTTP server, consumes jobs from the BullMQ echo queue (built in Phase 21), and proves end-to-end queue flow. It also adds a diagnostic `POST /v1/queue/test-echo` endpoint to the API for enqueuing test jobs.

The technical surface is well-understood: `NestFactory.createApplicationContext()` for non-HTTP NestJS apps, `@Processor` decorator with `WorkerHost` for BullMQ consumers, and a separate `nest-cli.json` config for independent builds. All dependencies are already installed (@nestjs/bullmq 11.0.4, bullmq 5.71.0). The existing codebase provides clear patterns for modules, controllers, logging, and throttling.

**Primary recommendation:** Follow existing codebase patterns exactly -- the worker is a thin `WorkerModule` importing only shared infrastructure modules, with an echo processor extending `WorkerHost`, and a `worker-cli.json` that overrides `entryFile` to `"worker"`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `POST /v1/queue/test-echo` endpoint enqueues a fixed-payload echo job -- no request body, no auth (open like ping)
- **D-02:** Endpoint lives in a new `QueueController` in the queue module -- keeps queue concerns together, separate from PingController
- **D-03:** Both ping and echo-test endpoints throttled at 6 requests per minute using existing throttler infrastructure
- **D-04:** Permanent diagnostic endpoint -- kept as a queue health check, not removed when real jobs arrive
- **D-05:** Fixed payload (no user input) -- worker logs a predetermined message proving the pipeline works
- **D-06:** `@Processor` classes live in `src/worker/processors/` -- worker-owned, never imported by API
- **D-07:** Clean split: `src/queue/` is shared infrastructure (module, constants, producer), `src/worker/` is consumer-side code (processors, WorkerModule, entry point)
- **D-08:** Pino `service` field distinguishes processes -- worker logs `"service": "worker"`, API logs `"service": "api"`
- **D-09:** Same Pino configuration for both, just the service identifier differs -- enables structured filtering with grep or log tools

### Claude's Discretion
- Exact fixed payload content for the echo job
- Worker bootstrap logging messages (startup, shutdown)
- WorkerModule provider registration order
- Test file structure and mock approach for echo processor

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| WORK-01 | `src/worker.ts` entry point using `NestFactory.createApplicationContext()` with SIGTERM graceful shutdown | Verified `createApplicationContext` exists in @nestjs/core. Existing `main.ts` provides SIGTERM/SIGINT pattern to follow. Worker uses `app.close()` for graceful shutdown. |
| WORK-02 | `WorkerModule` imports only CoreModule, ConfigModule, LoggerModule, and QueueModule (not full AppModule) | CoreModule is `@Global()`, ConfigModule uses `isGlobal: true`. WorkerModule needs its own `ConfigModule.forRoot()` and `LoggerModule.forRoot()` since those are registered in AppModule, not globally inheritable. QueueModule is `@Global()` but must still be imported at least once. |
| WORK-03 | Echo `@Processor` that logs received jobs, proving end-to-end queue flow | @nestjs/bullmq 11.0.4 provides `@Processor(queueName)` decorator and `WorkerHost` abstract class with `process(job)` method. QUEUE_NAMES.ECHO = "echo" constant already exists. |
| WORK-04 | `worker-cli.json` with `entryFile: "worker"` for independent build/start | Existing `nest-cli.json` structure confirmed. Worker config overrides `entryFile` from default "main" to "worker". `nest build --config worker-cli.json` produces `dist/worker.js`. |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/core | 11.1.12 | NestFactory.createApplicationContext | Installed, provides non-HTTP app context |
| @nestjs/bullmq | 11.0.4 | @Processor decorator, WorkerHost class | Installed, NestJS BullMQ integration |
| bullmq | 5.71.0 | Queue/Worker/Job types | Installed, underlying queue library |
| nestjs-pino | 4.5.0 | LoggerModule for worker process | Installed, structured logging |

### No New Dependencies
This phase requires zero new npm installations. All dependencies were installed in Phases 20-21.

## Architecture Patterns

### Worker Entry Point (`src/worker.ts`)

The worker uses `NestFactory.createApplicationContext()` instead of `NestFactory.create()`. This creates a NestJS dependency injection container without starting an HTTP server.

```typescript
// Source: @nestjs/core NestFactory API + existing main.ts pattern
import { NestFactory } from "@nestjs/core";
import { Logger } from "@nestjs/common";
import { Logger as PinoLogger } from "nestjs-pino";
import { WorkerModule } from "@worker/worker.module";

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(WorkerModule, {
    bufferLogs: true,
  });

  app.useLogger(app.get(PinoLogger));

  const logger = new Logger("Worker");
  logger.log("Worker service started");

  process.on("SIGTERM", async () => {
    logger.log("SIGTERM received, shutting down gracefully");
    await app.close();
    process.exit(0);
  });

  process.on("SIGINT", async () => {
    logger.log("SIGINT received, shutting down gracefully");
    await app.close();
    process.exit(0);
  });
}
bootstrap();
```

**Key difference from main.ts:** No `app.listen()`, no CORS, no ValidationPipe, no global pipes. The application context just boots the DI container and processors start consuming.

### WorkerModule Structure

```
src/worker/
  worker.module.ts        # Module definition
  processors/
    echo.processor.ts     # Echo job consumer
  test/
    processors/
      echo.processor.spec.ts
```

WorkerModule must declare its own `ConfigModule.forRoot()` and `LoggerModule.forRoot()` because these are registered inside `AppModule` -- the worker does not import AppModule. CoreModule is `@Global()` so it is available once imported. QueueModule is `@Global()` so it is available once imported.

```typescript
// WorkerModule imports
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ".env",
      load: [() => ({ REDIS_URL: process.env.REDIS_URL })],
    }),
    LoggerModule.forRoot(workerLoggerConfig),  // with service: "worker"
    CoreModule,
    QueueModule,
  ],
  providers: [EchoProcessor],
})
export class WorkerModule {}
```

### Processor Pattern (`@Processor` + `WorkerHost`)

```typescript
// Source: @nestjs/bullmq installed types (node_modules/@nestjs/bullmq/dist)
import { Processor, WorkerHost } from "@nestjs/bullmq";
import { Job } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

@Processor(QUEUE_NAMES.ECHO)
export class EchoProcessor extends WorkerHost {
  private readonly logger = new AppLogger(EchoProcessor.name);

  async process(job: Job<{ message: string }>): Promise<void> {
    this.logger.log("Echo job received", {
      jobId: job.id,
      message: job.data.message,
    });
  }
}
```

**Critical:** The processor class MUST extend `WorkerHost` and implement the abstract `process(job, token?)` method. The `@Processor` decorator takes the queue name string.

### Logger Config with Service Field (D-08, D-09)

The worker needs a modified logger config that adds `service: "worker"` to all log entries. The API should also add `service: "api"` for parity.

```typescript
// Worker logger config -- same as existing loggerConfig but with service field
export const workerLoggerConfig: Params = {
  pinoHttp: {
    // ... same as existing loggerConfig
    base: { service: "worker" },  // Pino base object merged into every log
  },
};
```

For the API side, update `loggerConfig` to include `base: { service: "api" }`.

**Note:** The `base` property in Pino config merges into every log entry. Setting `base: { service: "worker" }` means every log line includes `"service":"worker"` in JSON mode, or `[worker]` in pretty-print mode.

### QueueController (test-echo endpoint)

```typescript
// New controller in src/queue/controllers/queue.controller.ts
@Controller("v1/queue")
@UseGuards(ThrottlerGuard)
export class QueueController {
  constructor(private readonly queueProducer: QueueProducer) {}

  @Throttle({ default: { limit: 6, ttl: 60000 } })
  @Post("test-echo")
  public async testEcho(): Promise<IResponse<{ message: string }>> {
    await this.queueProducer.enqueueEcho({ message: "Echo test from API" });
    return createResponse([{ message: "Echo job enqueued" }]);
  }
}
```

**Throttler setup:** The existing codebase uses `ThrottlerModule.forRoot()` at the module level (see `QuoteTokenModule`). The queue module needs to import `ThrottlerModule.forRoot([{ name: "queue", ttl: 60000, limit: 6 }])` and apply `@UseGuards(ThrottlerGuard)` + `@Throttle()` on the controller.

### worker-cli.json

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": false,
    "entryFile": "worker"
  }
}
```

**Critical:** `deleteOutDir` must be `false` for the worker build config. If both `nest build` (main) and `nest build --config worker-cli.json` (worker) use `deleteOutDir: true`, they will delete each other's output. Only one should clean the dist directory.

### Build and Run Commands

```bash
# Build worker separately
nest build --config worker-cli.json

# Run worker in production
node dist/worker.js

# Build both (for CI/deployment)
nest build && nest build --config worker-cli.json
```

### Anti-Patterns to Avoid
- **Importing AppModule in WorkerModule:** Would pull in all HTTP modules, controllers, auth guards -- massive unnecessary overhead
- **Using `NestFactory.create()` for worker:** Would start an HTTP server; worker must use `createApplicationContext()`
- **Putting processors in `src/queue/`:** Processors are consumer-side; they belong in `src/worker/processors/` per D-06/D-07
- **Using `deleteOutDir: true` in worker-cli.json:** Would delete `dist/main.js` when building worker

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Queue job processing | Custom Redis polling loop | @nestjs/bullmq `@Processor` + `WorkerHost` | Handles connection management, job locking, error recovery |
| Process identification in logs | Custom log prefix/wrapper | Pino `base` config with `service` field | Native Pino feature, works with both JSON and pretty-print |
| Rate limiting | Custom counter middleware | @nestjs/throttler `ThrottlerGuard` + `@Throttle()` | Already in use for public endpoints |
| Separate build entry point | Custom webpack/esbuild config | `nest build --config worker-cli.json` with `entryFile` override | NestJS CLI native feature |

## Common Pitfalls

### Pitfall 1: WorkerModule missing ConfigModule.forRoot()
**What goes wrong:** Worker crashes on startup because ConfigService has no providers
**Why it happens:** ConfigModule.forRoot() is declared in AppModule, not globally available. Worker doesn't import AppModule.
**How to avoid:** WorkerModule must declare its own `ConfigModule.forRoot({ isGlobal: true, envFilePath: ".env", load: [...] })` with the same env loading as AppModule plus REDIS_URL.
**Warning signs:** `Nest could not find ConfigService element` error on worker startup

### Pitfall 2: deleteOutDir conflicts between build configs
**What goes wrong:** Building worker deletes `dist/main.js`, or building API deletes `dist/worker.js`
**Why it happens:** Both nest-cli.json configs have `deleteOutDir: true`
**How to avoid:** Set `deleteOutDir: false` in `worker-cli.json`. The main `nest-cli.json` can keep `deleteOutDir: true` since it runs first.
**Warning signs:** Missing entry point file after sequential builds

### Pitfall 3: Processor not discovered by BullMQ
**What goes wrong:** Jobs sit in queue but never get processed
**Why it happens:** Processor class not registered as a provider in WorkerModule, or QueueModule not imported
**How to avoid:** EchoProcessor must be in WorkerModule's `providers` array, and QueueModule (which registers the queue via `BullModule.registerQueue()`) must be imported
**Warning signs:** No errors, but jobs remain in "waiting" state

### Pitfall 4: Worker logger missing Pino setup
**What goes wrong:** Worker logs go to stdout as plain text or NestJS default format
**Why it happens:** `LoggerModule.forRoot()` not imported in WorkerModule, or `app.useLogger()` not called in worker bootstrap
**How to avoid:** Import LoggerModule with worker-specific config, call `app.useLogger(app.get(PinoLogger))` in worker.ts bootstrap
**Warning signs:** Log output doesn't match API's structured format

### Pitfall 5: ThrottlerModule scope confusion
**What goes wrong:** Throttle guard throws "Cannot determine scope" error
**Why it happens:** ThrottlerModule imported at wrong level or not at all in QueueModule
**How to avoid:** Import `ThrottlerModule.forRoot([...])` in QueueModule alongside the controller that uses it. Follow the existing QuoteTokenModule pattern.
**Warning signs:** 500 error when hitting the test-echo endpoint

### Pitfall 6: LoggerModule pinoHttp in worker context
**What goes wrong:** Worker crashes or logs warnings about pinoHttp in non-HTTP context
**Why it happens:** `LoggerModule.forRoot()` with `pinoHttp` config expects HTTP request/response objects. Worker has no HTTP server.
**How to avoid:** The worker logger config should use `pinoHttp: { autoLogging: false }` since there are no HTTP requests to log. The `base` and `transport` settings still work fine. Alternatively, use `{ pinoHttp: { enabled: false } }` if LoggerModule supports it -- but the safest approach is keeping `pinoHttp` config and just disabling `autoLogging`.
**Warning signs:** Warnings about missing `req`/`res` in log serializers

## Code Examples

### Echo Processor (verified pattern from installed @nestjs/bullmq types)
```typescript
// src/worker/processors/echo.processor.ts
import { Processor, WorkerHost } from "@nestjs/bullmq";
import { Job } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

@Processor(QUEUE_NAMES.ECHO)
export class EchoProcessor extends WorkerHost {
  private readonly logger = new AppLogger(EchoProcessor.name);

  async process(job: Job<{ message: string }>): Promise<void> {
    this.logger.log("Echo job processed", {
      jobId: job.id,
      message: job.data.message,
    });
  }
}
```

### QueueController (follows PublicQuoteController throttler pattern)
```typescript
// src/queue/controllers/queue.controller.ts
import { Controller, Post, UseGuards } from "@nestjs/common";
import { Throttle, ThrottlerGuard } from "@nestjs/throttler";
import { IResponse } from "@core/response/response.interface";
import { createResponse } from "@core/response/create-response.utility";
import { createHttpError } from "@core/errors/handle-error.utility";
import { QueueProducer } from "@queue/services/queue-producer.service";

@Controller("v1/queue")
@UseGuards(ThrottlerGuard)
export class QueueController {
  constructor(private readonly queueProducer: QueueProducer) {}

  @Throttle({ default: { limit: 6, ttl: 60000 } })
  @Post("test-echo")
  public async testEcho(): Promise<IResponse<{ message: string }>> {
    try {
      await this.queueProducer.enqueueEcho({ message: "Echo test from API" });
      return createResponse([{ message: "Echo job enqueued" }]);
    } catch (error) {
      throw createHttpError(error);
    }
  }
}
```

### Worker Logger Config (service field differentiation)
```typescript
// src/worker/config/worker-logger.config.ts
import { Params } from "nestjs-pino";

export const workerLoggerConfig: Params = {
  pinoHttp: {
    level: process.env.LOG_LEVEL || "info",
    autoLogging: false,  // No HTTP requests in worker
    base: { service: "worker" },
    transport:
      process.env.NODE_ENV !== "production"
        ? {
            target: "pino-pretty",
            options: {
              colorize: true,
              singleLine: true,
              translateTime: "SYS:standard",
              ignore: "pid,hostname",
            },
          }
        : undefined,
  },
};
```

### Echo Processor Test (follows existing queue-producer.service.spec.ts pattern)
```typescript
// src/worker/test/processors/echo.processor.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { Job } from "bullmq";
import { EchoProcessor } from "@worker/processors/echo.processor";

describe("EchoProcessor", () => {
  let processor: EchoProcessor;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [EchoProcessor],
    }).compile();

    processor = module.get<EchoProcessor>(EchoProcessor);
  });

  afterEach(() => jest.clearAllMocks());

  describe("process", () => {
    it("should log the echo job payload", async () => {
      const mockJob = { id: "test-1", data: { message: "hello" } } as Job<{ message: string }>;

      await expect(processor.process(mockJob)).resolves.toBeUndefined();
    });
  });
});
```

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | `package.json` jest section |
| Quick run command | `npm test -- --testPathPattern="worker\|queue.*controller" --no-coverage` |
| Full suite command | `npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| WORK-01 | Worker entry point boots and shuts down | manual-only | Manual: `node dist/worker.js` + SIGTERM | N/A -- bootstrap script, not unit testable |
| WORK-02 | WorkerModule imports correct modules only | unit | `npm test -- --testPathPattern="worker.module" --no-coverage` | Wave 0 |
| WORK-03 | EchoProcessor processes jobs and logs | unit | `npm test -- --testPathPattern="echo.processor" --no-coverage` | Wave 0 |
| WORK-04 | worker-cli.json produces dist/worker.js | manual-only | `nest build --config worker-cli.json && test -f dist/worker.js` | N/A -- build config |
| D-01 | QueueController test-echo endpoint | unit | `npm test -- --testPathPattern="queue.controller" --no-coverage` | Wave 0 |

### Sampling Rate
- **Per task commit:** `npm test -- --testPathPattern="worker\|queue" --no-coverage`
- **Per wave merge:** `npm test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/worker/test/processors/echo.processor.spec.ts` -- covers WORK-03
- [ ] `src/queue/test/controllers/queue.controller.spec.ts` -- covers D-01
- [ ] No framework install needed -- Jest already configured

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/node_modules/@nestjs/bullmq/dist/decorators/processor.decorator.d.ts` -- Processor decorator API, overloads
- `trade-flow-api/node_modules/@nestjs/bullmq/dist/hosts/worker-host.class.d.ts` -- WorkerHost abstract class with `process(job, token?)` method
- `trade-flow-api/src/main.ts` -- Existing bootstrap pattern (SIGTERM, bufferLogs, Pino logger)
- `trade-flow-api/nest-cli.json` -- Existing NestJS CLI config structure
- `trade-flow-api/src/queue/queue.module.ts` -- Phase 21 QueueModule with BullModule.forRootAsync()
- `trade-flow-api/src/queue/services/queue-producer.service.ts` -- QueueProducer.enqueueEcho() method
- `trade-flow-api/src/quote-token/quote-token.module.ts` -- ThrottlerModule.forRoot() usage pattern
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` -- @Throttle + @UseGuards(ThrottlerGuard) pattern
- `trade-flow-api/src/core/config/logger.config.ts` -- Existing Pino logger configuration
- `trade-flow-api/src/core/services/app-logger.service.ts` -- AppLogger wrapper class

### Secondary (MEDIUM confidence)
- NestJS documentation on `createApplicationContext` for non-HTTP contexts
- Pino `base` configuration property for service field injection

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all packages already installed, versions verified via `npm list`
- Architecture: HIGH -- patterns derived from existing codebase files and installed type definitions
- Pitfalls: HIGH -- identified from concrete code analysis (ConfigModule scope, deleteOutDir, LoggerModule HTTP context)

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable NestJS/BullMQ APIs, no breaking changes expected)
