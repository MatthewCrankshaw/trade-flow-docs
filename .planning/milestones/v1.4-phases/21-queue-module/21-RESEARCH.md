# Phase 21: Queue Module - Research

**Researched:** 2026-03-22
**Domain:** NestJS BullMQ module configuration, Redis connection, queue producer pattern
**Confidence:** HIGH

## Summary

Phase 21 creates the shared `QueueModule` that connects the NestJS API to Redis via BullMQ. The module uses `BullModule.forRootAsync()` for Redis connection configuration and `BullModule.registerQueue()` for queue registration, exports a `QueueProducer` service that encapsulates all queue operations, and defines queue name constants as the single source of truth.

All dependencies are already installed (Phase 20): `@nestjs/bullmq@11.0.4`, `bullmq@5.71.0`, `ioredis@5.10.1`. The `@queue/*` path alias and `src/queue/` placeholder directory are ready. This phase is purely about writing the module, constants, and producer service, then wiring it into `AppModule`.

**Primary recommendation:** Use `useFactory` with `ConfigService` injection for `BullModule.forRootAsync()`, passing `REDIS_URL` to an `IORedis` connection instance with `maxRetriesPerRequest: null` (required by BullMQ). The `QueueProducer` service uses `@InjectQueue()` decorator and exposes typed methods per queue.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `QueueModule` is `@Global()` -- any module can inject `QueueProducer` without explicit imports (same pattern as `CoreModule`)
- **D-02:** QueueModule exports only `QueueProducer` service -- no processor classes on the API side. Processors live exclusively in the worker (Phase 22)
- **D-03:** QueueModule contains `BullModule.forRootAsync()` (Redis connection) and `BullModule.registerQueue()` (queue registration) in one module
- **D-04:** Single `QueueProducer` service with typed methods per queue (e.g., `enqueueEcho(payload)`) -- central place to discover all queue operations
- **D-05:** QueueProducer injects `@InjectQueue(QUEUE_NAMES.ECHO)` internally -- consumers of the service don't need to know queue names
- **D-06:** `QUEUE_NAMES` typed const object in `src/queue/queue.constant.ts`: `export const QUEUE_NAMES = { ECHO: 'echo' } as const;`
- **D-07:** Only `ECHO` queue defined for Phase 21 -- future queues (EMAIL, PDF) added in their respective milestones
- **D-08:** `REDIS_URL` added to `ConfigModule.forRoot()` load array in `AppModule`, alongside existing `MONGO_URL` -- consistent env var loading
- **D-09:** `BullModule.forRootAsync()` uses `ConfigService.get<string>("REDIS_URL")` -- NO fallback default. Fail fast if REDIS_URL is not set
- **D-10:** QueueModule imported into `AppModule` -- API connects to Redis on startup

### Claude's Discretion
- Exact error message when REDIS_URL is missing
- Whether to use `useFactory` or `useClass` for BullModule.forRootAsync() configuration
- Test structure and mock approach for QueueProducer

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| QUEUE-01 | Shared `QueueModule` with `BullModule.forRootAsync()` using ConfigService for Redis connection | `BullModule.forRootAsync()` accepts `useFactory` with `ConfigService` injection; returns `{ connection: new IORedis(url) }` |
| QUEUE-02 | Queue name constants file (`QUEUE_NAMES`) as single source of truth for producer and consumer | `@InjectQueue(name)` and `@Processor(name)` both accept string queue name; `as const` object provides type safety |
| QUEUE-03 | `QueueModule` imported into `AppModule` -- API connects to Redis and can enqueue jobs | `@Global()` module with `BullModule` imports + `QueueProducer` export; added to `AppModule` imports array |
</phase_requirements>

## Standard Stack

### Core (already installed in Phase 20)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/bullmq | 11.0.4 | NestJS integration for BullMQ queues | Official NestJS queue package; provides `BullModule`, `@InjectQueue`, `@Processor` decorators |
| bullmq | 5.71.0 | Redis-backed job queue library | Industry standard for Node.js job queues; used by BullMQ NestJS module internally |
| ioredis | 5.10.1 | Redis client for Node.js | Required by BullMQ for Redis connectivity; supports URL connection strings |

### Supporting (no new installs needed)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @nestjs/config | 4.0.2 | ConfigService for env vars | Already global; used to read REDIS_URL |

**Installation:** No new packages needed. All were installed in Phase 20.

## Architecture Patterns

### Module File Structure
```
src/queue/
  queue.module.ts          # @Global() module with BullModule config + QueueProducer
  queue.constant.ts        # QUEUE_NAMES typed const object
  services/
    queue-producer.service.ts  # Single producer service with typed methods
  test/
    services/
      queue-producer.service.spec.ts  # Unit tests for QueueProducer
```

