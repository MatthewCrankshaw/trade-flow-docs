# Architecture Research — v1.8 Estimates

**Domain:** Estimates as a parallel document type alongside Quotes in Trade Flow
**Researched:** 2026-04-10
**Confidence:** HIGH (grounded in existing trade-flow-api source code)
**Scope:** Integration of a new `estimate` feature with existing `quote`, `quote-token`, `queue`, `worker`, `email`, `job`, `customer`, `business` modules

---

## Executive Summary

Estimates should ship as a **new `estimate` feature module that mirrors `quote`**, not as a polymorphic extension of `QuoteModule`. The existing code is structured around the explicit assumption that a quote is a single-purpose, status-driven document — entity fields, status enum, totals calculator, policy, transition service, line-item factories, number generator, public controller, and email template are all quote-shaped. Trying to sneak estimates into that module via a type discriminator would require touching every one of those files, inventing nullable fields, fragmenting the status enum, and forcing `QuoteTransitionService` to branch on type. A parallel `EstimateModule` keeps strict Controller → Service → Repository layering intact, follows existing precedent (`quote-token` is already a sibling to `quote`), and confines conversion to a single well-defined boundary.

A few targeted refactors convert today's single-use abstractions into shared ones:

- **Line items**: reuse the existing `QuoteLineItem*` stack unchanged for v1.8 by having estimates store line items in the same `quote_line_items` collection under a `parentType: "quote" | "estimate"` discriminator — cheaper than duplicating the bundle/tax/pricing factories.
- **Tokens**: generalise `quote-token` into a reusable `document-token` capable of pointing at either collection, using a minimal refactor (two weeks ago it was quote-only; now it needs a `documentType`). A fresh `EstimateTokenModule` is a worse long-term choice given customer-page infrastructure is the hardest part to get right.
- **Public controller**: separate `PublicEstimateController` — keep clear boundaries, URL namespaces (`/v1/public/estimate/:token`), and response shapes.
- **Queue**: add a new `ESTIMATE_FOLLOWUPS` queue name. Reuse infrastructure. Cancel-on-mutation is implemented via deterministic `jobId` per (estimateId, sequenceStep) and `queue.remove(jobId)`.
- **Convert to Quote** lives in a new `EstimateToQuoteConverter` service inside the **estimate** module, which depends on `QuoteCreator` (one-way) to avoid circular DI. No `forwardRef` required.
- **Revisions**: flat collection (not embedded), `parentEstimateId` + `revisionNumber`, compound index `{ parentEstimateId: 1, revisionNumber: -1 }` plus a partial unique index on `(parentEstimateId, isCurrent: true)`. History reads become simple; latest-revision reads stay fast.

Frontend: a new `src/features/estimates/` module mirroring `src/features/quotes/`. The unified "Create Document" dialog lives in a new shared location (`src/features/documents/CreateDocumentDialog.tsx`) that internally switches between estimate and quote mutations based on a toggle. No feature module rename or consolidation.

---

## System Overview

### Existing Backend Layout (verified from `trade-flow-api/src/`)

```
src/
├── quote/                         ← Owns quotes end-to-end
│   ├── quote.module.ts            ← Imports QuoteSettings, QuoteToken (forwardRef),
│   │                                Business, Customer, Email, Item, Job, TaxRate
│   ├── controllers/quote.controller.ts
│   ├── services/
│   │   ├── quote-creator.service.ts           (uses AuthorizedCreatorFactory)
│   │   ├── quote-retriever.service.ts
│   │   ├── quote-updater.service.ts
│   │   ├── quote-number-generator.service.ts  (quote_counters collection, Q-YYYY-NNN)
│   │   ├── quote-totals-calculator.service.ts
│   │   ├── quote-transition.service.ts        (Draft→Sent→Accepted/Rejected)
│   │   ├── quote-email-sender.service.ts
│   │   ├── quote-standard-line-item-factory.service.ts
│   │   ├── quote-bundle-line-item-factory.service.ts
│   │   ├── quote-line-item-creator.service.ts
│   │   ├── quote-line-item-retriever.service.ts
│   │   ├── bundle-config-validator.service.ts
│   │   ├── bundle-pricing-planner.service.ts
│   │   └── bundle-tax-rate-calculator.service.ts
│   ├── repositories/
│   │   ├── quote.repository.ts
│   │   └── quote-line-item.repository.ts
│   ├── entities/
│   │   ├── quote.entity.ts        ← { businessId, customerId, jobId, number, title,
│   │   │                              quoteDate, notes, status, validUntil, sentAt,
│   │   │                              acceptedAt, rejectedAt, declineReason, deletedAt,
│   │   │                              lineItems: ObjectId[] }
│   │   └── quote-line-item.entity.ts
│   ├── policies/                  ← QuotePolicy, QuoteLineItemPolicy
│   └── enums/                     ← QuoteStatus, quote-transitions.ts
│
├── quote-token/                   ← Public customer-facing infrastructure
│   ├── quote-token.module.ts      ← forwardRef(QuoteModule), ThrottlerModule
│   ├── controllers/public-quote.controller.ts   ← /v1/public/quote/:token
│   ├── guards/quote-session-auth.guard.ts       ← Token validation + firstViewedAt
│   ├── services/
│   │   ├── quote-token-creator.service.ts       (randomBytes(32), base64url, 30d expiry)
│   │   ├── quote-token-retriever.service.ts
│   │   ├── quote-token-revoker.service.ts
│   │   ├── public-quote-retriever.service.ts
│   │   └── quote-response-handler.service.ts    (accept/decline flow)
│   ├── repositories/quote-token.repository.ts
│   └── entities/quote-token.entity.ts   ← { token, quoteId, expiresAt, revokedAt,
│                                            sentAt, recipientEmail, firstViewedAt }
│
├── quote-settings/                ← Per-business email template config
│   └── quote-settings.module.ts   (imports business via forwardRef)
│
├── queue/                         ← BullMQ producer layer
│   ├── queue.module.ts            ← @Global, BullModule.forRootAsync, registerQueue
│   ├── queue.constant.ts          ← QUEUE_NAMES = { ECHO, STRIPE_WEBHOOKS }
│   └── services/queue-producer.service.ts  ← enqueueStripeWebhook with jobId, attempts, backoff
│
├── worker/                        ← Second entry point (dist/worker.js)
│   ├── worker.module.ts           ← ConfigModule, CoreModule, QueueModule, AuthModule,
│   │                                SubscriptionModule; providers: EchoProcessor, StripeWebhookProcessor
│   └── processors/
│       ├── echo.processor.ts
│       └── stripe-webhook.processor.ts
│
├── email/                         ← Resend + Maizzle
│   ├── services/quote-email-renderer.service.ts
│   └── templates/quote-email.html
│
├── job/, customer/, business/, item/, tax-rate/  ← shared domain modules
```

