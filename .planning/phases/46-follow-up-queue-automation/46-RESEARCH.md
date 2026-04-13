# Phase 46: Follow-up Queue & Automation - Research

**Researched:** 2026-04-13
**Domain:** BullMQ delayed jobs, Redis AOF persistence, NestJS processor pattern, Maizzle email templates
**Confidence:** HIGH

## Summary

Phase 46 is an infrastructure-and-backend-only phase that wires BullMQ delayed job scheduling into the estimate send lifecycle. The foundation (v1.4 worker infrastructure, STRIPE_WEBHOOKS processor pattern, Phase 42 `IEstimateFollowupCanceller` interface, Phase 44 `estimate_email_sends` collection) is already designed and waiting for this phase to fill in. There is zero UI work and no new npm dependencies required.

The phase has three distinct concerns: (1) a **scheduler** that enqueues 4 delayed jobs per send (3 follow-ups + 1 expiry), (2) two **processor classes** in the worker that execute those jobs (one for follow-ups, one for expiry), and (3) the **rebinding** of the `ESTIMATE_FOLLOWUP_CANCELLER` DI token from `NoopEstimateFollowupCanceller` to the real `BullMQEstimateFollowupCanceller`. A fourth concern is a Redis AOF startup check at worker boot — the infra gate required by FUP-08.

The STRIPE_WEBHOOKS processor (Phase 30/v1.6) is the canonical precedent for everything in this phase: queue constant addition, processor class structure (`@Processor` + `WorkerHost`), `WorkerModule` registration, and the deterministic-jobId deduplication pattern. Phase 46 follows that pattern exactly and introduces delayed-job mechanics on top.

**Primary recommendation:** Copy-adapt the STRIPE_WEBHOOKS processor pattern. Add `ESTIMATE_FOLLOWUPS` to `QUEUE_NAMES`, create `EstimateFollowupsModule` that rebinds the canceller token, and write `EstimateFollowupProcessor` + `EstimateExpiryProcessor` following the `WorkerHost` pattern exactly. The AOF check runs once at worker bootstrap before processor registration.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Follow-up Email Content (D-EMAIL)**
- D-EMAIL-01: All three follow-up steps (3d/10d/21d) use the same friendly, professional tone. No escalating urgency, no "last chance" language.
- D-EMAIL-02: Follow-up emails are brief reminders with a link, not full estimate summaries. Subject references the estimate number; body has a short message and a "View Estimate" button.
- D-EMAIL-03: One shared Maizzle template `estimate-followup.html` with a template variable for the step (`followupStep: "3d" | "10d" | "21d"`).
- D-EMAIL-04: All three CTAs appear in every follow-up email:
  - Primary: "Go Ahead" or "Happy to Proceed"
  - Secondary: "Message [Tradesperson First Name]"
  - Tertiary: "Not right for me" as text link
  - Disclaimer: "This is an estimate — the final cost may differ from this approximate figure."
- D-EMAIL-05: Follow-up emails use the same document-token link as the initial send (token references `rootEstimateId`, not per-revision id).

**Expiry Behaviour (D-EXP)**
- D-EXP-01: Auto-expiry is a silent transition. No email notification to trader.
- D-EXP-02: Customer sees friendly expired message on public page after expiry (Phase 45 owns the rendering).
- D-EXP-03: Expiry uses a per-estimate delayed BullMQ job at 720h (30 days). Deterministic jobId: `estimate-expiry:{estimateId}:{revisionNumber}`.
- D-EXP-04: `cancelAllFollowups()` cancels 4 jobs total: 3 follow-up jobIds + 1 expiry jobId.

**Scheduling Trigger (D-SCHED)**
- D-SCHED-01: Follow-ups scheduled immediately on send.
- D-SCHED-02: `scheduleFollowups()` lives inside `EstimateEmailSender` as step 9 in D-AUDIT-08 ordering.
- D-SCHED-03: No trader-facing UI for follow-ups. Zero UI in Phase 46.
- D-SCHED-04: On pure re-send path (same revision, status Sent/Viewed/Responded/SiteVisitRequested → Sent), follow-ups are NOT re-scheduled.
- D-SCHED-05: On revised-send path (new revision, Draft → Sent), step 8 cancels old-revision follow-ups, step 9 schedules new ones.
- D-SCHED-06: `scheduleFollowups()` uses deterministic jobIds. A second call for same estimate/revision is a silent no-op.

**Redis AOF Infrastructure Gate (D-AOF)**
- D-AOF-01: Runtime startup check: worker checks `CONFIG GET appendonly` on Redis connection. If not `"yes"`, log AppLogger.error (FATAL) and refuse to register the `ESTIMATE_FOLLOWUPS` processor. Worker still starts; other processors remain functional.
- D-AOF-02: Docker Compose Redis config updated with `appendonly yes` and `appendfsync everysec`.
- D-AOF-03: SC #4 smoke test is a manual procedure documented in the plan's verification section. Not an automated CI test.
- D-AOF-04: Railway production Redis must have AOF enabled before Phase 46 deploys.

### Claude's Discretion
- Exact copy variations between 3d/10d/21d follow-up steps (within the same friendly tone)
- `EstimateFollowupsModule` internal structure and provider wiring
- Whether `EstimateExpiryProcessor` is a separate processor class or handled as a special case within `EstimateFollowupProcessor`
- Error handling strategy for failed follow-up sends (retry policy, dead letter queue)
- Exact order of operations for the AOF startup check within worker bootstrap