### Pattern 1: BullModule.forRootAsync() with useFactory

**What:** Configure Redis connection asynchronously using ConfigService injection.
**When to use:** Always -- the app needs ConfigService to read REDIS_URL from environment.

```typescript
// Source: @nestjs/bullmq type definitions (verified from node_modules)
import { BullModule } from "@nestjs/bullmq";
import { ConfigService } from "@nestjs/config";
import IORedis from "ioredis";

BullModule.forRootAsync({
  useFactory: (configService: ConfigService) => {
    const redisUrl = configService.get<string>("REDIS_URL");
    if (!redisUrl) {
      throw new Error("REDIS_URL environment variable is not configured");
    }
    return {
      connection: new IORedis(redisUrl, {
        maxRetriesPerRequest: null, // Required by BullMQ
      }),
    };
  },
  inject: [ConfigService],
}),
```

**Why `useFactory` over `useClass`:** The factory is a 5-line function. Creating a separate factory class for this adds a file and boilerplate with zero benefit. The codebase has no precedent for `useClass` pattern in async module configuration.

### Pattern 2: Queue Registration

**What:** Register named queues so `@InjectQueue()` can inject them.
**When to use:** For each queue the API needs to produce jobs into.

```typescript
// Source: @nestjs/bullmq type definitions
BullModule.registerQueue({
  name: QUEUE_NAMES.ECHO,
}),
```

### Pattern 3: @Global() Module with Exports (follows CoreModule)

**What:** QueueModule is `@Global()` so any module can inject `QueueProducer` without importing QueueModule.
**When to use:** Exactly this phase -- matches the existing `CoreModule` pattern.

```typescript
// Source: trade-flow-api/src/core/core.module.ts (verified)
@Global()
@Module({
  imports: [
    BullModule.forRootAsync({ /* ... */ }),
    BullModule.registerQueue({ name: QUEUE_NAMES.ECHO }),
  ],
  providers: [QueueProducer],
  exports: [QueueProducer],
})
export class QueueModule {}
```

### Pattern 4: QueueProducer Service with Typed Methods

**What:** A single injectable service that encapsulates all queue operations.
**When to use:** Any module that needs to enqueue jobs injects `QueueProducer` and calls typed methods.

```typescript
// Source: @nestjs/bullmq @InjectQueue decorator (verified from node_modules)
import { InjectQueue } from "@nestjs/bullmq";
import { Queue } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";

@Injectable()
export class QueueProducer {
  constructor(
    @InjectQueue(QUEUE_NAMES.ECHO) private readonly echoQueue: Queue,
  ) {}

  public async enqueueEcho(payload: { message: string }): Promise<void> {
    await this.echoQueue.add("echo", payload);
  }
}
```

### Pattern 5: REDIS_URL in ConfigModule.forRoot() Load Array

**What:** Add REDIS_URL to the existing load callback in AppModule's ConfigModule.forRoot().
**When to use:** This phase -- extends the existing pattern.

```typescript
// Current (trade-flow-api/src/app.module.ts line 27-31):
load: [
  () => ({
    MONGO_URL: process.env.MONGO_URL,
  }),
],

// After Phase 21:
load: [
  () => ({
    MONGO_URL: process.env.MONGO_URL,
    REDIS_URL: process.env.REDIS_URL,
  }),
],
```

### Pattern 6: Queue Name Constants

**What:** Typed const object for queue names -- single source of truth.
**When to use:** Anywhere queue names are referenced (producer, consumer, processor).

```typescript
// Follows codebase constant naming pattern (UPPER_SNAKE_CASE keys, lowercase values)
export const QUEUE_NAMES = {
  ECHO: "echo",
} as const;

export type QueueName = (typeof QUEUE_NAMES)[keyof typeof QUEUE_NAMES];
```

### Anti-Patterns to Avoid
- **String literals for queue names:** Always use `QUEUE_NAMES.ECHO`, never `"echo"` directly. The constant ensures producer and consumer stay in sync.
- **Omitting `maxRetriesPerRequest: null`:** BullMQ requires this IORedis option. Without it, IORedis will throw after 20 retries and break queue operations.
- **Providing a fallback for REDIS_URL:** D-09 explicitly says fail fast. Do NOT use `?? "redis://localhost:6379"` in QueueModule. (Note: Phase 20 D-08 used a fallback in the `.env.example` documentation, but the QueueModule code itself must throw if REDIS_URL is missing.)

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Redis connection management | Custom IORedis wrapper | `BullModule.forRootAsync()` | Handles connection pooling, reconnection, cleanup automatically |
| Queue injection | Manual Queue instantiation | `@InjectQueue()` decorator | NestJS DI manages lifecycle; testable via mock injection |
| Queue registration | Direct `new Queue()` calls | `BullModule.registerQueue()` | Ensures queue is properly registered in NestJS DI container |