### Key Observations From Source

- `quote.module.ts` imports **9 other modules** including `forwardRef(() => QuoteTokenModule)` — token module imports quote module and vice versa. This existing circular forwardRef tells us the pattern for estimate↔document-token.
- `QuoteCreator.create()` uses `AuthorizedCreatorFactory.createFor(this.quoteRepository, this.quotePolicy)` then applies `QuoteTotalsCalculator` to the result. Any `EstimateCreator` follows the exact same shape.
- `QuoteNumberGenerator` uses an atomic `findOneAndUpdate` with `$inc: { lastNumber: 1 }` on `quote_counters`. Estimates get a parallel `estimate_counters` collection with the same implementation pattern — no shared abstraction needed yet (two callers is not a duplication problem).
- `QueueProducer` already demonstrates the delayed-job pattern with `jobId`, `attempts`, `backoff`, and `removeOnComplete` — we inherit all of this.
- `QuoteTokenCreator` generates `randomBytes(32).toString("base64url")` and writes a `quote_tokens` row with 30-day expiry. This is the piece that most clearly wants generalisation.

---

## Recommended Architecture

### 1. Module Structure — Separate `EstimateModule`

**Decision:** new `src/estimate/` module, mirroring `src/quote/`, with a new `src/document-token/` module replacing `src/quote-token/`.

**Why separate:**
- **Strict layering is enforced per module** (see `trade-flow-api/CLAUDE.md`): Controllers never talk to another feature's repository. A polymorphic `QuoteModule` would mean `QuoteCreator` branching on `documentType` and calling different downstream services — a violation of single-responsibility that every reviewer would flag.
- **Status lifecycles are genuinely different.** Quotes have Draft/Sent/Accepted/Rejected. Estimates have Draft/Sent/Viewed/Responded → SiteVisitRequested/Converted/Declined/Expired. Unioning these in one enum and one transition graph would make the graph untestable.
- **Field surface is different.** Estimates have contingency percentage, display mode, uncertainty notes, structured decline reasons, parent/revision fields. Quotes have validUntil, acceptedAt, rejectedAt, declineReason. A single `documents` entity would be ~50% nullable fields.
- **Conversion is a one-way, well-scoped operation** — a natural service boundary that does not require shared internals.

**Why NOT a type discriminator on `QuoteModule`:**
- Would require editing `QuoteStatus`, `quote-transitions.ts`, `QuoteTransitionService`, `QuotePolicy`, `QuoteCreator`, `QuoteUpdater`, `QuoteEmailSender`, `quote.entity.ts`, `quote.responses.ts`, `quote.controller.ts`, and every existing test.
- Database queries across the codebase would need `{ documentType: "quote" }` filters added retroactively — easy to miss, silent correctness bugs.
- Violates the principle applied everywhere else: `quote-settings` and `quote-token` are already separate feature modules, not embedded in `quote`.

**What gets reused vs duplicated:**

| Concern                        | Strategy                                                                 |
|--------------------------------|--------------------------------------------------------------------------|
| Controller → Service → Repo    | **Duplicate** (mirror the shape; no abstraction)                        |
| Policy pattern                 | **Duplicate** as `EstimatePolicy` (same shape as `QuotePolicy`)         |
| Line items                     | **Reuse** `QuoteLineItem*` with a `parentType: "quote" \| "estimate"` field; see §3 |
| Bundle factories               | **Reuse** (already generic over parent — pass parentId/parentType)      |
| Totals calculator              | **Reuse** `QuoteTotalsCalculator` (operates on line items, not quote fields) — rename to `DocumentTotalsCalculator` and move to a shared location, or extract interface. Pragmatic first pass: **duplicate** as `EstimateTotalsCalculator` if calculator reads any quote-specific fields |
| Number generator               | **Duplicate** as `EstimateNumberGenerator` with `estimate_counters` collection and `E-YYYY-NNN` format |
| Totals recalc-on-read          | **Duplicate** (stateless service)                                       |
| Transition service             | **Duplicate** (different state graph)                                    |
| Email sender                   | **Duplicate** as `EstimateEmailSender`; share `EmailModule` infrastructure |
| Email template                 | **New** `estimate-email.html` in `src/email/templates/`                 |
| Email renderer                 | **New** `EstimateEmailRenderer` sibling to `QuoteEmailRenderer`         |

### 2. Data Model

**Decision:** separate `estimates` collection, separate `estimate_counters`, shared `quote_line_items` collection with a `parentType` discriminator field.

**Estimate entity (new):**

```typescript
// src/estimate/entities/estimate.entity.ts
export interface IEstimateEntity extends IBaseEntity {
  businessId: ObjectId;
  customerId: ObjectId;
  jobId: ObjectId;
  number: string;                    // E-YYYY-NNN
  title: string;
  estimateDate: Date;
  notes?: string;
  status: EstimateStatus;

  // Price range
  contingencyPercent: number;        // 0..30, step 5, default 10
  displayMode: EstimateDisplayMode;  // "range" | "from"

  // Uncertainty
  uncertaintyReasons?: string[];     // ["site_inspection", "pipework", ...]
  uncertaintyNotes?: string;

  // Lifecycle timestamps
  sentAt?: Date;
  firstViewedAt?: Date;
  respondedAt?: Date;
  convertedAt?: Date;
  convertedToQuoteId?: ObjectId;
  declinedAt?: Date;
  declineReason?: EstimateDeclineReason;     // enum
  declineDetails?: string;
  siteVisitRequestedAt?: Date;
  siteVisitRequest?: {                       // structured request payload
    preferredWindows: string[];
    availabilityNotes?: string;
  };
  expiresAt?: Date;

  // Revisions
  parentEstimateId?: ObjectId;       // null for the original; set for revisions
  revisionNumber: number;            // 1 for original, 2+ for revisions
  isCurrent: boolean;                // only one current per parent chain

  deletedAt?: Date;
  lineItems?: ObjectId[];
}
```