### Deferred Ideas (OUT OF SCOPE)
- Trader-facing UI for viewing or managing scheduled follow-ups (invisible infrastructure)
- Configurable follow-up timing or delays (fixed 3/10/21-day schedule)
- Trader notification on auto-expiry (silent transition)
- Phase 45's public customer page response handling (Phase 45 scope)
- Phase 47's convert-to-quote / mark-as-lost (Phase 47 scope)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FUP-01 | Sending an estimate automatically schedules 3-day, 10-day, and 21-day follow-up emails via BullMQ delayed jobs on `ESTIMATE_FOLLOWUPS` queue | `EstimateFollowupScheduler.scheduleFollowups()` called from `EstimateEmailSender` step 9; `Queue.add()` with delay option |
| FUP-02 | Follow-up delays are relative in UTC (72h, 240h, 504h from send time) for DST safety | BullMQ `delay` option in milliseconds; `72 * 60 * 60 * 1000` etc — UTC-relative by design |
| FUP-03 | Follow-up jobs use deterministic jobId pattern for idempotent add and O(1) cancellation | BullMQ deduplicates by jobId on `Queue.add()`; `queue.remove(jobId)` for O(1) cancellation |
| FUP-04 | Each follow-up email includes the estimate summary and the same four response buttons as the initial send | `estimate-followup.html` Maizzle template with `followupStep` variable and 3-CTA structure |
| FUP-05 | Any exit transition cancels all pending follow-ups for that estimate chain | `BullMQEstimateFollowupCanceller.cancelAllFollowups()` removes 4 jobIds; wired via DI token |
| FUP-06 | Worker processor performs defence-in-depth state check before sending and silently no-ops if estimate is not in follow-up-worthy status | `EstimateFollowupProcessor.process()` re-reads estimate status from MongoDB before calling Resend |
| FUP-07 | Estimates auto-transition to Expired exactly 30 days after Sent timestamp | `EstimateExpiryProcessor` handles delayed job at 720h; calls `EstimateTransitionService` |
| FUP-08 | Production Redis has `appendonly yes` / `appendfsync everysec` persistence enabled | Worker startup check via `CONFIG GET appendonly`; Docker Compose and Railway config |
</phase_requirements>

## Standard Stack

### Core (already installed — no new npm dependencies required)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @nestjs/bullmq | 11.0.4 | `@Processor` decorator, `WorkerHost` class, `BullModule.registerQueue` | Installed v1.4; the `@Processor` + `WorkerHost` pattern is the project standard |
| bullmq | 5.x | `Queue` class for `add()` / `remove()` by jobId, `Job` type | Installed v1.4; provides delayed job support via `delay` option in milliseconds |
| ioredis | 5.x | `ConfigGet` for AOF startup check | Installed v1.4; used via `IORedis` connection obtained from `QueueModule` |

[VERIFIED: .planning/milestones/v1.4-ROADMAP.md] — BullMQ, ioredis, and @nestjs/bullmq were installed in Phase 20 (v1.4 milestone). All three are available with no new installation required.

[VERIFIED: .planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-RESEARCH.md] — `@nestjs/bullmq` version 11.0.4 and `bullmq` version 5.71.0 are the installed versions.

**Installation:** No new npm dependencies. Phase 46 reuses the v1.4 infrastructure stack.

**Version verification:** Exact installed versions confirmed via retrospective documentation — not re-verified against npm registry this session.

## Architecture Patterns

### Recommended Module Structure

```
src/estimate-followups/            (new module — mirrors worker processor pattern)
├── estimate-followups.module.ts   (rebinds ESTIMATE_FOLLOWUP_CANCELLER DI token)
├── services/
│   ├── estimate-followup-scheduler.service.ts   (enqueues 4 delayed jobs per send)
│   └── bullmq-estimate-followup-canceller.service.ts  (removes 4 jobs per cancellation)
└── test/
    └── services/
        ├── estimate-followup-scheduler.service.spec.ts
        └── bullmq-estimate-followup-canceller.service.spec.ts

src/worker/processors/
├── echo.processor.ts              (existing — Phase 22)
├── stripe-webhook.processor.ts   (existing — Phase 30)
├── estimate-followup.processor.ts (new — Phase 46)
└── estimate-expiry.processor.ts   (new — Phase 46; recommended as separate class)

src/email/templates/
└── estimate-followup.html         (new Maizzle template — single template, followupStep variable)
```

### Pattern 1: BullMQ Processor (WorkerHost) — STRIPE_WEBHOOKS Precedent

**What:** A NestJS class decorated with `@Processor(QUEUE_NAMES.X)` that extends `WorkerHost`. The `process()` method receives a `Job` and handles it.

**When to use:** Every queue consumer in this codebase follows this pattern. Phase 46 uses it for both `EstimateFollowupProcessor` and `EstimateExpiryProcessor`.