**Key insight:** `@nestjs/bullmq` handles all the Redis connection lifecycle. The only manual piece is the IORedis instance creation in the factory, which is required because BullMQ needs `maxRetriesPerRequest: null`.

## Common Pitfalls

### Pitfall 1: Missing maxRetriesPerRequest: null
**What goes wrong:** IORedis defaults to `maxRetriesPerRequest: 20`. BullMQ workers and queue listeners expect indefinite retries. Without `null`, operations fail with `ReplyError: max retries per request` under transient Redis disconnections.
**Why it happens:** This is an IORedis default that conflicts with BullMQ's design.
**How to avoid:** Always pass `{ maxRetriesPerRequest: null }` as the second argument to `new IORedis(url, options)`.
**Warning signs:** `ReplyError` exceptions in logs after brief Redis interruptions.

### Pitfall 2: IORedis Default Import
**What goes wrong:** TypeScript compilation fails or IORedis constructor is not a function.
**Why it happens:** IORedis uses a default export with CommonJS interop. The import must be `import IORedis from "ioredis"` (default import), not `import { IORedis } from "ioredis"` (named import).
**How to avoid:** Use `import IORedis from "ioredis"` consistently. The project has `esModuleInterop: true` and `allowSyntheticDefaultImports: true` in tsconfig.json which enables this.
**Warning signs:** `TypeError: IORedis is not a constructor` at runtime.

### Pitfall 3: BullModule.forRootAsync() Must Come Before registerQueue()
**What goes wrong:** If `registerQueue()` is listed in imports before `forRootAsync()`, the queue may not find the root connection configuration.
**Why it happens:** NestJS processes imports in order; the root connection must be established first.
**How to avoid:** Always list `BullModule.forRootAsync()` as the first BullModule import, followed by `BullModule.registerQueue()`.
**Warning signs:** `Connection is not established` or missing provider errors at startup.

### Pitfall 4: Forgetting to Add REDIS_URL to ConfigModule Load Array
**What goes wrong:** `ConfigService.get<string>("REDIS_URL")` returns `undefined` even when the env var is set.
**Why it happens:** The existing `ConfigModule.forRoot()` in AppModule uses a `load` callback that explicitly maps env vars. Variables not in the load callback may not be accessible via ConfigService depending on configuration.
**How to avoid:** Add `REDIS_URL: process.env.REDIS_URL` to the existing load callback in AppModule.
**Warning signs:** QueueModule throws "REDIS_URL not configured" despite the var being in `.env`.

## Code Examples

### Complete QueueModule Implementation

```typescript
// src/queue/queue.module.ts
// Source: Composed from @nestjs/bullmq types + CoreModule pattern
import { Global, Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bullmq";
import { ConfigService } from "@nestjs/config";
import IORedis from "ioredis";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { QueueProducer } from "@queue/services/queue-producer.service";

@Global()
@Module({
  imports: [
    BullModule.forRootAsync({
      useFactory: (configService: ConfigService) => {
        const redisUrl = configService.get<string>("REDIS_URL");
        if (!redisUrl) {
          throw new Error("REDIS_URL environment variable is not configured. Set REDIS_URL in your .env file.");
        }
        return {
          connection: new IORedis(redisUrl, {
            maxRetriesPerRequest: null,
          }),
        };
      },
      inject: [ConfigService],
    }),
    BullModule.registerQueue({ name: QUEUE_NAMES.ECHO }),
  ],
  providers: [QueueProducer],
  exports: [QueueProducer],
})
export class QueueModule {}
```

### Complete Queue Constants

```typescript
// src/queue/queue.constant.ts
export const QUEUE_NAMES = {
  ECHO: "echo",
} as const;

export type QueueName = (typeof QUEUE_NAMES)[keyof typeof QUEUE_NAMES];
```

### Complete QueueProducer Service

```typescript
// src/queue/services/queue-producer.service.ts
import { Injectable } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bullmq";
import { Queue } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

@Injectable()
export class QueueProducer {
  private readonly logger = new AppLogger(QueueProducer.name);

  constructor(
    @InjectQueue(QUEUE_NAMES.ECHO) private readonly echoQueue: Queue,
  ) {}

  public async enqueueEcho(payload: { message: string }): Promise<void> {
    await this.echoQueue.add("echo", payload);
    this.logger.log("Echo job enqueued", { message: payload.message });
  }
}
```