**Why a separate collection (not a shared `documents` collection):**
- Migration cost. Quotes exist in production already with ~real data. Moving them to `documents` is a destructive migration with no user benefit.
- Query plans. Indexes differ: estimates need `(parentEstimateId, revisionNumber)`, quotes don't.
- Field bloat. A shared collection means every document row carries every possible field, or nullable unions everywhere.

**Why shared `quote_line_items` with parentType:**
- Line items are genuinely the same thing for quotes and estimates (materials, labour, fees, bundles, tax calculations).
- The existing bundle factories, tax rate calculators, and pricing planner all operate on "a line item attached to a parent." Adding a `parentType` enum field is a minor migration, not a duplication.
- Reuses all existing code: `QuoteLineItemRepository`, `QuoteLineItemCreator`, bundle validators. Rename to `DocumentLineItem*` eventually, but v1.8 can keep names and just widen semantics.
- **Alternative considered and rejected:** a separate `estimate_line_items` collection. Would mean duplicating all bundle/tax logic, creating a parallel `EstimateBundleLineItemFactory`, etc. Cost: ~15 files duplicated for zero gain.

**Migration required:**
- Add `parentType: "quote"` to all existing rows in `quote_line_items` (single `updateMany`).
- Add partial index `{ parentType: 1, parentId: 1 }` to replace/augment existing `{ quoteId: 1 }`.
- Rename `quoteId` field to `parentId` OR keep `quoteId` and add a computed accessor. **Pragmatic:** keep `quoteId` column for quote rows, add `estimateId` column for estimate rows, make repository layer handle both. No schema migration of existing rows needed. Requires two fields instead of one but avoids a risky migration on live data.

### 3. Versioned Revisions

**Decision:** flat collection, `parentEstimateId` + `revisionNumber`, compound indexes.

```typescript
// Entity fields (from §2)
parentEstimateId?: ObjectId;   // null on the root estimate
revisionNumber: number;        // 1 on root, 2, 3, ... on revisions
isCurrent: boolean;            // exactly one current per chain
```

**Normalisation rule:** the **root** (original) estimate has `parentEstimateId === null`, `revisionNumber === 1`. A revision has `parentEstimateId === root._id`, `revisionNumber === 2..N`. "The latest revision" is the row with the highest `revisionNumber` in the chain (or equivalently, the row with `isCurrent: true`).

**Required Mongoose/MongoDB indexes:**

```javascript
// 1. Fast "get all revisions for an estimate chain" (audit, history panel)
db.estimates.createIndex({ parentEstimateId: 1, revisionNumber: -1 });

// 2. Fast "get the latest revision" — served by index 1 via limit(1),
//    or more directly:
db.estimates.createIndex(
  { parentEstimateId: 1, isCurrent: 1 },
  { partialFilterExpression: { isCurrent: true } }
);

// 3. Uniqueness: prevent two "current" revisions in the same chain
db.estimates.createIndex(
  { parentEstimateId: 1 },
  { unique: true, partialFilterExpression: { isCurrent: true } }
);

// 4. Existing: list estimates for a business
db.estimates.createIndex({ businessId: 1, createdAt: -1 });

// 5. Unique estimate number per business/year
db.estimates.createIndex(
  { businessId: 1, number: 1 },
  { unique: true, partialFilterExpression: { deletedAt: null } }
);

// 6. Fast job → estimates lookup (job detail page)
db.estimates.createIndex({ jobId: 1, createdAt: -1 });
```

**Why flat, not embedded:**
- **Embedded arrays have 16MB document limit.** Unlikely to hit with estimate revisions, but line items already reference estimates by id — embedding would duplicate that reference structure awkwardly.
- **Query patterns favour flat.** Looking up "the current revision" with an index is O(log n); walking an embedded array is O(n).
- **Line items already live in a separate collection.** If estimates were embedded in a parent "estimate chain" document, line items would have to reference the chain + the revision position, which is a worse foreign key.
- **Audit history is a feature, not a rare query.** Users see collapsed "History" section in the UI. This must be fast.
- **Revise → create new row** is cleaner than "mutate nested array." Each revision is immutable once superseded, which matches the UX promise of an audit trail.

**Revision creation flow (`EstimateReviser` service):**
1. Load the current revision.
2. In a session: insert a new estimate row copying fields, `parentEstimateId = root._id`, `revisionNumber = currentRevisionNumber + 1`, `isCurrent = true`.
3. Update the previous current revision: `isCurrent = false`.
4. Copy line items to the new revision (new `quote_line_items` rows referencing the new estimateId).
5. Reset the follow-up queue for the new revision (see §6).

(Not using Mongo transactions unless the codebase already does; the existing `MongoConnectionService` does not show transaction use. Two-step update with `isCurrent` flipping is acceptable because the partial unique index prevents duplicate currents — the second insert would fail before the previous row is downgraded. Correct approach: downgrade the old row first, then insert the new one. If insertion fails, upgrade the old row back. Document this in the service.)

### 4. Token Reuse — Generalise to `DocumentToken`

**Decision:** rename `quote-token` → `document-token` module, add a `documentType: "quote" | "estimate"` field to the token entity.

**Why generalise rather than duplicate:**
- `QuoteTokenCreator` is 30 lines; `QuoteSessionAuthGuard` is the real value — it encodes rate limiting, expiry, revocation, and `firstViewedAt` tracking. **Duplicating that is asking for two bugs in parallel.**
- Public customer UX is the most-tested, most-scrutinised part of the system (customers are non-technical; error states must be clean). One canonical implementation.
- Generalisation is small: add a `documentType` discriminator field; guard looks up the referenced document via the right repository.

**What the refactor looks like:**