**Example (canonical from Phase 30):**
```typescript
// Source: .planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-02-PLAN.md
import { Processor, WorkerHost } from "@nestjs/bullmq";
import { Job } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

@Processor(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)
export class EstimateFollowupProcessor extends WorkerHost {
  private readonly logger = new AppLogger(EstimateFollowupProcessor.name);

  constructor(
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateEmailSendCreator: EstimateEmailSendCreator,
    private readonly emailSenderService: EmailSenderService,
    private readonly estimateEmailRenderer: EstimateFollowupEmailRenderer,
  ) {
    super();
  }

  async process(job: Job<IFollowupJobData>): Promise<void> {
    const { estimateId, revisionNumber, step } = job.data;
    this.logger.log(`Processing follow-up step=${step}`, { estimateId, revisionNumber });

    // Defence-in-depth: re-read estimate status before sending
    const estimate = await this.estimateRetriever.findByIdOrFail(estimateId);
    const worthyStatuses = ["sent", "viewed", "responded", "site_visit_requested"];
    if (!worthyStatuses.includes(estimate.status)) {
      this.logger.log(`Follow-up skipped — estimate in status=${estimate.status}`, { estimateId });
      return;
    }

    // Send follow-up email and append to estimate_email_sends
    // ...
  }
}
```

### Pattern 2: BullMQ Delayed Job Scheduling

**What:** `Queue.add(name, data, options)` with a `delay` in milliseconds. BullMQ persists the job in Redis and fires it after the delay elapses. A `jobId` in options enables deduplication — if a job with the same `jobId` already exists, the new `add()` call is a silent no-op.

**When to use:** `EstimateFollowupScheduler.scheduleFollowups()` calls this 4 times per send.

**Example:**
```typescript
// Source: [CITED: docs.bullmq.io/guide/jobs/delayed]
import { Queue } from "bullmq";

// Delays (milliseconds)
const FOLLOWUP_DELAYS = {
  step1: 72  * 60 * 60 * 1000,  // 3 days  (FUP-02)
  step2: 240 * 60 * 60 * 1000,  // 10 days (FUP-02)
  step3: 504 * 60 * 60 * 1000,  // 21 days (FUP-02)
  expiry: 720 * 60 * 60 * 1000, // 30 days (D-EXP-03)
};

// Deterministic jobId patterns (FUP-03, D-EXP-03)
const followupJobId = (estimateId: string, revisionNumber: number, step: 1 | 2 | 3) =>
  `estimate-followup:${estimateId}:${revisionNumber}:${step}`;

const expiryJobId = (estimateId: string, revisionNumber: number) =>
  `estimate-expiry:${estimateId}:${revisionNumber}`;

// Add with deduplication — second call for same jobId is silent no-op
await queue.add("follow-up", jobData, {
  jobId: followupJobId(estimateId, revisionNumber, 1),
  delay: FOLLOWUP_DELAYS.step1,
});
```

### Pattern 3: BullMQ Job Cancellation

**What:** `Queue.remove(jobId)` removes a delayed job by its deterministic jobId. If the job does not exist (already processed, already removed), `remove()` returns without error — the call is safe to make unconditionally.

**When to use:** `BullMQEstimateFollowupCanceller.cancelAllFollowups()` calls this 4 times.

**Example:**
```typescript
// Source: [CITED: docs.bullmq.io/guide/queues/removing-jobs]
public async cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void> {
  const jobIds = [
    `estimate-followup:${estimateId}:${revisionNumber}:1`,
    `estimate-followup:${estimateId}:${revisionNumber}:2`,
    `estimate-followup:${estimateId}:${revisionNumber}:3`,
    `estimate-expiry:${estimateId}:${revisionNumber}`,
  ];
  await Promise.all(jobIds.map((id) => this.queue.remove(id)));
  this.logger.log("Cancelled 4 follow-up/expiry jobs", { estimateId, revisionNumber });
}
```

### Pattern 4: DI Token Rebinding (Module Override Pattern)

**What:** `EstimateFollowupsModule` provides `BullMQEstimateFollowupCanceller` under the `ESTIMATE_FOLLOWUP_CANCELLER` token, overriding the `NoopEstimateFollowupCanceller` that `EstimateModule` registers as default. `EstimateFollowupsModule` must be imported after `EstimateModule` in the consumer module for the override to take effect.

**When to use:** Whenever `EstimateFollowupsModule` is imported alongside `EstimateModule` — in `WorkerModule` and `AppModule`.

**Example:**
```typescript
// Source: [ASSUMED] — inferred from Phase 42 D-HOOK-02 and NestJS DI override mechanics
// src/estimate-followups/estimate-followups.module.ts
@Module({
  imports: [EstimateModule, QueueModule],
  providers: [
    EstimateFollowupScheduler,
    {
      provide: ESTIMATE_FOLLOWUP_CANCELLER,
      useClass: BullMQEstimateFollowupCanceller,
    },
  ],
  exports: [EstimateFollowupScheduler, ESTIMATE_FOLLOWUP_CANCELLER],
})
export class EstimateFollowupsModule {}
```

### Pattern 5: Queue Injection in Non-Processor Services

**What:** To inject a `Queue` instance into a service (not a processor), use `@InjectQueue(QUEUE_NAMES.X)` from `@nestjs/bullmq`. The module must import `BullModule.registerQueue({ name: QUEUE_NAMES.ESTIMATE_FOLLOWUPS })` to make the queue injectable.

**When to use:** `EstimateFollowupScheduler` and `BullMQEstimateFollowupCanceller` both need the queue to call `add()` and `remove()`.

**Example:**
```typescript
// Source: [CITED: docs.nestjs.com/techniques/queues#producers]
import { InjectQueue } from "@nestjs/bullmq";
import { Queue } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";

@Injectable()
export class EstimateFollowupScheduler {
  constructor(
    @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)
    private readonly followupsQueue: Queue,
  ) {}
}
```

### Pattern 6: Redis AOF Startup Check

