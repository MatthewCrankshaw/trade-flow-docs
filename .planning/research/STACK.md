# Technology Stack — v1.8 Estimates

**Project:** Trade Flow
**Milestone:** v1.8 Estimates (subsequent milestone — brownfield)
**Researched:** 2026-04-10
**Overall confidence:** HIGH

## Summary

**Zero net-new runtime dependencies are required for v1.8 Estimates.** Every capability the milestone needs — delayed BullMQ jobs, a slider UI primitive, self-referential Mongoose documents, and range-formatted money display — is already satisfied by libraries installed in v1.4 (worker infrastructure) or the existing Radix UI meta package. The work is pattern-level, not dependency-level.

This document's job is therefore to (1) confirm existing versions cover the new use cases, (2) surface the specific BullMQ APIs that must be used for the new delayed-follow-up feature, and (3) explicitly recommend **against** three tempting additions (a cron library, a fresh slider package, a money formatting helper library).

## Recommended Stack — Changes for v1.8

**Summary: no additions.** The tables below enumerate existing libraries that the new estimates feature will lean on, with confirmation that their currently-installed versions are sufficient.

### Backend (trade-flow-api) — Reused Libraries

| Library | Installed Version | v1.8 Use | Why Sufficient |
|---------|-------------------|----------|----------------|
| `bullmq` | 5.71.0 (from v1.4) | Delayed follow-up jobs (3/10/21 days) | `Queue.add()` accepts a `delay` option (milliseconds) in `JobsOptions` — this is the native primitive for scheduled-in-the-future jobs. No cron/scheduler library needed. |
| `@nestjs/bullmq` | 11.0.4 (from v1.4) | Register new `estimate-follow-up` queue; inject into `QueueProducer` | Same `BullModule.registerQueue()` + `@InjectQueue()` pattern already proven for the echo queue (v1.4) and Stripe webhook queue (v1.6). |
| `ioredis` | 5.10.1 (from v1.4) | Underlying Redis driver for the delayed set | Already configured with `maxRetriesPerRequest: null` (BullMQ requirement). No changes needed. |
| `mongoose` | 9.1.5 | Self-referential `parentEstimateId` + `revisionNumber` on Estimate entity | Mongoose supports self-referential `ObjectId` refs natively. Compound index `{ rootEstimateId: 1, revisionNumber: -1 }` will make "latest revision" queries O(log n). |
| `mongodb` (driver) | 7.0.0 | `estimate_counters` atomic counter collection | Same atomic `findOneAndUpdate({ $inc })` pattern already used by `quote_counters` in v1.2. |
| `dinero.js` | 1.9.1 | Computing `low = base` and `high = base * (1 + contingency)` for range display (API side) | v1 API has `multiply()` and `percentage()` — sufficient for the contingency math. |
| `luxon` | 3.5.1 | Computing the three delay offsets (`DateTime.now().plus({ days: 3 })`) and storing scheduled-at timestamps | Luxon `plus({ days })` is the idiomatic way. Already standardized across all DTOs (v1.6). |
| `class-validator` | 0.14.1 | Validating contingency range (0–30, integer) in Estimate DTOs | `@IsInt()`, `@Min(0)`, `@Max(30)` — existing decorators. Step-of-5 enforced at UI; API accepts any 0–30 integer defensively. |
| Resend SDK (via `EmailModule`) | v1.3 | Sending follow-up emails via existing EmailModule | `EmailService.sendEmail()` already wraps Resend; the follow-up processor will call it the same way the quote-sent and quote-accepted emails do. |

### Frontend (trade-flow-ui) — Reused Libraries