```typescript
// src/document-token/entities/document-token.entity.ts
export interface IDocumentTokenEntity extends IBaseEntity {
  token: string;
  documentType: "quote" | "estimate";   // new field
  documentId: ObjectId;                  // renamed from quoteId
  expiresAt: Date;
  revokedAt?: Date;
  sentAt: Date;
  recipientEmail: string;
  firstViewedAt?: Date;
}
```

**Migration for existing quote_tokens rows:**
```javascript
db.quote_tokens.updateMany({}, { $set: { documentType: "quote" }, $rename: { quoteId: "documentId" } });
db.quote_tokens.renameCollection("document_tokens");
```
One-shot, reversible, zero downtime.

**Guard generalisation:**
```typescript
@Injectable()
export class DocumentSessionAuthGuard implements CanActivate {
  constructor(private readonly tokenRepo: DocumentTokenRepository) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest();
    const token = req.params.token;
    const tokenDto = await this.tokenRepo.findByTokenOrFail(token); // throws 404/410
    req.documentToken = tokenDto;
    return true;
  }
}
```

The guard no longer knows or cares about document type. The public controller reads `req.documentToken.documentType` to dispatch.

**Cost/benefit:**

| Option                                   | Upfront cost | Ongoing cost | Risk |
|------------------------------------------|--------------|--------------|------|
| Reuse unchanged                          | 0            | n/a — impossible (entity hard-codes quoteId) | — |
| Generalise to `DocumentTokenModule` ✓    | Medium       | Low          | Medium migration, well-scoped |
| Separate `EstimateTokenModule`           | High         | High (two codebases to keep in sync) | Low each, High cumulative |

### 5. Public Controller — Separate `PublicEstimateController`

**Decision:** new `PublicEstimateController` at `/v1/public/estimate/:token`, separate from `PublicQuoteController`. Both use the shared `DocumentSessionAuthGuard`.

**Why separate (despite shared token module):**
- URL namespace clarity. Customers may see either URL; `/public/estimate/...` is self-documenting.
- Response shapes differ. Estimates return `contingencyPercent`, `displayMode`, `uncertaintyReasons`, `siteVisitRequest` fields that don't exist on quotes.
- Response actions differ. Estimate has four response buttons (site visit, quote me, question, not now); quote has accept/decline. Unifying into `PublicDocumentController` with type branches is the polymorphic-QuoteModule anti-pattern at the controller level.
- `QuoteResponseHandler` is already quote-specific (handles accept → `QuoteTransitionService.transition(SENT → ACCEPTED)`). A parallel `EstimateResponseHandler` is the right shape.

**Shared:**
- `DocumentSessionAuthGuard` (from §4)
- `ThrottlerGuard` with the same rate-limit config
- Error response shape (business name context, 404/410 handling)

### 6. BullMQ Follow-up Queue

**Decision:** new queue `ESTIMATE_FOLLOWUPS`, one job name per step, deterministic `jobId` scheme for idempotent cancellation.

**Queue registration:**

```typescript
// src/queue/queue.constant.ts
export const QUEUE_NAMES = {
  ECHO: "echo",
  STRIPE_WEBHOOKS: "stripe-webhooks",
  ESTIMATE_FOLLOWUPS: "estimate-followups",   // NEW
} as const;
```

```typescript
// src/queue/queue.module.ts — add
BullModule.registerQueue({ name: QUEUE_NAMES.ESTIMATE_FOLLOWUPS }),
```