**What:** Before registering the ESTIMATE_FOLLOWUPS processor, the worker checks `CONFIG GET appendonly` on the Redis connection. This uses the raw IORedis client exposed via the BullMQ connection.

**When to use:** Once at worker startup in `src/worker.ts` or in `WorkerModule`'s `onModuleInit()`.

**Example:**
```typescript
// Source: [ASSUMED] — derived from D-AOF-01 and IORedis command API
// Obtaining the IORedis client from the QueueModule connection:
const redis = await this.configService.get<string>("REDIS_URL");
const client = new IORedis(redis);
const result = await client.config("GET", "appendonly");
// result = ["appendonly", "yes"] or ["appendonly", "no"]
const aofEnabled = result[1] === "yes";
if (!aofEnabled) {
  this.logger.error("FATAL: Redis appendonly is not enabled. ESTIMATE_FOLLOWUPS processor will not be registered.");
  // Skip processor registration — other processors continue
}
```

### Anti-Patterns to Avoid

- **Relying on cancellation alone:** Follow-up processors MUST re-read estimate status at execution time. Cancellation can race with processor execution if a job fires milliseconds before `cancelAllFollowups()` is called. The defence-in-depth re-read is mandatory (FUP-06).
- **Storing delay as absolute timestamp:** Use millisecond delays relative to enqueue time (`delay` option), not absolute Unix timestamps. BullMQ computes the fire time as `enqueuedAt + delay`, which is inherently UTC-relative (FUP-02).
- **Using the `ECHO` or `STRIPE_WEBHOOKS` queue:** A new dedicated `ESTIMATE_FOLLOWUPS` queue is required per D-SCHED-01 and the pattern from Phase 30 D-02. Mixing queue domains prevents independent scaling and makes queue monitoring ambiguous.
- **Module import order:** `EstimateFollowupsModule` must be imported after `EstimateModule` in any module that uses both. If imported before, the `NoopEstimateFollowupCanceller` binding wins. Verify `AppModule` import order.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Delayed job scheduling | Custom `setTimeout` loop, cron-based scheduler | BullMQ `Queue.add({ delay })` | BullMQ persists jobs in Redis, survives process restarts (with AOF), provides retry semantics |
| Job deduplication | Separate DB table tracking scheduled jobs | BullMQ deterministic `jobId` | `Queue.add()` with same jobId is silently ignored; no extra state to maintain |
| Job cancellation | MongoDB flag checked by processor | BullMQ `Queue.remove(jobId)` | O(1) removal by jobId is the BullMQ design; defence-in-depth state check handles races |
| Redis config check | Environment variable `AOF_ENABLED=true` | `CONFIG GET appendonly` via IORedis | Runtime check is authoritative — env var can be wrong; Redis server state is the truth |
| Email retry | Manual retry logic in processor | BullMQ job `attempts` option | BullMQ handles retry with configurable backoff; failed jobs go to dead-letter queue |

**Key insight:** The BullMQ `jobId` pattern is the entire persistence and cancellation story for Phase 46. The deterministic jobId makes scheduling idempotent and cancellation O(1) without any additional tracking infrastructure.

## Runtime State Inventory

Phase 46 is new infrastructure, not a rename/refactor phase.

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | None — no existing follow-up job records exist pre-Phase 46 | None |
| Live service config | Redis Docker Compose service — needs `appendonly yes` / `appendfsync everysec` | Docker Compose config edit |
| OS-registered state | None | None |
| Secrets/env vars | `REDIS_URL` already exists in `.env` and Railway — no new secrets | None |
| Build artifacts | None | None |

**Nothing found requiring migration** — Phase 46 creates new infrastructure from zero state.

## Common Pitfalls

### Pitfall 1: Follow-up fires after estimate has already exited (orphaned job)

**What goes wrong:** Cancellation call races with processor execution. `cancelAllFollowups()` is called on day 2 after customer declines, but a step-1 follow-up job fires at day 3 after the cancellation window. Or: the worker was down when cancellation was attempted via `queue.remove()` and the queue recovered after Redis restart, re-materialising the delayed job.

**Why it happens:** `Queue.remove()` removes jobs from the Redis sorted set. If the job has already been picked up by a worker (in "active" state), `remove()` does not interrupt it. Also, cancellation is best-effort — a transient Redis connection failure during `cancelAllFollowups()` leaves jobs in place.

**How to avoid:** Every follow-up processor MUST re-read the estimate from MongoDB at execution time (FUP-06). If `estimate.status` is not in `[sent, viewed, responded, site_visit_requested]`, log at info level and return without sending. This is the "defence-in-depth" guard. Never rely solely on upfront cancellation.

**Warning signs:** Worker logs showing "Processing follow-up" for estimates in terminal status.

[CITED: .planning/research/PITFALLS.md §Pitfall 2]

### Pitfall 2: BullMQ `add()` with `jobId` deduplication does NOT prevent re-scheduling on repeat send

**What goes wrong:** On a pure re-send (same revision, D-SCHED-04), if `scheduleFollowups()` is accidentally called, BullMQ deduplicates by jobId and the original schedule survives — which is correct. But if `cancelAllFollowups()` was called first (incorrectly), then `scheduleFollowups()` would enqueue fresh jobs from the new send time, resetting the schedule. The logic that distinguishes pure re-send from revised-send (D-TKN-08) must be correctly threaded from `EstimateEmailSender` to `EstimateFollowupScheduler`.