### Updated AppModule (changes only)

```typescript
// src/app.module.ts -- additions
import { QueueModule } from "@queue/queue.module";

// In ConfigModule.forRoot() load array, add:
REDIS_URL: process.env.REDIS_URL,

// In imports array, add:
QueueModule,
```

### QueueProducer Test Pattern

```typescript
// src/queue/test/services/queue-producer.service.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { getQueueToken } from "@nestjs/bullmq";
import { QueueProducer } from "@queue/services/queue-producer.service";
import { QUEUE_NAMES } from "@queue/queue.constant";

describe("QueueProducer", () => {
  let producer: QueueProducer;
  const mockEchoQueue = { add: jest.fn() };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        QueueProducer,
        { provide: getQueueToken(QUEUE_NAMES.ECHO), useValue: mockEchoQueue },
      ],
    }).compile();

    producer = module.get<QueueProducer>(QueueProducer);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe("enqueueEcho", () => {
    it("should add a job to the echo queue", async () => {
      const payload = { message: "test" };
      await producer.enqueueEcho(payload);
      expect(mockEchoQueue.add).toHaveBeenCalledWith("echo", payload);
    });
  });
});
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `@nestjs/bull` (Bull v3) | `@nestjs/bullmq` (BullMQ v5) | NestJS 10+ | BullMQ has better TypeScript support, Flow producers, and is actively maintained |
| `connection: { host, port }` | `connection: new IORedis(url)` | BullMQ 4+ | URL strings are simpler and consistent with REDIS_URL env var pattern |

**Deprecated/outdated:**
- `@nestjs/bull` package: Replaced by `@nestjs/bullmq`. The project correctly uses the newer package.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 |
| Config file | `package.json` jest section |
| Quick run command | `npm test -- --testPathPattern queue` |
| Full suite command | `npm test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| QUEUE-01 | QueueModule configures BullModule with ConfigService | smoke | `npm run start:dev` (boots without error) | N/A -- startup test |
| QUEUE-02 | QUEUE_NAMES constant defines ECHO queue name | unit | `npm test -- --testPathPattern queue-producer` | Wave 0 |
| QUEUE-03 | API connects to Redis on startup with QueueModule in AppModule | smoke | `npm run start:dev` (boots, connects to Redis) | N/A -- startup test |

### Sampling Rate
- **Per task commit:** `npm run validate` (typecheck + lint)
- **Per wave merge:** `npm test` (full 306+ test suite)
- **Phase gate:** Full suite green + `npm run start:dev` boots without errors

### Wave 0 Gaps
- [ ] `src/queue/test/services/queue-producer.service.spec.ts` -- covers QUEUE-02 (enqueueEcho calls queue.add)
- [ ] Framework install: None needed -- Jest already configured with @queue/* moduleNameMapper alias

## Open Questions

1. **IORedis connection sharing between forRootAsync and registerQueue**
   - What we know: `BullModule.forRootAsync()` creates the root connection, `registerQueue()` inherits it automatically.
   - What's unclear: Whether BullMQ creates a new IORedis connection per queue or shares the root one.
   - Recommendation: Not a concern for Phase 21 (single queue). If connection count becomes relevant later, investigate BullMQ connection pooling docs.

## Sources

### Primary (HIGH confidence)
- `@nestjs/bullmq@11.0.4` type definitions -- `BullModule.forRootAsync()`, `SharedBullAsyncConfiguration`, `@InjectQueue`, `getQueueToken` APIs verified from `node_modules/@nestjs/bullmq/dist/`
- `ioredis@5.10.1` -- URL constructor verified with runtime test (accepts `redis://localhost:6379`)
- `trade-flow-api/src/core/core.module.ts` -- `@Global()` module pattern verified
- `trade-flow-api/src/app.module.ts` -- ConfigModule.forRoot() load array and imports array structure verified
- `trade-flow-api/src/email/services/email-sender.service.ts` -- ConfigService injection pattern verified

### Secondary (MEDIUM confidence)
- BullMQ `maxRetriesPerRequest: null` requirement -- widely documented in BullMQ ecosystem, consistent with library design

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all packages installed and type definitions verified locally
- Architecture: HIGH -- module pattern matches existing CoreModule; API verified from type definitions
- Pitfalls: HIGH -- `maxRetriesPerRequest` and IORedis import issues are well-documented and verified against installed versions

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- NestJS 11 + BullMQ 5 are current majors)