**Why a new queue (not reuse existing):**
- Queues are cheap in BullMQ (they're just Redis key prefixes).
- Isolates failure modes: a follow-up processor crash does not affect stripe webhook processing.
- Allows separate concurrency/rate limits if needed later.
- Mirrors the pattern already used for stripe vs echo.

**Follow-up scheduling (3/10/21 days):**

```typescript
// src/estimate/services/estimate-followup-scheduler.service.ts
@Injectable()
export class EstimateFollowupScheduler {
  private static readonly STEPS_DAYS = [3, 10, 21];

  constructor(
    @InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS) private readonly queue: Queue,
  ) {}

  public async schedule(estimateId: string, sentAt: DateTime): Promise<void> {
    for (const step of EstimateFollowupScheduler.STEPS_DAYS) {
      const runAt = sentAt.plus({ days: step });
      const delay = runAt.toMillis() - DateTime.now().toMillis();
      await this.queue.add(
        "estimate-followup",
        { estimateId, step },
        {
          jobId: this.jobIdFor(estimateId, step),   // deterministic, idempotent
          delay: Math.max(delay, 0),
          attempts: 3,
          backoff: { type: "exponential", delay: 5000 },
          removeOnComplete: { age: 86400, count: 1000 },
          removeOnFail: { age: 86400 * 7, count: 5000 },
        },
      );
    }
  }

  public async cancelAll(estimateId: string): Promise<void> {
    await Promise.all(
      EstimateFollowupScheduler.STEPS_DAYS.map((step) =>
        this.queue.remove(this.jobIdFor(estimateId, step)),
      ),
    );
  }

  public async reschedule(estimateId: string, sentAt: DateTime): Promise<void> {
    await this.cancelAll(estimateId);
    await this.schedule(estimateId, sentAt);
  }

  private jobIdFor(estimateId: string, step: number): string {
    return `estimate-followup:${estimateId}:${step}`;
  }
}
```

**Cancellation triggers** (the scheduler's `cancelAll` / `reschedule` is called from):
- `EstimateReviser` → `reschedule` (reset on revision per milestone spec)
- `EstimateToQuoteConverter` → `cancelAll` (converted, no more follow-ups)
- `EstimateResponseHandler.decline` → `cancelAll`
- `EstimateResponseHandler.requestSiteVisit` → `cancelAll`
- `EstimateExpirer` (future) → `cancelAll`

**Worker side — new processor:**

```typescript
// src/worker/processors/estimate-followup.processor.ts
@Processor(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)
export class EstimateFollowupProcessor extends WorkerHost {
  constructor(
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateEmailSender: EstimateEmailSender,
  ) { super(); }

  async process(job: Job<{ estimateId: string; step: number }>): Promise<void> {
    const estimate = await this.estimateRetriever.findByIdOrNull(job.data.estimateId);

    // Idempotency: if estimate is deleted, converted, declined, or no longer current,
    // silently no-op. The scheduler should have removed this job but we can't rely on
    // race-free cancellation.
    if (!estimate || !estimate.isCurrent || this.shouldSkipFollowup(estimate.status)) {
      return;
    }

    await this.estimateEmailSender.sendFollowup(estimate, job.data.step);
  }

  private shouldSkipFollowup(status: EstimateStatus): boolean {
    return [
      EstimateStatus.CONVERTED,
      EstimateStatus.DECLINED,
      EstimateStatus.EXPIRED,
      EstimateStatus.SITE_VISIT_REQUESTED,
    ].includes(status);
  }
}
```

Then add to `WorkerModule.providers`: `EstimateFollowupProcessor`.

**Why deterministic jobIds matter:**
- `queue.remove(jobId)` is O(1). Without deterministic IDs you'd have to list jobs and filter, which is slow and racy.
- Re-adding a job with the same ID while one exists is a no-op in BullMQ — automatic idempotency for retry scenarios.
- Pattern already proven in `QueueProducer.enqueueStripeWebhook` (`jobId: event.id`).

**Defence-in-depth:** the worker also checks estimate state on processing, so a race where a job fires between cancellation attempts still no-ops cleanly.

### 7. Estimate → Quote Conversion

**Decision:** new `EstimateToQuoteConverter` service in the `estimate` module that depends on `QuoteCreator` via standard DI. No circular dependency.

```typescript
// src/estimate/services/estimate-to-quote-converter.service.ts
@Injectable()
export class EstimateToQuoteConverter {
  constructor(
    private readonly estimateRetriever: EstimateRetriever,
    private readonly estimateUpdater: EstimateUpdater,
    private readonly estimateLineItemRetriever: QuoteLineItemRetriever, // reused
    private readonly quoteCreator: QuoteCreator,
    private readonly quoteLineItemCreator: QuoteLineItemCreator,
    private readonly followupScheduler: EstimateFollowupScheduler,
    private readonly accessControllerFactory: AccessControllerFactory,
    private readonly policy: EstimatePolicy,
  ) {}

  public async convert(authUser: IUserDto, estimateId: string): Promise<IQuoteDto> {
    const estimate = await this.estimateRetriever.findByIdOrFail(authUser, estimateId);

    // Authorization: converting is a write on the estimate
    this.accessControllerFactory.create(this.policy).canUpdate(authUser, estimate);

    // Build quote DTO from latest estimate revision, dropping contingency
    const quoteDto: IQuoteDto = {
      id: new ObjectId().toString(),
      businessId: estimate.businessId,
      customerId: estimate.customerId,
      jobId: estimate.jobId,
      number: "",                       // QuoteCreator assigns via QuoteNumberGenerator
      title: estimate.title,
      quoteDate: DateTime.now(),
      notes: estimate.notes,
      status: QuoteStatus.DRAFT,
      // No contingency — quotes are firm prices
    };

    // QuoteCreator handles number generation, policy check, persistence, totals
    const createdQuote = await this.quoteCreator.create(authUser, quoteDto);

    // Copy line items (base price; no contingency markup)
    const estimateLineItems = await this.estimateLineItemRetriever.findByParent(estimate.id);
    for (const li of estimateLineItems) {
      await this.quoteLineItemCreator.create(authUser, { ...li, parentId: createdQuote.id, parentType: "quote" });
    }

    // Back-link: mark estimate as converted
    await this.estimateUpdater.markConverted(authUser, estimate.id, createdQuote.id);

    // Cancel any pending follow-ups
    await this.followupScheduler.cancelAll(estimate.id);

    return createdQuote;
  }
}
```

**Circular dependency analysis:**
- `EstimateModule` imports `QuoteModule` (to access `QuoteCreator`, `QuoteLineItemCreator`).
- `QuoteModule` does **not** need to import `EstimateModule` — quotes know nothing about estimates. The back-link is stored on the estimate side (`convertedToQuoteId`), not the quote side.
- No `forwardRef` needed. One-way dependency: estimate → quote.

**Why the converter lives in `estimate/` not `quote/`:**
- It's an estimate-triggered operation. The UI action is "Convert this estimate." Consistent with GSD convention: services live where the entity they mutate lives.
- `QuoteCreator` stays pure — no special `estimateId` parameter polluting its signature.
- `EstimateUpdater.markConverted()` is where the estimate status transitions; that logic belongs in the estimate module.

**Rejected alternative:** `QuoteCreator.create()` takes an optional `estimateId` input.
- Pollutes the quote signature.
- Forces `QuoteModule` to depend on `EstimateModule` for the back-link update → circular dependency.
- Hides conversion semantics inside a "create" method.

### 8. Frontend Architecture

**Decision:** new `src/features/estimates/` mirroring `src/features/quotes/`. Shared create-dialog extracted to `src/features/documents/` (a tiny new feature module).

**New structure:**

```
src/features/
├── quotes/                        ← unchanged
├── estimates/                     ← NEW
│   ├── api/estimateApi.ts         ← injectEndpoints into apiSlice
│   ├── api/publicEstimateApi.ts   ← separate slice, no Firebase JWT (mirrors publicQuoteApi)
│   ├── components/
│   │   ├── EstimatesDataView.tsx
│   │   ├── EstimatesTable.tsx
│   │   ├── EstimatesCardList.tsx
│   │   ├── EstimateDetail.tsx
│   │   ├── EstimateHistoryPanel.tsx     ← collapsed revisions
│   │   ├── EstimateContingencySlider.tsx
│   │   ├── EstimateActionStrip.tsx      ← convert, revise, mark lost
│   │   ├── SendEstimateDialog.tsx
│   │   └── EstimateLineItemsCard.tsx    ← likely wraps QuoteLineItemsCard or imports it
│   ├── hooks/
│   │   ├── useEstimatesList.ts
│   │   ├── useEstimateActions.ts
│   │   └── useEstimateRevisions.ts
│   └── index.ts
│
├── documents/                     ← NEW — shared surface between quotes & estimates
│   ├── components/CreateDocumentDialog.tsx   ← type toggle, routes to create mutation
│   └── index.ts
│
├── (customer pages)
├── src/pages/CustomerEstimatePage.tsx        ← NEW public page, mirrors CustomerQuotePage
```

**Why separate `estimates/` feature module:**
- Existing convention: one feature module per domain. `quotes/`, `customers/`, `items/` are already independent.
- Redux tag invalidation is cleaner: `Estimate` tag separate from `Quote` tag.
- RTK Query endpoint naming is flatter and clearer.
- Lazy loading: estimates page bundle-splits from quotes.

**Why a shared `CreateDocumentDialog` is worth a new `documents/` module:**
- The create dialog is where the quote-vs-estimate decision happens. Putting it in `quotes/` or `estimates/` creates arbitrary ownership.
- It's a small surface (one dialog, maybe one hook) — a lightweight feature module.
- Naturally extends if we add a third document type (invoices) later.

**Rejected alternative:** replace `src/features/quotes/` with `src/features/documents/` and unify under tabs.
- Requires renaming everything, rewriting tests, invalidating muscle memory.
- Treats two similar-but-distinct domains as one, obscuring their differences.
- Users see them as different things on different pages; the code should match.

**Public customer-facing page:**
- Separate `CustomerEstimatePage.tsx`, separate route `/estimate/:token`.
- Uses new `publicEstimateApi` slice (same pattern as `publicQuoteApi`) so Firebase JWT headers are not leaked to public endpoints.
- Four response buttons component (`EstimateResponseButtons.tsx`) handles: Book site visit, Send quote, Ask question, Not right now.

### 9. Integration Points — Fields/References on Existing Entities

Estimates touch several existing modules. **What needs changing** in each:

| Module              | Change                                                                                                                    | Type     |
|---------------------|---------------------------------------------------------------------------------------------------------------------------|----------|
| **job**             | `Job` entity gains no required fields. `JobRetriever` is called from `EstimateCreator.validateEstimate()` (same pattern as `QuoteCreator`). No schema change. Job detail page (UI) gains an "Estimates" section next to "Quotes." | Minor    |
| **customer**        | `CustomerRetriever` called from `EstimateCreator.validateEstimate()` for inactive-check parity. Customer detail page (UI) gains estimates history. | Minor    |
| **business**        | `quote-settings` module today stores the quote email template. Either (a) extend `quote-settings` → rename to `document-settings` with `quoteEmailTemplate` + `estimateEmailTemplate`, or (b) new `estimate-settings` module. **Recommend (a)** — settings naturally live together. One migration + one Settings > Templates tab. | **Medium refactor** |
| **email**           | New `EstimateEmailRenderer` service, new `estimate-email.html` Maizzle template, new `estimate-followup-email.html` template. `EmailModule` providers list grows. | Additive |
| **queue**           | Add `ESTIMATE_FOLLOWUPS` to `QUEUE_NAMES`, register in `BullModule.registerQueue()`. Add `enqueueEstimateFollowup` to `QueueProducer` OR (better) inject `@InjectQueue(QUEUE_NAMES.ESTIMATE_FOLLOWUPS)` directly into `EstimateFollowupScheduler` and bypass `QueueProducer`. | Additive |
| **worker**          | Add `EstimateFollowupProcessor` to `WorkerModule.providers`. Worker module needs to import `EstimateModule` (for `EstimateRetriever` + `EstimateEmailSender`) — **or**, cleaner, create a minimal `EstimateWorkerModule` that exports only what the worker needs. | **Medium** |
| **quote-token**     | Rename → `document-token`. Widen entity with `documentType`. Migration: rename collection, add field, rename `quoteId` → `documentId`. Guard becomes `DocumentSessionAuthGuard`. `PublicQuoteController` updated to read `request.documentToken` instead of `request.quoteToken`. **Breaking change** in controller request typing — audit all callers. | **Medium refactor** |
| **quote-settings**  | See business row above. Rename to `document-settings` OR co-locate estimate template fields. | Medium  |
| **quote** (line items) | Add `parentType: "quote" \| "estimate"` + `estimateId?` field to `quote_line_items`. Update `QuoteLineItemRepository.findByParent()` to accept parent type + id. Backfill: `updateMany({}, { $set: { parentType: "quote" } })`. | **Medium migration** |
| **app.module.ts**   | Register new `EstimateModule`, `DocumentTokenModule` (replacing `QuoteTokenModule`), `DocumentSettingsModule` (replacing `QuoteSettingsModule`). | Trivial  |
| **openapi.yaml**    | Add endpoints: `GET /v1/estimate`, `POST /v1/estimate`, `PATCH /v1/estimate/:id`, `POST /v1/estimate/:id/send`, `POST /v1/estimate/:id/revise`, `POST /v1/estimate/:id/convert`, `GET /v1/public/estimate/:token`, `POST /v1/public/estimate/:token/respond`. | Additive |

### 10. Build Order — Suggested Phase Sequencing

Dependencies are surfaced in reverse (what blocks what):

```
Phase A. Foundations                          (prerequisite for everything)
├─ Rename quote-token → document-token
├─ Rename quote-settings → document-settings (estimate template field)
└─ Widen quote_line_items with parentType
    Why first: Both Phase B and all customer-facing work depend on the renamed
    token module. Doing it before estimate code means every new file uses the
    right names immediately — no rewrite later.

Phase B. Estimate module CRUD                 (no customer-facing, no queue)
├─ EstimateEntity, IEstimateDto, EstimateStatus enum
├─ EstimateRepository
├─ EstimatePolicy
├─ EstimateNumberGenerator (E-YYYY-NNN, estimate_counters)
├─ EstimateCreator, EstimateRetriever, EstimateUpdater, EstimateDeleter
├─ EstimateTotalsCalculator (with contingency)
├─ EstimateTransitionService
├─ EstimateController (CRUD, no send)
├─ Mongoose-style indexes (§3)
└─ API tests
    Why before email: can create/read/edit an estimate entirely server-side before
    worrying about customer communication. Unblocks frontend work for the create
    + list + detail experience.

Phase C. Revisions                            (depends on B)
├─ isCurrent / parentEstimateId / revisionNumber logic
├─ EstimateReviser service (copy line items, flip isCurrent, reset anything)
├─ Revisions endpoint: GET /v1/estimate/:id/revisions
└─ History partial unique index
    Why after base CRUD: revisions are a second-order concept on top of estimates.
    Getting the base data model right first avoids reworking revision logic.

Phase D. Frontend estimates CRUD              (parallel with Phase C)
├─ src/features/estimates/ scaffolding (api, components, hooks)
├─ EstimatesPage, EstimateDetailPage
├─ EstimateFormDialog with ContingencySlider
├─ src/features/documents/CreateDocumentDialog with type toggle
└─ Wire into job detail page "Estimates" section
    Can be built against Phase B alone. Revisions UI (history panel) can be a
    follow-on within this phase.

Phase E. Email + Send flow                    (depends on B + D for test harness)
├─ estimate-email.html Maizzle template
├─ EstimateEmailRenderer service
├─ EstimateEmailSender service
├─ POST /v1/estimate/:id/send endpoint
└─ SendEstimateDialog UI
    Why after CRUD: sending requires an estimate to exist and requires the
    settings module to host the template. Nothing before this depends on email.

Phase F. Public customer page                 (depends on A + E)
├─ DocumentSessionAuthGuard reused
├─ PublicEstimateController
├─ PublicEstimateRetriever service (sibling to PublicQuoteRetriever)
├─ EstimateResponseHandler (site visit, quote, question, decline)
├─ publicEstimateApi RTK Query slice
├─ CustomerEstimatePage (public, no Firebase)
└─ Four response buttons + structured forms
    Why late: customer-facing is the most-scrutinised surface. Ship it after the
    happy path (create → send) is verified so testers can see what customers see.

Phase G. Follow-up queue                      (depends on E + worker module)
├─ QUEUE_NAMES.ESTIMATE_FOLLOWUPS registration
├─ EstimateFollowupScheduler (producer side)
├─ EstimateFollowupProcessor (worker side, new entry in WorkerModule)
├─ Wire scheduler into EstimateEmailSender.send (on success → schedule 3/10/21)
├─ Wire cancelAll into EstimateReviser, EstimateToQuoteConverter, EstimateResponseHandler
└─ estimate-followup-email.html template + processor-side renderer
    Why after send: no reason to schedule follow-ups before basic send works.
    Why before convert: convert needs to call cancelAll, so the scheduler must exist.

Phase H. Convert to Quote                     (depends on B + existing QuoteCreator)
├─ EstimateToQuoteConverter service
├─ POST /v1/estimate/:id/convert endpoint
├─ EstimateUpdater.markConverted method
├─ Convert button in EstimateActionStrip UI
└─ E2E test: estimate created → sent → viewed → converted → follow-ups cancelled
    Why last (backend): depends on follow-up cancellation (G), on estimate
    persistence (B), and on quote creation (existing). Nothing else depends on it,
    so it can slip without blocking anything.

Phase I. Polish                               (parallel, ongoing)
├─ Mark as Lost endpoint + structured decline reasons reporting stub
├─ Estimate expiry background job (defers to future)
├─ Revision history panel UI
└─ Documentation / openapi.yaml update
```

**Minimum viable shipping order:** A → B → D → E → F → G → H. Revisions (C) and polish (I) can come in follow-ups.

**Critical path for "estimates work end to end":** A, B, E, F, G. Conversion (H) is the value-add feature that converts the product proposition into quotes, but can lag by days if needed.

---

## Trade-offs Discussed

| Decision                  | Chosen                                          | Alternative                                        | Trade-off                                                    |
|---------------------------|-------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------------|
| Module structure          | Separate `EstimateModule`                       | Discriminator on `QuoteModule`                      | +isolation, +lifecycle clarity; −duplication of ~10 service files |
| Documents collection      | Separate `estimates` collection                 | Shared `documents` collection                       | +no migration of live quotes, +focused indexes; −field drift  |
| Line item storage         | Shared `quote_line_items` with `parentType`     | New `estimate_line_items` collection                | +reuses bundle/tax logic; −migration of live line items       |
| Revisions                 | Flat with parent_id + revision_number           | Embedded array in parent                            | +query-friendly, +unbounded chains; −more rows to manage      |
| Token module              | Generalise to `document-token`                  | Separate `estimate-token` module                    | +one canonical guard, +less duplication; −breaking refactor   |
| Public controller         | Separate `PublicEstimateController`             | Unified `PublicDocumentController`                  | +clear URL namespaces, +typed responses; −two routing files   |
| Follow-up queue           | New `ESTIMATE_FOLLOWUPS` queue                  | Reuse existing queue with job types                 | +isolated failures, +separate rate limits; −new queue to monitor |
| Conversion location       | `EstimateToQuoteConverter` in estimate module   | `QuoteCreator` accepts optional `estimateId`        | +no circular DI, +clean signatures; −lives next to conversion trigger |
| Settings module           | Extend `quote-settings` → `document-settings`   | New `estimate-settings` module                      | +one settings API, +one Settings tab; −rename refactor       |
| Frontend feature layout   | New `src/features/estimates/` + small `documents/` for shared dialog | Unify into `src/features/documents/` with tabs | +minimal churn, +clear tag invalidation; −one small shared module |

---

## Confidence Assessment

| Area                       | Confidence | Basis                                                             |
|----------------------------|------------|-------------------------------------------------------------------|
| Existing quote layout      | HIGH       | Read directly: `quote.module.ts`, `quote.entity.ts`, `quote-creator.service.ts`, `quote-number-generator.service.ts` |
| Existing token layout      | HIGH       | Read directly: `quote-token.module.ts`, `quote-token-creator.service.ts`, `quote-token.entity.ts` |
| Existing queue + worker    | HIGH       | Read directly: `queue.module.ts`, `queue.constant.ts`, `queue-producer.service.ts`, `worker.module.ts` |
| BullMQ delayed-job pattern | HIGH       | Pattern visible in `QueueProducer.enqueueStripeWebhook`; jobId-as-idempotency is idiomatic BullMQ |
| MongoDB partial-unique indexes | HIGH   | Standard MongoDB feature; already used in codebase conceptually for soft delete |
| Convert-to-quote DI shape  | HIGH       | Follows existing pattern of cross-module dependencies (`QuoteModule` depends on `CustomerModule`, etc.) without forwardRef where no cycle exists |
| Line item `parentType` widening | MEDIUM | Shape of refactor is clear; actual migration cost depends on line item volume in prod (not verified) |
| Frontend feature split     | MEDIUM     | Pattern matches existing conventions; `documents/` feature module is new ground but small |
| Estimate lifecycle enum    | MEDIUM     | Derived from PROJECT.md milestone spec; exact transition validation rules need product confirmation |

---

## Open Questions for Roadmap

1. **Settings module rename scope** — is renaming `quote-settings` → `document-settings` in scope for v1.8, or do we instead tolerate an `estimate-settings` module and unify in a later milestone? Recommend rename in Phase A for cleaner long-term shape.
2. **Line items collection migration** — safe to backfill `parentType: "quote"` on production data during deploy, or stage through a feature flag? Depends on prod row count.
3. **Worker module import strategy** — does the worker import full `EstimateModule` (heavy) or a minimal `EstimateWorkerModule` exporting just `EstimateRetriever` + `EstimateEmailSender`? Minimal is cleaner but requires an extra module file.
4. **Transaction semantics for revisions** — the `isCurrent` flip is a two-write operation. The existing codebase does not appear to use MongoDB transactions. Acceptable to rely on the partial unique index as the correctness guard + a retry path? Or introduce transactions for this single case?
5. **Expiry job** — is `EstimateExpirer` (cron-style sweep) in v1.8 scope? The milestone spec mentions an `Expired` status but no automation details.
6. **Estimate PDF** — out of scope per milestone, consistent with quote PDF deferral.

---

## File Inventory — What Gets Created vs Modified

**New files (backend):**
- `src/estimate/estimate.module.ts`
- `src/estimate/controllers/estimate.controller.ts`
- `src/estimate/services/` — creator, retriever, updater, deleter, number-generator, totals-calculator, transition, email-sender, followup-scheduler, reviser, to-quote-converter, response-handler (public)
- `src/estimate/repositories/estimate.repository.ts`
- `src/estimate/entities/estimate.entity.ts`
- `src/estimate/data-transfer-objects/estimate.dto.ts`
- `src/estimate/requests/*.request.ts`
- `src/estimate/responses/estimate.responses.ts`
- `src/estimate/policies/estimate.policy.ts`
- `src/estimate/enums/estimate-status.enum.ts`, `estimate-transitions.ts`, `estimate-decline-reason.enum.ts`, `estimate-display-mode.enum.ts`
- `src/document-token/` (renamed from `quote-token/`) — all files
- `src/document-token/controllers/public-estimate.controller.ts` (NEW sibling to public-quote)
- `src/document-token/services/public-estimate-retriever.service.ts`
- `src/document-token/services/estimate-response-handler.service.ts`
- `src/document-settings/` (renamed from `quote-settings/`) — all files, `estimateEmailTemplate` field added
- `src/email/services/estimate-email-renderer.service.ts`
- `src/email/templates/estimate-email.html`
- `src/email/templates/estimate-followup-email.html`
- `src/worker/processors/estimate-followup.processor.ts`
- `src/estimate/test/**/*.spec.ts` (mirror structure)

**New files (frontend):**
- `src/features/estimates/` — api, components, hooks, index
- `src/features/documents/components/CreateDocumentDialog.tsx`
- `src/pages/EstimatesPage.tsx`
- `src/pages/EstimateDetailPage.tsx`
- `src/pages/CustomerEstimatePage.tsx`
- `src/services/publicEstimateApi.ts` (separate slice, no JWT)

**Modified (backend):**
- `src/app.module.ts` — register EstimateModule, rename-imports
- `src/quote/quote.module.ts` — imports update if QuoteTokenModule → DocumentTokenModule rename
- `src/quote/quote-token/*` → moved + renamed
- `src/quote/entities/quote.entity.ts` — no change
- `src/quote/repositories/quote-line-item.repository.ts` — widen to accept parentType
- `src/queue/queue.constant.ts` — add `ESTIMATE_FOLLOWUPS`
- `src/queue/queue.module.ts` — register new queue
- `src/worker/worker.module.ts` — add EstimateFollowupProcessor, import EstimateWorkerModule (or EstimateModule)
- `src/business/business.module.ts` — if document-settings rename crosses modules
- `openapi.yaml` — new estimate endpoints
- Migration scripts (one-shot): `quote_tokens` → `document_tokens`; `quote_line_items.parentType` backfill

**Modified (frontend):**
- `src/App.tsx` — new routes for estimates + public estimate page
- `src/components/layouts/DashboardLayout.tsx` — Estimates nav entry
- `src/features/quotes/components/QuoteActionStrip.tsx` — may surface "created from estimate X" breadcrumb
- `src/services/api.ts` — new tag types (`Estimate`, `EstimateRevision`, `EstimateFollowup`)
- Job detail page — add Estimates section alongside Quotes

---

## Sources

- `trade-flow-api/src/quote/quote.module.ts`
- `trade-flow-api/src/quote/entities/quote.entity.ts`
- `trade-flow-api/src/quote/services/quote-creator.service.ts`
- `trade-flow-api/src/quote/services/quote-number-generator.service.ts`
- `trade-flow-api/src/quote-token/quote-token.module.ts`
- `trade-flow-api/src/quote-token/services/quote-token-creator.service.ts`
- `trade-flow-api/src/quote-token/entities/quote-token.entity.ts`
- `trade-flow-api/src/queue/queue.module.ts`
- `trade-flow-api/src/queue/queue.constant.ts`
- `trade-flow-api/src/queue/services/queue-producer.service.ts`
- `trade-flow-api/src/worker/worker.module.ts`
- `trade-flow-api/CLAUDE.md` (layering rules, session auth docs)
- `.planning/codebase/ARCHITECTURE.md` (existing architecture analysis, 2026-02-21)
- `.planning/PROJECT.md` (v1.8 milestone spec and key decisions)