**Why it happens:** The send path detection lives in `EstimateEmailSender` (D-TKN-08). Phase 44 calls `cancelAllFollowups` in step 8 only on the revised-send path. Phase 46's `scheduleFollowups` in step 9 must also gate on the same path condition. If the condition is misread, follow-ups are either double-scheduled (deduped by jobId — harmless but confusing) or freshly scheduled on a re-send (resets the 3-day clock — wrong).

**How to avoid:** `EstimateEmailSender.send()` detects the send path via D-TKN-08 criteria. The planner must confirm that step 9 (`scheduleFollowups`) is called only on:
- `estimate.status === DRAFT AND estimate.revisionNumber === 1` (first-ever send — schedule fresh)
- `estimate.status === DRAFT AND estimate.revisionNumber > 1` (revised send — schedule fresh, after step 8 cancelled old revision's jobs)

Not on:
- `estimate.status IN (SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED)` (pure re-send — do not schedule, do not cancel)

**Warning signs:** Follow-up emails arriving on day 3 of a re-send when the original send was 8 days ago (customer gets a "still thinking about it?" email they weren't expecting).

### Pitfall 3: Redis AOF not enabled — delayed jobs silently lost on restart

**What goes wrong:** Without `appendonly yes`, Redis uses RDB snapshot persistence only. Delayed BullMQ jobs are stored in Redis sorted sets. If Redis restarts (deploy, crash, maintenance), any jobs not yet captured in the last RDB snapshot (typically every 5 minutes) are silently dropped. No error is thrown; the jobs simply never fire.

**Why it happens:** RDB persistence is asynchronous and periodic. There is always a window of potential data loss proportional to the snapshot interval. For delayed jobs that may fire 3-30 days in the future, this is catastrophic — a Redis restart on day 2 silently deletes all scheduled follow-ups.

**How to avoid:** AOF persistence (`appendonly yes`, `appendfsync everysec`) writes every Redis command to disk, guaranteeing at most 1 second of data loss on restart. The runtime startup check (D-AOF-01) is the enforcement mechanism — the ESTIMATE_FOLLOWUPS processor is not registered if AOF is off, making the failure immediately visible rather than a silent data-loss event days later.

**Warning signs:** Zero follow-up emails ever firing after a Redis restart. Worker startup logs showing "FATAL: Redis appendonly is not enabled."

[CITED: .planning/phases/46-follow-up-queue-automation/46-CONTEXT.md §D-AOF-01]

### Pitfall 4: Wrong IORedis type cast when injecting queue for `add()`/`remove()` calls

**What goes wrong:** The top-level `ioredis` package and BullMQ's bundled `ioredis` are different type namespaces. Passing a top-level `IORedis` instance directly to BullMQ connection options causes a TypeScript type error or a runtime mismatch.

**Why it happens:** This is a known BullMQ integration quirk in the project. The v1.4 milestone explicitly documented the workaround.

**How to avoid:** Use `as unknown as ConnectionOptions` cast when passing IORedis instance to BullMQ. This is the accepted pattern — not a code smell.

```typescript
// Source: [VERIFIED: .planning/milestones/v1.4-ROADMAP.md §Issues Resolved]
const connection = new IORedis(redisUrl) as unknown as ConnectionOptions;
```

**Warning signs:** TypeScript compilation errors involving `IORedis` and `ConnectionOptions` in queue/worker setup.

### Pitfall 5: `cancelAllFollowupsForChain` must cancel across ALL revisions

**What goes wrong:** Phase 47's `convert-to-quote` and `mark-as-lost` flows call `cancelAllFollowupsForChain(rootEstimateId)`. If only the current revision's follow-ups are cancelled, older revision jobs (that survived because the trader re-sent the same revision before revising) may still fire.

**Why it happens:** In theory, follow-ups are always scheduled per-revision and old-revision follow-ups are cancelled on revised-send. But defensive cancellation across all revisions in the chain ensures correctness even if a bug left orphaned jobs from a prior revision.

**How to avoid:** `cancelAllFollowupsForChain()` should walk all revisions in the chain (e.g., fetch all estimates with matching `rootEstimateId`) and call `cancelAllFollowups(estimateId, revisionNumber)` for each. Alternatively, use a known maximum revision number and iterate.

**Warning signs:** Follow-up emails firing for converted estimates. Customer receives "still thinking about your estimate?" email after they booked a site visit via Phase 47 convert.

### Pitfall 6: `EstimateFollowupProcessor` and `EstimateExpiryProcessor` must be registered in WorkerModule, not AppModule

**What goes wrong:** Processors registered in `AppModule` run in the API process, not the worker process. They compete with HTTP request handling for CPU and will not survive worker-only deployments.

**Why it happens:** NestJS DI allows processors to be registered anywhere, but the `@Processor` decorator connects to the BullMQ worker connection which must exist in the process. Worker and API are separate processes in this project.

**How to avoid:** Both processors are added to `WorkerModule.providers` only. `EstimateFollowupsModule` (which provides the scheduler and canceller) is imported by `AppModule` (so `EstimateEmailSender` can inject the scheduler) AND by `WorkerModule` (so the processors can access the canceller for any cleanup).

**Warning signs:** TypeScript compiles but follow-up emails never send. Worker logs show no "Processing follow-up" lines.

## Code Examples

Verified patterns from Phase 30 and project conventions:

### QUEUE_NAMES constant extension

```typescript
// Source: [VERIFIED: .planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-02-PLAN.md]
// src/queue/queue.constant.ts — existing pattern, add ESTIMATE_FOLLOWUPS
export const QUEUE_NAMES = {
  ECHO: "echo",
  STRIPE_WEBHOOKS: "stripe-webhooks",
  ESTIMATE_FOLLOWUPS: "estimate-followups",   // Phase 46 addition
} as const;
```

### EstimateFollowupScheduler service skeleton

```typescript
// Source: [ASSUMED] — derived from BullMQ docs and Phase 46 D-SCHED-06
@Injectable()
export class EstimateFollowupScheduler {
  private readonly logger = new AppLogger(EstimateFollowupScheduler.name);

  private static readonly FOLLOWUP_DELAYS_MS = {
    step1: 72  * 60 * 60 * 1000,  // 3d
    step2: 240 * 60 * 60 * 1000,  // 10d
    step3: 504 * 60 * 60 * 1000,  // 21d
    expiry: 720 * 60 * 60 * 1000, // 30d
  };

  constructor(
    @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)
    private readonly queue: Queue,
  ) {}

  public async scheduleFollowups(estimateId: string, revisionNumber: number): Promise<void> {
    const jobs = [
      { step: 1 as const, delay: EstimateFollowupScheduler.FOLLOWUP_DELAYS_MS.step1 },
      { step: 2 as const, delay: EstimateFollowupScheduler.FOLLOWUP_DELAYS_MS.step2 },
      { step: 3 as const, delay: EstimateFollowupScheduler.FOLLOWUP_DELAYS_MS.step3 },
    ];

    await Promise.all(
      jobs.map(({ step, delay }) =>
        this.queue.add(
          "follow-up",
          { estimateId, revisionNumber, step },
          {
            jobId: `estimate-followup:${estimateId}:${revisionNumber}:${step}`,
            delay,
          },
        ),
      ),
    );

    await this.queue.add(
      "expiry",
      { estimateId, revisionNumber },
      {
        jobId: `estimate-expiry:${estimateId}:${revisionNumber}`,
        delay: EstimateFollowupScheduler.FOLLOWUP_DELAYS_MS.expiry,
      },
    );

    this.logger.log("Scheduled 3 follow-ups + 1 expiry job", { estimateId, revisionNumber });
  }
}
```

### IEstimateFollowupCanceller interface (from Phase 42)

```typescript
// Source: [VERIFIED: .planning/phases/42-revisions/42-CONTEXT.md §D-HOOK-01]
// src/estimate/services/estimate-followup-canceller.interface.ts
export interface IEstimateFollowupCanceller {
  cancelAllFollowups(estimateId: string, revisionNumber: number): Promise<void>;
}

export const ESTIMATE_FOLLOWUP_CANCELLER = "ESTIMATE_FOLLOWUP_CANCELLER";
```

### EstimateEmailSender step 8 and step 9 ordering (from Phase 44)

```typescript
// Source: [VERIFIED: .planning/phases/44-email-send-flow/44-CONTEXT.md §D-AUDIT-08]
// D-TKN-08 path detection determines whether steps 8 and 9 run
// Step 8: cancelAllFollowups — only on revised-send path (revisionNumber > 1, status DRAFT)
// Step 9: scheduleFollowups  — only on first-ever send OR revised-send (status DRAFT)
// Pure re-send (status IN [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED]): neither step runs

// Rough ordering in EstimateEmailSender.send():
// 1. Resolve token
// 2. Render HTML
// 3. Send via Resend
// 4. Create estimate_email_sends row
// 5. Transition status via EstimateTransitionService
// 6. Update lastSentAt / lastSentTo
// 7. Extend token expiry
// 8. [revised-send only] cancelAllFollowups(previousRevision._id, previousRevision.revisionNumber)
// 9. [first-send or revised-send only] scheduleFollowups(estimate._id, estimate.revisionNumber)
```

### IFollowupJobData and IExpiryJobData (job payload shapes)

```typescript
// Source: [ASSUMED] — derived from Phase 46 deterministic jobId patterns
// src/estimate-followups/data-transfer-objects/followup-job.dto.ts
export interface IFollowupJobData {
  estimateId: string;
  revisionNumber: number;
  step: 1 | 2 | 3;   // 1=3d, 2=10d, 3=21d
}

export interface IExpiryJobData {
  estimateId: string;
  revisionNumber: number;
}
```

### Maizzle template variable shape

```typescript
// Source: [ASSUMED] — derived from D-EMAIL-03, D-EMAIL-04, D-EMAIL-05
// src/email/services/estimate-followup-email-renderer.service.ts
export interface EstimateFollowupEmailData {
  businessName: string;
  followupStep: "3d" | "10d" | "21d";    // minor copy variation per D-EMAIL-03
  estimateNumber: string;                  // "E-2026-001"
  viewUrl: string;                         // existing token URL (D-EMAIL-05)
  traderFirstName: string;                 // for "Message [First Name]" CTA (D-EMAIL-04)
  priceDisplay: string;                    // "£120 - £132" or "From £120"
}
```

### Follow-up-worthy status check in processor

```typescript
// Source: [ASSUMED] — derived from FUP-06 and EstimateStatus enum from Phase 41
private static readonly FOLLOWUP_WORTHY_STATUSES: EstimateStatus[] = [
  EstimateStatus.SENT,
  EstimateStatus.VIEWED,
  EstimateStatus.RESPONDED,
  EstimateStatus.SITE_VISIT_REQUESTED,
];
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Cron-based sweep for follow-ups | BullMQ delayed jobs per estimate | v1.4 worker infrastructure | Per-estimate precision vs batch latency; better reliability with AOF persistence |
| SendGrid for email delivery | Resend SDK | v1.3 Phase 18 migration | Simpler API, `{ data, error }` pattern vs try/catch SendGrid |
| Quote-only token | document-token with `documentType` discriminator | Phase 41 | Single guard handles both document types |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `EstimateFollowupsModule` DI rebinding pattern uses NestJS module override mechanics (later import wins) | Architecture Patterns §Pattern 4 | Module wiring may require different approach; planner must verify with NestJS DI docs |
| A2 | `Queue.remove(jobId)` is safe to call on non-existent jobIds (no-op, no error thrown) | Don't Hand-Roll + Pattern 3 | If it throws, `cancelAllFollowups` needs try/catch per jobId |
| A3 | `EstimateExpiryProcessor` is recommended as a separate class from `EstimateFollowupProcessor` | Architecture Patterns §module structure | Could be handled as a special case within one processor; planner chooses per Claude's Discretion |
| A4 | Redis `CONFIG GET appendonly` returns `["appendonly", "yes"]` array format via IORedis | Code Examples §AOF startup check | IORedis API response format for CONFIG GET — verify against IORedis docs or existing codebase usage |
| A5 | `@InjectQueue` injection for scheduler services (not processors) requires `BullModule.registerQueue` in the owning module | Architecture Patterns §Pattern 5 | If not registered in `EstimateFollowupsModule`, injection fails at runtime |
| A6 | BullMQ `jobId` deduplication on `add()` is handled by the queue layer — a duplicate `add()` returns without error | Architecture Patterns §Pattern 2 | If it throws on duplicate, scheduler needs existence check first |

## Open Questions

1. **Does `Queue.remove(jobId)` throw if the job does not exist?**
   - What we know: BullMQ docs describe `remove()` as returning `Promise<number>` (number of removed jobs). A jobId not found likely returns `0`.
   - What's unclear: Whether it throws or returns 0 for missing jobIds.
   - Recommendation: Planner wraps each `remove()` call in a try/catch per jobId, or validates against BullMQ docs before assuming no-op behaviour. Tagging as A2.

2. **Where does the AOF startup check live — `src/worker.ts` or `WorkerModule.onModuleInit()`?**
   - What we know: D-AOF-01 says "worker startup validates." Claude's Discretion covers "exact order of operations."
   - What's unclear: Whether the check should block processor registration (easier in `worker.ts` bootstrap) or be a module lifecycle hook.
   - Recommendation: Place it in `src/worker.ts` immediately after `NestFactory.createApplicationContext(WorkerModule)` but before the app starts listening. This matches the "refuse to register processor" language — the module is created but the processor registration can be skipped conditionally.

3. **What is the retry policy for failed follow-up email sends?**
   - What we know: Claude's Discretion covers error handling strategy for failed follow-up sends.
   - What's unclear: Whether to use BullMQ's built-in `attempts` + `backoff` option, or log and swallow failures.
   - Recommendation: Use BullMQ `attempts: 3` with `backoff: { type: "exponential", delay: 60000 }` (1-minute base, exponential). After 3 failures, the job moves to the BullMQ failed queue for inspection. Matches the STRIPE_WEBHOOKS pattern from Phase 30.

4. **Does `EstimateFollowupsModule` need to be imported in `AppModule` or only in `WorkerModule`?**
   - What we know: `EstimateEmailSender` (in the API process) injects `EstimateFollowupScheduler` via `@Inject(ESTIMATE_FOLLOWUP_CANCELLER)` and needs `scheduleFollowups()`. The processors run in the worker process.
   - What's unclear: The exact import graph.
   - Recommendation: `EstimateFollowupsModule` is imported in both `AppModule` (for scheduler injection into `EstimateEmailSender`) and `WorkerModule` (for processor + canceller access). The module exports `EstimateFollowupScheduler` and the `ESTIMATE_FOLLOWUP_CANCELLER` token.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Redis (Docker) | BullMQ delayed jobs, AOF check | Yes | 7.4 (from v1.4) | None — required |
| BullMQ | Queue.add() / Queue.remove() / Processor | Yes | 5.x (v1.4 installed) | None — already in use |
| @nestjs/bullmq | @Processor, WorkerHost, InjectQueue | Yes | 11.0.4 (v1.4 installed) | None — already in use |
| Resend SDK | Follow-up email delivery | Yes | Already installed | None — EmailSenderService wraps it |
| Maizzle | Follow-up email template rendering | Yes | Already in use for estimate-sent.html | None |

**Missing dependencies with no fallback:** None — all required infrastructure is in place from v1.4.

**Redis AOF state:** Docker Compose Redis does NOT have AOF enabled by default (v1.4 configured `maxmemory-policy noeviction` but not AOF). Phase 46 must add the `command:` configuration to the Docker Compose Redis service.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | `trade-flow-api/jest.config.js` |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern=estimate-followup` |
| Full suite command | `cd trade-flow-api && npm run test` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FUP-01 | `scheduleFollowups()` enqueues 4 delayed jobs | unit | `npm test -- --testPathPattern=estimate-followup-scheduler` | No — Wave 0 |
| FUP-02 | Delays are 72h / 240h / 504h in milliseconds | unit | same | No — Wave 0 |
| FUP-03 | jobIds are deterministic; second call is no-op | unit | same | No — Wave 0 |
| FUP-04 | Follow-up email uses correct template + CTAs | unit (renderer) | `npm test -- --testPathPattern=estimate-followup-email-renderer` | No — Wave 0 |
| FUP-05 | `cancelAllFollowups()` removes 4 jobIds | unit | `npm test -- --testPathPattern=bullmq-estimate-followup-canceller` | No — Wave 0 |
| FUP-06 | Processor no-ops on non-follow-up-worthy status | unit | `npm test -- --testPathPattern=estimate-followup.processor` | No — Wave 0 |
| FUP-07 | Expiry job transitions estimate to Expired | unit | `npm test -- --testPathPattern=estimate-expiry.processor` | No — Wave 0 |
| FUP-08 | AOF check refuses processor registration if disabled | unit (mock IORedis) | `npm test -- --testPathPattern=worker` | No — Wave 0 |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npm run test -- --testPathPattern=estimate-followup`
- **Per wave merge:** `cd trade-flow-api && npm run ci`
- **Phase gate:** Full suite green (`npm run ci` passes in both repos) before `/gsd-verify-work`

### Wave 0 Gaps
All test files for Phase 46 are new:
- [ ] `src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts`
- [ ] `src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts`
- [ ] `src/worker/test/processors/estimate-followup.processor.spec.ts`
- [ ] `src/worker/test/processors/estimate-expiry.processor.spec.ts`

Mocking approach (per project convention): Pure unit tests with mocked Queue (`{ add: jest.fn(), remove: jest.fn() }`), mocked `EstimateRetriever`, and mocked `EmailSenderService`. No `mongodb-memory-server` — per Phase 42 D-HOOK-04 explicit instruction.

## Security Domain

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No — background worker, no HTTP surface | N/A |
| V3 Session Management | No — no sessions in worker | N/A |
| V4 Access Control | Yes — follow-up processor must confirm businessId policy before sending | Re-use `EstimatePolicy.canRead()` pattern; confirm businessId on fetched estimate |
| V5 Input Validation | Partial — job payload is internal; estimate data comes from MongoDB (already validated) | No external input; jobId format is canonical |
| V6 Cryptography | No — existing token reuse (D-EMAIL-05); no new tokens generated | N/A |

**Key security note:** The follow-up processor sends email to `estimate.lastSentTo` (or the stored recipient email). It MUST re-fetch the estimate from MongoDB at execution time (FUP-06) rather than storing the recipient email in the job payload. This ensures the email goes to the current recipient even if it was updated after scheduling. This also means the processor never sends to a stale email address.

## Sources

### Primary (HIGH confidence)
- `.planning/phases/46-follow-up-queue-automation/46-CONTEXT.md` — Locked decisions, deterministic jobId patterns, DI token, email content, AOF requirements
- `.planning/phases/42-revisions/42-CONTEXT.md` §D-HOOK — `IEstimateFollowupCanceller` interface, DI token `ESTIMATE_FOLLOWUP_CANCELLER`, `NoopEstimateFollowupCanceller` pattern
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` §D-AUDIT-08 — Step ordering in `EstimateEmailSender.send()`, step 8 (cancel) and step 9 (schedule) placement
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` §D-TKN-06/07/08 — Send path detection (pure re-send vs revised-send vs first-send)
- `.planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-02-PLAN.md` — Canonical `@Processor(QUEUE_NAMES.X) extends WorkerHost` pattern with full code example
- `.planning/milestones/v1.6-phases/30-stripe-checkout-and-webhooks/30-CONTEXT.md` §D-02/D-03 — Queue constant pattern, processor registration in WorkerModule
- `.planning/milestones/v1.4-ROADMAP.md` — BullMQ/ioredis infrastructure, IORedis type cast workaround, `deleteOutDir:false`, dual entry-point pattern
- `.planning/research/PITFALLS.md` §Pitfall 2 — Deterministic jobId pattern, orphaned follow-up prevention, `queue.remove(jobId)` cancellation

### Secondary (MEDIUM confidence)
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` §D-TXN — `EstimateTransitionService`, `EstimateStatus` enum values, `→ Expired` transition readiness
- `.planning/RETROSPECTIVE.md` §v1.4, §v1.6 — Pattern maturity confirmation, IORedis workaround, BullMQ deduplication precedent

### Tertiary (LOW confidence)
- [CITED: docs.bullmq.io/guide/jobs/delayed] — BullMQ `delay` option in milliseconds (known from training data; not re-verified this session)
- [CITED: docs.bullmq.io/guide/queues/removing-jobs] — `Queue.remove(jobId)` API (known from training data; not re-verified this session)
- [CITED: docs.bullmq.io/guide/jobs/job-ids] — Deterministic jobId deduplication (referenced in PITFALLS.md as HIGH confidence)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all dependencies confirmed installed from v1.4/v1.6 retrospective docs
- Architecture: HIGH — processor pattern confirmed from Phase 30 plan code; DI token pattern confirmed from Phase 42; step ordering confirmed from Phase 44
- Pitfalls: HIGH — Pitfalls 1/3 confirmed from official research; Pitfalls 2/4/5/6 confirmed from project retrospectives and CONTEXT docs
- Test map: HIGH — follows established project Jest pattern; all tests are new (Wave 0)

**Research date:** 2026-04-13
**Valid until:** 2026-05-13 (stable BullMQ API; 30 days)