| Library | Installed Version | v1.8 Use | Why Sufficient |
|---------|-------------------|----------|----------------|
| `radix-ui` (meta) | 1.4.3 | Contingency slider (0–30% in 5% steps) | The meta `radix-ui` package re-exports **all** Radix primitives including `Slider`. **No separate `@radix-ui/react-slider` install is needed.** Import path: `import { Slider } from "radix-ui"`. The primitive accepts `min`, `max`, `step`, and `value`/`onValueChange` — exactly what the contingency UX needs. Tick-mark labels are rendered by the consumer (absolutely positioned divs over the track), not by Radix. |
| `dinero.js` | 2.0.0-alpha.14 | "From £X" and "£X – £Y" range display | Existing `formatCurrency` helper in `src/lib/` already uses v2 Dinero; the new `formatRange(low, high)` helper composes two calls. No new formatting library. |
| `react-hook-form` + `valibot` | 7.71.1 / 1.2.0 | Estimate creation form (document type toggle, base price, contingency, display mode) | Existing form stack handles conditional fields fine. New `estimateSchema.ts` alongside `quoteSchema.ts` in `src/lib/forms/schemas/`. |
| `@reduxjs/toolkit` (RTK Query) | 2.11.2 | New `estimatesApi` slice + `publicEstimateApi` (mirror of `publicQuoteApi`) | Same slice pattern already used for quotes. |
| `sonner` | 2.0.7 | Follow-up scheduling confirmation toasts | Existing toast system. |
| `luxon` | (standardized v1.6) | UI-side "scheduled for" display, countdown helpers | Already standardized across UI. |

## Explicit Non-Additions (What NOT To Install)

This section is as important as the inclusions — it prevents scope creep and accidental duplication.

| Tempting Addition | Why NOT | Use Instead |
|-------------------|---------|-------------|
| `node-cron`, `agenda`, `@nestjs/schedule` | BullMQ **already handles** time-deferred execution via the `delay` option. A cron library would add a second scheduling system competing with BullMQ for the same "fire at time T" responsibility. Pitfall: duplicated, harder-to-reason-about scheduling. | `await queue.add(jobName, payload, { delay: millisUntilFire, jobId: stableKey })` |
| `@radix-ui/react-slider` (standalone) | The project already imports Radix primitives from the unified `radix-ui@1.4.3` meta package (shadcn/ui's February 2026 recommendation). Adding the standalone package would create a second source of the same component. | `import { Slider } from "radix-ui"` |
| `numeral.js`, `accounting.js`, `currency.js` | `dinero.js` + its currency data are already in the UI; the existing `formatCurrency` utility covers all UK/GBP needs. A range formatter is two lines, not a new library. | Extend `src/lib/format-currency.ts` with `formatRange(low: Dinero, high: Dinero)` |
| `mongoose-sequence`, `@typegoose/auto-increment` | v1.2 already solved atomic per-business counters with the hand-rolled `quote_counters` collection and `findOneAndUpdate({ $inc })`. Copy that exact pattern for `estimate_counters`. | Mirror `QuoteCounterRepository` as `EstimateCounterRepository` |
| `rrule`, `later.js` | Follow-ups are fixed offsets (3, 10, 21 days), not recurring rules. No recurrence library needed. | Hard-code the offsets as a constant array. |
| `ts-pattern`, `xstate` for status lifecycle | The existing quote-status transition pattern (enum + guard function) is enough. Introducing a state-machine library just for estimate statuses is over-engineering for eight states. | Mirror `quote-status-transition.utility.ts` as `estimate-status-transition.utility.ts` |
| `mongoose-version`, `mongoose-history` | Plugin buries versioning logic. Hand-rolled `rootEstimateId` + `parentEstimateId` + `revisionNumber` (three fields) is ~20 lines and perfectly legible, and aligns with existing repo conventions. | See Pattern 2 below. |

## Key Patterns for v1.8

### Pattern 1: BullMQ Delayed Follow-Up with Idempotency Key

**What:** Schedule the 3-day, 10-day, and 21-day follow-up emails at the moment an estimate is sent. Use stable `jobId`s so re-sending (or a revision reset) is idempotent and cancellable.

**Why:** BullMQ's `delay` option is the primitive for "run this job at a future time." Using `jobId` makes duplicates impossible (second `add()` with the same `jobId` is silently dropped by BullMQ), which is exactly the idempotency semantics we want. `removeOnComplete`/`removeOnFail` keeps the Redis delayed set from growing unboundedly.

**Example (QueueProducer extension):**

```typescript
// src/queue/services/queue-producer.service.ts -- new methods added for v1.8
import { Injectable } from "@nestjs/common";
import { InjectQueue } from "@nestjs/bullmq";
import { Queue } from "bullmq";
import { QUEUE_NAMES } from "@queue/queue.constant";
import { AppLogger } from "@core/services/app-logger.service";

const FOLLOW_UP_OFFSETS_DAYS = [3, 10, 21] as const;
const DAY_IN_MS = 24 * 60 * 60 * 1000;

@Injectable()
export class QueueProducer {
  private readonly logger = new AppLogger(QueueProducer.name);

  constructor(
    @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOW_UP)
    private readonly estimateFollowUpQueue: Queue,
    // ...other queue injections
  ) {}

  public async scheduleEstimateFollowUps(params: {
    estimateId: string;
    revisionNumber: number;
  }): Promise<void> {
    const { estimateId, revisionNumber } = params;

    await Promise.all(
      FOLLOW_UP_OFFSETS_DAYS.map((days) =>
        this.estimateFollowUpQueue.add(
          "follow-up",
          { estimateId, revisionNumber, offsetDays: days },
          {
            // Stable, deterministic jobId: second call for the same
            // (estimate, revision, offset) triple is a silent no-op.
            jobId: `estimate-follow-up:${estimateId}:r${revisionNumber}:d${days}`,
            delay: days * DAY_IN_MS,
            removeOnComplete: { age: 60 * 60 * 24 * 7 }, // keep 7 days for audit
            removeOnFail: { age: 60 * 60 * 24 * 30 },    // keep 30 days for triage
            attempts: 3,
            backoff: { type: "exponential", delay: 60_000 },
          },
        ),
      ),
    );

    this.logger.log("Estimate follow-ups scheduled", { estimateId, revisionNumber });
  }

  public async cancelEstimateFollowUps(params: {
    estimateId: string;
    revisionNumber: number;
  }): Promise<void> {
    const { estimateId, revisionNumber } = params;
    await Promise.all(
      FOLLOW_UP_OFFSETS_DAYS.map(async (days) => {
        const jobId = `estimate-follow-up:${estimateId}:r${revisionNumber}:d${days}`;
        const job = await this.estimateFollowUpQueue.getJob(jobId);
        if (job) await job.remove();
      }),
    );
  }
}
```

**Reset-on-revision flow:**
1. Revising an estimate bumps `revisionNumber` → new `jobId` namespace → new scheduled jobs.
2. Before scheduling the new revision's jobs, call `cancelEstimateFollowUps()` for the previous `revisionNumber` to keep Redis clean.
3. "Responded" / "Converted" / "Declined" / "Expired" transitions all call `cancelEstimateFollowUps()` for the active revision.

**Worker-side defensive check (belt and braces):** The `EstimateFollowUpProcessor` should re-query the estimate and confirm (a) the estimate is still in a follow-up-worthy status and (b) `estimate.revisionNumber === job.data.revisionNumber` before sending. This handles races where cancellation and job execution overlap, and the case where `cancel` silently missed the job because BullMQ had already moved it from `delayed` to `active`.

**Sources:** [BullMQ — Delayed jobs](https://docs.bullmq.io/guide/jobs/delayed), [BullMQ — Auto-removal of jobs](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs), existing v1.4 QueueProducer pattern.

### Pattern 2: Self-Referential Mongoose Versioning (Invisible to Users)

**What:** An `Estimate` document has optional `parentEstimateId` (self-ref to immediate predecessor) and mandatory `revisionNumber` (integer, starts at 1). A "revision" is a new document whose `parentEstimateId` points to the previous revision and whose `rootEstimateId` points to the original root (denormalized for fast lookup).

**Why invisible:** The UI shows a single logical estimate and calls "resend with changes" an edit. Internally we keep the old revision immutable for audit. The History collapsed section in the UI reads `findMany({ rootEstimateId }).sort({ revisionNumber: 1 })`.

**Entity sketch:**

```typescript
// src/estimate/entities/estimate.entity.ts
@Schema({ collection: "estimates", timestamps: true })
export class EstimateEntity {
  @Prop({ type: Types.ObjectId, ref: "EstimateEntity", default: null, index: true })
  parentEstimateId: Types.ObjectId | null;

  @Prop({ type: Types.ObjectId, ref: "EstimateEntity", required: true, index: true })
  rootEstimateId: Types.ObjectId; // set to self._id on root creation, copied to all revisions

  @Prop({ type: Number, required: true, default: 1, min: 1 })
  revisionNumber: number;

  // ... other fields (businessId, jobId, number, status, lineItems, basePriceAmount,
  //                   contingencyPercent, displayMode, ...)
}

// Compound indexes:
EstimateSchema.index({ rootEstimateId: 1, revisionNumber: -1 }); // "latest revision" query
EstimateSchema.index({ parentEstimateId: 1 }); // optional chain walk for audit
EstimateSchema.index({ businessId: 1, status: 1, updatedAt: -1 }); // list views
```

**Design decision — store BOTH `parentEstimateId` and `rootEstimateId`:**

| Field | Purpose |
|-------|---------|
| `parentEstimateId` | Immediate predecessor for audit trail (History section shows chain) |
| `rootEstimateId` | Fast "give me the latest revision of this logical estimate" query without chain walking |

The cost is one extra indexed ObjectId per document. The benefit is that every list / detail / public-token fetch runs a single indexed query instead of a recursive lookup or `$graphLookup`.

**Root creation:** When a root estimate is created, run a second write to set `rootEstimateId = self._id`. Cleaner alternative: use a Mongoose `pre('save')` hook to set `rootEstimateId` to `_id` if it's unset.

**"Latest revision" repository method:**

```typescript
// EstimateRepository
public async findLatestRevision(rootEstimateId: string): Promise<IEstimateDto | null> {
  const doc = await this.model
    .findOne({ rootEstimateId: new Types.ObjectId(rootEstimateId) })
    .sort({ revisionNumber: -1 })
    .exec();
  return doc ? this.toDto(doc) : null;
}

public async findRevisionHistory(rootEstimateId: string): Promise<IEstimateDto[]> {
  const docs = await this.model
    .find({ rootEstimateId: new Types.ObjectId(rootEstimateId) })
    .sort({ revisionNumber: 1 })
    .exec();
  return docs.map((d) => this.toDto(d));
}
```

**Public-token resolution note:** The public-facing URL should encode the `rootEstimateId` (or a token bound to it), not a specific revision ID. Customer always sees the latest revision. This mirrors v1.3's quote token pattern.

**Sources:** Mongoose self-reference uses a vanilla `ref` — see [Mongoose — Populate](https://mongoosejs.com/docs/populate.html). Existing v1.2 quote pattern for the repository and counter structure.

### Pattern 3: Radix Slider from the Unified `radix-ui` Package

**What:** The contingency slider is a single-thumb Radix Slider with `min=0`, `max=30`, `step=5`, plus consumer-rendered tick labels.

**Why from the meta package:** The project's v1.7 STACK.md lists `radix-ui@1.4.3` (the unified meta package) alongside individual primitives. shadcn/ui's February 2026 changelog marks the unified package as the forward-compatible import path. Slider is available from this package today — **no new dependency install is needed.**

**Example:**

```tsx
// src/features/estimates/components/ContingencySlider.tsx
import { Slider } from "radix-ui"; // unified package, no new install
import { cn } from "@/lib/utils";

interface ContingencySliderProps {
  value: number;             // 0..30 (percent)
  onChange: (value: number) => void;
}

const TICKS = [0, 5, 10, 15, 20, 25, 30] as const;

export function ContingencySlider({ value, onChange }: ContingencySliderProps) {
  return (
    <div className="w-full">
      <Slider.Root
        className="relative flex h-5 w-full touch-none select-none items-center"
        min={0}
        max={30}
        step={5}
        value={[value]}
        onValueChange={([next]) => onChange(next)}
      >
        <Slider.Track className="relative h-1.5 w-full grow rounded-full bg-muted">
          <Slider.Range className="absolute h-full rounded-full bg-primary" />
        </Slider.Track>
        <Slider.Thumb
          aria-label="Contingency percent"
          className="block h-5 w-5 rounded-full border-2 border-primary bg-background shadow focus:outline-none focus:ring-2 focus:ring-ring"
        />
      </Slider.Root>
      <div className="mt-2 flex justify-between text-xs text-muted-foreground">
        {TICKS.map((tick) => (
          <span key={tick} className={cn(tick === value && "font-semibold text-foreground")}>
            {tick}%
          </span>
        ))}
      </div>
    </div>
  );
}
```

**Accessibility:** Radix's Slider primitive handles arrow-key navigation, `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, and RTL direction automatically. The `aria-label` on `Slider.Thumb` above is the only thing the consumer needs to add.

**Step-of-5 enforcement:** The `step={5}` prop makes the thumb snap to 0/5/10/…/30 regardless of where the user drags. Keyboard arrow keys respect the step too. No manual `Math.round()` needed in the `onValueChange` handler.

**Sources:** [Radix Primitives — Slider](https://www.radix-ui.com/primitives/docs/components/slider), [shadcn/ui — February 2026 Unified Radix UI Package changelog](https://ui.shadcn.com/docs/changelog/2026-02-radix-ui).

### Pattern 4: Range vs "From £X" Display with Dinero

**What:** Two display modes for the same pair of money values. Both endpoints are always computable from `(basePriceAmount, contingencyPercent)`; the display mode is a per-estimate flag (`"range" | "from"`).

**Computation (UI side, consistent with v1.2 quote totals pattern):**

```typescript
// src/features/estimates/lib/estimate-range.ts
import { dinero, multiply, type Dinero } from "dinero.js";

export function computeEstimateRange(
  base: Dinero<number>,
  contingencyPercent: number, // 0..30
): { low: Dinero<number>; high: Dinero<number> } {
  // Low = base. High = base * (100 + contingencyPercent) / 100.
  // Use scale=2 so multiply by 110 becomes multiply by 1.10.
  const high = multiply(base, { amount: 100 + contingencyPercent, scale: 2 });
  return { low: base, high };
}
```

```typescript
// src/lib/format-currency.ts -- extension
import type { Dinero } from "dinero.js";
import { formatCurrency } from "./format-currency";

export function formatRange(
  low: Dinero<number>,
  high: Dinero<number>,
  mode: "range" | "from",
  locale = "en-GB",
): string {
  if (mode === "from") {
    return `From ${formatCurrency(low, locale)}`;
  }
  return `${formatCurrency(low, locale)} – ${formatCurrency(high, locale)}`;
}
```

**Why this is enough:** `dinero.js` v2's `multiply({ amount, scale })` does integer-safe scaling without float rounding errors. The only "new" utility is the two-line `formatRange` helper. No currency-formatting library needed — `formatCurrency` already wraps `Intl.NumberFormat` via dinero's `toDecimal()`.

**API-side note:** Persist `basePriceAmount` (Dinero-v1 shape) and `contingencyPercent` (integer) on the entity; compute `low`/`high` for responses the same way quote totals are recomputed on read (v1.2 decision: "Quote totals recalculated on every read"). **Do not denormalize** `lowAmount`/`highAmount` — they're derivable and would drift.

## Alternatives Considered (and Rejected)

| Area | Chose | Rejected | Why Not |
|------|-------|----------|---------|
| Scheduled jobs | BullMQ `delay` | `@nestjs/schedule` (cron) | Adds a second scheduler; cron has no per-job cancellation, no retry/backoff, no idempotency via jobId. BullMQ already owns this domain in Trade Flow. |
| Scheduled jobs | BullMQ `delay` | BullMQ Repeatable Jobs | Repeatable = recurring pattern (every N minutes). We need one-shot at specific future offsets. `delay` is correct. |
| Slider | `radix-ui` meta (already installed) | `@radix-ui/react-slider` standalone | Duplicate source of the same component; goes against the February 2026 shadcn/ui unified-package recommendation. |
| Slider | Radix Slider | `react-range`, `rc-slider`, native `<input type="range">` | Native lacks proper styling + multi-mark labels without wrappers; third-party adds dependency; Radix is already in the project and has a11y built in. |
| Versioning | Flat `rootEstimateId` + `parentEstimateId` | Event-sourcing / CQRS | Massive over-engineering for a document with ~5 edit events per lifetime. |
| Versioning | Hand-rolled three-field approach | `mongoose-version` / `mongoose-history` plugin | External plugin buries the versioning logic and is untyped; three indexed fields are perfectly legible and align with existing repo conventions. |
| Range money | Computed from base + percent on read | Store `lowAmount` + `highAmount` directly | Violates v1.2 single-source-of-truth decision ("recalculated on every read"). Denormalized totals drift when contingency changes. |
| Counter | `estimate_counters` collection (mirror of v1.2) | `mongoose-sequence` | v1.2 already proved the atomic-`$inc` pattern works; adding a plugin duplicates that. |
| Status transitions | Enum + transition table utility | `xstate` / `ts-pattern` | Eight states with linear transitions — state-machine library is overkill. |
| Idempotency | Stable `jobId` strings | Lua scripts / application-level lock collection | BullMQ's `jobId` dedup is the cheapest and most reliable option, and it's built in. |

## Version Currency Check

| Library | Installed | Latest (verified April 2026) | Action |
|---------|-----------|------------------------------|--------|
| `bullmq` | 5.71.0 | 5.7x current | No upgrade needed for v1.8; `delay` API is stable since BullMQ 4.x. |
| `@nestjs/bullmq` | 11.0.4 | 11.x | No upgrade needed. |
| `ioredis` | 5.10.1 | 5.x | No upgrade needed. |
| `mongoose` | 9.1.5 | 9.x | No upgrade needed; self-refs are classic Mongoose. |
| `radix-ui` (meta) | 1.4.3 | 1.4.x | Sufficient; Slider primitive is included. |
| `@radix-ui/react-slider` (standalone) | **not installed — do not add** | 1.3.6 on npm | Do not install; use `Slider` from the `radix-ui` meta package instead. |
| `dinero.js` (UI) | 2.0.0-alpha.14 | Still alpha as of April 2026 | **Flag only:** same alpha pin as v1.2 quotes. Not upgrading for v1.8 — stay on same version as quote code for consistency. |
| `dinero.js` (API) | 1.9.1 | v1 is LTS / frozen | No change. Continues to handle server-side money. |
| `luxon` | 3.5.1 | 3.x | No upgrade needed. |

**Confidence:** HIGH for BullMQ, Radix, Mongoose (verified via official docs and v1.4 phase research that inspected `node_modules`). MEDIUM for "latest" dinero.js UI since it remains on an alpha — however this is consistent with v1.2's choice and changing it is out of scope for v1.8.

## Installation (None Required)

```bash
# Zero new packages. v1.8 is entirely pattern work on top of v1.4 + v1.3 infrastructure.
```

If a developer is tempted to run `npm install` for this milestone: stop and re-check this document. The only commits that should touch `package.json` in v1.8 are accidental lockfile churn — and those should be reverted.

## Integration Notes

- **New queue constant:** Add `ESTIMATE_FOLLOW_UP: "estimate-follow-up"` to `src/queue/queue.constant.ts`. Register in `QueueModule` via `BullModule.registerQueue({ name: QUEUE_NAMES.ESTIMATE_FOLLOW_UP })`.
- **New worker processor:** Add `EstimateFollowUpProcessor` to `WorkerModule` (mirror `StripeWebhookProcessor` from v1.6). Inject `EstimateRetriever`, `EmailService`, and the `QueueProducer` (for chained scheduling if ever needed).
- **New module:** `src/estimate/` mirroring `src/quote/` structure (controllers, services, repositories, DTOs, responses, entities, policies). Add `@estimate/*` path alias to both `tsconfig.json`, `jest.config.js`, and worker `tsconfig`.
- **Shared document-type enum:** Neither quote nor estimate should own it. Put `DocumentType` enum in `src/core/enums/` and reference from both features.
- **Email templates:** Add new Maizzle templates (`estimate-sent.njk`, `estimate-follow-up-3d.njk`, `estimate-follow-up-10d.njk`, `estimate-follow-up-21d.njk`, `estimate-site-visit-requested.njk`, etc.) under the existing templates directory. Same Maizzle pipeline as v1.3 quote emails.
- **Public token infrastructure:** Reuse v1.3 `PublicTokenService` pattern. Create a parallel `EstimateSessionAuthGuard` (mirror of `QuoteSessionAuthGuard`). Prefer parallel for now; generalize in a future refactor if both guards stay identical.
- **UI routing:** Add public customer-facing route `/estimates/view/:token` paired with a dedicated `publicEstimateApi` RTK Query slice that (like `publicQuoteApi`) does not inject Firebase auth headers.

## Sources

- [BullMQ — Delayed jobs guide](https://docs.bullmq.io/guide/jobs/delayed) — verified `delay`, `jobId`, `removeOnComplete` semantics
- [BullMQ — Auto-removal of jobs](https://docs.bullmq.io/guide/queues/auto-removal-of-jobs) — `KeepJobs` `{ age, count }` options
- [BullMQ — JobsOptions API reference](https://api.docs.bullmq.io/interfaces/v1.JobsOptions.html) — full options shape
- [Radix Primitives — Slider](https://www.radix-ui.com/primitives/docs/components/slider) — confirmed `min`, `max`, `step`, `value`, `onValueChange` props and a11y behaviour
- [@radix-ui/react-slider on npm](https://www.npmjs.com/package/@radix-ui/react-slider) — version cross-check (1.3.6 latest; NOT to be installed)
- [shadcn/ui — February 2026 Unified Radix UI Package changelog](https://ui.shadcn.com/docs/changelog/2026-02-radix-ui) — confirms `radix-ui` meta package is the current idiomatic import surface
- [Mongoose — Populate (self-refs)](https://mongoosejs.com/docs/populate.html)
- Trade Flow v1.4 Phase 21 RESEARCH.md — confirms installed BullMQ versions (`bullmq@5.71.0`, `@nestjs/bullmq@11.0.4`, `ioredis@5.10.1`) and existing `QueueProducer` pattern
- Trade Flow `.planning/codebase/STACK.md` (v1.7 snapshot) — confirms `radix-ui@1.4.3` meta package and all other installed versions referenced in this document
- Trade Flow v1.2 Quote implementation — pattern source for `estimate_counters`, totals recomputation, module structure
- Trade Flow v1.3 Send Quotes implementation — pattern source for Maizzle templates, public token infrastructure, Resend email integration

---

*Stack research for v1.8 Estimates — 2026-04-10*
